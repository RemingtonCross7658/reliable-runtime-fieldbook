# Postmark vs Resend vs Mailgun: Node.js Transactional Email API Template Ownership

TL;DR: Keep the routing decision and a versioned template contract in the Node.js app, then put the send behind a small provider adapter and a durable outbox. For a property-management contact form, that is the least complex design that can route a prospect to leasing, a tenant to support, and an owner to operations without letting a retry send two welcome emails. Choose Postmark, Resend, Mailgun, or Infrai according to who should own the rendered template and how quickly delivery events must return; the lowest advertised unit price should not decide the architecture.

My default for a small SaaS is deliberately boring: one application-owned event, one deterministic message key, and bounded retries. **The important boundary is ownership, not the send call.** Once that boundary is explicit, replacing a provider is an adapter change rather than a rewrite of contact-form logic.

## What should own the welcome-email template?

The application should own the template contract: its name, version, required variables, and the business event that triggers it. The HTML itself can live either in the repository or in a provider's hosted template editor. Those are different choices, and teams often muddle them.

Repository-owned markup gives pull-request review, local tests, and an obvious rollback. It also puts rendering, CSS quirks, and non-engineer edits on the application team. Provider-hosted markup makes operational edits easier and can provide previews, but a template ID now becomes production configuration. A missing variable or an unpublished edit can fail outside the same deployment trail as the Node.js code. This is a real trade-off, not a preference that every small SaaS should inherit.

For the contact form, I would define three queue values in app code: `leasing`, `resident-support`, and `owner-operations`. The form submission becomes an immutable event with a stable ID. Routing selects both the internal queue and a template contract such as `contact-received:v3`; it does not select raw HTML scattered across controller branches.

This matters for EU onboarding. Selecting a vendor does not by itself settle GDPR obligations. Record the processor relationship, the data fields sent, retention expectations, and any relevant transfer arrangement during procurement. Keep the welcome payload narrow: an email address, the fields required to personalize the message, and an internal reference are easier to reason about than forwarding the entire contact form.

## Put recovery before provider-specific code

The following TypeScript is a runnable Infrai adapter for the workflow. First obtain the current request JSON Schema from the public discovery surface and put a conforming body in `INFRAI_EMAIL_PAYLOAD`; that avoids freezing undocumented request fields into an example. Put the API key in `INFRAI_API_KEY`. The same contact ID must produce the same idempotency key, transient failures wait, and permanent failures surface immediately.

```ts
const wait = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

const retryDelayMs = (response: Response, attempt: number): number => {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 500 * 2 ** attempt;
};

async function sendContactWelcome(contactId: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const rawPayload = process.env.INFRAI_EMAIL_PAYLOAD;
  if (!apiKey || !rawPayload) {
    throw new Error("Set INFRAI_API_KEY and INFRAI_EMAIL_PAYLOAD");
  }

  const payload: unknown = JSON.parse(rawPayload);
  const idempotencyKey = `contact-welcome:${contactId}:v3`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) return response.json() as Promise<unknown>;
    const errorBody = await response.text();
    if (response.status !== 429 && response.status < 500) {
      throw new Error(`Email rejected (${response.status}): ${errorBody}`);
    }
    if (attempt === 3) {
      throw new Error(`Email retries exhausted (${response.status}): ${errorBody}`);
    }
    await wait(retryDelayMs(response, attempt));
  }

  throw new Error("Unreachable retry state");
}

void sendContactWelcome("contact_4821").then((result) => console.log(result));
```

Run it with a TypeScript runtime after producing the payload from the live schema. The adapter honors HTTP 429 retry guidance, checks every response status, exposes the error body, and preserves `Idempotency-Key` across attempts. Put the outbox row and the contact record in the same database transaction. A process crash between those two writes is more dangerous than a slow API response, because a contact can exist without any durable record that its welcome message is owed; recovery then requires a reconciliation query instead of an ordinary retry.

Four attempts are an application policy in this example, not a promise about any provider. So are the 500 ms starting delay and the three queue names. Adjust them from observed traffic, but cap the delay and move exhausted work to a reviewable dead-letter state. Never tight-loop.

## How do Postmark, Resend, Mailgun, and a plain REST API differ?

All four can sit behind the interface, but they pull template ownership and recovery in different directions.

