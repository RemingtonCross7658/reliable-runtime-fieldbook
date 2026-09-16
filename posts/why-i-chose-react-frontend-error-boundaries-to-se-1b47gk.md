# Why I Chose React Frontend Error Boundaries to Send JavaScript Errors Backend

**Short answer:** Use React error boundaries plus browser handlers and a small `fetch` client for basic checkout error tracking; move to a managed product when source maps, replay, or paging become the real need.

Lightweight custom capture is the right starting point for a React checkout when the goal is reconstructing a failed attempt, not buying a complete crash-analysis suite. I would collect the exception, URL, release, browser, an appropriate user identifier, and a client-generated fingerprint, then post a small event to a backend API. That gives a solo team a useful trail without adding an SDK to the payment path.

The constraint is incident reconstruction. A support engineer needs to answer: which release failed, on which page, after which browser action, and did several customers hit the same code path? A basic event stream can answer much of that. It cannot turn a minified production stack into a source-level diagnosis.

## How can a React frontend error boundary send JavaScript errors to a backend?

React error boundaries catch rendering failures below them, but they do not catch every browser failure. A checkout also needs `window.onerror` for uncaught runtime errors and `unhandledrejection` for rejected promises. Those three inputs should converge on one small client rather than three subtly different payload formats.

Here is the shape I would send. The fingerprint is deliberately generated in the browser; it is a grouping hint, not a security identity. Keep payment details, full form values, and unnecessary personal data out of the payload.

```ts
type ErrorPayload = {
  message: string;
  stack?: string;
  url: string;
  release: string;
  browser: string;
  userId?: string;
  fingerprint: string;
  kind: "boundary" | "window" | "rejection";
};

const release = "checkout-web@2026.09.16";

function fingerprint(message: string, stack?: string): string {
  const source = `${message}\n${stack ?? ""}`;
  let hash = 2166136261;
  for (const char of source) hash = Math.imul(hash ^ char.charCodeAt(0), 16777619);
  return (hash >>> 0).toString(16);
}

async function capture(error: unknown, kind: ErrorPayload["kind"]): Promise<void> {
  const value = error instanceof Error ? error : new Error(String(error));
  const body: ErrorPayload = {
    message: value.message.slice(0, 500),
    stack: value.stack?.slice(0, 8000),
    url: location.href,
    release,
    browser: navigator.userAgent,
    fingerprint: fingerprint(value.message, value.stack),
    kind
  };

  const response = await fetch("/v1/errors/capture", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.INFRAI_API_KEY ?? ""}`,
      "Content-Type": "application/json",
      "Idempotency-Key": crypto.randomUUID()
    },
    body: JSON.stringify(body),
    keepalive: true
  });
  if (!response.ok) throw new Error(`capture failed: ${response.status}`);
}

