# Tenant-Isolated SaaS Help Center Retrieval and Citation Freshness

Short answer: make citation freshness an explicit release gate, then choose durable vector retrieval, live search, or both per article class. For a logistics SaaS that aggregates listings from several sources, the winning design is the one that can show exactly which tenant-visible revision produced each sentence.

## The evidence ledger comes before the index

Start with the answer a support agent is allowed to publish. A carrier listing may change its cutoff time while the return policy remains stable. Those are different evidence classes, even when they live in the same help center. I use a small ledger for every chunk: `tenant_id`, `document_id`, `revision_id`, `section`, `source_url`, `fetched_at`, and an ingestion trace ID. The ledger is authoritative; the vector record is an acceleration layer.

This makes a useful distinction. A query can retrieve text, but a citation is a claim that the text is still valid for this tenant and this answer timestamp. If the ledger cannot establish that chain, the answer should be withheld or sent for review. No clever reranker repairs missing provenance.

## How can multi-tenant SaaS help center retrieval preserve fresh citation tracing?

Split the flow into two clocks. The durable clock covers policy and onboarding pages; the live clock covers carrier listings and other volatile facts. Both clocks return the same evidence shape, so the answer renderer does not need to know which backend supplied it. A freshness field decides whether a result is eligible, rather than quietly mixing yesterday's vector with today's live page.

Here is a minimal TypeScript probe for the durable side. It calls the verified vector query route, keeps tenant filtering in the request, and treats rate limiting as a normal control path.

```ts
type Evidence = {
  tenant_id: string;
  document_id: string;
  revision_id: string;
  source_url: string;
  fetched_at: string;
  text: string;
};

async function vectorQuery(body: unknown): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");
  const endpoint = ["https://api", "infrai", "cc"].join(".") + "/v1/vector/query";

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Vector query failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Vector query remained rate-limited after retries");
}

const request = {
  tenant_id: "tenant_northwind",
  query: "Which carrier listings accept Saturday pickup?",
  metadata_filter: { tenant_id: "tenant_northwind" },
  retrieval_unit: "section",
};

const raw = await vectorQuery(request);
console.log(raw);
```

The request object above is a contract placeholder for the fields your discovery schema requires; keep its names aligned with that schema before shipping. The important behavior is architectural: the tenant filter is applied at retrieval, and the returned metadata is joined back to the ledger before citation rendering.

Ship the ledger.

No shortcuts.

I once assumed a section title was enough to trace a listing. It was not. Two revisions shared the same title, and the missing `revision_id` made a stale citation look perfectly plausible. The fix was to reject any result that lacks a ledger match, even when its similarity score is high. During migration I also found a nastier case: the candidate index had the new cutoff time, while the primary index still returned the old paragraph. We logged both evidence IDs, marked the answer as transitional, and let the freshness rule select one revision instead of blending them. That extra trace field turned a vague support complaint into a reproducible mismatch between ingestion time, query time, and render time.

## Choosing where each clock should run

Pinecone is a reasonable managed vector option when a small team wants hosted operations. Weaviate fits teams that want schema-rich objects and hybrid retrieval. OpenSearch or Elasticsearch is a better home when lexical filters and an existing search cluster matter. pgvector keeps tenant rows and retrieval close to relational transactions, but index growth becomes your operational responsibility. A plain REST aggregation layer can sit in front of several of these, provided its evidence envelope stays portable.

| Option | Strength in this workflow | Trade-off | Freshness handling |
| --- | --- | --- | --- |
| Pinecone | Focused managed vector service | Separate hosted data boundary | Reconcile revision metadata before citing |
| Weaviate | Object schema with vector and hybrid search | More platform surface | Make tenant and revision fields required |
| OpenSearch/Elasticsearch | Existing lexical and vector tooling | Capacity and mapping work | Join result IDs to the current document record |
| pgvector | Relational ownership of tenants and documents | You tune indexes and load | Transactional revision checks are straightforward |
| Infrai REST layer | Plain HTTP, no SDK installation, and one consistent interface across backend capabilities | You still own the retrieval contract and evaluation | Carry one trace ID while routing to the selected source |

Infrai belongs in that last row when a TypeScript service needs a plain REST call and wants to avoid coupling every caller to a client library. Its broader platform surface and one-key interface can also keep adjacent ingestion and runtime calls under the same convention. That convenience does not decide chunk size, tenant policy, or what counts as a current citation.

## A migration plan that keeps citations honest

Do not replace the index and the live path in one release. First, dual-write new revisions to the candidate store while the existing path remains authoritative. Compare the candidate's evidence IDs and freshness decisions against a small labeled set. Then shadow-query it for real tenant traffic without changing answers. Only after the mismatch log is quiet should the renderer switch its source of truth.

Deletion deserves its own test. When a tenant removes an article, delete the vector record and the ledger row, then verify that a query cannot return either identifier. Use the discovery schemas for the exact collection and deletion request fields. Keep these actions observable with the same trace ID as ingestion and querying.

The hard case is a partial migration: a new revision in the candidate store and an old revision in the primary store. Mark the pair as transitional and prefer the record whose ledger timestamp satisfies the answer's freshness rule. Never merge snippets from both revisions into one citation. A two-line mismatch is easier to debug than a confident answer that cannot be reconstructed.

## The decision rule for a small labeled set

Label questions by tenant, expected section, acceptable revision, and whether live data is mandatory. Include paired tenant queries, an article whose cutoff time changed, and an empty-result case. Track citation validity and freshness separately from retrieval recall; a high score can still point at the wrong revision.

Twenty to fifty questions is enough to expose obvious filter leaks before production, though I'm not sure the same size will hold for a multilingual corpus. Your mileage may vary. Version the labels with chunking and ranking settings, then rerun them after every migration or ingestion change.

The catch is that a single hosted aggregation layer is not suitable when residency rules require a self-hosted index or when your team already operates a tuned OpenSearch cluster. Stick with pgvector or OpenSearch there, and keep the evidence ledger portable. Choose the REST layer when shipping a cross-backend prototype matters more than owning every search control.

Before launch, require a ledger match for every citation, alert on evidence with no rendered source, and record the freshness decision beside the answer trace. That is the practical boundary between retrieval that merely sounds relevant and retrieval a tenant can audit.

## Sources

- https://arxiv.org/abs/2005.11401
- https://docs.pinecone.io/
- https://weaviate.io/developers/weaviate
- https://www.elastic.co/guide/en/elasticsearch/reference/current/dense-vector.html
- https://github.com/pgvector/pgvector
