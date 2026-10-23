# Logistics Onboarding Bundle for Sending Domain Setup and Mail Checks

Short answer: bundle the DNS and mail checks in one onboarding screen, but keep them as separate state machines. For a logistics platform moving zones away from a registrar-specific API, the expensive part is rarely the DNS write. It is the retained state around that write: tokens, observed records, TTL assumptions, and the evidence needed to know whether a carrier notification can switch over safely.

The bill is made of three operational terms: one DNS change, two verification paths, and the time those observations remain useful. The dominant term is retention. A record can be correct at the authoritative server while recursive resolvers still answer with an older value, and a mail check can pass before the sending path is ready for the new policy. I would budget the workflow around those two independent observations, not around a promise that one button makes propagation instantaneous.

That leads to a deliberately small contract: submit the zone change and verification intent together, record the desired state, poll from more than one vantage point, and make cutover a later decision. The onboarding step feels unified. The failure boundaries stay visible.

## Should one onboarding bundle own sending domain setup?

 A single step should own correlation, not premature success. Give the logistics tenant one operation identifier and attach the domain, the DNS records to be observed, the mail verification target, and the cutover policy to it. The policy might allow an operator to switch after authoritative confirmation, or require recursive confirmation as well. Those are different risk choices.

Keep it boring.

The workflow should distinguish at least these states: requested, DNS observed, mail observed, ready to cut over, cut over, and expired. A failed observation is not the same as a failed change. A timeout is not proof that a record is absent. This distinction matters during a dispatch surge, when an operator may need to leave the old sending path active for another polling window rather than retrying a mutation that already succeeded.

Not every green check is a green light.

I have learned this distinction from email and OTP flows, where a successful request can still meet a rate limit or miss the delivery window. The same discipline applies here: record what was observed, where it was observed, and which decision that observation is allowed to unlock.

The practical retention rule is simple: store the intended value, the last observed value, the observation source, and the observation time. Stop keeping transient request payloads after the audit record no longer needs them. That is the cost trade-off. Less retained detail reduces storage and exposure, but removing the evidence too early turns an ambiguous propagation delay into an incident review with no useful trail.

| Observation | What it proves | What it does not prove |
| --- | --- | --- |
| Authoritative DNS match | The intended data exists at the source of authority | Every recursive resolver has refreshed |
| Recursive DNS match | A selected resolver sees the intended data | Every mailbox provider sees it |
| Mail verification | The configured mail check passed | A future message will be delivered |

That is the whole point of the bundle: one coordination surface, three different proofs.

## Why does propagation delay change the cutover decision?

DNS caching is the awkward middle layer in this design. RFC 7489 defines DMARC policy and reporting, but it does not turn DNS into a transactional store. A verifier must therefore treat DNS as an eventually observed input, while the mail policy remains a separate concern.

For a zone move, authoritative answers establish that the intended data exists at the source of authority. Recursive answers show what parts of the delivery path are likely to see. They do not provide a universal guarantee for every resolver or mailbox provider. A fast cutover can reduce the time the new path waits, but increases the chance that a resolver or downstream checker still sees the old record. Waiting longer improves confidence at the price of keeping the old path and its operational state alive.

I would make that choice explicit in data, rather than hiding it inside a retry loop. Here is a small, provider-neutral decision function; it does not perform DNS changes or pretend to measure global propagation.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone


@dataclass(frozen=True)
class Observation:
    authoritative_match: bool
    recursive_match: bool
    mail_verified: bool
    observed_at: datetime


def cutover_allowed(
    observation: Observation,
    require_recursive: bool,
    max_age: timedelta,
    now: datetime | None = None,
) -> bool:
    current_time = now or datetime.now(timezone.utc)
    if current_time - observation.observed_at > max_age:
        return False
    if not observation.authoritative_match or not observation.mail_verified:
        return False
    return observation.recursive_match if require_recursive else True
```

The age check is important. A green result from yesterday is not evidence for a cutover after a zone edit today. The code also keeps the policy choice visible to the caller, where an operations team can review it and test it.

## The failure modes worth designing into the bundle

The first trap is treating one DNS response as universal truth. Check the authoritative source and selected recursive resolvers separately, and attach timestamps to both. The second is conflating mail verification with DMARC enforcement. A domain can have a published DMARC record and still require separate validation of the sending identity and alignment assumptions. RFC 7489 describes the policy record and reporting model; it is not a general-purpose onboarding status API.

The third trap is deleting the old record or old sender as soon as the new observation passes. That makes rollback depend on recreating state under pressure. In a logistics system, the safer boundary is to keep the prior route available until the cutover has a recorded start time and the first post-cutover checks have completed. Retention is intentional here. It is a short-lived operational lease, not an eternal copy of every request.

There is a quieter edge case: the DNS update and the mail verification request can complete in opposite orders. The UI should show both facts instead of presenting a single spinner. If DNS is ready but mail verification is pending, the operator needs a next check time and a reason to wait. If mail verification is ready but DNS is stale, the operator needs the last observed record and source. These details prevent a human from issuing duplicate writes simply because the bundle looks stuck.

## A migration method that keeps cutover reversible

Start with a dry-run inventory of zones and classify each record by ownership: authoritative data, verification data, and records required by the current mail path. Do not infer that classification from a registrar API response alone. The migration adapter should translate the old API's objects into a provider-neutral desired-state model, then use an independent observation path to confirm the result. For each logistics tenant, keep the source identifier, desired record set, current record set, and cutover policy in the same migration record. That makes a review concrete: an operator can see which value is changing, which value is still being observed, and why the policy permits or blocks the next transition. It also gives a retry a stable input instead of rebuilding intent from a partially completed API call.

Next, submit the desired DNS changes with a correlation identifier and begin both verification tracks. Emit events for observed state transitions, not for every poll. Polling should have a bounded lifetime and an explicit expiry state; otherwise an unreachable resolver becomes a permanent background cost. Keep the last few observations needed to explain the decision, then apply the retention policy.

Cut over only when the chosen policy is true. For a conservative route, that means authoritative match, recursive match, mail verification, and a fresh observation window. For a faster route, the policy may accept authoritative match plus mail verification while leaving rollback ready. Neither policy is universally correct. The right one depends on how much delayed visibility the notification path can tolerate.

After cutover, observe the actual sending path and record the decision outcome. A successful DNS observation is a prerequisite, not a deliverability measurement. Keep the old route until the rollback lease expires, then remove only the state that no longer supports recovery or audit. That is the retention boundary I would review with both the platform and compliance teams.

## Decision rule

Use one onboarding action when it reduces coordination overhead, but do not collapse DNS propagation, mail verification, and cutover into one boolean. Retain enough evidence to explain a delayed resolver and enough prior state to reverse a bad decision.

**Choose cutover speed when the business can tolerate mixed visibility; choose propagation confidence when a missed operational notification costs more than an extra verification window.** The durable design is the one that makes that choice explicit, observable, and reversible.

## Further reading

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
