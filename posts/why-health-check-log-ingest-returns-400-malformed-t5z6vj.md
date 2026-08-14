# Why health check log ingest returns 400: malformed JSON and schema drift in Node.js

Validate the log envelope in the process that runs the probe, not at the collector that rejects it. When a health check emits structured JSON and the ingest endpoint answers 400 Bad Request, the payload is rarely corrupt in the byte sense — it's schema drift. A timestamp serialized as a Date object, a level field that arrived as the number `30`, a service name that was `undefined` at build time. Use one typed envelope, one serializer, and one validation step, and the troubleshooting turns into reading a rejection message instead of guessing.

The system I'll use throughout is a property management platform rolling out a new late-fee pricing rule behind a flag. The rollout needs a synthetic health check: every two minutes, price a sample of units under both flag variants and confirm the rule returns a number, in range, within budget. Those probe results are the only evidence the rollout is safe. Losing them to a 400 is worse than having no check at all, because a silent gap reads like success.

## What should a health check log line contain so a JSON ingest endpoint doesn't return 400 bad request?

Six fields carry almost all the value, and every one of them is a place where drift happens.

| Field | Shape that survives ingest | How it drifts into a 400 |
|---|---|---|
| `timestamp` | RFC 3339 string in UTC, e.g. `2026-08-12T09:14:02.118Z` | A `Date` instance, epoch milliseconds as a number, or a local time with no offset |
| `level` | Lowercase string from a closed set: `debug`, `info`, `warn`, `error` | Numeric severity from a logging library, or `INFO` when the schema declares lowercase |
| `service` | Stable machine id, one per deployable | Display names with spaces, or a value that resolves to `undefined` and disappears |
| `status` | `pass` or `fail`, nothing else | Booleans, `"ok"`, or a nested result object |
| `duration_ms` | Finite number | `NaN` from arithmetic on a missing start time, which `JSON.stringify` writes as `null` |
| `flag` | Small object with rule name and variant | Free-text messages that push the flag state into unstructured prose |

Everything else is optional. Resist the urge to attach the whole probe context — a full pricing breakdown per unit turns a 900-byte line into a 40 KB one and can trip a body size limit that surfaces as, yes, a 400.

## Where malformed JSON actually comes from

The word "malformed" hides several distinct failures, and they need different fixes.

Encoding failures are the smallest group. A circular reference throws inside `JSON.stringify` before anything is sent; a `BigInt` throws for the same reason. Those crash your probe, so you find them fast.

The dangerous group is the one that produces valid JSON with the wrong shape. `JSON.stringify` silently drops keys whose value is `undefined`, so a required `service` field vanishes when an environment variable is unset, and the collector answers 400 with "missing required property: service". Non-finite numbers become `null`. A `Date` becomes an ISO string only because `Date.prototype.toJSON` exists — nest that same date inside a `Map` and it disappears entirely.

Then there's transport shape. Many ingest APIs accept newline-delimited JSON, one object per line, while others want a JSON array or an object with an `events` key. Sending an array to an NDJSON endpoint parses fine as JSON and still fails validation. So does sending NDJSON with `content-type: application/json`, and so does a trailing newline in a strict parser. I've also seen double serialization — a library that stringifies the object, then a client that stringifies the string — produce a body that is technically valid JSON and semantically a quoted blob.

Read the response body. A 400 that names the offending field is a gift, and the fastest troubleshooting loop is to log the first few hundred characters of that rejection next to the line that caused it.

## Implementing the envelope in Node.js

Put the envelope construction and the schema check in the same process as the health check, before anything is queued or batched. The cost is a few microseconds per line. The benefit is that a bad line fails in code you own, with a stack trace, rather than in a collector you can only observe through status codes.

```ts
// Health probe for the "seasonal-late-fee" pricing rule.
// One line per probe run, shipped as newline-delimited JSON.
type Level = "debug" | "info" | "warn" | "error";
const LEVELS = new Set<Level>(["debug", "info", "warn", "error"]);

interface ProbeLog {
  timestamp: string;            // RFC 3339, UTC
  level: Level;
  service: string;              // stable id, not a display name
  event: "pricing_rule_health_check";
  status: "pass" | "fail";
  duration_ms: number;
  flag: { rule: string; variant: "control" | "treatment" };
  unit_sample?: number;
}

function toEnvelope(input: Partial<ProbeLog>): ProbeLog {
  const level = LEVELS.has(input.level as Level) ? (input.level as Level) : "info";
  const duration = Number(input.duration_ms);
  const service = process.env.SERVICE_ID;
  if (!service) throw new Error("SERVICE_ID unset: the log envelope would lose a required field");

  return {
    timestamp: new Date(input.timestamp ?? Date.now()).toISOString(),
    level,
    service,
    event: "pricing_rule_health_check",
    status: input.status === "fail" ? "fail" : "pass",
    duration_ms: Number.isFinite(duration) ? Math.round(duration) : 0,
    flag: {
      rule: input.flag?.rule ?? "seasonal_late_fee",
      variant: input.flag?.variant === "treatment" ? "treatment" : "control",
    },
    ...(Number.isFinite(input.unit_sample) ? { unit_sample: input.unit_sample } : {}),
  };
}

// One serializer for the whole service. Non-finite numbers are the common
// accident, so they get an explicit value instead of a silent null.
function encode(lines: ProbeLog[]): string {
  return lines
    .map((line) => JSON.stringify(line, (_key, value) =>
      typeof value === "number" && !Number.isFinite(value) ? 0 : value))
    .join("\n");
}

async function ship(lines: ProbeLog[], batchId: string, attempt = 0): Promise<void> {
  const body = encode(lines);
  const res = await fetch(`${process.env.LOG_INGEST_URL}/ingest/logs`, {
    method: "POST",
    headers: {
      "content-type": "application/x-ndjson",
      authorization: `Bearer ${process.env.LOG_INGEST_TOKEN}`,
      // Same key on every retry, so a duplicated batch is stored once.
      "idempotency-key": batchId,
    },
    body,
  });

  if (res.status === 400) {
    // The rejection names the field that drifted. Keep it next to the payload.
    const detail = (await res.text()).slice(0, 400);
    console.error("log ingest rejected", { detail, sample: body.slice(0, 200) });
    return;                        // Retrying a schema rejection just repeats it.
  }
  if (res.status === 429 && attempt < 4) {
    const retryAfter = Number(res.headers.get("retry-after")) * 1000;
    const backoff = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter
      : 2 ** attempt * 250 + Math.random() * 250;
    await new Promise((r) => setTimeout(r, backoff));
    return ship(lines, batchId, attempt + 1);
  }
  if (!res.ok) console.error("log ingest not accepted", res.status);
}
```

