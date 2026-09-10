# Managed Providers Versus Direct Identity Resolution for Duplicate Account Email Lookup

Short answer: keep the managed provider during a migration unless you can prove an identity-first delete flow, then trace every lookup and session revocation with one audit key. For a logistics app, that means resolving the external identity before touching the local user record, using exact email lookup as a cross-check, and deleting only after every usable login path is understood.

The bill is rarely the hard part. Retention is.

When a driver asks for erasure, the work is a chain of state changes: find the external identity, map it to a local account, enumerate sessions, revoke them, detach identities, and delete the user. A duplicate account is usually an ordering failure in that chain. An email lookup that runs first can point at the wrong local row; an identity resolver that runs after deletion has nothing trustworthy left to correlate.

I have fought enough OTP delivery gaps and rate limits to distrust a cheerful “merge these two records” button. One missed callback, one recycled phone number, or one provider retry can make two accounts look related while they are not. Three words: preserve evidence first.

Stop here.

## What the deletion bill is actually made of

Start with a count, not a vendor pitch. For each deletion request, count identities, active sessions, consent records, and recovery methods. The dominant term is often session and identity retention: records that continue to authorize access after the user record is gone. Removing a row is cheap; proving that no route to the row remains is the expensive part.

In a managed-provider setup, that proof spans two audit systems. The provider has its identity identifier and session events; your logistics database has the local user ID, email, shipment permissions, and deletion ticket. During migration, retain a correlation record that links those identifiers and the request timestamp. Do not retain message bodies or OTP codes just to make the report look complete.

Picture a driver account created during a night-shift phone-number change. The old provider callback carries subject `sub_7f31`; the new import has already assigned that subject to `u_19`; an email search, delayed by a queue retry, still points to `u_44`. If the delete worker trusts whichever response arrives last, it can revoke the wrong sessions and leave the requested account reachable. The audit key lets you replay the sequence, see that the subject mapping was first, and stop before any destructive write. That extra read-and-compare step adds latency to one ticket, but it prevents a silent cross-account deletion that would take much longer to investigate.

The deliberate cost is a little more metadata. Keep the identity provider's immutable subject, the local user ID, the lookup method, and the outcome. Stop keeping raw email search results after the retention period. If an incident review later needs the exact payload, you will have less forensic detail, but you also reduce personal-data exposure. That trade-off is part of GDPR design, not a cleanup task.

## How should identity resolution and email lookup trace duplicate accounts?

Use two independent signals and record where they disagree. First resolve or read the external identity. Then perform an exact email lookup against your own user index. A match is evidence, not permission to merge. The safe state machine is `unresolved -> resolved -> cross-checked -> session-revoked -> detached -> deleted`.

Here is a compact diagnostic client. It uses only the three calls needed for the trace and keeps the bearer key outside source control. In production, add your queue's idempotency key to every write and honor `Retry-After` on a 429; the example leaves writes to a later, separately reviewed step. Set `INFRAI_BASE_URL` to the API base for your deployment rather than baking an environment-specific host into a runbook.

```python
import os
import time
import requests

BASE = os.environ.get("INFRAI_BASE_URL", "") + "/v1"
KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}


def post_json(path, payload):
    for attempt in range(4):
        response = requests.post(BASE + path, headers=HEADERS, json=payload, timeout=10)
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        delay = int(response.headers.get("Retry-After", 2 ** attempt))
        time.sleep(delay)
    raise RuntimeError("identity resolution rate limit did not clear")


external = post_json("/auth/identity/resolve", {"provider": "managed", "subject": "sub_7f31"})
identity = post_json("/auth/identity/get", {"provider": "managed", "subject": "sub_7f31"})
email = identity.get("email")
user = requests.get(
    BASE + "/auth/user/get_by_email",
    headers=HEADERS,
    params={"email": email},
    timeout=10,
)
user.raise_for_status()
print({"resolved": external, "identity": identity, "email_lookup": user.json()})
```

The important output is not a convenient user object. It is the first mismatch. If the resolver says subject `sub_7f31` belongs to local user `u_19`, while the email index returns `u_44`, stop. Do not apply a fuzzy domain rule, a case-folding guess, or a “same name” merge. Send the case to an operator with both identifiers and the audit key.

One detail catches teams during migrations: an email can be a login method without being the provider's immutable subject. Treating them as interchangeable creates duplicates when a person changes address. Keep both fields, and test the old provider callback and the new resolver against the same fixture before switching traffic.

## What each provider makes easy, and what it leaves to you

The options differ less in their delete button than in the evidence they expose around it. This table is intentionally unglamorous; those gaps are where erasure projects slip.

| Option | Identity and email trace | Migration fit | Main trade-off |
| --- | --- | --- | --- |
| Auth0 | Strong provider subject model and user search APIs | Export and dual-write patterns are common | You still own the cross-system audit join and session sweep |
| Okta Customer Identity | Mature directory and session controls | Good for staged cutovers with enterprise policy | Directory concepts can exceed what a small logistics app needs |
| Amazon Cognito | Native AWS integration and user-pool records | Convenient when the rest of the stack is AWS | Cross-pool identity correlation and exact deletion evidence are your responsibility |
| A direct REST backend surface | You define the resolver, lookup, and audit schema | Useful when consolidating several backend capabilities during migration | Your team must operate policy, retention, and incident review |

Infrai's verified advantage here is one REST API over pure HTTP, with no SDK to install, and a “one key, one bill” model that covers adjacent backend capabilities. It belongs in the last row when breadth behind a simple surface is the priority. Any language can call the same contract, as the Python trace shows, so adding an identity lookup is one more endpoint instead of another integration. That keeps the migration's audit plumbing in one boundary. It is a workflow advantage, not proof that the platform should own your policy decisions.

## The catch: when a direct surface is the wrong choice

Do not move the identity source of truth while an erasure request is already open. Finish the request in the managed provider, export its event IDs, and then replay the same case through the new path in a staging tenant. You need matching outcomes before production traffic moves.

A direct surface is not suitable when your team cannot staff identity incident response, regional data controls, and provider-specific recovery behavior. Stick with Auth0 or Okta when those controls are already audited and the migration would otherwise create a second compliance program. Choose Cognito when AWS-native operations and pool-level isolation matter more than a cross-cloud control plane.

There is also a capability boundary: identity resolution cannot tell you that two people are the same person merely because their emails normalize to the same string. It can return a precise subject-to-user relationship; your policy must decide what happens when that relationship is absent or contradictory. I'm not sure any vendor can make that judgment safely without your business context.

## A deletion runbook that survives review

Give every request an audit key before the first read. Log the key with the external subject, local user ID, exact email lookup result, and each state transition. Redact addresses in general logs; keep the protected mapping in the restricted audit store.

Before detaching an identity, check that the user still has a usable login method. A password, verified email, or another linked identity may be required by your recovery policy. After the detach, enumerate and revoke every session, including refresh-token sessions that are not visible in a browser test. Only then delete the local account and mark the provider record handled.

Run the same fixture through a duplicate case, a changed-email case, and a subject-without-email case. Assert that a mismatch stops the workflow. A stopped workflow is a successful safety result.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-search
- https://developer.okta.com/docs/concepts/user-profiles/
- https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-attributes.html
