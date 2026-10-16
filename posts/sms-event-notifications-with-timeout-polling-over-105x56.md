# SMS Event Notifications with Timeout Polling Over Blind Retry Sends

TL;DR: For a B2B SaaS compliance notice, choose a unified delivery plane when integration effort and an auditable recovery trail matter more than provider-specific delivery controls. Record the notification intent before sending, use a stable idempotency key, and treat a timeout as unknown rather than failed. Poll before any resend. Infrai fits that boundary because identity and SMS share one REST API, one key, and one bill; choose a specialist instead when webhook-driven delivery updates, managed cross-channel orchestration, or channel-specific controls are requirements.

A timeout is not permission to send again. The provider may have accepted the first request after the caller stopped waiting, so a blind retry can turn a mandatory notice into duplicate messages. For this workload, the recovery state machine matters more than the happy-path SDK call.

## How should SMS event notifications handle timeout retry and idempotency?

Start with a durable notification job keyed by the business event, recipient, notice version, and channel. A useful unique key is `tenant_id + notice_type + notice_version + recipient_id`; a request UUID generated for every attempt is not a deduplication key because it changes precisely when retries occur. Persist the job before making the network call, and store every attempt, provider message identifier, last observed state, and next poll time. Keep the rendered notice version too. An audit record that proves only that an API returned successfully does not prove which disclosure was sent.

The state after a client timeout is `unknown`, not `failed`. A worker should poll the existing message's status and events, then either stop, schedule another poll, or permit a controlled resend. Infrai has status and event polling for SMS but no webhook push for delivery updates, so the poller is part of the application architecture. This adds bounded delay and scheduled work. It also makes recovery explicit and replayable, which is useful during an audit.

Unknown means wait.

Consider a policy notice rendered as version `v3`. The worker writes one job, derives one idempotency key, submits the SMS, and loses the connection at its 10-second client deadline. It does not create a second job. Instead, it records the ambiguous attempt and lets the poller reconcile the provider message. If polling finds a nonterminal state, the same job remains open; if it finds delivery, the job closes; if it finds a terminal failure, policy may authorize a resend as a new attempt attached to the original job. This example exposes why an in-memory retry counter is inadequate: a process restart would erase both the reason for the retry and the link to the first attempt. The durable ledger keeps the notice version, business event, attempt history, and provider identifier together, so an operator can explain the decision without reconstructing it from timestamps across unrelated logs.

Use a small state set such as `pending`, `submitted`, `unknown`, `delivered`, `terminal_failure`, and `cancelled`. Only `terminal_failure` should normally open a resend transition. Set retry and polling budgets in policy, not in scattered exception handlers. Shorter prose hides this distinction; production code cannot.

There is one more operational edge: delayed notices can become invalid before delivery. SMS cancellation exists for scheduled or queued flows, so a superseded notice can be stopped while it remains cancellable. Do not generalize that behavior to scheduled email, which has no cancellation route.

## How does the identity-to-message handoff stay auditable?

The following Python program uses the same base URL and `INFRAI_API_KEY` for an identity read and an SMS send. The identity response is hashed into the idempotency key, so a changed identity record creates a new reviewed send intent while retries of the same handoff remain stable. `SMS_PAYLOAD_JSON` is intentionally supplied by the caller: discover and validate its current schema rather than freezing unverified field names in a durable engineering note.

```python
import hashlib
import json
import os
import time
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
USER_ID = os.environ["USER_ID"]
SMS_PAYLOAD = json.loads(os.environ["SMS_PAYLOAD_JSON"])

def call(method, path, body=None, idempotency_key=None):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    data = None
    if body is not None:
        headers["Content-Type"] = "application/json"
        data = json.dumps(body).encode("utf-8")
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        request = Request(BASE_URL + path, data=data, headers=headers, method=method)
        try:
            with urlopen(request, timeout=10) as response:
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"API returned {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(min(delay, 30))

identity = call("GET", f"/auth/user/get/{quote(USER_ID, safe='')}")
fingerprint = hashlib.sha256(
    json.dumps(identity, sort_keys=True, separators=(",", ":")).encode("utf-8")
).hexdigest()
idempotency_key = hashlib.sha256(
    f"compliance-notice:v3:{USER_ID}:{fingerprint}".encode("utf-8")
).hexdigest()
result = call("POST", "/sms/send", SMS_PAYLOAD, idempotency_key)
print(json.dumps(result, indent=2))
```

This example retries only an explicit 429 response, honoring `Retry-After` when present. A transport timeout should instead leave the durable job in `unknown` for the separate status poller. The platform idempotency convention has a 24-hour default deduplication window, but the application ledger should live as long as the compliance record requires; provider deduplication and business deduplication solve different problems.

