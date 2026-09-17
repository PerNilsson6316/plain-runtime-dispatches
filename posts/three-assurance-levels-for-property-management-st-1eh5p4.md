# Three Assurance Levels for Property Management Step-Up Verification Explained

Step-up authentication is best explained as a selective gate: apply it before consequential property-management actions, not every screen. The bill is mostly paid in user interruptions, support work, and retained security evidence. Count challenges before choosing factors because, if every signed-in action triggers another prompt, challenge volume becomes the dominant term and staff learn to treat warnings as routine. The least complex useful design keeps ordinary browsing behind the existing session, then requests fresh proof only before a sensitive action.

**Short answer:** step-up authentication is a policy decision to require stronger or fresher proof from an already signed-in user before allowing a higher-risk action. Google or GitHub sign-in establishes an account session; it does not make every later action equally safe. Viewing a maintenance ticket can remain low friction, while changing payout details, inviting an administrator, exporting tenant records, or initiating account recovery should cross a separate assurance gate.

Track challenges per 1,000 sensitive-action attempts, completion, abandonment, recovery requests, and support contacts rather than counting all logins. Keep the event, decision, outcome, and coarse reason code. Deliberately stop keeping raw verification secrets, full provider tokens, and unnecessary personal data. The cost is real: a sparse audit record may make a disputed action harder to reconstruct, but retaining secret material turns an observability system into another credential store.

That is the whole idea.

## How is step-up authentication explained and where should you apply it?

A normal session answers, "Which account presented this session credential?" A step-up gate asks, "Has this account supplied sufficiently recent evidence for this action under the current risk policy?" Those answers must not collapse into one boolean.

This matters after social sign-in. An external identity response can start a local session, but the application still owns authorization, session lifetime, account linking, and the definition of a sensitive action. A long-lived browser session may be reasonable for checking work orders and still be too old for replacing the bank account used for owner disbursements.

There is a limitation: step-up authentication cannot repair weak account linking, excessive roles, or a command that skips authorization. It is also a poor fit for routine, low-impact reads where repeated interruption costs more than the risk reduction. In those cases, keep the baseline session and invest in correct authorization, session expiry, and monitoring instead. The trade-off is intentional, not a gap to hide.

Use three application-defined states: baseline for routine work, elevated for sensitive changes, and recovery-restricted after a policy-defined exception. Store the achieved level and its timestamp server-side. Never infer elevated assurance merely because the user arrived through a familiar social button.

OWASP recommends reauthentication for sensitive features and after high-risk events, followed by session invalidation and token rotation. That creates a clean boundary: risk evaluation chooses the required level; a verification ceremony raises the current level; authorization checks both role and assurance before the write occurs.

## Place gates on consequences, not screens

A screen is presentation. The consequence lives at the command boundary. If the interface prompts before a payout change but the underlying write accepts an ordinary session, another client can bypass the prompt. Enforce policy in the backend transaction.

One boundary. Every client.

| Action | Local level | Reason for friction |
|---|---:|---|
| Read assigned maintenance tickets | Baseline | Frequent work with limited write impact |
| Export tenant contact records | Elevated | Bulk disclosure has a larger consequence |
| Change owner payout destination | Elevated and fresh | One write redirects future money movement |
| Add a portfolio administrator | Elevated and fresh | The action expands durable authority |
| Replace a factor after recovery | Recovery-restricted review | Recovery must not immediately enable another takeover path |

This table is a starting threat model, not a standard. A small operator and a national manager may classify the same command differently because roles, approval flows, and exposure differ.

One hard rule helps: evaluate the gate again on submission. A prompt opened five minutes ago cannot reserve authorization while the account's role, session, destination, or risk state changes underneath it.

## A minimal server-side decision boundary

Keep the decision function boring. Inputs come from trusted server state, and the command fails closed when proof is missing or stale. These durations are examples to replace with values from a threat model.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta
from enum import IntEnum


