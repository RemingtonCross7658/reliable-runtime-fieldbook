# Training-Artifact Backups: Restore Throughput Before Private Object Storage Costs

Pick the private object store that can hand a whole training run back inside the recovery window you already promised, and treat the per-GB storage rate as the tiebreaker it actually is. An edtech team retraining a scoring model every night ends up with sharded datasets, two or three checkpoints, an eval log and a manifest — somewhere between 200 GB and 4 TB per run, in a handful of very large files. Those backups are cheap to write and easy to misjudge: restore throughput is what decides whether the recovery window holds, and storage costs only break the tie afterwards.

A backup nobody has read back is a hypothesis.

The flow is unremarkable, which is the point. A training job writes shards and checkpoints to local disk; a packer walks that directory, computes a SHA-256 per file, and emits a manifest naming every object with its byte length, its digest and the date the run may be deleted; an uploader pushes the set under one prefix per run id, never mixed with another run. Retention rules on the storage side expire objects by that prefix. Recovery runs the manifest backwards: read it first, fetch only the objects it names, verify every digest, and let a training job point at the restored directory only after the last check passes.

## How should you size a restore of private object backups before comparing storage costs?

Divide the bytes by the window you promised. 2 TB inside 90 minutes is roughly 3 Gbps sustained, and a single TLS stream on a single connection will not get there on most networks — you get there with several concurrent transfers, and for individual multi-gigabyte shards, with ranged reads inside one object. That is the number that decides whether a candidate is even in the running. Everything else is arithmetic you can do later.

Object count is the other half, and it changes the shape of the answer completely. Forty shards of 50 GB and four hundred thousand small files hold the same bytes and behave nothing alike: in the second case each object costs a round trip before a single useful byte arrives, request charges start to matter more than transfer charges, and the link sits idle while the client waits on latency. Training artifacts usually land on the friendly side of that split, which is exactly why large-file throughput — not request pricing — is the axis worth optimizing for this workload. If the same bucket also holds app data backups full of small user files, size that path separately instead of assuming one drill covers both. Archive tiers add a third dimension: S3's archive storage classes require an explicit restore request before an object can be read, and that request has its own latency budget that has nothing to do with your bandwidth.

Two knobs, then: concurrency and part size.

## A restore worker that saturates the link

The worker below is deliberately generic — it talks HTTP to a pre-signed URL, so it runs unchanged against any S3-compatible endpoint or an internally hosted one. It reads one manifest entry, pulls it in 64 MiB ranges across eight lanes, hashes the bytes in order as they land, and refuses to declare success on a digest mismatch.

```ts
import { createHash } from "node:crypto";
import { open } from "node:fs/promises";

type Entry = { key: string; bytes: number; sha256: string };
type Manifest = { runId: string; retainUntil: string; entries: Entry[] };

const PART = 64 * 1024 * 1024;   // 64 MiB ranges: big enough to amortize per-request latency
const LANES = 8;                 // concurrent range reads inside one object

async function fetchRange(url: string, start: number, end: number): Promise<Uint8Array> {
  const res = await fetch(url, { headers: { Range: `bytes=${start}-${end}` } });
  if (res.status !== 206) throw new Error(`range ${start}-${end}: expected 206, got ${res.status}`);
  return new Uint8Array(await res.arrayBuffer());
}

async function restoreEntry(entry: Entry, url: string, dest: string): Promise<number> {
  const file = await open(dest, "w");
  const digest = createHash("sha256");
  const ranges: [number, number][] = [];
  for (let at = 0; at < entry.bytes; at += PART) {
    ranges.push([at, Math.min(at + PART, entry.bytes) - 1]);
  }

  const started = Date.now();
  for (let i = 0; i < ranges.length; i += LANES) {
    const lane = ranges.slice(i, i + LANES);
    const parts = await Promise.all(lane.map(([s, e]) => fetchRange(url, s, e)));
    for (let j = 0; j < parts.length; j++) {
      digest.update(parts[j]);                                  // ranges stay in order, so does the hash
      await file.write(parts[j], 0, parts[j].length, lane[j][0]);
    }
  }
  await file.close();

  if (digest.digest("hex") !== entry.sha256) {
    throw new Error(`${entry.key}: digest mismatch — do not promote this run`);
  }
  return entry.bytes / ((Date.now() - started) / 1000);         // bytes per second, log it every time
}

export async function restoreRun(manifest: Manifest, sign: (key: string) => Promise<string>): Promise<void> {
  for (const entry of manifest.entries) {
    const rate = await restoreEntry(entry, await sign(entry.key), `./restore/${entry.key}`);
    console.log(`${manifest.runId} ${entry.key} ${(rate / 1e6).toFixed(1)} MB/s`);
  }
}
```

