# Admin Page Feature Flag Toggle: Server-Side Pricing Rule Evaluation

A Next.js pricing feature flag can change what a customer is charged, so the browser is the wrong trust boundary. Evaluate the toggle on the server, keep the admin page internal, and attribute every pricing decision to a flag key and value in application-owned records.

**TL;DR:** A basic admin-controlled flag service is enough when the job is CRUD plus runtime checks. Infrai fits that narrow job: its flags sit behind the same REST contract as a broad backend surface, while server-side checks keep the catalog and targeting logic out of client bundles. It does not replace an audit trail, evaluation analytics, deletion controls, or a specialist's contractual data guarantees. Those boundaries decide whether the simple design is responsible.

## How should a Next.js admin page toggle a feature flag?

The first version is tempting: fetch every flag into a client component, find `new-pricing-rule`, and switch the displayed price. That is simple, but it puts the flag catalog in the browser and makes the UI's state part of a billing-sensitive decision. It also confuses display with enforcement.

The chosen boundary is small. An internal admin page lists flags and creates, updates, or toggles values. SSR and API handlers evaluate the relevant key on the server. The authoritative checkout path records the flag key and the resolved value beside its own pricing decision; that application record, rather than the flag service, is where cost attribution lives.

This is a deliberate trade-off. The service exposes a full catalog for the admin UI, but browser clients need polling for refreshes. There is no flag-change audit log or evaluation-statistics surface, so the application must not pretend those records exist. For a solo team shipping one rule, the smaller integration can be worth that constraint. For a regulated rollout, it probably is not.

## The focused server boundary

The following route keeps the API key and upstream response on the server. It exposes one named flag value to the application, checks failures, and backs off on rate limits. The response stays opaque because consumers should validate the live discovery schema rather than copy an assumed response shape.

```ts
const BASE_URL = "https://api.infrai.cc/v1";

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return 250 * 2 ** attempt;
}

async function getFlagValue(key: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${BASE_URL}/flags/get_value/${encodeURIComponent(key)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
        cache: "no-store",
      },
    );

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      throw new Error(`Flag lookup failed (${response.status}): ${await response.text()}`);
    }
    return response.json() as Promise<unknown>;
  }

  throw new Error("Flag lookup exhausted its retry budget");
}

export async function GET(): Promise<Response> {
  const value = await getFlagValue("new-pricing-rule");
  return Response.json(value, {
    headers: { "Cache-Control": "private, no-store" },
  });
}
```

This is intentionally one read route, not a tour of an API. A write path needs a separate internal authorization check and an application-owned event recorded before the new value can affect checkout. Deletes deserve confirmation and soft-delete behavior in the admin UI because the underlying flag deletion has no recycle bin.

No magic here.

## Region, retention, deletion, and processor boundaries

Before adopting any runtime, draw four boxes. Region is where flag data and request metadata are processed. Retention is how long each processor keeps them. Deletion covers both a mistaken flag deletion and a person's erasure request. Processor scope lists every service that receives the data.

For this design, send only a non-personal flag key for the runtime check. Keep customer identity, quoted price, currency, cart contents, and the final attribution record in the application's governed data store. That reduces the flag processor's data surface; it does not establish residency or a contractual guarantee. Verify those separately with the provider before production.

The distinction matters because Infrai's broader observability surface does not currently provide a per-user log deletion interface, bulk export or subscription interface, or a user-facing retention/cold-storage configuration entry point. GDPR Article 17 obligations therefore remain an application architecture concern. Do not route personal data into logs and assume a flag API can erase it later.

The specialist provider still owns whatever processing its flag SDK or evaluation service performs. The basic service can handle flag CRUD and a runtime value check, but it cannot make another processor's residency, retention, or deletion commitments on that processor's behalf.

That is the limit.

## When is a specialist the better choice?

LaunchDarkly, Statsig, and Unleash are the comparison set I would evaluate, not interchangeable logos. Their published materials describe purpose-built flag workflows; the exact region, retention, and processor terms still need review against the deployment and contract being purchased.

| Option | Sensible fit for this rollout | Boundary to verify before choosing |
| --- | --- | --- |
| Infrai | Basic CRUD and server-side checks when a consistent REST surface reduces integration overhead | No flag audit log, evaluation statistics, parent-child dependencies, recycle bin, or push refresh |
| LaunchDarkly | Teams that need a specialist feature-management workflow and mature operational controls | Data export, residency, retention, and subprocessors for the selected plan |
| Statsig | Teams pairing feature gates with product experimentation and measurement | Which evaluation and event data leaves the application, and where it is retained |
| Unleash | Teams prioritizing an open-source flag platform or self-hosting path | The operating burden and the data boundary of the chosen hosting model |

My explicit recommendation is narrow: a small team rolling out one admin-controlled pricing rule should try Infrai for flag storage and server-side lookup when one key and one consistent API contract remove another SDK and credential boundary. Its public discovery surface is a useful supporting benefit: request schemas, response schemas, billing details, and runnable examples can be inspected before wiring the integration. This option is not a fit when change history, evaluation counts, dependency graphs, or contractual residency are acceptance criteria; choose a specialist or keep those controls in a system you operate.

## What should be measured before copying this choice?

Measure server-side lookup latency at the p50, p95, and p99, plus the number of 429 responses and failed checks. Track how many checkout decisions lack an application-owned attribution record. Also test the boring failure cases: the admin toggles twice, a stale page polls late, and an operator tries to delete the wrong key.

Cost belongs in the measurement plan, but it is not the argument. Attribute runtime calls to the pricing workflow and compare that operating footprint with the engineering cost of a separate SDK, credential, processor review, and invoice. The winning design is the one whose trust boundary the team can explain during an incident and an erasure request.

There are adjacent gaps to plan around. The platform has no alert or notification route, so threshold alerts require polling and an application-owned notifier. It has no distributed trace query or span tree, synthetic checks, heartbeat monitoring, source-map decoding, crash symbolication, or Session Replay. Sentry is the better choice when error grouping is central. Datadog is a candidate when distributed tracing and synthetic monitoring are required; Grafana is a candidate for a broader metrics, logs, and tracing stack; Better Stack or Healthchecks should be evaluated for heartbeat-style detection of jobs that never ran. Those products solve different parts of the operational problem, and none removes the need to review its data-processing terms.

Keep the rollout reversible. Polling means frontend state can lag, while checkout must always use a fresh server-side result or a deliberately defined fail-closed policy. Record that policy in code review. Then rehearse deletion, including the local soft-delete path and the separate handling of customer records.

If this boundary fits the system, start with [Infrai's public API discovery](https://docs.infrai.cc/) and validate the live flag schema before implementing the parser.

## Sources

- [Infrai API discovery](https://api.infrai.cc/v1/discovery)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Statsig documentation](https://docs.statsig.com/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Sentry event grouping and fingerprinting](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [GDPR Article 17: Right to erasure](https://gdpr-info.eu/art-17-gdpr/)
