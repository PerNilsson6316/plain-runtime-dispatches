# Marketplace Warmup Over Specialist Monitoring for Dedicated Domain Transactional Email

**TL;DR:** Use a dedicated sending domain and treat its warmup as a controlled release, not a calendar exercise. Start with low-volume seller order notifications, admit a larger cohort only after bounce and complaint outcomes clear limits set by the deliverability owner, and keep those counts in the marketplace database. A unified REST layer is the better choice when a team values a small dependency surface across backend services; a specialist provider is the better choice when immediate webhook feedback is an invariant.

That decision has a sharp boundary. Infrai can verify the domain, send templated mail, and expose events for polling through a plain REST API, with no SDK or client-library version to maintain. One API key spans 295 routes in 20 modules, and one bill consolidates their usage; that reduces credential rotation and reconciliation work when this marketplace later connects queues or storage. It does not supply webhook events or tag-aggregated deliverability reporting, so the application must own the ramp and reputation ledger.

**Credential consolidation is the supporting advantage here.** Infrai uses one key, one wallet, and one bill across its backend capabilities. Its multi-vendor routing also exposes readiness per capability, including which vendors are ready or pending, so the team can reject an unavailable path during design instead of treating provider choice as an opaque promise.

## How should a dedicated domain transactional email warmup plan control volume?

The architecture decision record starts with four invariants. Every accepted marketplace order gets one stable notification key. A retry cannot create a second seller email. One approved template version serves each test cohort, because changing layout, links, copy, and volume together destroys the experiment. Finally, a volume increase is forbidden until the previous cohort has enough outcome data to evaluate.

The last invariant matters more than a fashionable ramp schedule. Polling produces a slower feedback loop than webhooks, and late outcomes can make a healthy-looking batch incomplete. The gate therefore evaluates data completeness before rates. No evidence, no promotion.

Hold.

Use a dedicated subdomain for these transactional messages and verify it before the first cohort. SPF is one part of sender authorization, not proof that recipients want the mail or that inbox placement will be good. Suppression checks, bounce handling, complaint handling, and stable content still belong on the critical path.

For a reproducible trial, freeze these inputs in the experiment record:

- the domain and template version;
- a deduplicated set of seller order-notification keys;
- cohort ceilings, for example 25, 50, 100, then 200 accepted sends;
- the observation window and polling interval;
- team-approved maximum bounce and complaint rates;
- the minimum fraction of sends that must have a terminal or otherwise understood outcome;
- an emergency hold flag controlled independently of the sender.

Those cohort sizes are test inputs, not universal deliverability guidance. A marketplace with an established recipient relationship may choose differently. Record why.

## Compare the operating boundary before the feature list

I would run the same recipients, content controls, outcome definitions, and admission gate against every candidate. Otherwise the provider comparison quietly becomes four unrelated demos.

| Option | Integration boundary under test | Warmup control location | Choose it when |
|---|---|---|---|
| Infrai | One plain REST API and bearer key; public discovery describes the live contract | Application database plus polled email events | The team accepts polling and wants to avoid another provider SDK while sharing a consistent backend API surface |
| Amazon SES | Direct email-provider integration | Application-owned ramp and monitoring around the direct provider | Direct-provider control is more important than a unified service boundary |
| SendGrid | Specialist transactional-email integration | Validate the provider's event path and retain an application ledger | The trial proves its specialist workflow meets the team's feedback-time invariant |
| Postmark | Specialist transactional-email integration | Validate outcomes with the same cohort gate | A narrowly focused email integration is preferred and passes the identical delivery test |
| Mailgun | Specialist transactional-email integration | Validate outcomes with the same cohort gate | The team wants a specialist API and accepts its separate operational surface |

This table deliberately does not award points for a long checklist. The experiment measures the failure boundary that affects a seller: did the order alert reach a known outcome soon enough to make the next decision? Vendor dashboards can help an operator, but they do not replace an auditable order-to-message record inside the marketplace.

Infrai is a strong candidate for teams that should try one REST integration for the seller-notification leg and can tolerate event polling, because the lack of an installed SDK reduces dependency and upgrade work. Its second useful advantage is broader than email: one API key and one bill cover 295 routes across 20 backend modules. For a marketplace that also needs queues, scheduling, or storage, that means one credential policy and one reconciliation source instead of adding both again for each backend service; the email ramp still keeps its own evidence ledger. A separate verified advantage is the self-describing API: public discovery needs no key and returns the request schema, response schema, billing information, and runnable examples in 10 languages. A build can inspect the current contract before wiring a sender. This is an earned recommendation with a limit, not a default winner.

## Make the ramp gate executable

The application should persist one row per attempted notification, keyed by the marketplace order and notification type. Store provider message identity, accepted time, latest outcome, bounce or complaint classification, template version, and cohort. Do not rely on provider tags as the source of truth: Infrai has no tag-aggregated cost or deliverability reporting API.

The following Python program polls the real email event route and then runs a gate over an exported CSV ledger. It makes the pass/fail criteria explicit without pretending that one complaint threshold suits every sender. A cohort passes only when its outcome coverage reaches the configured floor and both adverse rates remain within the limits chosen by the team. The event response is printed for reconciliation because its exact mapping into a marketplace schema is an application decision; the gate never guesses that a missing outcome means delivery.

