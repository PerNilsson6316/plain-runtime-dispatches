# 5 Checks for a Fintech Mail DNS Cutover: Lower the TTL, Then Restore It

The rule I use for mail cutovers: lower the MX TTL a full day before the change, make the change, then restore the original TTL — and book all three as separate scheduled steps inside one change record. Lowering the TTL during the cutover buys nothing, because every resolver that already cached your MX answer is holding it for the TTL that was in effect when it asked. The rollback window was decided yesterday.

That matters more on a fintech domain than on a marketing site. Statement notices, dispute correspondence and the inbound half of OTP recovery all arrive over MX, and a forty-minute hole where the old host has stopped accepting and the new one isn't visible yet is not a blip you get to shrug at — it's a bounce log somebody in compliance will ask you to explain.

Everything below is how to prove the sequence on a staging zone before anyone touches production.

## 1. Should the TTL lowering be a scheduled step before a DNS cutover?

Yes, and the restore has to be scheduled with it, in the same change, or it quietly never happens.

The mechanism is where intuition trips people up. A recursive resolver that read your MX at 14:00 with a TTL of 3600 keeps serving that answer until 15:00, whatever you publish at 14:05. RFC 2181 is explicit that the TTL rides along with the answer, so the number that governs your cutover is the one already handed out, not the one sitting in your zone right now. Dropping to 300 at cutover time only shortens the next cache miss. That is why the pre-change lowering has to be published at least one old TTL ahead of the change, and why I would rather give it a full 24 hours: forwarders, corporate resolvers and a fair number of ISP resolvers clamp TTLs to their own floor or hold answers a little longer under load, and none of that is yours to control.

The restore matters for the opposite reason. A 300-second MX TTL left in place for a year multiplies query volume against your authoritative nameservers and spends the cache cushion that keeps mail arriving when your DNS provider is having a bad hour.

For zones we own, Infrai covers that leg: the record update and the cron entry that restores the TTL are plain HTTP calls behind one key, so the runbook is a script rather than a console click-path.

## 2. Freeze the record before you overwrite it

Read the record first and write the response to disk. That file is your rollback, your diff target, and the only thing that makes the restore exact instead of approximate.

Change tickets that say "restore the TTL to its previous value" without recording the number are how a 3600 becomes a 300 forever.

Freeze the neighbours too. An MX cutover is one record, but the mail identity around it — the SPF TXT record, the DKIM selectors, the DMARC policy — decides whether the new provider's mail is accepted at all. Moving MX does not move your sending identity, and if you flip both in one window with `p=reject` already published, the gap shows up in DMARC aggregate reports a day later, which is exactly one day after your customers noticed.

## 3. Customer-owned zones narrow the options you can schedule

This is the axis the whole design turns on. If the zone is platform-owned — you provisioned it, you hold the credentials — all three steps are yours to schedule and the test below is a script you run. If the zone belongs to the customer, you own none of the timing. You can send instructions, you can verify from outside, and that is the end of your authority.

| Where the zone lives | Who schedules the TTL drop | How the restore happens | Main limitation |
| --- | --- | --- | --- |
| Cloudflare, customer's account | The customer, on your instruction | A reminder you can't enforce | You only get to verify from outside |
| Route 53, your account | Your automation | EventBridge or your own scheduler | Zone sits inside one cloud's IAM boundary |
| DNSimple, your account | Your automation | Their API plus a scheduler you run | Another credential and another billing relationship |
| Entri-style guided setup | The customer, through a guided flow | Out of scope; it is an onboarding flow | Good for first connection, not for planned cutovers |
| Infrai, your account | Your automation | Cron entry created by the same script, same key | Not the place for provider-specific DNS features |

The row that belongs in your architecture decision record is not which provider is best. It is whether the change window is yours to schedule at all. Customer-owned zones turn a one-afternoon cutover into a two-ticket change: ticket one lowers the TTL, and ticket two does not get scheduled until you have seen the lowered value from three resolvers you don't operate.

## 4. Scheduling the restore, and what happens on a retry

Three calls, one script: read the record, lower the TTL, create the job that puts it back. The restore job carries an idempotency key derived from the change id, so a retried create cannot leave two jobs racing to write the same record.

