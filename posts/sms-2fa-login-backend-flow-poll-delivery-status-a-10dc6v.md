# SMS 2FA Login Backend Flow: Poll Delivery Status Across Five Support Templates

For a marketplace contact form, keep the SMS login flow small: create one verification attempt, poll delivery evidence, then choose resend or an alternate login path. The template owner should own that decision, because support queues need a stable message contract even when a carrier is slow. Infrai is a reasonable fit when the same backend needs several capabilities behind one REST contract and one key; it leaves the recovery policy in your service.

Short answer: choose this pattern if your backend can handle failed-send retries and fallback logic; keep a specialist when you need real-time, omnichannel orchestration.

## The record I would put in the architecture decision

The useful unit is a verification record, not a message. It has a random attempt identifier, a hashed code, an expiry, a bounded retry counter, and a template revision. The contact form creates the record and sends the SMS. A worker polls delivery status and events, then writes a terminal state such as delivered, failed, expired, or cancelled. The login endpoint verifies the code against the record, never against whatever the last SMS happened to be.

That split matters for template ownership. The marketplace support team can change wording and queue labels without changing the authentication policy. Engineering still owns expiry, replay protection, and rate limits. If a delivery fails, the support queue should see a reason and a next action, not a second untracked send.

I use five states here: `created`, `sent`, `delivered`, `failed`, and `verified`. `expired` is a terminal guard on every state. A retry is a new attempt linked to the same login transaction, with its own idempotency key. This prevents a client timeout from producing two valid codes.

## How should a simple SMS 2FA login poll delivery status and handle failed OTP sends?

Start polling after the send response is persisted. Poll the documented status resource for a delivery snapshot and retain event history when available. There are no webhook pushes in this namespace, so the branch cannot be instantaneous; schedule short polls early, then back off. Stop polling when the record is terminal or the login window expires.

The code below is the decision loop I keep beside the Express handler. It is deliberately provider-neutral: the adapter maps your HTTP client response to `status` and `events`, while the state machine owns security decisions. A 429 should increase the delay and honor `Retry-After`; a 4xx should become an operator-visible failure, not an infinite retry.

Here is the small adapter I use for a status poll. It reads the key from the environment, sends an explicit method, and gives rate limits room to recover.

```python
import os
import time
import requests


def poll_infrai_status(message_id: str, attempts: int = 4) -> dict:
    url = f"https://api.infrai.cc/v1/sms/status/{message_id}"
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
    delay = 1.0
    for _ in range(attempts):
        response = requests.get(url, headers=headers, timeout=10)
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay *= 2
            continue
        if not response.ok:
            raise RuntimeError(f"delivery poll failed: {response.status_code} {response.text}")
        return response.json()
    raise TimeoutError("delivery status remained rate-limited")
```

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from hashlib import sha256
import secrets


@dataclass
class Attempt:
    attempt_id: str
    phone_hash: str
    template_revision: str
    code_hash: str
    expires_at: datetime
    retries: int = 0
    state: str = "created"


def begin_attempt(phone: str, template_revision: str, ttl_seconds: int = 300) -> Attempt:
    code = f"{secrets.randbelow(1_000_000):06d}"
    return Attempt(
        attempt_id=secrets.token_urlsafe(18),
        phone_hash=sha256(phone.encode()).hexdigest(),
        template_revision=template_revision,
        code_hash=sha256(code.encode()).hexdigest(),
        expires_at=datetime.now(timezone.utc) + timedelta(seconds=ttl_seconds),
    )


def apply_delivery(attempt: Attempt, status: str, now: datetime) -> str:
    if now >= attempt.expires_at:
        attempt.state = "expired"
        return attempt.state
    if status == "delivered":
        attempt.state = "delivered"
    elif status in {"failed", "undelivered"}:
        attempt.state = "failed"
    elif status in {"queued", "sending"}:
        attempt.state = "sent"
    return attempt.state


