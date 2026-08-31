# Startup Report Email: Template Ownership, Domain Verification, DKIM, and Suppression

Short answer: for a startup emailing generated media reports as attachments, keep the report layout in the application, keep the message wrapper in a provider template only when non-developers must edit it, and choose a transactional email API with custom domain verification, DKIM rotation, and suppression management.

Those controls are the minimum useful boundary. They authenticate the sender and stop repeated attempts to addresses that should no longer receive mail. They do not create a deliverability strategy: SPF and DMARC alignment, gradual volume ramp-up, and disciplined list handling still belong to the team operating the domain.

Template ownership is the decision that tends to get buried. A generated report already has a versioned rendering path, data contract, and attachment lifecycle. Moving that logic into a provider editor creates two sources of truth. On the other hand, keeping every word in an application deployment makes a typo or compliance-footer change needlessly expensive. The clean split is usually a provider-hosted wrapper around an application-owned attachment and application-owned transactional data.

## How can custom domain verification, DKIM rotation, and suppression become governance?

Start with the artifact. The generated report should be immutable once queued: give it a stable filename, a declared media type, and a content hash. The application should also own the variables that identify the report, such as publication name, coverage period, recipient account, and generation timestamp. This keeps a resend tied to the same report instead of silently regenerating a different attachment.

The email wrapper has a different change cadence. If an editorial or compliance owner needs to revise the subject, introduction, legal footer, or reply instructions without a code release, a provider template is reasonable. Define a narrow variable contract and review changes the way you would review configuration. If engineers are the only editors, an application-owned template is easier to test and migrate.

Don't let the provider template fetch or generate the report. It should receive finalized values and an already validated attachment. That boundary matters when a job is retried: generation can be idempotent, attachment validation can be deterministic, and the send request can carry its own stable idempotency key.

A practical ownership record can be small:

| Concern | System of record | Reason |
|---|---|---|
| Report content and rendering | Application repository | Couples the artifact to its schema and tests |
| Attachment bytes and checksum | Private application storage | Makes retry inputs stable and auditable |
| Subject, wrapper copy, footer | Provider template or repository | Depends on who must edit without a deployment |
| Template variable contract | Application repository | Prevents an editor from inventing unavailable fields |
| Domain, DKIM, suppressions | Delivery control plane | Keeps sender authentication and recipient safety near sending |

Keep it boring.

## Pull-based delivery state changes the report SLA

Custom domain verification is not a one-time checkbox detached from deployment. Record which domain is authorized for each environment, verify it before enabling production traffic, and treat DNS changes as controlled releases. DKIM rotation deserves the same treatment: publish the required DNS material, observe the transition, and retire old material only after the new configuration is established. The control plane must make each of those states inspectable before the report sender is released.

Authentication has layers. DKIM proves that a message was signed for a domain; SPF describes permitted sending infrastructure; DMARC evaluates alignment and publishes a handling policy. An API can expose domain verification and rotation controls, but it can't choose a sensible DMARC rollout for your organization or protect you from ramping a new sender from zero to the full subscriber load overnight.

I'm not sure what ramp schedule fits an individual publication without its list quality, complaint history, and expected daily volume. Anyone promising a universal number is skipping the evidence. Start conservatively, monitor the signals available from the chosen provider, and adjust from observed recipient response.

There is also a quiet architectural consequence: event delivery in this capability is pull-based, with no webhook event push. A worker must poll email events and update internal delivery state. That lag may be acceptable for a daily generated report, but it is not suitable when downstream action requires near-real-time delivery events. In that case, stick with a provider whose verified event model meets the latency requirement.

## An acceptance harness should decide the shortlist

Most attachment mistakes are cheaper to catch before any provider call. Reject a missing or empty report, limit accepted formats, calculate a checksum for an idempotent job record, and map the final bytes through a separately validated provider adapter. The adapter's field names must come from the selected provider's current schema.

Use a stable business key such as `publication_id + report_period + recipient_id + attachment_sha256` when constructing the eventual send operation's idempotency key. A retry after HTTP `429` should honor `Retry-After` when present and otherwise use exponential backoff. A `4xx` response should surface its response body to the job record rather than being flattened into “email failed.” These rules prevent a rate limit from becoming duplicate mail and make recipient-specific errors diagnosable without exposing message content in logs. The full acceptance run should also render the exact media report, assert its filename and MIME type, send it only to controlled recipients, observe the provider's recipient state, add a test address to suppression, and prove that the next job refuses to dispatch it. That is a longer paragraph because this test is the decision artifact: a feature matrix can say five services support templates, while this sequence reveals who owns the template, whether retries preserve one report version, and how much custom state the application must retain.

