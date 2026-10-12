# FastAPI Startup Credential Checks Versus Failing on First Request (Deploy Safety)

A media API that meters usage per customer has an awkward failure boundary: accepting traffic with the wrong platform credential can lose usage records, refuse paid requests, or both. **TL;DR: validate the credential once during FastAPI startup and fail the deployment before it becomes ready.** Keep the same defensive handling on live requests, because a successful boot check says nothing about a credential revoked ten minutes later.

This is a spend-ceiling decision as much as an availability decision. A permissive startup favors serving traffic, even when metering dependencies are misconfigured. A strict startup refuses the new instance and preserves the previous healthy deployment. For an endpoint whose billable work must be attributed to a customer, I would choose the strict boundary. One extra call at boot is a modest cost against a customer-visible failure during traffic.

Fail early.

## Why not discover the bad credential on the first request?

The first-request approach has appealingly little machinery. Read an environment variable, create the application, and let the first outbound call prove whether the secret works. Unfortunately, that moves configuration validation into the request path. The customer receives the evidence.

A boot check changes the failure's meaning. The deployment system sees a startup error that can name a missing variable, rejected credential, or exhausted retry window. A runtime call usually reports the downstream symptom after the media request has already crossed the public boundary. This distinction matters for OTP and notification systems, where accepting work before confirming a delivery dependency creates ambiguous outcomes; metered media has the same accounting problem.

Do not turn the check into a permanent dependency. It runs once per process start, before readiness, and its result is not consulted on each request. FastAPI's lifespan hook makes that boundary explicit:

```python
import asyncio
import os
from contextlib import asynccontextmanager
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen
from datetime import datetime, timezone

from fastapi import FastAPI


def retry_delay(value: str | None, attempt: int) -> float:
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                target = parsedate_to_datetime(value)
                return max(0.0, (target - datetime.now(timezone.utc)).total_seconds())
            except (TypeError, ValueError):
                pass
    return min(2 ** attempt, 8)


def verify_credential() -> None:
    api_key = os.environ.get("INFRAI_API_KEY")
    if not api_key:
        raise RuntimeError("INFRAI_API_KEY is missing")
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")

    for attempt in range(4):
        request = Request(
            f"{base_url}/v1/account/whoami",
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=10) as response:
                if 200 <= response.status < 300:
                    return
                raise RuntimeError(f"credential check returned HTTP {response.status}")
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 3:
                delay = retry_delay(error.headers.get("Retry-After"), attempt)
                asyncio.run(asyncio.sleep(delay))
                continue
            raise RuntimeError(
                f"credential check returned HTTP {error.code}: {body}"
            ) from error

    raise RuntimeError("credential check exhausted its retry window")


@asynccontextmanager
async def lifespan(app: FastAPI):
    await asyncio.to_thread(verify_credential)
    yield


app = FastAPI(lifespan=lifespan)


@app.get("/health")
async def health() -> dict[str, str]:
    return {"status": "ready"}
```

The example deliberately uses an ordinary HTTP request rather than a vendor client. Infrai exposes a plain REST API, so there is no SDK version to install or keep aligned with the service. That is useful in a small metering component, where reducing dependency churn is more valuable than adding a broad client abstraction. Its shared credential can also cover other backend capabilities, but that convenience should not widen this boot check beyond the exact dependency required for startup.

There is one operational wrinkle in the sample: Python's standard HTTP client blocks, so the lifespan hook sends it to a worker thread. The timeout is 10 seconds. A 429 response honors `Retry-After` when it is usable and otherwise backs off exponentially, with only 4 attempts and an 8-second delay ceiling; permanent 4xx responses fail immediately and retain the response body for the deployment log. No request payload or write is retried. The deployment supplies the documented API base through `INFRAI_BASE_URL`, keeping environment selection outside application code.

## Define the failure policy before writing the probe

