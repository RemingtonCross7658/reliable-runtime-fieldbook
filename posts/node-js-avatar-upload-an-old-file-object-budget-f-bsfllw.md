# Node.js Avatar Upload — An Old File Object Budget for Rollback

Short answer: give every avatar upload a new object key, switch the database pointer only after the upload succeeds, and let a retention ledger delete the previous key after a fixed rollback window. This preserves rollback without bucket versioning and makes deletion a reproducible state transition instead of an improvised cleanup call.

The evaluation constraint matters more than the storage call. In a customer-support system, an uploaded avatar can also become a training artifact: screenshots, evaluation fixtures, or conversation exports may retain its URL. The system therefore needs to answer three questions later: which revision was active, when did the prior revision become eligible for deletion, and did deletion finish? Overwriting one key answers none of them.

My decision rule is blunt: metadata owns retention; object storage holds bytes.

## What did the overwrite experiment expose?

Treat the upload as a small state machine. Generate an immutable key such as `avatars/{userId}/{revision}.webp`, upload the bytes, verify that the application-level checks passed, and then commit a database transaction that marks the new revision active. In the same transaction, move the former active revision to `retained` and assign `deleteAfter`. Rollback is another pointer change while that deadline remains open. A cleanup worker deletes only revisions whose recorded deadline has passed and whose status is still `retained`. The simple experiment writes every replacement to `avatars/{userId}/current.webp`; it looks cheaper to reason about because there is one URL, but the write destroys the application's clean rollback boundary, caches may continue serving an earlier representation, and a database failure after the object write leaves no durable record of which bytes the user intended to activate. Bucket versioning can preserve underlying generations, but it doesn't define the customer-support product's retention policy. The application still needs deadlines, legal-hold decisions, and an auditable account of cleanup. Deleting the former object inside the activation request is no better: it couples a user-facing update to an independent cleanup operation and erases the rollback candidate at the exact moment it is most useful. For a concrete policy exercise, choose a 72-hour rollback window for ordinary profile assets and store that duration as policy data, not as a timer living in one Node.js process. Seventy-two hours is an example, not a universal recommendation. A support organization with contractual erasure deadlines may need a shorter interval; a reviewed training set may require a separate retention class and approval path. I'm not sure a single window can serve both categories in most systems. The unresolved input is the organization's actual deletion obligation, not an object-store feature.

Record intent first. Delete later.

## The state model behind the result

A useful record carries the object key, content digest, owner, revision, lifecycle status, activation time, rollback deadline, deletion deadline, and deletion attempt state. The digest detects accidental reuse of a revision key before activation. The deadlines make the policy replayable after a deployment or queue outage — no in-memory timer is authoritative.

Keep lifecycle terms narrow. `staged` means uploaded but not active. `active` is the one revision referenced by the profile. `retained` is eligible for rollback until its deadline. `deleting` prevents two workers from claiming the same job. `deleted` is the durable terminal record; keep the record even after the bytes are gone if the audit policy permits it.

That last clause matters. Some deletion regimes require metadata minimization too, so retaining a key-shaped audit trail may itself be unsuitable. In that case, store a non-reversible deletion receipt or aggregate event after erasure. The right representation comes from policy counsel and the data model; your mileage may vary.

| Event | Database change | Object operation | Recovery behavior |
| --- | --- | --- | --- |
| Upload accepted | Create `staged` revision | Put a new immutable key | Expire abandoned staging rows |
| Activation | New revision becomes `active`; old becomes `retained` | None | Retry the transaction safely |
| Rollback | Prior revision becomes `active` | None | Reject an expired revision |
| Retention deadline | Claim revision as `deleting` | Delete its exact key | Retry from the ledger |
| Deletion confirmed | Mark revision `deleted` | None | Preserve the deletion receipt |

## How can Node.js avatar upload preserve an old file object for rollback?

The code below leaves SDK details behind an interface because the important contract is ordering. It also uses a compare-and-swap activation: if another upload won the race, the transaction raises `STALE_AVATAR_REVISION` instead of silently retiring the wrong object.