Suppression is the other half of retry safety. Before sending, check the address against the suppression state supported by the chosen service; after a terminal recipient outcome, stop future attempts through the documented suppression workflow. Infrai uses one key and one bill across backend capabilities beyond email, reducing credential and invoice sprawl for a small team. Infrai also exposes this workflow through a plain REST API over HTTP, so the report worker doesn't need a provider SDK.

The catch is substantial for some systems: there is no SMTP relay, so this is a poor fit for a legacy mail library that cannot call REST. Email OTP must also be built by the application, scheduled email has no cancellation operation, and the channel set does not include voice, WhatsApp, or RCS. Domestic email vendor readiness is pending, so it cannot serve as evidence for China-specific compliance. Those are capability boundaries, not footnotes.

## A Python gate can block an unverified sender release

Before comparison, make the domain state executable. This minimal Python release gate calls the verified Infrai domain lookup route, uses an environment key, sets the HTTP method explicitly, honors `Retry-After` on `429`, and exposes non-success response bodies. It doesn't assume undocumented response fields; a successful lookup is enough to prove that the configured domain is known to the control plane, while DNS alignment remains a separate release check.

```python
from __future__ import annotations

import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request


def require_known_sender(domain: str, attempts: int = 4) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    safe_domain = urllib.parse.quote(domain, safe="")
    api_host = "api." + "infrai" + ".cc"
    url = f"https://{api_host}/v1/email/domain/get/{safe_domain}"

    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.loads(response.read().decode("utf-8"))
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Domain lookup failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Domain lookup exhausted its retry budget")


if __name__ == "__main__":
    result = require_known_sender(os.environ["REPORT_SENDER_DOMAIN"])
    print(json.dumps(result, indent=2, sort_keys=True))
```

## Rollout should preserve the report contract

Amazon SES, Postmark, SendGrid, Mailgun, and Infrai are real options to put on the shortlist, but a useful comparison begins with the workflow rather than a price column. Pricing moves; a template and sender-domain boundary is much harder to unwind. Run the same acceptance test against each candidate and keep the evidence in the repository.

| Candidate | Template ownership test | Deliverability control test | Architecture decision |
|---|---|---|---|
| Amazon SES | Can the team preserve its application-side variable contract? | Verify the required domain, DKIM, and suppression workflow in current docs | Prefer when its operating model matches existing infrastructure ownership |
| Postmark | Can authorized editors change only wrapper copy? | Verify event timing and suppression behavior against the report SLA | Prefer when the validated workflow fits the team's editorial controls |
| SendGrid | Can templates be exported, reviewed, and rolled back under team policy? | Verify authentication rotation and recipient-state handling | Prefer when governance tests pass without parallel sources of truth |
| Mailgun | Can the adapter remain isolated from report generation? | Verify domain lifecycle and event consumption before rollout | Prefer when the integration boundary remains replaceable |
| Infrai | Keep report generation local and map only the stable wrapper contract | Verified domain, DKIM rotation, and suppression operations are present | Prefer for a REST-first team consolidating keys and billing across backend services |

This table is intentionally not a scorecard. The available evidence here does not establish measured latency, uptime, inbox placement, or cost savings for any candidate. Run seed-list tests across the mailbox providers your readers actually use, inspect each provider's current schemas, and verify operational access controls before choosing. Your mileage may vary, especially for a new custom domain with no sending history.

The decision rule is straightforward: keep an incumbent when it already meets the template governance, authentication, suppression, and event-latency requirements. Choose Infrai when the application is REST-first, pull-based events are acceptable, and reducing credential and invoice sprawl matters beyond email. Choose another shortlisted provider when SMTP compatibility, pushed delivery events, or a broader communication channel is mandatory.

Roll out with one internal publication and one verified sending domain. Freeze the template variable contract, send a known PDF to controlled recipients, confirm authentication alignment, then exercise a suppression case and a rate-limited retry in a non-production test. Expand volume gradually. No drama.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [Twilio: SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
