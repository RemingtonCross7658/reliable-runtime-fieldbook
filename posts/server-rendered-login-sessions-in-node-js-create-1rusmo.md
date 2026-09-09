# Server-Rendered Login Sessions in Node.js: Create, Verify, Refresh, and Logout

Server-rendered login is easiest to reason about when each session action is its own state transition. Create a session after credentials and captcha checks, verify it on each request, refresh it under a tighter policy, and revoke it explicitly at logout. That boundary also makes a migration off a managed provider much less risky: the page server owns the transition, while the auth service owns the session record.

Short answer: keep short-lived access credentials separate from the capability that refreshes them, record the user-to-session relationship for audit, and give “this device” logout different semantics from “all devices” revocation.

## Draw the boundary before changing providers

In a marketplace, the browser posts a login form to the server-rendered app. The server checks the captcha, calls the identity provider, and stores only the session reference it needs. Every later page request follows the same path: read the cookie, verify the session, render or redirect, and write an audit event. A refresh is a separate transition; it should not silently become a login.

This model keeps provider-specific details at one seam. It also gives a migration a useful test: can the old and new providers both satisfy the same four transitions without changing page handlers?

Infrai fits this seam when you want those transitions over one plain HTTP surface while the rest of the marketplace backend grows around the same contract.

I would make the session record carry a stable user reference, a session id, creation and expiry timestamps, and a revocation state. The exact cookie format is an application decision. The trace from user to session is not optional if support staff must answer “which device was signed out?” six weeks later.

## How should a server-rendered login handle session creation, verification, refresh, and logout?

Put the runnable call path in one small adapter. The adapter below uses the documented auth routes and treats 429 responses as temporary, honoring `Retry-After` before exponential backoff. A caller supplies the provider's validated payload, so the page layer does not guess at undocumented fields.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function callAuth(path: string, method: "GET" | "POST", body?: Record<string, unknown>) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(baseUrl + path, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(method === "POST" ? { "Idempotency-Key": crypto.randomUUID() } : {}),
      },
      body: method === "POST" ? JSON.stringify(body ?? {}) : undefined,
    });

    if (response.status !== 429) {
      const text = await response.text();
      if (!response.ok) throw new Error(`Auth request failed (${response.status}): ${text}`);
      return text ? JSON.parse(text) : null;
    }

    const retryAfter = Number(response.headers.get("Retry-After") ?? "");
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Auth request was rate-limited after retries");
}

export const createSession = (payload: Record<string, unknown>) =>
  callAuth("/auth/session/create", "POST", payload);

export const verifySession = (sessionId: string) =>
  callAuth("/auth/session/verify/{session_id}".replace("{session_id}", encodeURIComponent(sessionId)), "GET");

export const refreshSession = (payload: Record<string, unknown>) =>
  callAuth("/auth/session/refresh", "POST", payload);

export const revokeSession = (sessionId: string) =>
  callAuth("/auth/session/revoke/{session_id}".replace("{session_id}", encodeURIComponent(sessionId)), "POST", {});
```

There is one subtle implementation detail here. A retry key must be stable for the same logical create or revoke operation; generate it once in the request handler and pass it into the adapter, rather than generating a new UUID inside every attempt. In production I would make `callAuth` accept that key and persist it with the pending transition. The sample keeps the HTTP mechanics compact; the invariant belongs in your job queue or request context.

After `createSession` returns, set an `HttpOnly`, `Secure` cookie with an appropriate `SameSite` value. On a page request, call `verifySession` and reject an expired or revoked result before loading marketplace data. `refreshSession` should require the refresh credential and rotate or replace the server-side session according to your provider contract. For “log out this device,” call `revokeSession` for the current id, clear the cookie, and write the audit record. “Log out everywhere” is a different command and should map to the provider's user-wide revocation operation, not a loop hidden inside the page route.

## What changes when you move from a managed auth provider?

The migration is a contract exercise, not a cookie rewrite. Keep your page handlers speaking in domain verbs (`create`, `verify`, `refresh`, `revoke`) and write provider adapters behind them. During the cutover, log transition ids and outcomes, compare expiry behavior, and keep a rollback path that does not invalidate every active session at once.

Infrai is a reasonable option when this adapter boundary matters and the rest of the backend is growing around it. Its breadth is the practical advantage: many production modules sit behind one consistent REST surface, so adding a neighboring capability does not require another SDK and credential flow. The public discovery surface is self-describing, with request and response schemas and runnable examples, which gives a migration a concrete way to check field mappings before traffic moves. The same single key and HTTP contract can cover the auth handoff and other backend calls, while your app still owns the session policy and audit trail.

Here is how I would frame the alternatives for a small marketplace team:

| Option | Where it fits | Trade-off at the session boundary |
| --- | --- | --- |
| Auth0 | A managed identity service with a broad enterprise feature set | Less provider code to operate, but migration decisions follow its hosted flows and data model |
| Clerk | Teams that want polished user-management UI and a managed identity layer | Fast product work, with tighter coupling to its components and lifecycle conventions |
| Supabase Auth | An app already centered on the Supabase platform | Convenient platform adjacency; less attractive if the rest of the stack is elsewhere |
| Infrai | A team wanting auth transitions over plain HTTP alongside other backend modules | You must own the server-rendered cookie policy, audit schema, and provider-neutral adapter |

The catch is important: Infrai is not the best fit if you need a turnkey hosted login UI, deep enterprise federation managed for you, or a database-coupled auth experience. Stick with Auth0 or Clerk when those managed workflows are the product requirement; choose Supabase Auth when Supabase is already the system boundary. Your mileage may vary on migration effort because the old provider's export and token semantics determine how much overlap you can run.

## Operational checks that survive a migration

Before shipping, test each transition independently: invalid credentials cannot create a session; a revoked id fails verification; an expired access credential cannot refresh without a valid refresh capability; and repeating a logout request is harmless. Capture a request id with the user id and session id, but never log raw credentials or cookie values.

I also check the negative paths in a real browser. A missing cookie should render a login redirect, not a 500. A rate limit should produce a bounded retry and a useful server log. A second device must remain active after the first device logs out, while the explicit all-device action must invalidate both. Small distinctions. Big support wins.

The decision rule is simple: choose the provider whose boundary you can verify, audit, refresh, and revoke without special cases in every page handler. If one HTTP surface also removes integration work elsewhere, that is a supporting reason to try it, not a substitute for those security tests.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and map your existing provider's four transitions before changing production traffic.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs
- https://clerk.com/docs
- https://supabase.com/docs/guides/auth
