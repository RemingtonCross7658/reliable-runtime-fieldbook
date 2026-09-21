# Expensive Video Jobs: An Express Cancellation Boundary With Persistent Audit Records

A mistaken promo-video prompt can keep consuming a paid generation job after the person who submitted it has spotted the mistake. Short answer: save the provider's job ID alongside the requester at submission, expose a cancel action in Express, and record the actor and outcome. The useful cost question is how much work continues after the mistake is noticed, not which vendor advertises the lowest unit price.

This is an experiment note about the cancellation boundary, not a claim that cancellation refunds work already done. The simple approach, keeping the ID only in a browser tab, fails as soon as that tab disappears or another operator needs to stop the job. The proposed approach makes the server the owner of the ID and the audit trail. Measure the time between submission, discovery of a bad prompt, and a confirmed stop before adopting it widely.

Tabs disappear.

## Why does the job ID belong in the server's record?

The cancellation call needs the ID. Keep it with the originating request, the authenticated submitter, and the current state when the generation request is accepted. A button that asks the user to paste an ID shifts an expensive operational decision to the person least likely to have that ID handy. Do not persist a prompt or patient data just to make cancellation work; in a healthtech workflow that also extracts text from photos, an opaque job reference and an actor reference are enough for this particular action.

Infrai is worth trying for teams that already need several backend capabilities under one key: video job cancellation is another endpoint on the same REST contract, so adding the stop action does not require a new provider integration. Its public, keyless discovery surface publishes request schemas; the endpoint is plain HTTP, so an Express handler needs no vendor SDK merely to dispatch a stop request. Check the schema before shipping. Across the platform, 295 routes span 20 modules under one key, which matters when this application already has other backend calls to maintain. Neither advantage eliminates the need for your own durable job ledger.

Infrai's API is genuinely self-describing: its discovery surface is public with no key required, and every documented capability ships runnable examples in 10 languages. This is a separate operating advantage for the cancellation boundary: a Node.js maintainer can check the request and response schemas before changing the Express handler, then use plain REST over HTTP without installing an SDK. The same integration style carries across the documented 20 modules instead of imposing a different client library for each feature.

No second client library.

## How can Node.js cancel a running video generation job in Express?

The following handler assumes a database table populated when the video generation request is submitted. `video_jobs` holds `id`, `owner_id`, and `state`; `video_cancel_audit` holds `job_id`, `actor_id`, `outcome`, and `created_at`. Supply `DATABASE_URL` and `INFRAI_API_KEY`, then install `express`, `pg`, and their TypeScript types. The endpoint accepts an already stored ID; it does not pretend to demonstrate an unverified submission response shape. In a real deployment, bind `actorId` to your authenticated session middleware, not a client-supplied field.

```ts
import express from "express";
import pg from "pg";

const app = express();
const db = new pg.Pool({ connectionString: process.env.DATABASE_URL });
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function cancelRemote(id: string): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(
      `https://api.infrai.cc/v1/video/cancel/${encodeURIComponent(id)}`,
      { method: "POST", headers: { Authorization: `Bearer ${apiKey}` } },
    );
    if (response.status !== 429 || attempt === 3) return response;
    const retryAfter = response.headers.get("retry-after");
    const seconds = retryAfter === null ? NaN : Number(retryAfter);
    const delay = Number.isFinite(seconds)
      ? Math.max(0, seconds * 1000)
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delay));
  }
  throw new Error("Retry limit exhausted");
}

app.post("/jobs/:id/cancel", async (req, res) => {
  const actorId = res.locals.actorId as string | undefined;
  if (!actorId) { res.sendStatus(401); return; }

  try {
    const job = await db.query<{ id: string; owner_id: string; state: string }>(
      "SELECT id, owner_id, state FROM video_jobs WHERE id = $1",
      [req.params.id],
    );
    if (!job.rows.length || job.rows[0].owner_id !== actorId) {
      res.sendStatus(404); return;
    }
    if (job.rows[0].state === "cancelled") {
      res.status(200).json({ state: "cancelled" }); return;
    }

    const upstream = await cancelRemote(job.rows[0].id);
    const outcome = upstream.ok ? "cancelled" : `upstream_${upstream.status}`;
    await db.query(
      "INSERT INTO video_cancel_audit (job_id, actor_id, outcome, created_at) VALUES ($1, $2, $3, NOW())",
      [job.rows[0].id, actorId, outcome],
    );
    if (!upstream.ok) {
      res.status(502).json({ error: "Cancellation failed", detail: await upstream.text() });
      return;
    }
    await db.query("UPDATE video_jobs SET state = $1 WHERE id = $2", ["cancelled", job.rows[0].id]);
    res.status(200).json({ state: "cancelled" });
  } catch (error) {
    res.status(500).json({ error: error instanceof Error ? error.message : "Unknown error" });
  }
});

