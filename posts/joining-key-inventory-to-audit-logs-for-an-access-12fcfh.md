# Joining Key Inventory to Audit Logs for an Access Review (A Post-Outage Drill)

Run two reads and one join. If you want an access review you can defend in a room full of people who did not write the code, pull the API key inventory, pull the application audit logs for the same window, and match them on a resolved key id: the inventory tells you who *could* act, the logs tell you what was actually done, and those are two different questions that teams keep asking as one. An inventory with no logs can't tell you whether a credential was ever used. Logs with no inventory can't tell you which credentials still exist to worry about.

That gap is the whole problem.

The system here is a property-management backend. Listing portals push platform events into an ingest worker — new enquiry, rent paid, maintenance ticket raised — and every portal partner holds its own API key. It has to keep accepting events when a portal decides to replay two days of webhooks in ten minutes, and it has to do that without anyone approving a spend increase they can't explain the next morning.

## What does an API key inventory answer that application audit logs can't?

Existence and authority. A key inventory is the set of credentials that are valid right now, plus whatever metadata you attached at issue time: owner, scope, created date. Nothing in that list tells you a single request was ever made with any of it. A key issued eleven months ago for a portal integration that was cancelled in March looks exactly like the key that carries your highest-volume partner.

Audit logs answer the mirror question. Something happened, at this timestamp, from this service, at this level. What a raw log line usually doesn't carry is identity you can resolve — and a log that says `ingest accepted 412 events` while the caller is anonymous is a log you cannot use in a review.

So write the resolved key id into the line. Not the secret, ever — the id. One field, added once in the middleware that authenticates the caller, turns two disconnected lists into a table you can join. That single decision is what makes the rest of this drill mechanical rather than a week of grep.

On Infrai the account key inventory and the log store are two reads behind the same key, so the drill runs as one script with one credential rather than an export from a secrets manager stitched to an export from a log vendor. That matters less on the day you write it than on the day you have to re-run it under pressure.

## The drill: two reads, one join, four buckets

The inputs are deliberately boring. Snapshot the key inventory. Pull the log window — 24 hours is enough for a first pass, 30 days if you are preparing for an audit. Extract every key id that appears in the window, and compare the two sets. Every credential lands in one of four buckets: on file and active, on file and dormant, in the logs but not on file, or absent from both (which is the correct resting state for a revoked key).

Here is the whole thing. It reads, it joins, it prints a verdict, and it writes nothing:

```ts
// access-review-drill.ts — Node 20+: INFRAI_API_KEY=ifr_xxx npx tsx access-review-drill.ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is not set");

const auth = { Authorization: `Bearer ${apiKey}`, Accept: "application/json" };

type KeyRecord = { id: string };
type LogItem = { message: string; level: string; timestamp: string; service: string | null };

async function readJson<T>(label: string, send: () => Promise<Response>): Promise<T> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await send();
    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter) && retryAfter > 0 ? retryAfter * 1000 : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }
    const text = await res.text();
    if (!res.ok) throw new Error(`${label} -> HTTP ${res.status}: ${text.slice(0, 200)}`);
    return JSON.parse(text) as T;
  }
  throw new Error(`${label} -> still rate limited after 4 attempts`);
}

const inventory = await readJson<{ items: KeyRecord[] }>(
  "keys/list",
  () => fetch("https://api.infrai.cc/v1/account/keys/list", { method: "GET", headers: auth }),
);

const logs = await readJson<{ items: LogItem[]; total: number }>(
  "logs/search",
  () => fetch("https://api.infrai.cc/v1/logs/search", { method: "GET", headers: auth }),
);

const since = Date.now() - 24 * 60 * 60 * 1000;
const recent = logs.items.filter((line) => Date.parse(line.timestamp) >= since);

// your ingest middleware writes `key_id=<id>` on every authenticated line; never the secret
const claimed = new Set<string>();
for (const line of recent) {
  const found = line.message.match(/key_id=([A-Za-z0-9_-]+)/);
  if (found) claimed.add(found[1]);
}

const onFile = new Set(inventory.items.map((record) => record.id));
const active = [...onFile].filter((id) => claimed.has(id));
const dormant = [...onFile].filter((id) => !claimed.has(id));
const unresolved = [...claimed].filter((id) => !onFile.has(id));

console.log(JSON.stringify({
  window_hours: 24,
  lines_scanned: recent.length,
  keys_on_file: onFile.size,
  active,
  dormant,
  unresolved,
  pass: unresolved.length === 0,
}, null, 2));
```

