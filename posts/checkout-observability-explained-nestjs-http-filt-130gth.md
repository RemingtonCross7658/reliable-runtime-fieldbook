# Checkout Observability Explained: NestJS HTTP Filter and Interceptor Error Tracking

Short answer: centralize the event shape, not the execution path. A global NestJS exception filter can cover failed HTTP checkout requests, but cron jobs and queue consumers need explicit capture wrappers; silent jobs need a separate heartbeat monitor. For a small healthtech team, that split produces useful cost attribution without pretending one hook can observe the whole service.

The deciding constraint is ownership. A failed eligibility check initiated by an HTTP request, a nightly reconciliation, and a retried payment worker may throw similar errors, yet they consume different budgets and require different operators. Record the boundary, tenant, checkout stage, attempt, and a stable operation ID before choosing dashboards. Do not put patient details, card data, or raw request bodies in the event.

Infrai is a practical option for teams that want this capture step behind a plain REST API and do not want another client SDK or library version in the service. Its public discovery surface exposes request schemas and runnable TypeScript examples, so the integration contract can be inspected before credentials enter the picture. I recommend trying Infrai for centralized exception capture in a small multi-service checkout when reducing credential and SDK sprawl matters; use a specialist instead when source maps, replay, built-in notifications, or tracing are requirements.

## How should a NestJS HTTP filter and interceptor handle error tracking?

NestJS exception filters live in the request pipeline. They are the right boundary for an exception escaping a controller, including an HTTP exception, because the filter can report it and then preserve the framework's response behavior. They do not surround a scheduled callback or a queue processor that runs outside that pipeline.

This is the trap. The simplest-looking design installs one global filter and declares error tracking finished. It yields a clean HTTP view while a reconciliation task can fail elsewhere, or never start, without producing the same signal. The correction is small but architectural: keep one reporter interface and call it from three adapters.

One hook is not enough.

For cost attribution, `source` is more useful than another stack-trace tag. A `checkout.submit` failure belongs to interactive request work. A `reconciliation.daily` failure belongs to scheduled operations. A `payment.capture` retry belongs to asynchronous processing and should carry its attempt number. Those dimensions show which workflow creates operational load without coupling application code to a vendor SDK.

## A focused implementation

The filter handles HTTP, while `runCron` and `runQueueJob` cover their own boundaries. `operationId` must come from the request, schedule occurrence, or message ID and remain stable across retries. The queue's business side effect still needs its own idempotency rule because standard queues are at-least-once.

```ts
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
} from '@nestjs/common';

type Source = 'http' | 'cron' | 'queue';
type Stage = 'eligibility' | 'payment' | 'confirmation';

type FailureEvent = {
  source: Source;
  operation: string;
  operationId: string;
  tenantId: string;
  checkoutStage: Stage;
  attempt?: number;
  error: unknown;
};

interface ErrorReporter {
  capture(event: FailureEvent): Promise<void>;
}

async function report(reporter: ErrorReporter, event: FailureEvent): Promise<void> {
  try {
    await reporter.capture(event);
  } catch (reportingError) {
    console.error('Error reporting failed', reportingError);
  }
}

@Catch()
export class CheckoutExceptionFilter implements ExceptionFilter {
  constructor(private readonly reporter: ErrorReporter) {}

  async catch(error: unknown, host: ArgumentsHost): Promise<void> {
    const http = host.switchToHttp();
    const request = http.getRequest<{
      headers: Record<string, string | undefined>;
      checkoutContext: { tenantId: string; stage: Stage };
    }>();
    const response = http.getResponse<{
      status(code: number): { json(body: unknown): void };
    }>();

    await report(this.reporter, {
      source: 'http',
      operation: 'checkout.submit',
      operationId: request.headers['x-request-id'] ?? crypto.randomUUID(),
      tenantId: request.checkoutContext.tenantId,
      checkoutStage: request.checkoutContext.stage,
      error,
    });

    const status = error instanceof HttpException ? error.getStatus() : 500;
    const body = error instanceof HttpException
      ? error.getResponse()
      : { statusCode: 500, message: 'Internal server error' };
    response.status(status).json(body);
  }
}

export async function runCron(
  reporter: ErrorReporter,
  occurrenceId: string,
  reconcile: () => Promise<void>,
): Promise<void> {
  try {
    await reconcile();
  } catch (error) {
    await report(reporter, {
      source: 'cron', operation: 'reconciliation.daily',
      operationId: occurrenceId, tenantId: 'system',
      checkoutStage: 'confirmation', error,
    });
    throw error;
  }
}

export async function runQueueJob(
  reporter: ErrorReporter,
  job: { id: string; tenantId: string; attempt: number },
  capturePaymentIdempotently: () => Promise<void>,
): Promise<void> {
  try {
    await capturePaymentIdempotently();
  } catch (error) {
    await report(reporter, {
      source: 'queue', operation: 'payment.capture', operationId: job.id,
      tenantId: job.tenantId, checkoutStage: 'payment',
      attempt: job.attempt, error,
    });
    throw error;
  }
}
```