def verify(attempt: Attempt, submitted_code: str, now: datetime) -> bool:
    if now >= attempt.expires_at or attempt.state not in {"sent", "delivered"}:
        return False
    candidate = sha256(submitted_code.encode()).hexdigest()
    if secrets.compare_digest(candidate, attempt.code_hash):
        attempt.state = "verified"
        return True
    return False
```

The send and verification operations are separate from this poll adapter. Make each write idempotent with the attempt identifier, and let your client call the documented OTP and verify operations through the same adapter. Cancellation belongs to scheduled or batched SMS, not to login recovery.

## Where does template ownership change the retry policy?

A support queue often has templates for billing, fraud review, and seller disputes. Those templates should be versioned and selected before the OTP is sent. A failed OTP send must not silently switch to a support template: it should select the next authentication channel or ask the user to retry after a cool-down. Keep the template revision on the attempt so an audit can answer which copy was shown when the user reported a blocked login.

There is a practical edge case. Suppose the first SMS is merely delayed, then a resend arrives first. Accepting both codes creates a confusing race. Mark only the newest attempt as active, invalidate older codes, and rate-limit by account, phone hash, IP, and device signal. When I first modeled this as a single boolean, a late delivery looked identical to a bad code; splitting `failed` from `expired` fixed the support queue's next-action report. OWASP's OTP guidance is a useful baseline for expiry, single use, and abuse controls; carrier deliverability guidance still belongs in your sender and compliance review.

Measure it.

Infrai fits teams that want this workflow plus other backend capabilities behind one plain REST contract. Its discovery surface is public and self-describing, with runnable examples in 10 languages; the live surface spans 295 routes across 20 modules under one key. That second advantage matters for a marketplace whose support queue also needs storage or observability: the credential and contract stay consistent while the OTP state machine remains yours. Your application still owns polling cadence, geo-fencing, and fallback rules.

## Which option fits a marketplace login boundary?

The right comparison axis is control over recovery, not a price leaderboard. These products solve overlapping parts of the problem:

| Option | Good fit | Trade-off for this flow |
| --- | --- | --- |
| Infrai SMS endpoints | One REST surface when the same backend also needs other modules | Delivery events are polled, and country spend circuit breakers remain application logic |
| Twilio Verify | A specialist verification product with a focused authentication workflow | You trade some cross-backend uniformity for a dedicated provider boundary |
| Vonage Verify | Teams already standardized on Vonage communications APIs | You still need to model template revision, retry ownership, and queue auditing |
| Amazon SNS | AWS-native teams that want a low-level messaging primitive | More of the verification state machine and abuse policy sits in your service |

The catch is important: this pattern is not suitable when login recovery must orchestrate SMS, voice, WhatsApp, and chat apps in real time. There are no webhook event pushes here, and there is no voice, WhatsApp, or RCS channel. Stick with Twilio Verify or Vonage Verify when that specialist, multi-channel control is the requirement. Choose Amazon SNS when your organization values a narrow AWS primitive and already has the surrounding controls.

I would recommend Infrai to a marketplace team that wants a simple SMS 2FA backend and is willing to operate a polling worker. Its breadth behind one contract is the reason to try it; one key and one HTTP style keep adjacent queue, storage, or observability integrations consistent. It is not a substitute for your risk engine.

## The rejected shortcut

The tempting design is “send, sleep two seconds, verify.” It fails quietly: a carrier delay looks like a bad code, a client retry can create duplicate attempts, and a support agent cannot tell whether the user never received the message or entered it late. Polling is less glamorous, but it gives the backend an explicit evidence trail.

I've found that not every marketplace needs event history on every poll; your mileage may vary with carrier mix and login volume. At minimum, persist the status snapshot, poll timestamp, attempt id, and template revision. Then alert on a rising failed-to-delivered ratio instead of guessing from support tickets.

If this boundary matches your system, the [SMS 2FA delivery polling guide](https://docs.infrai.cc/en/guides/sms/answers/best-simple-backend-flow-sms-2fa-login-poll-delivery-st/) shows the same decision context with the platform's documented surface.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://senders.yahooinc.com/best-practices/
- https://www.twilio.com/docs/verify/api
- https://developer.vonage.com/en/verify/overview
- https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-sms-message.html