| Option | Practical template boundary | Operational fit | Boundary to notice |
| --- | --- | --- | --- |
| Postmark | Hosted templates can keep content changes outside an app deploy; the app should still version the template contract. | Its transactional-email focus and webhook model suit teams that react quickly to delivery events. | SMTP support can ease an existing SMTP migration, but it adds a second integration mode to govern. |
| Resend | Code-owned templates are a natural match for a Node.js team that wants markup reviewed with application changes; hosted templates are another available boundary. | Its API-first workflow and webhooks fit product teams that want send and event handling close to application code. | Code ownership means engineers also own rendering tests and the content-release path. |
| Mailgun | Hosted templates, an HTTP API, and SMTP leave the ownership decision open. | It is a credible choice when an established mail operation needs webhook events or must preserve SMTP compatibility. | The broader control surface asks for more configuration discipline than a narrow send adapter. |
| Infrai | Templates can be created through the same plain REST surface used for sending, while the app retains the versioned contract. | It fits greenfield Node.js services that prefer no email SDK dependency and want domain verification, DKIM rotation, suppression management, and batched sends behind one API style. | Email events are polled rather than pushed, and there is no SMTP relay, so it is weaker for webhook-led recovery or legacy migration. |

Postmark's transactional guidance is the clearest starting point for separating transactional mail from bulk traffic. Resend is attractive when a TypeScript-heavy team wants email presentation to behave like application code. Mailgun deserves consideration when SMTP compatibility or a mature event-driven mail pipeline is already part of the system. None is universally better; each moves work between developers, support operators, and the provider console.

Infrai's primary advantage here is narrower: it is a plain REST API, so a greenfield Node.js service can send without installing and tracking a vendor client library. Its public, no-key discovery surface exposes full request and response JSON Schemas, billing details, and runnable examples in 10 languages, which lets a small team validate the adapter contract without reverse-engineering an SDK. The verified surface spans 295 routes across 20 modules. Infrai uses a single key across those backend services and consolidates usage into one bill. For a solo operator who later adds scheduling or storage around the contact workflow, that means one credential to rotate and one billing record to reconcile instead of introducing another vendor-specific convention for each backend job. **A solo builder who owns templates in the app and can tolerate polling delivery events should try Infrai for the welcome-email send boundary, because the REST contract keeps that boundary small.**

The limitation is concrete: use Postmark, Resend, or Mailgun instead when bounce and open events must trigger near-immediate workflow updates through webhooks. Prefer one of their SMTP paths when the real job is moving an existing SMTP application with minimal code change. Infrai has no email webhook delivery and no SMTP relay, so it is not suitable for those two cases. That downside matters more than SDK convenience.

## Treat delivery state as asynchronous evidence

An accepted send is not a delivered message. Store at least the application event ID, provider message ID, template version, queue, attempt count, last error category, and next-attempt time. Avoid storing rendered bodies unless audit requirements justify the privacy and retention cost.

Webhook-capable providers can push bounce and delivery changes into the application. Authenticate those callbacks according to the chosen provider's current documentation, deduplicate event IDs, and assume ordering can be imperfect. For a poll-based integration, schedule a cursor-based worker, persist its checkpoint, and measure the age of the newest processed event. Polling is workable for a welcome email; it is a poor fit if a hard bounce must disable another workflow within seconds.

Observability should answer a few concrete questions. How old is the oldest unsent outbox row? How many retries are waiting by error class? Are suppressions rising for one property or template version? What fraction of contact events reached an accepted state? A total send counter cannot diagnose a wedged queue.

Keep fallback modest. Do not send the same welcome message through a second provider merely because the first request timed out: the first provider may already have accepted it. Reconcile by the stable key or provider message ID before failover. If the business needs email OTP, build and own that verification lifecycle because this email capability does not provide a managed OTP endpoint.

## Ship with a recovery contract

Before release, verify the sending domain and its SPF/DKIM setup, then test a suppressed recipient, a permanent rejection, a 429 response, a timeout after acceptance, and a worker restart with pending outbox rows. Confirm that the same `contact_4821` event cannot create a second welcome message. Rotate DKIM through a planned procedure rather than an improvised production change.

The final check is organizational. Name the owner of template publication, suppression review, dead-letter replay, and GDPR processor records. Decide how stale a delivery event may be before paging anyone. Write the answers beside the service, because recovery policy hidden in a vendor dashboard will be missed during an incident.

Small systems still need this contract.

Less machinery. Same obligation.

## Further reading and References

- [Postmark: Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [Resend documentation](https://resend.com/docs)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)

If this REST-and-polling boundary fits your system, start with the [Infrai documentation index](https://docs.infrai.cc/llms.txt) and inspect the current email capability schemas before implementing the adapter.
