# How to Rotate Realtime Connection Tokens — Multiplayer Quiz Failure Handling

+Short answer: rotate a realtime connection token by opening an authenticated replacement connection, resuming from the last applied event, and retiring the old connection only after the replacement is ready. For a multiplayer compliance quiz inside a fintech workspace, this keeps the online roster and answer stream coherent even when expiry, reconnect, and backfill happen at nearly the same moment. A reconnect without a cursor is guesswork; a token refresh without a connection handoff leaves an avoidable gap.

The data flow is small enough to reason about. The client obtains a short-lived connection token, opens the realtime channel, and applies only events whose sequence is newer than its local cursor. Before expiry, it asks the application backend for another token. It then starts a second channel with the cursor, waits for the server to backfill missed events and declare the channel live, switches the active generation, and closes the first channel. The quiz UI derives `who is online` from the ordered event state rather than from socket open/close notifications.

## How should a multiplayer quiz game handle realtime connection token rotation?

Treat rotation as a state transition, not an in-place string replacement. The token authenticates a connection attempt; the event cursor protects continuity. Those are different jobs. The clean invariant is: at most one connection generation may update the visible quiz state, but a newer generation may warm up while the current one is still serving reads.

Don't delete the old channel as soon as the refresh request starts. Token issuance, network setup, authentication, and backfill all take time, and any one of them can outlast the remaining token window. Start rotation early enough to absorb the latency budget your own telemetry shows, add randomized scheduling so a room of clients does not refresh in one burst, and keep the old channel until the new channel sends an application-level ready message. WebRTC defines connection and data-channel machinery, but token lifetime and replay semantics remain application policy; they need an explicit protocol above the transport.

A useful server contract has four messages. `resume` carries the last applied sequence. `event` carries a strictly increasing sequence and an immutable event identifier. `ready` states that backfill is complete and live delivery has begun. `resync` tells the client that its cursor is outside retained history, so it must fetch an authoritative room snapshot before processing more events. This contract works with a hosted realtime service, a self-managed gateway, or a peer connection because it does not confuse transport status with quiz state.

Keep presence leased. A participant becomes online after authenticated activity establishes or renews a server-side lease; they become offline when that lease expires, not merely because one browser connection closes. During handoff, two valid connections can briefly represent the same participant. Key the lease by participant and session, and make renewal idempotent, so overlap doesn't produce a duplicate name in the workspace roster.

Short overlap is intentional.

## Build the handoff as a generation-aware state machine

The following focused example runs without a vendor SDK. Its `Dialer` is the narrow transport boundary: a production adapter can wrap a WebSocket or an `RTCDataChannel`, while the state machine owns rotation, deduplication, cursor advancement, and stale-generation rejection. The fake dialer makes the failure paths deterministic enough to execute in a test runner.

```ts
type QuizEvent = {
  id: string;
  sequence: number;
  kind: "presence" | "answer" | "score";
  payload: Record<string, unknown>;
};

type ServerMessage =
  | { type: "event"; event: QuizEvent }
  | { type: "ready"; through: number }
  | { type: "resync" };

type Connection = {
  send(message: { type: "resume"; after: number }): void;
  close(): void;
  onMessage(handler: (message: ServerMessage) => void): void;
  onClose(handler: () => void): void;
};

type Dialer = (token: string) => Promise<Connection>;
type TokenIssuer = () => Promise<{ token: string; expiresAtMs: number }>;

class QuizSession {
  private activeGeneration = 0;
  private cursor = 0;
  private seen = new Set<string>();
  private connection?: Connection;
  private rotating?: Promise<void>;

  constructor(
    private readonly issueToken: TokenIssuer,
    private readonly dial: Dialer,
    private readonly apply: (event: QuizEvent) => void,
    private readonly loadSnapshot: () => Promise<number>,
  ) {}

  start(): Promise<void> {
    return this.rotate();
  }

  rotate(): Promise<void> {
    if (!this.rotating) {
      this.rotating = this.openNext().finally(() => {
        this.rotating = undefined;
      });
    }
    return this.rotating;
  }

  private async openNext(): Promise<void> {
    const generation = this.activeGeneration + 1;
    const { token } = await this.issueToken();
    const next = await this.dial(token);

    await new Promise<void>((resolve, reject) => {
      let settled = false;

      next.onClose(() => {
        if (!settled) reject(new Error("replacement closed before ready"));
      });

      next.onMessage(async (message) => {
        if (message.type === "resync") {
          this.cursor = await this.loadSnapshot();
          this.seen.clear();
          next.send({ type: "resume", after: this.cursor });
          return;
        }

        if (message.type === "event") {
          this.accept(message.event, generation);
          return;
        }

        if (!settled && message.type === "ready") {
          settled = true;
          const previous = this.connection;
          this.activeGeneration = generation;
          this.connection = next;
          previous?.close();
          resolve();
        }
      });

      next.send({ type: "resume", after: this.cursor });
    });
  }

  private accept(event: QuizEvent, generation: number): void {
    if (generation < this.activeGeneration) return;
    if (event.sequence <= this.cursor || this.seen.has(event.id)) return;

    this.apply(event);
    this.seen.add(event.id);
    this.cursor = event.sequence;
  }
}
```