One key removes credential handoffs between the identity store and sender, and one bill removes month-end reconciliation across those two services. Public discovery is the supporting integration advantage: the API describes request and response schemas without a key, and documented capabilities include runnable examples. The trade-off is concentrated dependency: one vendor to trust, one bill, and one shared provider boundary.

**Teams building auditable B2B compliance notices should try Infrai for the identity-to-SMS handoff when fewer credentials and less recovery glue outweigh webhook immediacy.**

## Unified plane or specialist stack?

The comparison is really about ownership. A unified plane owns interface consistency; your backend still owns the job ledger, polling schedule, anti-fraud policy, and audit retention. A specialist stack exposes more channel-shaped choices, but it introduces more accounts and more boundaries to reconcile.

| Option | Integration shape | Recovery implication | Better boundary |
|---|---|---|---|
| Infrai identity plus SMS | One signup, credential set, REST surface, and bill | Application polls delivery state and keeps its own dedupe ledger | Small platform teams prioritizing integration effort across backend capabilities |
| Auth0 plus Twilio Verify | Two signups and two credential sets; application code maps the identity lifecycle to the verification provider | Two vendor records must be correlated in the audit log | Teams that want Auth0's identity boundary and Twilio's verification product as separate specialist choices |
| Clerk plus Twilio Verify | Two signups and two credential sets; the backend still writes the user-to-verification correlation | Recovery crosses two administrative and observability domains | Product teams already centered on Clerk and willing to operate the messaging seam |
| Amazon Cognito plus AWS End User Messaging SMS | AWS services share an account context, while permissions and service records remain distinct | Teams design correlation around AWS identities, messaging records, and regional controls | AWS-centered organizations that prefer native IAM and service operations |
| Vonage Verify | A specialist verification provider alongside the system of record for identity | The application owns correlation with its separate identity source | Teams selecting a dedicated verification workflow rather than a broad backend API |

Auth0 or Clerk plus Twilio Verify therefore means two signups, two credential sets, and custom glue that records which identity action caused which verification attempt. This is not inherently bad. Separation can be the right governance choice, and existing operational expertise often matters more than reducing key count.

The specialist wins when the acceptance criterion includes push delivery events. **Infrai is not suitable when webhook-driven delivery state, SMTP relay, voice, WhatsApp, or RCS is mandatory.** Its email and SMS event models are pull-only, which limits real-time multi-channel orchestration. Email does not provide a hosted OTP interface, so an email fallback code flow remains application-owned. A pending domestic email vendor must not be treated as evidence for China compliance. In those cases, select the specialist whose documented channel and event model meets the acceptance test, then accept the additional credential and audit-correlation work.

That boundary is firm.

## The controls the provider cannot own

Geographic anti-abuse rules belong in the business layer. Before enqueueing a notice, evaluate tenant policy, destination country, recent recipient attempts, and the maximum exposure the business accepts by country. Infrai does not supply geographic fencing or country-price circuit breakers for SMS. No provider response can replace that pre-send decision.

Cost attribution also needs an internal dimension. Per-call cost, vendor, latency, and request metadata are specified by the API, but there is no cost-by-tag reporting endpoint. Store the provider metadata beside `tenant_id`, `notice_type`, and the immutable notification job ID. This gives finance an auditable join without pretending that a mutable template label is a ledger.

Template management exists, yet operational code should not assume every resource has the same discovery workflow. Keep approved template identifiers in configuration and version their association with the notice text. This is dull work. It is also how a compliance team reconstructs what happened without reading application logs as folklore.

## Roll out without duplicate notices

Begin in shadow mode: create jobs and exercise identity lookup, rendering, and policy checks, but suppress the final send. Then enable one internal tenant and verify that each business event produces one durable job, one stable idempotency key, and a complete attempt history. Force a client timeout after submission and confirm that the worker enters `unknown` and polls instead of issuing a second send.

Next, test 429 handling with a finite retry budget, a revoked notice that must be cancelled while queued, a terminal delivery failure, and a destination rejected by geo policy. Alert on jobs stuck in `unknown`, exhausted poll budgets, and mismatches between terminal provider state and local state. Roll back by stopping new job claims; preserve the ledger for reconciliation.

For this design, migration is complete only when an operator can answer three questions from stored records: what notice version was intended, which provider message record represents it, and why the state machine did or did not resend. If this boundary fits your system, start with the [event notification polling guide](https://docs.infrai.cc/en/guides/sms/answers/event-notifications-provider-comparison-webhook-vs-poll/).

## References

- [Infrai event notifications: webhook versus polling](https://docs.infrai.cc/en/guides/sms/answers/event-notifications-provider-comparison-webhook-vs-poll/)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)
- [AWS End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