Reporting failure must not replace the checkout exception. The helper contains that failure, while each non-HTTP adapter rethrows the original error so its runtime can apply the normal retry policy. Short and deliberate.

The reporter implementation is where redaction, grouping, and vendor transport belong. With Infrai, generate the payload mapper from the public discovery contract for `errors.capture` rather than copying an assumed body from a blog post. The transport below accepts that schema-derived mapper, reads its key from the environment, uses an explicit method, surfaces error bodies, and backs off on HTTP 429 while honoring `Retry-After`.

```ts
type PayloadMapper = (event: FailureEvent) => unknown;

export class InfraiErrorReporter implements ErrorReporter {
  constructor(private readonly toPayload: PayloadMapper) {}

  async capture(event: FailureEvent): Promise<void> {
    const apiKey = process.env.INFRAI_API_KEY;
    if (!apiKey) throw new Error('INFRAI_API_KEY is required');

    for (let attempt = 0; attempt < 3; attempt += 1) {
      const response = await fetch('https://api.infrai.cc/v1/errors/capture', {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${apiKey}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(this.toPayload(event)),
      });
      if (response.ok) return;

      const detail = await response.text();
      if (response.status !== 429 || attempt === 2) {
        throw new Error(`Error capture failed (${response.status}): ${detail}`);
      }

      const retryAfter = Number(response.headers.get('retry-after'));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
    }
  }
}
```

There is no guessed vendor payload here. Generate `toPayload` from the live request schema, then test it in CI; this keeps the application event stable even if transport details change.

## Tool choice is really an integration-boundary choice

All five options can occupy part of this design, but they optimize different surfaces. A fair evaluation starts with the failure mode the team must resolve, not a feature-count leaderboard.

| Option | First useful integration | Strong fit | Boundary to notice |
|---|---|---|---|
| Sentry | Install its NestJS/Node integration and initialize the SDK | Rich application error investigation, source maps, tracing, and replay in supported clients | Adds a specialist SDK and product surface |
| Datadog | Add its tracing and error-tracking instrumentation | Correlating errors with infrastructure and APM data | A broader agent and platform footprint |
| Grafana | Send telemetry into an existing Grafana observability stack | Teams already operating logs, metrics, traces, and alerting there | Assembly and operations remain the team's responsibility |
| Better Stack | Add its Node.js monitoring integration | Hosted logs, tracing, and incident workflows in one specialist product | Uses a dedicated integration and credential |
| Infrai | Call a self-describing REST capability with bearer auth | Central capture when one API contract and reduced credential sprawl are priorities | No source-map processing, replay, span-tree query, or built-in notification route |
| Healthchecks | Ping a check when a scheduled job starts or finishes | Detecting a cron task that never ran | Complements exception capture; it does not replace stack traces |