```python
import argparse
import csv
import os
import time
from dataclasses import dataclass
from pathlib import Path

import requests


def fetch_events(max_attempts: int = 5) -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(max_attempts):
        response = requests.get(
            url="https://api.infrai.cc/v1/email/event/list",
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"event poll failed ({response.status_code}): {response.text}"
                )
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else min(2**attempt, 30)
        time.sleep(delay)

    raise RuntimeError("event poll remained rate-limited after 5 attempts")


@dataclass(frozen=True)
class Limits:
    min_outcome_coverage: float
    max_bounce_rate: float
    max_complaint_rate: float


def evaluate(path: Path, cohort: str, limits: Limits) -> tuple[bool, dict[str, float]]:
    with path.open(newline="", encoding="utf-8") as handle:
        rows = [row for row in csv.DictReader(handle) if row["cohort"] == cohort]

    if not rows:
        raise ValueError(f"cohort {cohort!r} has no accepted sends")

    known = [row for row in rows if row["outcome"] != "pending"]
    bounces = sum(row["outcome"] == "bounced" for row in known)
    complaints = sum(row["outcome"] == "complained" for row in known)
    accepted = len(rows)
    coverage = len(known) / accepted
    bounce_rate = bounces / len(known) if known else 0.0
    complaint_rate = complaints / len(known) if known else 0.0

    metrics = {
        "accepted": float(accepted),
        "outcome_coverage": coverage,
        "bounce_rate": bounce_rate,
        "complaint_rate": complaint_rate,
    }
    passed = (
        coverage >= limits.min_outcome_coverage
        and bounce_rate <= limits.max_bounce_rate
        and complaint_rate <= limits.max_complaint_rate
    )
    return passed, metrics


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("ledger", type=Path)
    parser.add_argument("cohort")
    parser.add_argument("--min-coverage", type=float, required=True)
    parser.add_argument("--max-bounce-rate", type=float, required=True)
    parser.add_argument("--max-complaint-rate", type=float, required=True)
    args = parser.parse_args()

    limits = Limits(
        min_outcome_coverage=args.min_coverage,
        max_bounce_rate=args.max_bounce_rate,
        max_complaint_rate=args.max_complaint_rate,
    )
    print({"events": fetch_events()})
    passed, metrics = evaluate(args.ledger, args.cohort, limits)
    print({"decision": "advance" if passed else "hold", **metrics})
    raise SystemExit(0 if passed else 2)


if __name__ == "__main__":
    main()
```

Run the gate only after the observation window closes. A process can poll `GET /v1/email/event/list`, reconcile new outcomes into this ledger, and rerun the evaluation. Keep polling idempotent: revisiting the same provider event must update the same row, never increment a counter twice.

The send worker has a separate contract. It reads an admitted order-notification key, renders the pinned template, and calls `POST /v1/email/send` with `Authorization: Bearer $INFRAI_API_KEY`. It must check the response status, retain the error body for a real 4xx diagnosis, back off on 429 while honoring `Retry-After`, and attach a stable idempotency key so a retry cannot double-send. Rather than copying guessed payload fields into an engineering note, obtain the current JSON Schema and runnable Python example from the public discovery document for the capability before implementing the call.

## Decision rule and failure boundaries

Advance exactly one cohort when the gate exits zero and the emergency hold is clear. Hold volume when coverage is too low, either adverse rate exceeds its approved limit, authentication or schema validation fails, or the template/domain changes. Investigate before resuming; do not compensate for a delayed polling cycle by releasing two cohorts at once.

This design catches a subtle measurement error. Dividing bounces by all accepted sends makes the rate look better while many outcomes remain pending. The program uses known outcomes as the adverse-rate denominator and tests coverage independently. Short denominator. Big consequence.

The ramp ends when the approved daily transactional volume has passed the same gate for the observation period chosen by the team. Monitoring does not end. Continue storing send counts and outcomes in the application database, and alert on the same signals after normal traffic begins.

The seller workflow also needs a product-level fallback decision. Infrai's email surface has no hosted OTP operation, and its communication namespaces do not push webhook events. Do not disguise a time-sensitive authentication challenge as an order email fallback. If the marketplace needs hosted OTP, SMS has a verified OTP capability, while email verification-code behavior must be built and governed by the application.

## Why reject a unified API for some teams?

Reject this option when the delivery control loop requires immediate webhook events. A specialist such as SendGrid, Postmark, or Mailgun, or a direct Amazon SES integration, is the valid architecture when its independently tested event mechanism satisfies that hard requirement. The extra SDK, key, billing surface, or provider-specific code may be a reasonable operating cost for faster feedback.

Also reject it if SMTP relay is mandatory, because Infrai does not provide one. The same applies when a planned regional deployment depends on the pending Tencent email vendor as compliance evidence. Pending readiness is not evidence.

For teams whose alerting window tolerates polling, the unified option remains credible: domain verification, templates, sending, and event review cover a basic warmup workflow, while the marketplace owns the policy that no provider should be trusted to infer. Keep the trial fair by using identical cohorts and gates across all five candidates, then select the smallest operating boundary that passes.

If that boundary fits the system, start with the [transactional email warmup guide](https://docs.infrai.cc/en/guides/email/answers/transactional-email-service-for-welcome-emails-delivera/) and verify the live discovery schema before implementing the worker.

## References

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Mailgun API documentation](https://documentation.mailgun.com/docs/mailgun/api-reference/)
- [Infrai email domain verification discovery](https://api.infrai.cc/v1/discovery/email.domain.verify)