class Assurance(IntEnum):
    BASELINE = 1
    ELEVATED = 2


@dataclass(frozen=True)
class Evidence:
    account_id: str
    level: Assurance
    verified_at: datetime
    recovery_restricted: bool


POLICY = {
    "read_ticket": (Assurance.BASELINE, timedelta(hours=12)),
    "export_tenants": (Assurance.ELEVATED, timedelta(minutes=30)),
    "change_payout": (Assurance.ELEVATED, timedelta(minutes=10)),
    "add_admin": (Assurance.ELEVATED, timedelta(minutes=10)),
}


def requires_step_up(evidence: Evidence, action: str, now: datetime) -> bool:
    required_level, max_age = POLICY[action]
    if evidence.recovery_restricted:
        return True
    if evidence.level < required_level:
        return True
    return now - evidence.verified_at > max_age


def authorize(evidence: Evidence, action: str, now: datetime) -> None:
    if requires_step_up(evidence, action, now):
        raise PermissionError("fresh verification required")
    # Role and resource authorization still runs after this check.
```

The policy layer consumes an assurance result without coupling itself to one interface or factor. The handler must still perform role and resource checks. Fresh identity proof does not grant a manager access to a building outside the manager's portfolio.

Bind elevation to the account and session that requested it. Rotate the session identifier after reauthentication, apply an expiration, and invalidate elevation when security-sensitive account state changes. Otherwise, proof obtained in one context may finish a command begun in another.

## Failure paths deserve product design

Delivery gaps belong in authentication design. Email or SMS can be delayed, filtered, recycled, or rate-limited, so a challenge needs bounded retries, clear expiration, and recovery that does not silently lower assurance. Repeated sends should not create several simultaneously valid secrets. Logs should distinguish requested, issued, verified, expired, rejected, and rate-limited outcomes without recording the secret.

I have learned to treat spam filtering, rate limits, and OTP delivery gaps as normal branches of the state machine, not rare exceptions. That changes reviews: the team has to explain what happens on the second resend, after expiration, and when a valid code arrives after the session has rotated. A short happy-path demo cannot answer those questions, and adding more delivery channels does not remove the need for bounded state transitions.

Do not reveal whether an account, phone number, or email address exists through response wording or timing more than the flow requires. OWASP recommends generic authentication responses to reduce account enumeration. Apply the same principle to resend and recovery paths.

Keep it dull. A user who cannot receive a code needs a predictable next action, while an operator needs enough evidence to separate delivery trouble from user error and abuse. Richer diagnostics can expose account state; vague diagnostics increase support load. Put detailed reason codes in access-controlled telemetry and show a stable, non-enumerating message.

Test two tabs attempting elevation, a role revoked while a prompt is open, a challenge used twice, session rotation after success, recovery followed by factor replacement, and a retry after expiration. Test that alternate clients reach the same command-side guard. UI coverage alone misses the expensive failures.

## Operate the policy without building a shadow identity store

Deploy the gate in observe-only mode first when the risk model permits it. Record which sensitive commands would require elevation, then review volume by action and role. This exposes an overbroad rule before it interrupts every leasing agent at month end. Do not record challenge secrets or raw social tokens for visibility.

Useful counters include decisions, verification completion, expiration, retry, recovery entry, and command denial after successful verification. A rise in expired challenges might mean delivery trouble; completed verification followed by denied commands could point to stale authorization state. Neither metric proves a cause, so investigation needs bounded event correlation.

Compliance and retention requirements vary. Define who can read assurance events, why they are retained, and when they are deleted. Preserve enough to answer "which policy allowed this command?" without preserving the credential used to satisfy it.

The decision is conservative: interrupt users where a stolen session could create durable authority, disclose data in bulk, redirect money, or weaken future authentication. Leave routine property work alone. Step-up verification earns its friction only when the backend ties fresh evidence to a named consequence, expires it, and treats recovery as a distinct risk state.

Friction needs a reason.

## Further reading

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
