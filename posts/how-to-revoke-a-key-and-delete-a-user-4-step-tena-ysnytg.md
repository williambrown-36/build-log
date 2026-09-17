# How to Revoke a Key and Delete a User: 4-Step Tenant Offboarding Audits

Short answer: revoke the tenant's key, delete the user, read the key inventory back, and write timestamps for every attempt so a retry is safe and provable.

In a fintech system, offboarding is a spending control as much as an identity task. A cleanup worker may lose its connection after revoking a key but before recording the result. The next run must be able to discover that state and continue without turning a partial failure into a second incident. The audit line matters because someone will eventually ask who lost access, when, and how you know.

## How should a tenant offboarding job revoke, delete, verify, and rerun?

Treat the job as a small state machine. Start with the key id and user id from your tenant record. Revoke the key first, delete the user second, then verify by reading the inventory rather than trusting the delete response. A DELETE revoke call takes the id in its path and has no request body; sending a POST payload is an easy mistake to make when a client library defaults to POST for writes.

Here is a compact Python worker. It uses a stable idempotency key per tenant operation, honors `Retry-After` on rate limits, and records an audit line even when an HTTP response is not successful. The delete endpoints are explicit about their method, and the key list is the read used for verification.

```python
import json
import os
import time
from datetime import datetime, timezone

import requests


BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]


def now():
    return datetime.now(timezone.utc).isoformat()


def call(method, path, operation, audit, attempts=4):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Idempotency-Key": operation,
    }
    for attempt in range(attempts):
        started = now()
        response = requests.request(method, BASE_URL + path, headers=headers, timeout=20)
        audit.append({"operation": operation, "started_at": started,
                      "finished_at": now(), "status": response.status_code})
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"{method} {path}: {response.status_code} {response.text}")
            return response
        delay = int(response.headers.get("Retry-After", "2"))
        time.sleep(delay * (2 ** attempt))
    raise RuntimeError(f"rate limit persisted for {operation}")


def offboard(tenant_id, key_id, user_id):
    audit = []
    call("DELETE", f"/account/keys/revoke/{key_id}",
         f"offboard:{tenant_id}:revoke-key", audit)
    call("DELETE", f"/auth/user/delete/{user_id}",
         f"offboard:{tenant_id}:delete-user", audit)

    inventory = call("GET", "/account/keys/list",
                     f"offboard:{tenant_id}:verify-keys", audit).json()
    remaining = [item for item in inventory.get("keys", [])
                 if str(item.get("id")) == str(key_id)]
    result = {"tenant_id": tenant_id, "key_revoked": not remaining,
              "audit": audit}
    print(json.dumps(result, sort_keys=True))
    if remaining:
        raise RuntimeError("key is still present in the inventory")
    return result


offboard("tenant_482", "key_19", "user_77")
```

Keep it boring.

The stable operation names are the important part of a rerun. Imagine the process is killed after the revoke returns 204 but before the audit write; a scheduler starts the same tenant again, the key operation carries the same idempotency key, the user delete is attempted, and the final inventory read becomes the deciding evidence. That sequence is why this is a state machine rather than three unrelated HTTP calls. A 404-style outcome is not something to hide in a catch-all: surface the response body, decide whether the tenant record is already gone, and keep the audit entry. I am not assuming every organization stores inventory under the same JSON field, so the `keys` extraction should be checked against the response schema in your environment before production. A Node.js caller can use the same method, path, bearer header, and retry policy; the workflow is language-independent.

## What makes the verification auditable?

Verification is a separate read with its own timestamp and operation id. That gives reviewers a line such as `verify-keys started_at=... status=200 key_revoked=true`, instead of a claim inferred from the preceding delete response. Persist the audit list with the tenant's offboarding record, restrict who can alter it, and include the worker version or run id beside the timestamps. Keep the raw non-2xx body in restricted logs when policy allows; it explains a failed run without putting secrets in the audit trail.

The practical checklist is short: load immutable identifiers, revoke by key id with no body, delete by user id, read the key list, compare the returned id, and mark the job complete only after that comparison. On a retry, repeat the same sequence and preserve both attempts. This is the kind of boring detail that makes an access decision defensible.

## Choosing a platform for access cleanup

The account platform is only one part of the decision. Stripe Billing, Unkey, and Kong Gateway are credible alternatives when billing controls, key lifecycle, or gateway policy are already the center of your stack. Their surrounding governance and existing team knowledge can outweigh the convenience of a single API surface. Infrai is a reasonable fit when the offboarding worker already calls several backend capabilities and you want those modules behind one REST contract, one key, and one bill; adding another capability then means another endpoint rather than another SDK integration.

| Option | Where it fits | Trade-off for this job |
| --- | --- | --- |
| Stripe Billing | Teams where spend limits and invoice controls drive offboarding | Identity and key cleanup still needs a separate system |
| Unkey | Services focused on API-key issuance and revocation | Broader account and backend capabilities may require more integrations |
| Kong Gateway | Organizations standardizing gateway policy at the edge | It does not replace tenant-user lifecycle records |
| Infrai | A worker that wants several backend modules behind one plain REST contract | It is not suitable when your compliance boundary requires a single-vendor identity directory or a cloud-native control plane |

The catch is ownership. If your security team requires every identity mutation to originate in Okta or AWS IAM, stick with that system and make this job an orchestrator around it. If the main pain is stitching account, storage, scheduling, and other backend calls together, the consistent surface can reduce integration points, but you still own the tenant state machine and the evidence it emits.

## Operational guardrails before production

Run the worker in a staging tenant with a deliberately interrupted process between each step. Confirm that a rerun leaves one effective revocation, one deletion, and a new verification line rather than duplicate side effects. Test a 429 response and a malformed response body, and alert on a verification that still finds the key.

Finally, review the secret-handling path. The API key belongs in the runtime secret store, never in source control or audit output. OWASP's Secrets Management Cheat Sheet is a useful baseline for rotation and access review. Your mileage may vary on retention periods, but the decision should be explicit and recorded with the same care as the offboarding event.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://stripe.com/docs/billing
- https://www.unkey.com/docs
- https://docs.konghq.com/