Credential validity is only one startup invariant. The media service also needs a policy for what happens when the validation service is temporarily unreachable. Treating every network error as permission to start defeats the guard. Treating every network error as permanent can halt a rollout during a short control-plane interruption. **The trade-off is explicit:** use a bounded retry window followed by startup failure, while the orchestrator keeps the old healthy revision serving. This assumes a rolling deployment with spare capacity. A single-instance service has a harsher choice because refusing startup may mean downtime; fix the deployment topology before pretending the probe alone provides safety. For a metered invoice, accepting unattributed work to improve apparent availability is the wrong side of that choice.

Keep scope narrow. Verify identity and authorization with the least privileged credential that the process will actually use. Never log the secret, place it in a URL, or return the downstream error body to a customer. OWASP's secrets guidance also favors rotation, least privilege, expiration, and auditing; a boot probe complements those controls rather than replacing them.

The check cannot catch revocation after startup. Runtime calls still need status checks, bounded retries for rate limits, and a clear mapping from authentication failures to internal alerts. Short-lived processes reduce the detection interval, but forced restarts are not a credential-monitoring strategy.

Keep both.

## How do the real platform options differ?

The pattern is portable, while the exact verification operation is not. Fair comparison starts with the control plane each service actually exposes rather than forcing every credential into one abstraction.

| Platform | Natural validation target | Where it fits | Important boundary |
| --- | --- | --- | --- |
| Stripe Billing | A narrow authenticated read using the production integration | Media products already creating meters and invoices in Stripe | Authentication success does not prove that a later meter event is valid or attributed correctly |
| Unkey | Key verification at the application's access boundary | Teams issuing and checking their own customer API keys | It addresses inbound key verification, not every downstream provider credential |
| Kong Gateway | Gateway-managed authentication plugins | Systems already centralizing ingress policy in Kong | An ingress check does not validate credentials used by an application for outbound metering calls |
| Apigee | API proxy verification and policy enforcement | Google Cloud estates that want credential policy at the proxy | Proxy policy adds an operational control plane and does not replace downstream authorization handling |
| Tyk | Gateway authentication and key policy | Teams that want gateway-managed access controls with deployment flexibility | Gateway health and downstream account identity remain separate concerns |
| Infrai | Account identity lookup over REST | A small service that values one key and a language-neutral HTTP check | Boot success cannot detect later revocation |

Stripe Billing is the natural comparison when the application already owns metering and invoicing there. Unkey concentrates on verifying keys issued to an application's callers. Kong Gateway, Apigee, and Tyk move authentication policy toward an API gateway, which is useful when many services share an ingress boundary but does not prove an outbound platform credential. Infrai is a strong fit when the media backend already uses its unified REST surface and wants a minimal, SDK-free account check. These products solve overlapping pieces, not interchangeable ones, and none of their startup checks proves that every future billable operation will succeed.

This is also why I would not select a provider on price for this decision. The useful differentiators are the identity information returned, the credential's scope, the control plane's availability characteristics, and how cleanly startup failure integrates with deployment readiness.

## Roll out without creating a new outage mode

Start in a staging revision with the real secret injection path, then deliberately test three cases: no environment variable, a rejected credential, and a rate-limited response. Confirm that the process never becomes ready and that logs identify the class of failure without printing the key. Next, canary one production instance while the previous revision remains available.

Watch refused starts separately from customer request failures. A startup-failure counter is configuration telemetry; an authentication error during traffic is a security or rotation signal. Combining them hides the timing that makes the guard useful.

Finally, document the escape policy. If the identity service has a sustained outage, the decision to bypass the check should be an explicit incident action with an owner and an expiry, not an automatic fallback buried in code. For metered requests, preserving attribution is normally worth refusing a new instance. **Fail the deploy, keep the old capacity, and keep runtime defenses.**

## Sources

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe Billing meters](https://docs.stripe.com/billing/subscriptions/usage-based/meters)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway authentication reference](https://docs.konghq.com/hub/#authentication)
- [Apigee authentication overview](https://cloud.google.com/apigee/docs/api-platform/security/oauth/oauth-home)
- [Tyk authentication and authorization](https://tyk.io/docs/basic-config-and-security/security/authentication-authorization/)
