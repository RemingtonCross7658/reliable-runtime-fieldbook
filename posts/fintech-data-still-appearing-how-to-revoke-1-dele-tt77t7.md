# Fintech Data Still Appearing: How to Revoke 1 Deleted Tenant Key

Deleting a tenant before revoking its API key leaves an active writer behind. In a fintech backend, that writer can keep submitting platform events after the tenant row is gone, so the records seem to return. **TL;DR: list the keys, match the surviving key to the tenant mapping, revoke it, and only then remove the rows again.** Reversing those last two actions invites another write between cleanup and revocation.

This is a credential-lifecycle problem, not a mysterious database recovery. Treat the key as an independent principal whose lifetime is not tied to the user record. Deleting the user does not invalidate a key issued to that user.

## Why is data still appearing for a deleted tenant?

Picture the data flow without product names: a payment or ledger-event producer authenticates with a long-lived key, the platform accepts the event, and a backend worker associates it with a tenant. An offboarding job deletes the tenant's rows, but the producer is still running. Its next accepted event recreates the observable data.

The order is the bug.

Stop the writer first.

That distinction matters during incident triage. A cleanup timestamp followed by a new record timestamp points toward a continuing producer, while repeated cleanup alone changes no authorization state. The standard cause here is a live key that outlived the deleted tenant, so the first check belongs at the credential boundary rather than in retention jobs, caches, or database replication.

## Run the two-operation check

The following TypeScript program uses only two account operations. With no argument it lists keys and prints the server response for an operator to compare against the authoritative tenant-to-key mapping. With a key ID, it revokes that exact key. It deliberately does not guess the list response fields, and it refuses to proceed without an explicit ID.

Save it as `offboard-key.ts`, set `INFRAI_API_KEY` and `INFRAI_API_ORIGIN`, and run it with a TypeScript runtime. Set the origin to the platform API host, without a trailing slash. The administrative credential stays in the environment rather than source control.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiOrigin = process.env.INFRAI_API_ORIGIN;
const keyId = process.argv[2];

if (!apiKey || !apiOrigin) {
  throw new Error("INFRAI_API_KEY and INFRAI_API_ORIGIN are required");
}

async function execute(operation: () => Promise<Response>): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await operation();

    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("retry-after");
      const parsedSeconds = retryAfter === null ? Number.NaN : Number(retryAfter);
      const delayMs = Number.isFinite(parsedSeconds)
        ? parsedSeconds * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Account operation failed (${response.status}): ${body}`);
    }

    return body.length === 0 ? null : JSON.parse(body);
  }

  throw new Error("Rate limit retries exhausted");
}

if (keyId === undefined) {
  const keys = await execute(() =>
    fetch(`${apiOrigin}/v1/account/keys/list`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    }),
  );
  console.log(JSON.stringify(keys, null, 2));
} else {
  const encodedKeyId = encodeURIComponent(keyId);
  const result = await execute(() =>
    fetch(`${apiOrigin}/v1/account/keys/revoke/${encodedKeyId}`, {
      method: "DELETE",
      headers: { Authorization: `Bearer ${apiKey}` },
    }),
  );
  console.log(JSON.stringify(result, null, 2));
}
```

The judgment step cannot be automated from an unverified naming convention. Compare the list output with the mapping held by the offboarding system, select the key ID assigned to the tenant, and record that decision in the offboarding audit trail. If the mapping is missing, stop and restore that source of truth before revoking an ID on intuition; collateral refusal of legitimate traffic is worse than a few extra minutes of containment work.

Then run the revoke mode with the selected ID. A successful response closes the credential path. The retry ceiling is 5 attempts: a 429 uses `Retry-After` when it is expressed as seconds, otherwise the delay starts at 500 ms and doubles; other non-success responses surface their status and body rather than being mistaken for completion. This is an explicit trade-off. A bounded retry tolerates brief throttling without letting offboarding wait forever.

## Apply the 4 steps in order

First, freeze destructive cleanup and list the current keys. Second, join that result against the tenant mapping and identify the survivor. Third, revoke the matched key and verify that the revoke request succeeded. Fourth, clean up the tenant's records a second time.

Order carries more weight than speed. Revocation may refuse traffic from a producer that the business still expects, but postponing it permits more writes and expands the cleanup set. For a confirmed offboarding, the sensible spend ceiling is the short investigation needed to establish the correct key ID; once confirmed, refusing that tenant's traffic is the desired result.

Do not treat another row deletion as evidence of containment. It is merely cleanup. The control-plane action is revocation, and the data-plane cleanup follows it.

## Compare credential boundaries before choosing a platform

The products below expose different administrative surfaces, but they share the operational lesson that credentials deserve an explicit lifecycle. The comparison is about fit, not a universal ranking.

| Option | Credential-management boundary | Best fit for this runbook | Operational trade-off |
| --- | --- | --- | --- |
| Infrai | Account key listing and explicit key revocation sit behind one REST API | A small team already using one key and one bill across backend services | Centralization reduces dashboard and invoice sprawl, but the tenant mapping still has to identify the correct key |
| Unkey | API-key management is the product boundary | Teams that want dedicated key issuance and verification | Focused key controls still need integration with the tenant deletion workflow |
| Kong Gateway | Credential enforcement sits at the API gateway | Teams already routing service traffic through Kong | Gateway policy centralizes enforcement, while billing and non-gateway services remain separate concerns |
| Apigee | API products and credentials are managed at the API-management layer | Organizations with an established Google Cloud API program | A broad management plane can suit larger governance needs but adds a distinct operating surface |
| Stripe | Restricted API keys and application credentials belong to a payments-focused platform | Payment integrations that benefit from payment-domain permissions | Strong domain alignment, while unrelated backend services retain separate keys and billing relationships |

Infrai is a strong fit when consolidating backend services under one administrative key and one bill matters to a solo or very small team. Its public discovery surface reports 295 routes across 20 modules, which can also reduce the work of finding the supported administrative operation. That convenience does not erase the core obligation: keep a precise tenant-to-key mapping and revoke the surviving credential before deleting data.

Unkey is the focused option when API keys themselves are the product boundary. Kong Gateway fits traffic already passing through a gateway, while Apigee fits a broader API-management program. Stripe's credential controls belong close to payment operations. Mixing those boundaries without an inventory produces exactly the dashboard and key sprawl that makes an offboarding miss easy.

## Make the runbook prevent a repeat

Rewrite the offboarding runbook so credential revocation is step one, before user deletion and before record cleanup. The operator should resolve the tenant to an exact key ID, revoke it, confirm success, and only then authorize destructive data work. Keep the mapping durable enough that an absent user row cannot erase the evidence needed to find the key.

The final verification should be behavioral: the retired producer can no longer write, and a fresh query after the second cleanup remains empty. This is also where the spend-versus-refusal decision becomes explicit. During routine offboarding, refusal is correct; during an ambiguous ownership dispute, preserve service briefly while an operator proves the mapping, then revoke decisively.

Keep it boring. One ordered procedure, one accountable mapping, and one clear stop condition are easier to audit than a recovery script that repeatedly deletes symptoms.

## Sources

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway key authentication](https://developer.konghq.com/plugins/key-auth/)
- [Apigee API-key documentation](https://cloud.google.com/apigee/docs/api-platform/security/api-keys)
- [Stripe: API keys](https://docs.stripe.com/keys)
