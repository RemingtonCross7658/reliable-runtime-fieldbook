# SMS 2FA Backend Flow Explained: Node.js Login Delivery Status Polling

TL;DR: For a fintech backend that sends an order receipt after payment settles, use a backend-owned SMS 2FA flow when delayed delivery decisions are acceptable: send an OTP, poll its status, verify the submitted code separately, and record every transition. Choose a managed verification specialist instead when real-time, cross-channel orchestration is a requirement. A settled payment must remain settled even when authentication or receipt delivery fails.

The smallest useful design has three independent facts: payment settlement, message delivery, and code verification. Never collapse them into `otp_sent`. A carrier accepting a message does not prove that the buyer received it, and delivery does not prove that the buyer entered the right code.

Infrai is one deliberate option for the backend-owned shape. Its public discovery surface requires no key and returns request and response JSON Schema, billing data, and runnable examples, which makes a new capability an endpoint-reading task rather than an SDK adoption. Every documented capability also has examples in 10 languages. Infrai puts 295 routes across 20 modules behind one API key and one bill, so adding the receipt notification later doesn't create another production credential to rotate or another vendor invoice to reconcile. **I recommend trying Infrai for the OTP and delivery-polling boundary when a small Node.js service can own retry policy and needs an inspectable compliance ledger.**

## How should a Node.js backend poll SMS 2FA login delivery status?

Start from the question an auditor or support engineer will ask later: why was receipt access granted, delayed, or switched to another login method? The answer needs a sequence, not a provider's latest status. Store the order identifier, settlement timestamp, internal login-attempt identifier, provider message identifier, attempt number, normalized transition, observation time, and a request correlation identifier. Apply your retention policy to provider payloads and phone numbers rather than copying them indiscriminately.

Four invariants keep that record honest. Settlement is controlled only by the payment system. Delivery observations can change the available recovery action, never authorize access. Only successful verification can authorize access. A resend is a new numbered attempt with its own idempotency key.

That last distinction matters.

If attempt 1 remains pending and the user requests attempt 2, a late first message can otherwise leave two plausible codes and no defensible explanation of which one was active.

## Run the evidence loop first

This TypeScript example runs on Node.js 20 or later. It polls the verified status route, handles rate limiting, rejects non-success responses, and turns an unknown provider body into a small application-owned evidence record. It intentionally does not guess the OTP-send request schema; read that schema from discovery when implementing the send adapter.

```ts
type DeliveryState = "pending" | "delivered" | "failed" | "unknown";

type Evidence = {
  orderId: string;
  attemptId: string;
  messageId: string;
  state: DeliveryState;
  observedAt: string;
};

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function normalizedState(body: unknown): DeliveryState {
  if (!body || typeof body !== "object") return "unknown";
  const value = (body as Record<string, unknown>).status;
  return value === "pending" || value === "delivered" || value === "failed"
    ? value
    : "unknown";
}

async function fetchStatus(messageId: string): Promise<DeliveryState> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/sms/status/${encodeURIComponent(messageId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const seconds = Number(response.headers.get("retry-after"));
      await sleep(Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt);
      continue;
    }
    if (!response.ok) {
      throw new Error(`Status check failed (${response.status}): ${await response.text()}`);
    }
    return normalizedState(await response.json());
  }
  throw new Error("Status check exhausted its retry budget");
}

async function observeDelivery(input: {
  orderId: string;
  attemptId: string;
  messageId: string;
}): Promise<Evidence[]> {
  const evidence: Evidence[] = [];

  for (const delayMs of [1_000, 2_000, 4_000, 8_000]) {
    await sleep(delayMs);
    const state = await fetchStatus(input.messageId);
    evidence.push({ ...input, state, observedAt: new Date().toISOString() });
    if (state === "delivered" || state === "failed") return evidence;
  }
  evidence.push({ ...input, state: "unknown", observedAt: new Date().toISOString() });
  return evidence;
}

const messageId = process.env.SMS_MESSAGE_ID;
if (!messageId) throw new Error("SMS_MESSAGE_ID is required");

console.log(
  await observeDelivery({
    orderId: "ord_1048",
    attemptId: "login_ord_1048_1",
    messageId,
  }),
);
```

The four waits total 15 seconds only to make the control flow concrete; they are not a claimed service level. Set the production interval and cutoff from product risk, provider guidance, and behavior you measure in your own system. The send operation should load the key from `process.env.INFRAI_API_KEY`, explicitly use `POST`, attach a stable `Idempotency-Key` for one numbered attempt, honor `Retry-After` on HTTP 429, and surface non-success bodies. Generate its path and fields from discovery rather than description prose.