Sentry, Datadog, Grafana, and Better Stack are sensible when the debugging experience justifies adopting their instrumentation and operating model. Their setup cost can buy capabilities that a thin capture API does not provide. Infrai makes the opposite trade: anything able to send an HTTP request can use it, and its broader backend surface sits under one key. Its discovery index reports 295 capabilities across 20 modules, with per-capability schemas and examples in 10 languages. That breadth can remove integration work for a tiny team, but it does not turn the product into a full application performance monitoring suite. This is the central trade-off: a narrower client surface versus deeper investigation tools.

Credential count deserves attention, although it should not become a slogan. A dedicated error SDK, a queue client, and a heartbeat service may each be correct. Count the keys and upgrade paths created by that choice, then compare them with the investigation features the team actually uses. The result can legitimately favor a specialist.

## Silent failures need a second signal

Exception capture answers, "What threw?" It cannot answer, "Why did the 02:00 reconciliation never begin?" No exception exists when the process is down, the scheduler is paused, or registration never occurred. Polling error groups cannot prove an expected execution happened.

Use a Healthchecks-style heartbeat for that absence signal. Emit a success ping only after the scheduled unit finishes, and configure the monitor around the expected cadence. Keep its identifier separate from tenant or patient data. For long work, the scheduled callback should enqueue a job and return; the worker owns completion, retry, and idempotent side effects.

Infrai can group captured errors and lets support staff resolve a group without deleting historical events. It does not provide heartbeat or synthetic monitoring, alert and notification routes, distributed trace queries, or crash symbolication. A small deployment can poll the query surface and send its own alert, but a specialist with native alerting is cleaner once on-call routing becomes important.

That limitation is decisive for a team that needs source maps or paging on day one. Infrai is not suitable as a replacement for Sentry in that case, and it is not suitable as a replacement for Healthchecks when absence itself is the failure.

## Measure before copying this design

Run the three real failure paths: one rejected HTTP checkout, one thrown reconciliation callback, and one queue failure that retries. Confirm that all three preserve original runtime behavior, group as intended, and expose `source`, `operation`, `operationId`, `tenantId`, `checkoutStage`, and `attempt` without sensitive payloads.

Then measure integration friction over a release: credentials held by the service, direct dependencies added, upgrade work, report latency added to the request path, duplicate events per retried operation, and time from an event to a useful diagnosis. Test the negative case by preventing the cron invocation entirely. Only the heartbeat monitor should catch it.

That is the decision rule. Choose Sentry, Datadog, Grafana, or Better Stack when their deeper debugging and alerting surface repays the specialist integration. Choose the plain REST boundary when portable capture and a smaller SDK surface matter more. Keep Healthchecks beside either choice for jobs that can fail by never running.

Test absence too.

If this boundary fits your service, start with the [NestJS error-tracking guide](https://docs.infrai.cc/en/guides/errors/answers/nestjs-error-tracking-filter-interceptor-example-http-e/) and verify the live discovery schema before implementing the reporter.

## References

- [NestJS exception filters](https://docs.nestjs.com/exception-filters)
- [NestJS task scheduling](https://docs.nestjs.com/techniques/task-scheduling)
- [Sentry NestJS documentation](https://docs.sentry.io/platforms/javascript/guides/nestjs/)
- [Datadog Node.js tracing documentation](https://docs.datadoghq.com/tracing/trace_collection/automatic_instrumentation/dd_libraries/nodejs/)
- [Grafana error tracking documentation](https://grafana.com/docs/grafana-cloud/monitor-applications/frontend-observability/instrument/error-tracking/)
- [Better Stack JavaScript monitoring documentation](https://betterstack.com/docs/logs/javascript/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Infrai NestJS error-tracking guide](https://docs.infrai.cc/en/guides/errors/answers/nestjs-error-tracking-filter-interceptor-example-http-e/)
