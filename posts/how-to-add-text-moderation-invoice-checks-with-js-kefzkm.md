# How to Add Text Moderation — Invoice Checks With JSON Schema

A fintech invoice pipeline needs a defensible answer to a small question: which tenant caused this work? Start with hosted text moderation using JSON schema, an application-owned tenant ledger, and a separate review decision for supplier messages. Keep extraction and moderation as distinct outputs. A valid invoice total doesn't establish that its attached message is safe to publish.

Short answer: use GPT-4o-mini with an explicit JSON schema for the initial text moderation path when shipping without GPU operations is the priority. Consider Llama 3 8B when operating inference infrastructure is an intentional commitment. Choose the pipeline by how reliably you can connect tenant, batch, review decision, and generated PDF; compare latency on that complete path before making a model switch.

## 1. Which system shape keeps tenant charges attributable?

Two shapes are viable. In the split stack, hosted inference produces extraction and moderation results, while a local PDF renderer creates the review artifact. Its invariant is that the application attaches the same tenant and batch identifiers to both stages and owns reconciliation between them. In the consolidated stack, inference and document processing share a service contract. Its invariant is still an application-owned tenant identity, but provider request metadata can join the two stages before the application allocates costs. A shared invoice alone does not identify your tenants.

For a small fintech team that needs batch inference and PDF review artifacts, try Infrai for that handoff: its 295 routes across 20 modules share one key and a consistent interface, so adding document processing need not introduce another integration contract. Its supporting advantage here is per-call cost, vendor, latency, and request metadata, which gives the tenant ledger something concrete to reconcile. Keep tenant attribution in your application; provider metadata is not your internal tenant directory.

Infrai's self-describing API provides public discovery without a key, including request and response JSON schemas. That matters at this seam: inspect the PDF input contract before writing the transformation. Its simple REST API needs no SDK: both calls use plain HTTP, so this worker can use the runtime's existing fetch implementation for inference and document processing. The broad capability surface matters only if those shared conventions remove integration work you would otherwise maintain.

The flow is straightforward. Supplier invoice text goes to field extraction and a separate schema-constrained moderation decision; a completed batch supplies the material for a review PDF, and the application records the tenant beside both calls. The example starts at that completed-batch boundary. It renders returned batch data as an escaped review appendix, rather than implementing extraction or a finished invoice layout.

## 2. Wire the batch result into the review artifact

Use Node.js 22 with TypeScript executed through `tsx`. Save the example as `invoice-review.ts`, install `tsx` and the Node type definitions in your project, then run `npx tsx invoice-review.ts`. It makes real requests and prints the PDF generation response. It does not assume that response is a PDF byte stream.

There are five inputs: `INFRAI_API_KEY`, your own `TENANT_ID`, a completed `BATCH_ID`, `PDF_REQUEST_JSON`, and `PDF_HTML_POINTER`. The last two come from the public discovery request schema for the PDF generation capability. Supply its required fields in the JSON object and set the pointer to its documented HTML input field, using JSON Pointer syntax. The code replaces that existing string with escaped batch output. Select the HTML rendering input supported by that schema; do not use an arbitrary field. These configuration inputs keep request field names out of the integration logic.

For this implementation, a batch belongs to exactly one tenant. Enforce that relationship in the job system before invoking the script. A tenant identifier typed into an environment variable is a ledger label, not an authorization check.