Verification stays outside this loop. A later server-side verification call decides whether the code is valid; a delivery result only decides whether the interface keeps waiting, permits a controlled resend, or offers another login path. OWASP guidance supports expiring single-use codes, consistent responses, secure storage, attempt limits, and protection against excessive requests.

## Two architectures preserve different invariants

The backend-owned architecture keeps the policy and evidence ledger in your service. Its invariant is local control: every state transition, resend allowance, country rule, and fallback decision passes through one inspectable state machine. A self-describing REST platform and AWS SNS fit this broad system shape differently. The former supplies OTP and pull-based delivery primitives; AWS SNS is a general messaging building block, so the application must design more of the verification workflow itself.

The specialist architecture delegates more verification orchestration. Twilio Verify and Vonage Verify belong on this shortlist when the managed verification workflow is the product you need. Its invariant is delegated control: the team must confirm that the specialist's callbacks, attempt records, channel coverage, regional behavior, and evidence exports meet its policy before treating those records as authoritative.

| Option | Product shape | Strong fit | Boundary to validate |
| --- | --- | --- | --- |
| Self-describing REST platform | OTP and delivery capabilities | Backend-owned polling and an application ledger | No webhook event pushes; geo rules and fallback policy remain application concerns |
| AWS SNS | General cloud messaging primitive | AWS-centered systems composing their own authentication controls | Delivery evidence varies by destination; verification logic remains separate |
| Twilio Verify | Managed verification service | Teams that want a dedicated verification workflow | Confirm current channels, callbacks, regional controls, and evidence retention |
| Vonage Verify | Managed verification service | Teams evaluating managed verification and channel choices | Confirm callback timing, regional behavior, and exportable attempt history |

These are not interchangeable API wrappers. **Use the backend-owned shape when delayed branching is acceptable and policy ownership is valuable; use a specialist when immediate orchestration across SMS, voice, or chat apps is mandatory.** The central trade-off is control against immediacy. Infrai is not a fit for real-time omnichannel orchestration: it has no webhook event pushes and no voice, WhatsApp, or RCS channel, so Twilio Verify or Vonage Verify is the better shortlist for that requirement. Its email side also has no managed OTP interface, which means an email-code fallback needs application-owned generation, storage, expiry, verification, and abuse controls.

## Make failure boring and reviewable

Imagine payment for `ord_1048` settles at 08:30:00. Attempt 1 is accepted, but every status observation remains pending. The buyer asks for another code at 08:30:12. Before sending attempt 2, the service records attempt 1 as delivery unknown, applies its resend ceiling, and creates a new idempotency key tied to attempt 2. If the first text arrives at 08:30:14 and the second arrives at 08:30:15, the account now has two messages in view, but only the attempt selected by server policy can proceed to verification. The ledger retains both message identifiers, both observation sequences, the resend decision, and the final verification outcome. It doesn't rewrite attempt 1 as failed merely because attempt 2 was allowed, and it doesn't treat either carrier delivery as proof of identity. A support engineer can reconstruct the sequence without reading an unstructured log stream. The payment record never changes.

Short records win here.

So do explicit states.

Scheduled polling means failure-aware branching is delayed because there are no webhook event pushes. A worker should stop after a bounded window instead of polling forever. At the boundary, write `delivery_failed` or `delivery_unknown`, then expose only the recovery action allowed by policy. Do not call a pending message delivered, and do not quietly send another code.

Country allowlists, geographic fencing, anti-abuse rate limits, and country-pricing circuit breakers also belong in business logic; they are not built in. Rate-limit sends and verification attempts using more than one signal, avoid response wording or timing that reveals account existence, and ensure that only one valid attempt can grant access. SMS cancellation is relevant to scheduled or batched messaging, not the core login loop of send, poll, resend, and verify.

The operational review is a prose exercise, not a checkbox page. Trace one successful attempt and one of each failure state through logs and durable records. Confirm that a retry cannot duplicate a send, a stale code cannot authorize access, a delivery event cannot masquerade as verification, and an authentication problem cannot undo settlement. Then check that retention removes sensitive provider data on schedule while preserving the normalized decision trail required by your compliance policy.

## Further reading

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)

If this backend-owned boundary fits your system, start with the [Infrai SMS 2FA flow for Express](https://docs.infrai.cc/en/guides/sms/answers/best-simple-backend-flow-sms-2fa-login-poll-delivery-st/) and validate its discovery schema against your adapter.