One detail deserves scrutiny: the example applies backfilled events from the warming generation before `ready`. That is safe only if the server preserves sequence order for the resumed stream and the client has a single serialized message handler. If an adapter can deliver callbacks concurrently, enqueue them before calling `accept`. If the server can interleave live events ahead of older backfill, buffer by sequence or make `ready` include enough information to prove the gap is closed. I'm not sure which guarantee a generic provider will make; its protocol documentation and a forced-gap test are what resolve that uncertainty.

The `seen` set is a second guard, not the primary ordering mechanism. Bound it in production by evicting identifiers below a confirmed checkpoint, otherwise a long quiz session grows memory indefinitely. Persist the cursor only after the corresponding event mutation commits. For an in-memory score view that means updating both in one synchronous reducer; for a durable local store it means one transaction. If the cursor advances first and the process stops before applying the answer, reconnect will faithfully skip data the UI never received.

## Failure handling starts with replay, not retries

Retries answer a narrow question: can the client attempt the operation again? Replay answers the important one: after it reconnects, which accepted answers, presence renewals, and score changes are missing? Every reconnect should therefore carry the last committed cursor, including reconnects caused by ordinary network loss rather than scheduled token rotation. Use exponential backoff with jitter for repeated attempts, but let token errors return to the backend token issuer instead of retrying a rejected credential against the realtime endpoint.

Classify outcomes by what the client can safely preserve. A failed token request leaves the current connection active until its natural end, so the UI can show a reconnecting state without erasing the roster. A replacement that closes before `ready` is discarded and retried with a fresh token. A replay gap triggers snapshot recovery. An event already seen is ignored. An event ahead of the expected sequence is held while the missing range is requested. None of these cases should synthesize an `offline` participant event: presence comes from the server lease, not from a transient client transport observation.

For the compliance quiz, imagine sequence 40 records Mina's answer, rotation begins, and sequence 41 updates the score while the replacement is authenticating. The new connection resumes after 40, receives 41, then receives `ready`. If the old connection also delivers 41 during overlap, the identifier and cursor prevent a second score update. If retained history begins at 45, `resync` causes a room snapshot to replace local state and returns its checkpoint. This is why reconnect and backfill are one design decision — separating them creates a quiet failure where the screen looks connected but is stale. Test those exact boundaries with a scripted transport: pause delivery immediately before `ready`; deliver event 41 on both generations; make token issuance finish after the current channel closes; return `resync`; and reorder an event around the snapshot checkpoint. Assert on the visible roster, selected answers, score, cursor, and count of active generations after each step, including the instant when both channels exist. Then run the script again with sequence 41 omitted rather than duplicated; the client must request or recover the gap instead of advancing to sequence 42. Avoid asserting only that a socket is open. That check can pass while the quiz is missing the deciding answer.

Observability should follow the same model. Record rotation start, token issued, replacement connected, replay range, ready, old connection retired, and snapshot recovery as structured events with a room-safe correlation identifier and generation number. Never log connection tokens. Track time from rotation start to `ready`, replayed event count, duplicate count, snapshot recovery count, and reconnect attempts. Those signals expose a shrinking expiry margin and an undersized retention window without putting credentials in logs.

## Choose overlap or pause based on the quiz's consistency cost

Overlap gives the smoothest handoff and keeps answers flowing, but it costs a brief second connection per active participant and forces the server to tolerate duplicate session presence. It is not suitable when the transport or account enforces a strict one-connection-per-session rule. In that case, pause answer submission, close the current channel, open the replacement, replay from the committed cursor, and enable input only after `ready`. The pause is visible, yet its state model is easier to prove.

A single long-lived token is simpler still, but stick with it only when its exposure window matches the application's security policy and revocation needs. Rotating aggressively without replay support is the wrong trade: it increases connection churn while leaving continuity undefined. Likewise, peer-to-peer data channels can reduce reliance on a central event path for some game traffic, but a fintech workspace may still need an authoritative service for authentication, ordered scoring, presence leases, and recoverable history. The W3C WebRTC Recommendation specifies the peer connection and data-channel surface; it does not choose that application authority model.

Cost belongs in this decision, but don't reduce it to a connection unit price. Model concurrent connections during overlap, token issuance requests, retained event bytes, replay reads, snapshot reads, and telemetry volume. Then set two operational budgets: the maximum acceptable handoff time and the maximum recoverable event gap. Those numbers determine rotation lead time and retention far more directly than a feature checklist does. Your mileage may vary because room size, round length, and mobile network behavior change both.

Before deployment, walk one complete lifecycle in staging: join, answer, rotate, overlap, replay, retire, lose the network, reconnect, and recover from an expired cursor. Confirm that authorization is re-evaluated when the replacement connects, that a removed participant cannot resume, that logs contain no tokens, and that the roster converges after every transition. During rollout, start with a small share of rooms and compare the `ready` latency and recovery counters with the budgets. Roll back the client policy if the margin collapses; do not weaken replay correctness to hide the delay.

That's the ship criterion.

## References

- W3C, WebRTC Recommendation: https://www.w3.org/TR/webrtc/