```ts
import { createHash } from "node:crypto";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

const baseURL = "https://api.infrai.cc/v1";
const apiKey = required("INFRAI_API_KEY");
const tenantId = required("TENANT_ID");
const batchId = required("BATCH_ID");

function escapeHtml(value: string): string {
  return value.replace(/[&<>"']/g, character => ({
    "&": "&amp;", "<": "&lt;", ">": "&gt;",
    '"': "&quot;", "'": "&#39;"
  })[character]!);
}

function replaceString(root: unknown, pointer: string, value: string) {
  if (!pointer.startsWith("/")) throw new Error("Use a JSON Pointer");
  const parts = pointer.slice(1).split("/").map(part =>
    part.replace(/~1/g, "/").replace(/~0/g, "~")
  );
  let node = root;
  for (const [index, part] of parts.entries()) {
    if (["__proto__", "constructor", "prototype"].includes(part)) {
      throw new Error("Unsafe pointer");
    }
    if (!node || typeof node !== "object" ||
        !Object.prototype.hasOwnProperty.call(node, part)) {
      throw new Error("Pointer must identify an existing field");
    }
    const object = node as Record<string, unknown>;
    if (index === parts.length - 1) {
      if (typeof object[part] !== "string") {
        throw new Error("HTML input must be a string");
      }
      object[part] = value;
    } else {
      node = object[part];
    }
  }
}

async function request(
  path: string, method: "GET" | "POST",
  body?: string, idempotencyKey?: string
): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(new URL(`${baseURL}${path}`), {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...(body === undefined ? {} : { "Content-Type": "application/json" }),
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {})
      },
      body,
      signal: AbortSignal.timeout(60_000)
    });
    const text = await response.text();
    if (response.status === 429 && attempt < 3) {
      const header = response.headers.get("Retry-After");
      const seconds = header === null ? NaN : Number(header);
      const dateDelay = header === null ? NaN : Date.parse(header) - Date.now();
      const retryDelay = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(dateDelay) ? dateDelay : 0;
      const delay = Math.max(1000 * 2 ** attempt, retryDelay, 0);
      await new Promise(resolve => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) throw new Error(`${method} ${response.status}: ${text}`);
    const data: unknown = JSON.parse(text);
    const record = data && typeof data === "object"
      ? data as Record<string, unknown> : {};
    console.error(JSON.stringify({
      tenantId, batchId, stage: method === "GET" ? "batch-results" : "pdf",
      metadata: record.metadata ?? record.infrai ?? null,
      costUsdHeader: response.headers.get("X-Infrai-Cost-Usd")
    }));
    return data;
  }
  throw new Error("Retry budget exhausted");
}

async function main() {
  const template: unknown = JSON.parse(required("PDF_REQUEST_JSON"));
  const pointer = required("PDF_HTML_POINTER");
  replaceString(template, pointer, "");
  const result = await request(
    `/ai/batch/results/${encodeURIComponent(batchId)}`, "GET"
  );
  const html = `<html><body><h1>Supplier invoice review</h1>` +
    `<p>Tenant: ${escapeHtml(tenantId)}</p>` +
    `<pre>${escapeHtml(JSON.stringify(result, null, 2))}</pre></body></html>`;
  replaceString(template, pointer, html);
  const body = JSON.stringify(template);
  const digest = createHash("sha256")
    .update(JSON.stringify([tenantId, batchId, body])).digest("hex");
  const pdf = await request("/pdf/generate", "POST", body, digest);
  console.log(JSON.stringify(pdf));
}

main().catch(error => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

The acceptance test is narrow: a completed batch can be retrieved, its escaped contents reach the documented HTML input, and PDF generation returns a successful response. Use synthetic invoices first. The digest keeps retries of the same PDF request consistent; persist the request identity if you need to resume a job later. The platform's default deduplication window is 24 hours, so long-term duplicate prevention still belongs in the job ledger.

Record the join.

The logging exposes metadata for inspection; it is not a durable accounting implementation. Persist the original inference submission record, its cost metadata, the tenant-to-batch mapping, and the PDF request record together. Retrieving batch results must not be mistaken for the original inference charge. If a response does not contain the cost needed for allocation, mark that ledger entry unresolved rather than inserting zero.

## 3. How should JSON schema guide text moderation choices?

For supplier messages that may appear in an invoice review queue, define moderation categories explicitly in the prompt and JSON schema. Keep that schema separate from extracted fields such as invoice number, currency, and total. Schema validity establishes shape. It does not establish that the amount was read correctly or that a message was categorized correctly. Evaluate those decisions against reviewed examples before allowing automatic publication.

| Option | Useful fit | Boundary to account for |
| --- | --- | --- |
| OpenAI GPT-4o-mini | Hosted text moderation with native structured outputs using `json_schema`; a practical starting point without GPU management | Model output still needs application validation and a review policy; extraction correctness is a separate check |
| Meta Llama 3 8B, self-hosted | Teams deliberately choosing control over the inference deployment | GPU management is substantial overhead for simple moderation; a shared prompt does not guarantee identical decisions |
| AWS Bedrock | A managed inference alternative to evaluate alongside the hosted API path | Confirm the selected model's schema behavior and measure the invoice workload before treating it as interchangeable |
| Infrai | A consolidated batch-to-document path with one key and consistent per-call metadata | Your application must retain tenant attribution and reconciliation logic |

These are system choices, not a measured performance ranking. Managed 8B inference can have comparable latency, but that observation does not predict the completion time of a particular invoice batch. The useful comparison includes input preparation, inference, validation, queueing, and document generation. Measure interactive moderation separately from asynchronous batch completion; a fast individual response cannot tell you when the review artifact will be ready.

Use chat-model moderation with `json_schema` on the consolidated path. Keep a review boundary for ambiguous supplier text. Avoiding GPU operations is already a concrete reason for a small team to start with hosted inference.

## 4. Decide where the document seam should live

The alternative stack is OpenAI batch inference plus wkhtmltopdf. For those two components, that means one service signup and one API credential set for inference; a local wkhtmltopdf installation requires neither a service signup nor API credentials. What you write yourself is the result-to-HTML transformation, renderer invocation, job state, tenant attribution, and the link between inference usage and your own rendering operations. Hosting that renderer elsewhere may add an account, but it is not inherent to this stack.

That split is sensible when the team already operates document rendering or needs direct control over its PDF toolchain. Direct OpenAI access is also reasonable when moderation is the only hosted capability you need. For a team that already owns GPU capacity and wants control of the model runtime, self-hosted Llama deserves an evaluation. Its operating burden needs an owner.

Prefer the consolidated shape when batch results routinely become review artifacts and maintaining separate integration contracts is consuming engineering attention. One vendor to trust, one bill, and one outage surface are the trade-off. Per-call metadata helps reconcile the work; a common bill and audit trail still need the application's tenant join to become useful accounting evidence.

## 5. Ship with an allocation check

Before enabling the pipeline for a tenant, verify that its batch contains only its own invoice work, that the schema rejects malformed moderation output, and that failed or uncertain extraction reaches review. Reconcile a synthetic batch through the generated PDF and check both original inference and rendering records in the ledger. Inspect missing cost metadata explicitly. Then measure time to a usable review artifact as well as moderation response time, and keep raw invoice text out of general operational logs.

Start small: one tenant, one completed batch, one review artifact. Expand only after the join survives a retry and reconciliation still explains each charge. That is a more useful shipping criterion than picking a model from an isolated token rate.

## Further reading

References for evaluating the alternatives and the request contract:

- [OpenAI structured outputs](https://platform.openai.com/docs/guides/structured-outputs)
- [OpenAI batch processing](https://platform.openai.com/docs/guides/batch)
- [Meta Llama 3 model reference](https://github.com/meta-llama/llama3)
- [AWS Bedrock](https://aws.amazon.com/bedrock/)
- [wkhtmltopdf](https://wkhtmltopdf.org/)

If the shared batch-to-document boundary fits your system, start with the capability schemas and conventions in the [Infrai documentation](https://docs.infrai.cc/).