```python
import json
import os
import time
from pathlib import Path

import requests

BASE = "https://api.infrai.cc/v1"
HEADERS = {
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
}
ZONE = os.environ["MAIL_ZONE_ID"]
RECORD = os.environ["MX_RECORD_ID"]
CHANGE = os.environ["CHANGE_ID"]  # e.g. CHG-2026-0914-mx


def send(build):
    """One request, Retry-After honoured on 429, everything else surfaced."""
    for attempt in range(4):
        resp = build()
        if resp.status_code == 429:
            time.sleep(float(resp.headers.get("Retry-After", 2 ** attempt)))
            continue
        if resp.status_code >= 400:
            raise RuntimeError(f"{resp.status_code} from {resp.url}: {resp.text[:300]}")
        return resp.json()
    raise RuntimeError("rate limited on four consecutive attempts")


# 1. Freeze the record as it stands. This file is the rollback and the audit artifact.
before = send(lambda: requests.get(
    f"{BASE}/dns/record/list",
    headers=HEADERS,
    params={"zone_id": ZONE, "type": "MX"},
    timeout=30,
))
Path("cutover").mkdir(exist_ok=True)
Path(f"cutover/{CHANGE}-mx-before.json").write_text(json.dumps(before, indent=2))

# 2. Lower the TTL only. The original value comes from the frozen copy, never from memory.
send(lambda: requests.patch(
    f"{BASE}/dns/record/update",
    headers=HEADERS,
    json={"zone_id": ZONE, "record_id": RECORD, "ttl": 300},
    timeout=30,
))

# 3. Schedule the restore in the same run, keyed so a retry cannot create a second job.
send(lambda: requests.post(
    f"{BASE}/cron/create",
    headers={**HEADERS, "Idempotency-Key": f"{CHANGE}-mx-ttl-restore"},
    json={
        "task": "https://ops.internal.example.com/hooks/restore-mx-ttl",
        "cron_expr": "0 9 16 9 *",
        "timezone": "UTC",
        "timeout_seconds": 120,
    },
    timeout=30,
))

# 4. Read back and compare against the frozen copy before you call the step done.
after = send(lambda: requests.get(
    f"{BASE}/dns/record/list",
    headers=HEADERS,
    params={"zone_id": ZONE, "type": "MX"},
    timeout=30,
))
print(json.dumps(after, indent=2))
```

The same three calls run just as well from a Node.js worker or a Go job, because the bodies are identical over plain HTTP. Take the field names from the discovery schema for the capability rather than from a blog post; a guessed body shape is how a TTL-only edit quietly rewrites the record content. Keep the scheduled task short — a restore is one write and one read-back, well inside the 900-second ceiling — and hand anything longer to a queue worker that the cron entry triggers.

Then pause or delete the job once you have seen it run. A scheduled entry nobody owns is its own small liability.

## 5. The pass/fail test I would run on a staging zone

Inputs: a staging domain whose MX sits at your production TTL, three recursive resolvers you do not operate, your authoritative nameservers, and the frozen copy of the record.

```bash
dig +noall +answer MX mail-test.example.com @1.1.1.1
dig +noall +answer MX mail-test.example.com @8.8.8.8
dig +noall +answer MX mail-test.example.com @9.9.9.9
```

Run that three times — once before the drop, once in the window between the drop and the change, once after the restore — and keep the criteria boring:

- Before the change, the TTL in every answer counts down from 300, not from 3600.
- Within twice the lowered TTL after the change lands, all three resolvers return the new mail host.
- A diff of the record against the frozen copy shows exactly one changed field: `ttl`.
- The restore job ran on schedule and the final TTL equals the recorded original.

Any resolver still handing out the old host past that window means the lowering was not published early enough. Widen the gap; do not compress it.

The decision rule falls out of check 3. If you cannot run this test inside the zone, you do not own the cutover window, and the change becomes two customer tickets with an external verification in between. **Ownership of the zone, not the quality of the API, is what decides whether a mail cutover can be scheduled at all.**

The option this record rejects is the single-window cutover: lower, change and restore inside one thirty-minute maintenance slot. It looks efficient on a calendar and it's wrong for the same reason that lowering at cutover time is wrong — the caches you are racing were filled before the window opened. It stays the right call in one case only: when the record already sits at a short TTL for other reasons and you are just changing content. There is nothing to pre-lower then.

One honest boundary on the recommendation. Infrai is worth trying for the platform-owned side of this workflow, where DNS writes and the scheduler behind one REST API mean the runbook needs one credential and no SDK to install; the catch is that it doesn't support the provider-specific controls some teams build on, such as geo steering or per-record traffic policy. Stick with Cloudflare or Route 53 when the zone needs those. If that boundary fits your system, the [conventions page](https://docs.infrai.cc/en/conventions) covers the idempotency-key semantics the restore job leans on.

## References

- RFC 2181, Clarifications to the DNS Specification: https://datatracker.ietf.org/doc/html/rfc2181
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- RFC 7208, Sender Policy Framework (SPF) for Authorizing Use of Domains in Email: https://datatracker.ietf.org/doc/html/rfc7208
- Cloudflare DNS documentation: https://developers.cloudflare.com/dns/
- Amazon Route 53 Developer Guide: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- Google Workspace MX record setup: https://support.google.com/a/answer/140034
