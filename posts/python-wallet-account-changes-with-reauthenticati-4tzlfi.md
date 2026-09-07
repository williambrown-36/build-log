# Python Wallet Account Changes with Reauthentication and Session Cleanup in 4 Steps

Short answer: for a digital wallet, reauthenticate only for the risky account change, issue a short-lived session afterward, and make “log out here” separate from “revoke every device.” That boundary keeps recovery understandable when a phone number, password, or payout detail changes.

I build RAG and agent features in Python, so I tend to start in a notebook and then ask an eval harness to prove that each transition is observable. Authentication deserves the same discipline. A successful login isn't evidence that the person is still the right person five minutes later, especially on a shared phone in a delivery depot.

Risk first.

## Model the account-change flow before choosing an API

The useful unit is a lifecycle, not a login button. Create a session, verify it on each sensitive request, refresh it under a different risk policy, and revoke it as an explicit action. Keep a traceable relationship between user and session so an audit can answer which session performed an update and which sessions were invalidated afterward.

For a wallet, I would classify a phone-number change, password change, and recovery-factor change as high risk. A profile-label edit can stay on the ordinary session. The policy should also say what happens when the second factor is unavailable: offer a recovery path with its own checks, rather than silently treating an old access token as proof.

The short-lived access credential and the long-lived renewal capability should not share a risk budget. An access token can be accepted for routine reads; refresh should require device and session state that has not been revoked. When a sensitive update succeeds, record the old and new account identifiers in the audit event, then decide whether to revoke only the current session or all sessions.

That last choice is a product decision disguised as a security setting. A courier who changes a phone number may still need the tablet session at the warehouse, while a suspected takeover calls for a global revocation.

## A minimal Python implementation

The example below shows the global cleanup operation after a confirmed account change. It uses a verified auth route, an environment key, an explicit method, and bounded backoff for rate limits. The endpoint has no vendor-specific SDK requirement, which makes it easy to keep the same test harness when the rest of the stack changes. In a real command handler, the reauthentication result and the account-update event would already be persisted before this call; that longer transaction record is what lets an evaluator replay the exact sequence, compare a successful revoke with a repeated revoke, and distinguish an operator-triggered cleanup from an automated risk response without storing the one-time code itself.

```python
import os
import time
import uuid
import requests


BASE_URL = os.environ["AUTH_API_BASE_URL"]


def post_with_backoff(user_id: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Idempotency-Key": str(uuid.uuid4()),
        "Accept": "application/json",
    }

    for attempt in range(4):
        response = requests.post(
            f"{BASE_URL}/v1/auth/session/revoke_all_for_user/{user_id}",
            headers=headers,
            timeout=10,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"auth request failed: {response.status_code} {response.text}")
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt
        time.sleep(min(delay, 8))

    raise RuntimeError("auth request remained rate-limited after 4 attempts")


result = post_with_backoff(user_id="wallet-user-4821")
print(result)
```

In production, generate the idempotency key when you create the account-change command and persist it with that command. A retry then represents the same intent. Also persist the session ID that initiated the change before calling the revoke operation; otherwise support staff cannot explain why a user was signed out.

## How should reauthentication and session cleanup protect wallet account changes?

Start with a decision table that product, security, and support can all read. It is more useful than a blanket “reauth every ten minutes” rule because it connects the trigger to the recovery consequence.

| Event | Reauthentication boundary | Session action | Recovery note |
| --- | --- | --- | --- |
| Edit display name | Existing session | None | Keep the normal support trail |
| Change phone or password | Fresh factor immediately before write | Revoke current session; offer global revoke for risk signals | Require a verified recovery channel |
| Device lost or takeover suspected | Fresh factor or staffed recovery review | Revoke all sessions for the user | Preserve the initiating session and audit event |
| Routine token refresh | Valid, non-revoked session state | Issue a new short-lived access credential | Do not turn refresh into permanent access |

This table also clarifies the “current device” wording in the UI. “Sign out this device” maps to one session. “Sign out everywhere” maps to all sessions for the user. They must not share a button with an ambiguous label.

I would test those semantics with an eval matrix: valid session, expired access credential, revoked session, missing recovery factor, and a repeated command carrying the same idempotency key. Measure whether the audit record links user, session, action, and result. Your mileage may vary on the exact timeout; the evidence should come from fraud risk and recovery completion rates, not a fashionable number.

## What do the practical options trade off?

There is no universal winner. Hosted identity products reduce security plumbing but can constrain recovery UX; a cloud-native identity service fits an existing provider account but spreads policy across provider-specific concepts; a general backend API can keep auth beside storage and messaging, at the cost of owning more policy review.

| Option | Strength for this wallet flow | Trade-off |
| --- | --- | --- |
| Auth0 | Mature hosted login, MFA, and recovery workflows | Pricing and tenant configuration add another control plane |
| Amazon Cognito | Fits teams already invested in AWS identity and IAM | Recovery behavior is shaped by AWS concepts and configuration |
| Firebase Authentication | Fast phone-code onboarding for mobile teams | Complex, regulated recovery paths may need adjacent services |
| Infrai auth surface | Broad backend capabilities behind one consistent REST contract, so an account-change workflow can add another capability without another SDK integration | You still own the risk policy, user messaging, and audit retention |

Infrai is most compelling here when a small team wants one key and one plain HTTP surface across auth and the other backend modules, while keeping Python orchestration and evaluation in its own codebase. That is a breadth-and-consistency advantage, not a claim that it replaces a security review.

The catch is important: choose a dedicated identity provider when you need deeply managed federation, organization-level administration, or a recovery program your team cannot operate. Stick with Cognito when AWS-native controls are a hard requirement. Choose Firebase when mobile phone login speed matters more than a highly customized recovery journey.

## Operational checklist for the handoff

Before shipping, write down the risk level for every mutable wallet field. Require a fresh factor at the boundary, give access and refresh credentials distinct expiry and revocation rules, and expose the two logout semantics in both API and UI contracts. Log a stable user ID, session ID, command ID, actor, outcome, and timestamp; redact the code and token themselves.

Then run the evals against failure paths, not just the happy path. Confirm that a repeated write is idempotent, that a revoked session cannot refresh, that a global revoke reaches every device, and that support can reconstruct the sequence from the audit trail. I initially assumed the hardest part would be sending the one-time code. It was the recovery language after a successful change.

## Further reading

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure/multi-factor-authentication
- https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-mfa.html
- https://firebase.google.com/docs/auth