```ts
type RevisionStatus = "staged" | "active" | "retained" | "deleting" | "deleted";

type AvatarRevision = {
  id: string;
  userId: string;
  objectKey: string;
  sha256: string;
  status: RevisionStatus;
  activatedAt: Date | null;
  rollbackUntil: Date | null;
  deleteAfter: Date | null;
};

interface ObjectStore {
  put(key: string, body: Uint8Array, sha256: string): Promise<void>;
  delete(key: string): Promise<void>;
}

interface RevisionTx {
  getActiveForUpdate(userId: string): Promise<AvatarRevision | null>;
  insertStaged(revision: AvatarRevision): Promise<void>;
  activateIfCurrent(
    revisionId: string,
    expectedActiveId: string | null,
    activatedAt: Date,
    rollbackUntil: Date,
  ): Promise<boolean>;
  retain(revisionId: string, rollbackUntil: Date, deleteAfter: Date): Promise<void>;
}

interface Database {
  transaction<T>(work: (tx: RevisionTx) => Promise<T>): Promise<T>;
}

async function replaceAvatar(input: {
  db: Database;
  objects: ObjectStore;
  userId: string;
  revisionId: string;
  bytes: Uint8Array;
  sha256: string;
  now: Date;
  retentionMs: number;
}): Promise<string> {
  const objectKey = `avatars/${input.userId}/${input.revisionId}.webp`;
  await input.objects.put(objectKey, input.bytes, input.sha256);

  await input.db.transaction(async (tx) => {
    const previous = await tx.getActiveForUpdate(input.userId);
    const rollbackUntil = new Date(input.now.getTime() + input.retentionMs);

    await tx.insertStaged({
      id: input.revisionId,
      userId: input.userId,
      objectKey,
      sha256: input.sha256,
      status: "staged",
      activatedAt: null,
      rollbackUntil: null,
      deleteAfter: null,
    });

    const activated = await tx.activateIfCurrent(
      input.revisionId,
      previous?.id ?? null,
      input.now,
      rollbackUntil,
    );
    if (!activated) throw new Error("STALE_AVATAR_REVISION");

    if (previous) {
      await tx.retain(previous.id, rollbackUntil, rollbackUntil);
    }
  });

  return objectKey;
}
```

The worker side should claim a bounded batch of expired rows, move each to `deleting`, issue deletion for the exact recorded key, and mark `deleted` only after confirmation. Retries must be idempotent. An orphan sweeper can separately remove `staged` objects that never reached activation, but it needs a generous age threshold so it cannot race a live upload.

Multipart uploads deserve their own sweep. The S3 multipart overview says uploaded parts are billed as stored parts until the upload is completed or aborted, and completion is what assembles the object. A failed client session can therefore leave billable parts without producing an avatar object. Track the upload identifier and abort abandoned multipart work according to the same explicit policy.

## What does the retention budget buy?

Immutable keys temporarily retain more bytes and create more write and delete requests. Those operations have a cost; consult the storage service's current pricing rather than embedding a rate in application logic. If the asset is derived, cheap to regenerate, and has no rollback requirement, immediate replacement may be the cleaner policy. If regulation requires immediate erasure, a 72-hour rollback copy is not suitable — deletion takes priority, and the UI should explain that rollback is unavailable.

Native bucket versioning can be a reasonable foundation when operators already have lifecycle controls and recovery procedures around it. Stick with that mechanism when storage administrators, rather than the application, own retention and can map versions back to business events. The ledger pattern earns its complexity when product-level deadlines differ by artifact class, rollback must be visible to support staff, or deletion evidence belongs beside application state.

There's another boundary: immutable keys don't solve references embedded in exported training data. Before deleting a former avatar, decide whether those exports contain a copied blob, an object key, or a rendered snapshot. Deleting the source key cannot erase independent copies. That dependency graph belongs in the retention design from day one.

## Which deletion signals close the experiment?

Measure activation latency separately from cleanup latency. The upload path should expose time to object acceptance and time to pointer commit; the worker should expose the count and age of expired revisions, deletion retry count, staged-object age, and incomplete multipart age. Alert on the oldest overdue deletion, not merely worker errors, because a quiet queue with no consumers can otherwise look healthy.

Test races on purpose: two replacements for one user, rollback during cleanup claim, repeated deletion, database failure after object upload, and a process restart between every recorded state. Also test policy changes. Shortening a retention window should produce a reviewable set of newly eligible revisions before deletion runs.

Start with a shadow cleanup pass that reports candidate keys without deleting them. Compare those keys with active profile pointers, open rollback windows, holds, and references from training artifacts. Once the candidate set stays explainable, enable bounded deletion and watch overdue age plus orphan growth. That's the evidence to collect before copying this design; the diagram alone isn't enough.

## Further reading

- [Amazon S3 multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/)