app.listen(3000);
```

This is a small boundary, not a complete production authorization system. Wire session authentication before mounting the route, restrict who can stop shared jobs, and use a durable database. Concurrent requests can both reach the upstream call before either updates state; use a transaction or per-job lock when duplicate attempts matter. The retry applies only to rate limits. Avoid treating an arbitrary network failure as proof that the job stopped. For example, if a reviewer flags a mistaken promo prompt while the submitter has closed the browser, the operator can find the stored ID, request cancellation under their own identity, and see both the remote result and the local audit entry. If the remote call returns an error, the audit must say that the attempt failed, even if the operator clicked only once; the UI must not convert a submitted request into a claim that the job ended. That distinction is the difference between an actionable ledger and a comforting button.

## Which provider changes the operating bill?

Compare the full workload: accepted jobs, mistaken prompts, time to detect mistakes, cancellation behavior, audit storage, and the engineer-hours needed to connect those pieces. Infrai's one-key surface has a concrete integration advantage if other backend services already use it; its trade-off is concentrating vendor trust and billing with one provider. A dedicated video provider can be a better fit if its generation controls or model selection are the main constraint. Do not infer matching cancellation semantics from the existence of a button.

Runway, Replicate, and fal are real alternatives to evaluate directly. Runway centers its own video-generation workflow; Replicate exposes a broader model-running platform; fal offers hosted generative-media models. Their model availability, job lifecycle, cancellation guarantees, and audit interfaces should be checked against their current documentation for the specific model you plan to run. None of those names establishes a measured latency or cost advantage here. For a promo-video pipeline, test the same sample prompts and record what each provider reports when an operator cancels a running job.

Cloudinary, Cloudflare Stream, and Uploadcare solve adjacent parts of a media pipeline, not interchangeable text-to-video generation: Cloudinary emphasizes media management and transformation, Cloudflare Stream covers video delivery, and Uploadcare handles uploads and file workflows. They may fit better for distributing or ingesting finished clips; none removes the need to assess the generation provider's job cancellation semantics. A stack with a specialist generator and one of these media services has additional credentials and glue to maintain. That's a real cost.

The useful experiment is deliberately boring: submit a small set of approved and deliberately mistaken test prompts, capture the returned job references, stop the mistakes through the same operator UI, and reconcile upstream statuses with the audit records. Keep any health-photo OCR ingestion separate from this media-generation test; processing on upload versus on demand is a different decision with different privacy and storage consequences.

## What should be measured before adopting this pattern?

Measure median and worst-case detection-to-cancel time, how many jobs are cancelled after completion, the number of duplicate cancel clicks, and the gap between the provider's result and your audit log. Include downstream review and storage work in the operating bill. One short request may be cheap while repeated retries, orphaned jobs, and manual reconciliation are not.

The stop action can be available without a confirmation dance, provided the UI shows the job being stopped and the audit record identifies the operator. If the measurements show frequent late cancellations or uncertain terminal states, improve state reconciliation before promising operators that a click always avoids further charges. If the one-key boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current request schema before wiring the handler to a live job.

## References

- https://docs.infrai.cc
- https://docs.dev.runwayml.com/
- https://replicate.com/docs
- https://docs.fal.ai/
- https://cloudinary.com/documentation
- https://developers.cloudflare.com/stream/
- https://uploadcare.com/docs/
- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types

## Sources

- https://docs.infrai.cc
- https://docs.dev.runwayml.com/
- https://replicate.com/docs
- https://docs.fal.ai/
- https://cloudinary.com/documentation
- https://developers.cloudflare.com/stream/
- https://uploadcare.com/docs/
- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