The bucket stays private; `sign()` is your own service minting a short-lived URL after its own authorization check, so the recovery client never holds the account credential and the full signed URL never reaches a log line. If a human ever pulls one artifact through a browser, the response's `Content-Disposition` header is what decides whether it saves under the object key or gets rendered inline, and getting that wrong turns a 40 GB checkpoint into a very bad tab.

Eight lanes is a starting point, not a law. Widen it until the throughput number the worker prints stops improving — the ceiling might be the NIC, the CPU doing SHA-256, or a per-connection limit on the endpoint, and which one binds first varies enough that your mileage may vary. Handle 429 and 503 by backing off and honoring `Retry-After`, and key every write to the run id so a retried restore converges on the same directory rather than a second half-populated one.

## The manifest is the retention policy

Lifecycle rules are an executor. The manifest is the record. S3 lifecycle configuration filters objects by prefix, tag or size and then transitions or expires them, and every major store offers some version of that mechanism — which makes it easy to assume the bucket is the source of truth about what you kept. It isn't. Write `retainUntil` into the manifest at the moment the run is packed, store the manifest somewhere you can query without listing the bucket, and an auditor asking "what should have existed on March 3rd" gets an answer from a database instead of an archaeology project. That separation is what makes the retention policy reproducible: the rule and the record are written from the same input, so a rule that silently stopped matching a prefix shows up as a discrepancy instead of a surprise.

The catch is that a time floor in the billing model can quietly invert your policy. If a store bills a minimum storage duration per object and your checkpoints churn every three days, you are paying for the floor, not for your retention — a real trade-off for hot, short-lived artifacts, and a non-issue for the quarterly snapshots you keep for a year anyway. Versioning and object lock are separate mechanisms with separate switches; without them, an overwrite of a checkpoint key is not something a lifecycle rule can undo. Stick with an immutable-retention product when a regulator, not an engineer, is the one asking.

## Where the billing shapes actually diverge

| Store | Billing shape that dominates a large-file restore | What to verify in your own drill |
|---|---|---|
| AWS S3 | Per-GB data transfer out plus request charges; archive classes need an explicit restore before a read | Which class the artifacts landed in, and the retrieval tier's latency |
| Cloudflare R2 | No egress charge; operations are metered as Class A and Class B | How many operations a full-set restore actually issues |
| Backblaze B2 | Egress allowance tied to the volume stored; S3-compatible API | The allowance math against one complete restore, and multipart part sizes |
| Wasabi | Capacity pricing with a minimum storage duration per object | Effective cost of short-lived checkpoints under that floor |

Three shapes, really: transfer-metered, operation-metered, and capacity-metered with a time floor. Which one hurts is a property of your data, not of the vendor — object count decides whether operations matter, object lifetime decides whether a duration floor matters, and total bytes decide whether transfer matters. All four expose an S3-compatible API, so the client library is rarely the thing that differentiates them, and the table above is a test plan rather than a ranking.

## What to measure before you promise a recovery time

Run the full-set drill quarterly, into an empty target, from the network where recovery would actually happen — if the training cluster sits in the US and the archive sits in the EU, the drill has to cross that boundary too, because the transfer path and the data-residency paperwork both change at that line. Record wall clock, sustained bytes per second, object count, request count, retry count and which invoice lines moved, and reconcile against the manifest rather than a bucket listing, since a listing can only tell you what is there and never what should have been. Run the shape you actually have and the shape you might grow into: one pass with the big shards, one with a directory full of small files. Write down the lane count and part size that produced the number, because a recovery-time promise made with eight lanes is not a promise about a single-stream client. Then do it again next quarter with the same fixture, and compare. A store that survives that twice is one you can put in a runbook; a spreadsheet of per-GB rates is not.

## Further reading

- [MDN: Content-Disposition response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)
- [MDN: HTTP range requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Range_requests)
- [RFC 9110, HTTP semantics: range requests](https://www.rfc-editor.org/rfc/rfc9110#name-range-requests)
- [AWS S3: managing the lifecycle of objects](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [AWS S3: multipart upload limits](https://docs.aws.amazon.com/AmazonS3/latest/userguide/qfacts.html)
- [Cloudflare R2 pricing](https://developers.cloudflare.com/r2/pricing/)
- [Backblaze B2 cloud storage pricing](https://www.backblaze.com/cloud-storage/pricing)
- [Wasabi storage pricing and policies](https://wasabi.com/cloud-object-storage/pricing)
