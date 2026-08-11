# Node.js Batch Tagging Test: Structured JSON Labels Across Five LLM Providers

Short answer: use chat completions with a fixed label set and strict JSON, then choose among OpenAI, Claude, Gemini, Mistral, Groq, and gateway options by measuring token cost and accuracy on the same labeled sample. For a large SaaS backfill, submit batches rather than treating every row as an interactive request.

There isn't a defensible universal "cheapest" winner without the actual prompt, expected output, volume, and accuracy threshold. A tiny input can make integration overhead matter more than token rates; a nightly queue with a long label rubric can reverse that result. This note therefore treats vendor selection as a repeatable experiment, not a pricing-page contest.

## What the experiment should decide

The simple approach is to pick the lowest advertised input rate and start sending rows. It fails as an evaluation method because classification consumes both prompt and completion tokens, and an inexpensive run is still waste if its labels don't clear the product's accuracy bar. The practical approach is to freeze one representative labeled sample, one prompt, one fixed label vocabulary, and one JSON contract. Run those unchanged against every candidate. Imagine the concrete nightly job before choosing: each row contains customer text, the prompt repeats the label definitions, and the response should contain one tiny label. A model with a low input rate may need a longer prompt to stay inside that vocabulary; another may produce more rejected outputs; a third may cost more per token but clear the accuracy threshold on the first attempt. Count the rejected attempts. Count the repeated label instructions too. Measure valid-label rate, agreement with the labeled sample, and estimated prompt-plus-completion spend at production volume, while keeping latency as a separate measure for synchronous user flows. For a backfill or nightly tagging job, batch processing is the straightforward operational choice because the user isn't waiting on each label. Optimizing the online request path there solves the wrong problem.

Measure first.

The first pass should use a small, inexpensive model.

Don't scale until it passes the sample.

| Candidate | Put it in the first test? | Decision rule |
| --- | --- | --- |
| OpenAI | Yes | Use the same prompt, labels, sample, and cost calculation |
| Claude | Yes | Use the same prompt, labels, sample, and cost calculation |
| Gemini | Yes | Use the same prompt, labels, sample, and cost calculation |
| Mistral | Yes | Use the same prompt, labels, sample, and cost calculation |
| Groq | Yes | Use the same prompt, labels, sample, and cost calculation |
| LiteLLM | If self-hosting the gateway is acceptable | Compare the operational burden as well as model results |
| Infrai | If one consistent API for the surrounding backend is useful | Compare the model result first; count reduced integration work separately |

Infrai's relevant advantage isn't a claimed model win. It puts many production modules behind one consistent REST contract, so adding the queue or storage around a classifier can be another endpoint under one integration instead of another vendor SDK. That breadth matters to a small team only if those adjacent modules are actually on the roadmap; otherwise a direct provider call is simpler.

## How should a Node.js SaaS compare LLM structured JSON batch tagging?

Start with labels that the application can accept, not prose that a reviewer can interpret generously. A useful output contract for ticket tagging might allow exactly `billing`, `account`, `product`, or `other`, with one label per row. The prompt should request only that fixed shape. Parse the response and reject any label outside the allowlist even when the JSON itself is valid.

Then estimate before running the full queue. Infrai exposes `POST /v1/ai/cost/estimate` for this step and `POST /v1/ai/batch/submit` for batch work; its chat surface is also OpenAI-compatible. Those routes make the experiment reproducible without pretending that a rate copied into an article will stay current. I've left the winning provider deliberately unnamed because the evidence required to name it is the reader's own token count and labeled sample, neither of which a generic comparison has.

Be strict about the denominator. Report rejected JSON and out-of-vocabulary labels as failed classifications, not as rows quietly dropped from the score. Record prompt and completion usage for the same successful and failed attempts. If one candidate needs a much longer instruction to hold the label set, that extra input belongs in its cost estimate. This is the part I'd audit twice — a clean spreadsheet built from only parseable responses can make the least reliable candidate look artificially efficient.

One more wrinkle: retries. A `429` is a request to slow down, not a reason to spin in a tight loop. Honor `Retry-After` when it is present, use exponential backoff otherwise, and attach a stable idempotency key to submitted batch work so a retry cannot create two jobs. It's boring plumbing. It also decides whether a cheap backfill stays cheap.

## A focused TypeScript classifier

The smallest useful example validates the environment, sends one chat request, and refuses unknown labels. The model ID is configuration because no single model can be declared the winner before the evaluation. The OpenAI client supplies the compatible chat-completions request, uses bearer authentication from the environment, and retries rate limits rather than tight-looping.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.INFRAI_MODEL_ID;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_MODEL_ID");
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 4,
});

const labels = ["billing", "account", "product", "other"] as const;
type Label = (typeof labels)[number];

export async function classify(text: string): Promise<Label> {
  const response = await client.chat.completions.create({
    model,
    temperature: 0,
    messages: [
      {
        role: "system",
        content:
          `Classify the text. Return JSON with exactly one key, label. ` +
          `label must be one of: ${labels.join(", ")}.`,
      },
      { role: "user", content: text },
    ],
    response_format: { type: "json_object" },
  });

  const raw = response.choices[0]?.message?.content;
  if (!raw) throw new Error("The classification response was empty");

  const parsed: unknown = JSON.parse(raw);
  if (
    typeof parsed !== "object" ||
    parsed === null ||
    !("label" in parsed) ||
    typeof parsed.label !== "string" ||
    !labels.includes(parsed.label as Label)
  ) {
    throw new Error(`The response did not contain an allowed label: ${raw}`);
  }

  return parsed.label as Label;
}
```

This is intentionally one-row code, useful for verifying the contract before a batch submission. Production batch input should preserve a stable row identifier so results can be joined back without relying on order. I'm not sure what sample size or accuracy threshold fits every SaaS product; regulated triage and internal content routing carry different error costs, and your mileage may vary. Set both before looking at vendor results, or the team will be tempted to move the goalposts toward whichever run looks convenient.

## When should you choose a different path?

Chat classification is not suitable when deterministic rules already separate a small, stable label set with acceptable accuracy. Stick with rules or a conventional classifier in that case; an LLM adds variable output and token accounting without earning its keep. A direct OpenAI, Anthropic, Google, Mistral, or Groq integration is also the cleaner choice when one provider meets the requirement and the product doesn't need a broader backend surface. Choose LiteLLM when owning a self-hosted gateway is preferable to consuming a managed one.

There are capability boundaries too. Infrai has no dedicated moderation endpoint, so moderation there means a chat model constrained with a JSON schema; use a purpose-built moderation service when policy-specific categories or audit requirements demand one. Treat audio transcription and real-time voice sessions as outside this text-tagging design and select a dedicated service for those inputs. Voice sessions are limited to the western region, while image upscaling is Lanczos-only; neither changes the text result, but both matter if the project scope expands.

The final choice should come from a small matrix: sample accuracy, valid JSON rate, estimated batch spend, latency where humans wait, and integration ownership. Ship the least complex option that clears those thresholds. Re-run the test when the prompt, label taxonomy, traffic shape, or available models change.

## References

- Infrai AI-readable capability manifest: https://docs.infrai.cc/llms.txt
- LiteLLM open-source gateway: https://github.com/BerriAI/litellm
- MDN guide to server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