Four details in there matter more than they look. The `service` lookup throws instead of defaulting, because a default hides a misconfigured deployment for weeks. The replacer normalizes non-finite numbers at the single point where objects become bytes, which is the only place a rule like that can be enforced. The 400 branch keeps a sample of the body, so the next person debugging this reads evidence rather than reproducing it. And the retry path separates two rejection classes that get conflated: a 400 is a schema problem that will fail identically forever, while a 429 is backpressure that clears on its own — retry the second with exponential backoff behind a stable idempotency key, never the first.

A structural note on where this code lives: the encoder belongs behind the same interface as the appender or transport your logging library already uses, so application code keeps calling `log.info(...)` and never learns about NDJSON. Logback's appender model is the clearest documented version of that separation, and the shape translates to Node.js transports.

## Signal quality versus noise during a flag rollout

Here is where uptime logging goes wrong even after the 400s are gone. A flag rollout multiplies the probe surface: two variants, several property portfolios, a handful of rule branches, and suddenly the health check emits sixty lines a minute of which fifty-nine say `pass`. Nobody reads that. Worse, the storage bill and the query latency both scale with the noise, and the one `fail` line that mattered is three pages deep in a search result.

The split that works is boring. Aggregate counters go to metrics, per-probe detail goes to logs, and the flag variant is a low-cardinality dimension on both. A counter of `pass`/`fail` per variant answers "is the treatment worse than control" in one query and costs almost nothing to keep. Logs then only need to carry the failures plus a thin sample of successes — enough to prove the probe itself is alive. In the property management case that means one line per failed unit price, a sampled line every tenth successful run, and a counter incremented on every run without exception. Sampling successes is safe precisely because the counter is authoritative for rates; the log is there for the narrative, not the arithmetic. Keep the sampling decision in the probe, not in a collector rule, so the sampled-out lines never cost bandwidth. And never sample failures. The moment you do, a rollout comparison built on log counts starts lying, and you'll believe the treatment is healthier than it is.

Feature flags add one more requirement that generic logging advice misses: the flag state at evaluation time has to be in the line. A pricing rule that read `treatment` from a cached flag client two minutes after the rollout was paused produces a result that looks anomalous and is actually correct. Without the variant recorded per probe, that's unresolvable. Martin Fowler's toggle taxonomy is the useful frame here — a release toggle that lives for days deserves this instrumentation, while a long-lived ops toggle probably deserves its own dashboard.

The catch is that this whole approach assumes probe results are yours to shape. If your health checks come from a hosted uptime service that posts a fixed payload, you can't add a `flag` field, and you're stuck correlating by timestamp — workable, but coarse. High-frequency probes across thousands of units are also not a good fit for per-line logging at all; that's a metrics problem with logs attached only on state transitions. And if you need per-request causal chains across services, structured logs with a shared `trace_id` are a weak substitute for real tracing. They let you join records by hand. They don't reconstruct a call graph.

## Failure modes to alert on before the flag goes live

Ship the validator first, in a build that logs rejections but doesn't change the envelope. Give it a day; the rejection samples tell you which field actually drifts in your fleet, and it's usually not the one you'd bet on. Then freeze the schema in a JSON Schema document, keep it in the same repository as the probe, and run it in a unit test against a fixture of every envelope your code can emit — including the `undefined` service case and the `NaN` duration case, since those are the two that produce valid JSON and invalid semantics. Version the schema and treat a field removal as a breaking change, because a collector that validates strictly will reject old lines from a straggler instance during a rolling deploy.

For the rollout itself, wire the ingest failure count into the same alert that watches the pricing rule. A health check that stops reporting must page exactly like a health check that reports failures — otherwise the quietest possible failure mode is the one where your evidence pipeline broke and the flag stayed on. Then delete the extra fields you added while debugging. They were noise the day after they were useful.

## References

- https://www.rfc-editor.org/rfc/rfc3339
- https://www.rfc-editor.org/rfc/rfc9110#name-400-bad-request
- https://www.rfc-editor.org/rfc/rfc9457
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify
- https://json-schema.org/draft/2020-12/json-schema-core.html
- https://opentelemetry.io/docs/specs/otel/logs/data-model/
- https://martinfowler.com/articles/feature-toggles.html
- https://logback.qos.ch/manual/appenders.html