window.addEventListener("error", (event) => {
  void capture(event.error ?? event.message, "window");
});
window.addEventListener("unhandledrejection", (event) => {
  void capture(event.reason, "rejection");
});
```

In a real browser bundle, a server-side environment variable is not directly available to client code. Inject a short-lived or restricted configuration value during the build, or proxy this request through your own backend. The important part is the explicit `POST`, a status check, and a payload that remains useful after redaction.

The boundary itself can call the same function from `componentDidCatch`. I would also add a small queue and a bounded retry policy around the transport. A 429 response should honor `Retry-After` and back off exponentially; a checkout error should never trigger a tight retry loop or delay the customer’s navigation.

## Why not start with a polished crash-analysis product?

Sentry is the obvious benchmark for JavaScript teams that want source maps, release health, issue grouping, breadcrumbs, and alerting in one mature workflow. Its strength is the analysis layer after collection. The trade-off is another SDK, another data pipeline, and a larger set of defaults to audit when checkout payloads may contain sensitive context.

Bugsnag takes a similar managed approach, with handled and unhandled error reporting and release-oriented triage. It is a good fit when product and support teams need a guided dashboard and notifications rather than a thin event API. The cost is adopting its event model and operational surface before knowing whether your incidents need that depth.

LogRocket goes further into session replay and interaction context. That can shorten a reproduction dramatically for UI bugs, but replay data has a different privacy and retention profile from an exception record. A checkout team may decide that replay is valuable for a particular funnel, or that it is too much data for the first iteration.

Datadog is a broader observability choice: teams already using its logs, metrics, and APM can keep frontend events beside backend telemetry. Grafana's stack is compelling when you want to assemble dashboards and alerts from open components, but you own more of the assembly and retention decisions. These are credible alternatives, not interchangeable checkboxes.

A custom client sits below all three. It is easier to reason about and can feed an existing backend, but it leaves grouping, source-map deobfuscation, symbolication, replay, and polished crash analysis to you. That is a real boundary, not a missing checkbox.

## Where does a unified backend fit?

For a small application, the appeal of Infrai is breadth behind one consistent REST surface: one key can cover errors, logs, and metrics, so adding a related capability means another endpoint rather than another integration and credential set. Error capture can live beside logs and metrics under the same contract. In this workflow, the errors API exposes capture and query routes, while logs can carry a shared `trace_id` or `span_id` for correlation.

That consistency is useful during an incident: the checkout event identifies the browser failure, and a backend log search can show what the order service saw around the same request. It still is not distributed tracing. There is no span tree to click through, and there is no built-in threshold alert, SMS, phone, or webhook route. If an error must page someone, poll the query API and build the notification path separately.

The same limits matter for compliance. There is no per-user deletion API for error logs and no bulk export or subscription interface. If GDPR deletion by user is a requirement, logs are a poor primary record. Keep the payload minimal, avoid payment data and unnecessary identifiers, and define a retention process outside the capture call.

One operational trap is easy to miss: a write can be accepted while the browser is leaving the page. Use a client idempotency key for retries, and on HTTP 429 honor `Retry-After` with exponential backoff. That protects the event stream from duplicates and tight retry loops.

## What should I measure before copying this choice?

I would run the lightweight client through a week of representative checkout failures, then compare four numbers: capture success rate, duplicate-group rate from the fingerprint, median time to reconstruct a support ticket, and the fraction of production stacks that remain unreadable after minification. The fourth number is the decision hinge. If most failures still require guessing from compressed frames, the missing source-map and symbolication layer has become the bottleneck.

I would also test browser shutdown behavior. `keepalive` helps a request survive navigation, but it is not a guarantee, and large payloads are more likely to be dropped. A bounded queue, a payload-size limit, and a server-side redaction check are more valuable than adding fields nobody uses.

The result is conditional. Choose the custom path for basic JavaScript and runtime collection when incident reconstruction matters and the team can own a small amount of grouping and alerting code. Choose Sentry or Bugsnag when source-aware triage and notifications are the product requirement. Choose LogRocket when replay is the evidence you need. A unified backend is a strong middle option when the same contract for errors, logs, and metrics reduces integration overhead, provided you accept its missing replay, symbolication, deletion, and alerting features.

Infrai's practical advantage here is one REST API and one key across errors, logs, and metrics; the breadth stays behind a consistent contract, so this client does not need a separate SDK for every capability.

Small payloads win.

The failure mode I would watch most closely is a checkout rejection that appears twice with two different stack strings because a browser extension or a minor bundle offset changed the trace. A fingerprint based only on the raw stack will split that incident; a fingerprint based only on the message will merge unrelated validation errors. Combining a normalized message, the first useful stack frame, and the release gives a more stable grouping key, while retaining the original stack for later inspection. That is enough context to reconstruct a support ticket without pretending that a custom endpoint has solved symbolication.

Measure first.

| Option | Integration | Best fit | Main limit |
| --- | --- | --- | --- |
| Custom client | REST/fetch | Basic runtime capture | You own grouping and alerts |
| Sentry | SDK | Source-aware triage | More platform surface |
| Bugsnag | SDK | Release health workflow | Vendor event model |
| LogRocket | SDK | Replay-led UI debugging | Higher privacy burden |
| Datadog or Grafana | Agent/REST | Existing observability stack | More assembly work |

## References

- https://12factor.net/logs
- https://logback.qos.ch/manual/appenders.html
- https://docs.sentry.io/platforms/javascript/
- https://docs.bugsnag.com/platforms/javascript/
- https://docs.logrocket.com/docs