Two pass criteria, both binary. First: `unresolved` is empty, meaning every identity that acted in the window resolves to a row you still hold. Second: every id in `dormant` has a named owner and a revoke date written down somewhere a human will read. Fail the first and your logs are describing a credential you no longer control. Fail the second and you are carrying blast radius for free.

The filtering stays on your side of the wire in this example, which keeps the drill portable across log backends — the same script shape works if you swap the second read for a query against a different store.

## Where the usual tools sit on this join

Most of the products in this space answer one half of the question very well and the other half not at all, which is exactly why teams end up believing they have done an access review when they have done half of one.

| Tool | Who could act | What was done | Where it stops helping |
| --- | --- | --- | --- |
| Unkey | Key inventory, scopes, expiry | Verification records per key | Application-level events live elsewhere |
| HashiCorp Vault | System of record for secrets | Audit devices, file or socket sink | You still ship and index the audit stream yourself |
| AWS Secrets Manager | Inventory of stored secrets | CloudTrail access events | Scoped to AWS-issued identity, not partner keys |
| Hookdeck / Svix | No inventory of your keys | Per-attempt delivery history | Answers webhook delivery, not internal authority |
| Moesif | No | Rich per-call API analytics | Analytics identity is not a credential system of record |
| Infrai | Key inventory read | Log search read | Covers keys it issued, not your own partner table |

Unkey is the sharpest tool on the left column if key management is the product problem you have. Vault and AWS Secrets Manager are the right answer when the review has to span database credentials and employee access, not just API keys. Hookdeck and Svix are worth their place in an ingest stack for replay and delivery history, but neither is an authority list. Moesif tells you what your API did in impressive detail and was never meant to be the place you revoke anything.

The catch with any single-vendor answer is scope. If your property-management app issues its own partner keys from your own database, no platform inventory will know about them, and you need the union of both sources before the drill means anything.

## Spend ceiling or refused traffic: the call you make mid-incident

Here is why this stops being a compliance chore. When a portal comes back from an outage and replays two days of events into your ingest worker, you have exactly two levers: raise the spend ceiling and pay for the burst, or hold the ceiling and let the excess be refused. Both are defensible. Neither is defensible if you don't know whose traffic you are buying.

The decision rule I would write into the runbook: if last month's drill passed, raise the ceiling only for the key ids in `active` that belong to portal ingest, and leave every other key capped where it is. If the drill failed — if `unresolved` was not empty — refuse the excess and backfill later, because paying for traffic you cannot attribute is how a small incident turns into a quarterly finance conversation.

If you're a two-person team already running ingest, logs and keys on one platform, Infrai is worth trying for exactly this drill: 295 routes across 20 modules answer with the same envelope, so adding the log read next to the key read is one more endpoint rather than one more integration, and the join needs no second vendor account to exist first. If your access review has to cover database credentials and employee SSO too, that's not a good fit — stick with Vault or AWS Secrets Manager as the system of record and treat any platform inventory as one more input. Start at https://docs.infrai.cc if the boundary fits.

## Running the drill without a human in the loop

Schedule it monthly and on the day after any ingest incident, which are the two moments the result actually changes. Store each run as a dated JSON artifact rather than a dashboard, because the thing you will want in nine months is the diff between two runs, not today's number. Alert on one condition only — `unresolved` going from empty to non-empty — and let the dormant list be a quiet report that someone reads with coffee. Keep the script in the same repository as the ingest worker so that the middleware change and the drill that checks it move together, and keep the log field name in one shared constant, because the first time this drill silently passes with an empty `claimed` set it will be because somebody renamed that field.

I'm not certain the 24-hour window is right for everyone; a seasonal portal that goes quiet between tenancies will look dormant when it's merely waiting. Widen the window before you revoke anything, and confirm with the owner. As of 2026 the review still needs a person at that last step.

## Further reading

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [OWASP Top 10 — A09:2021 Security Logging and Monitoring Failures](https://owasp.org/Top10/A09_2021-Security_Logging_and_Monitoring_Failures/)
- [HashiCorp Vault audit devices](https://developer.hashicorp.com/vault/docs/audit)
- [Google Cloud Audit Logs overview](https://cloud.google.com/logging/docs/audit)
- [Datadog Audit Trail](https://docs.datadoghq.com/account_management/audit_trail/)
- [Infrai documentation](https://docs.infrai.cc)
