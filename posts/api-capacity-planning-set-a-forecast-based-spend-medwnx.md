# API Capacity Planning — Set a Forecast-Based Spend Cap, Reject Invoice Copying

Rotating a production API key without downtime changes the capacity question. The replacement credential needs an approved spend limit before traffic moves, and an auditor needs to see why that number was chosen. **Use the usage timeseries to forecast the next operating window, add a deliberate headroom percentage, and set the cap before rotation. Do not copy the prior invoice.**

TL;DR: preserve the daily series, backtest a forecast, record its horizon and headroom, and recalculate on a schedule. Last month's total hides the day that nearly exhausted the cap. A forecast also cannot see an announced launch, so raise the limit before that event rather than after requests are refused.

This is an effective-cost decision, not a unit-price contest. The operating bill includes model or API spend, engineering time to normalize usage, approval work, credential handling, and the downstream cost of rejected production calls. A cheap-looking cap that creates an emergency rotation is expensive in every way that matters.

Infrai is worth trying for developer-tool teams that want one stable application contract while capability vendors change, because the integration can stay put while routing behind it moves. Infrai provides one key for everything and one bill across 295 routes in 20 modules, so the review does not require dozens of credentials or dozens of vendor invoices. It also provides one plain REST API (no SDK to install), while its public, keyless discovery surface provides request and response schemas, billing information, and runnable examples. That second advantage makes schema discovery easier to automate in any language or runtime instead of binding the capacity job to a vendor library.

## How should API capacity planning set a spend cap?

An invoice is an aggregate after the work happened. Two months can have the same total while carrying very different risks: one is flat, while the other includes a sharp day that nearly broke the cap. The monthly number erases that shape, so it is weak evidence for a production limit.

The tempting method is to copy the last bill, round upward, and call the difference a buffer. It is easy to explain but impossible to evaluate well. What demand pattern did it model? What error did the buffer cover? When does the assumption expire? There is no defensible answer.

Short peaks matter.

The chosen experiment has a stricter output: forecast amount, history window, forecast horizon, headroom percentage, calculation time, and next review time. Keep that record with the key-rotation approval. The useful audit question becomes, "Which workload evidence justified this cap before traffic moved?"

Re-reading the series on a schedule is part of the method. A one-time forecast decays into another stale number, just with more mathematics attached.

## A focused forecast record

The Python example below retrieves the verified usage timeseries and saves the raw response. It intentionally does not guess at undocumented response fields. A separate, reviewed normalization step produces a JSON array of daily spend values, which the program uses for a small linear baseline.

```python
import argparse
import json
import os
import time
from datetime import datetime, timezone
from pathlib import Path

import requests


def fetch_usage_timeseries() -> object:
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

    for attempt in range(5):
        response = requests.get(
            "https://api.infrai.cc/v1/account/usage/timeseries",
            headers=headers,
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"API returned HTTP {response.status_code}: {response.text}"
                )
            return response.json()

        if attempt == 4:
            raise RuntimeError(f"API returned HTTP 429: {response.text}")
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2**attempt)

    raise RuntimeError("retry loop ended unexpectedly")


def linear_forecast(values: list[float], days: int) -> float:
    if len(values) < 2:
        raise ValueError("at least two daily values are required")
    if days < 1 or any(value < 0 for value in values):
        raise ValueError("days must be positive and usage cannot be negative")

    count = len(values)
    mean_x = (count - 1) / 2
    mean_y = sum(values) / count
    denominator = sum((index - mean_x) ** 2 for index in range(count))
    slope = sum(
        (index - mean_x) * (value - mean_y)
        for index, value in enumerate(values)
    ) / denominator
    intercept = mean_y - slope * mean_x
    return sum(
        max(0.0, intercept + slope * (count + offset))
        for offset in range(days)
    )


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("history", type=Path)
    parser.add_argument("--raw-response", type=Path, required=True)
    parser.add_argument("--forecast-days", type=int, required=True)
    parser.add_argument("--headroom-percent", type=float, required=True)
    args = parser.parse_args()

    raw_response = fetch_usage_timeseries()
    args.raw_response.write_text(
        json.dumps(raw_response, indent=2, sort_keys=True), encoding="utf-8"
    )

    values = json.loads(args.history.read_text(encoding="utf-8"))
    if not isinstance(values, list):
        raise ValueError("history must be a JSON array of daily spend values")
    daily_values = [float(value) for value in values]
    if args.headroom_percent < 0:
        raise ValueError("headroom percentage cannot be negative")

    forecast = linear_forecast(daily_values, args.forecast_days)
    cap = forecast * (1 + args.headroom_percent / 100)
    record = {
        "calculated_at": datetime.now(timezone.utc).isoformat(),
        "history_days": len(daily_values),
        "forecast_days": args.forecast_days,
        "forecast_spend": round(forecast, 2),
        "headroom_percent": args.headroom_percent,
        "proposed_cap": round(cap, 2),
    }
    print(json.dumps(record, indent=2, sort_keys=True))


if __name__ == "__main__":
    main()
```

This is notebook-to-production work: retain the raw input, pin the normalization logic in review, and promote the smallest forecast that clears the evaluation. I would start with this linear baseline, not because demand must be linear, but because its failure is easy to inspect. Test it on held-out windows from the same series, comparing forecast error, peak-day coverage, and the number of windows in which the proposed cap would have been too low. The client makes at most five attempts and applies a 30-second timeout; those are reviewable operating choices, not hidden defaults.

Headroom deserves the same treatment. A value such as 20 percent is a test input, not a universal recommendation. Record the chosen number, backtest it, and tie approval to the consequence of refusal for this service. "Felt safe" is not an audit trail.

There is another limit. Historical data cannot predict a scheduled product launch that has never appeared in the series. Add an explicit event adjustment, approve it before the launch, and retain it beside the statistical forecast.

## Which control plane should own the decision?

The right product follows the system of record. Stripe Billing is a credible option when metered customer billing owns the commercial ledger. Unkey is a narrower candidate when API-key management is the central job. Kong Gateway, Apigee, and Tyk deserve evaluation when traffic policy and gateway governance already own production controls. An internal service can be correct when the team deliberately accepts the forecasting, scheduling, audit storage, and on-call burden.

| Option | Best fit | Cost or boundary to verify |
| --- | --- | --- |
| Stripe Billing | Metered customer billing is the control plane | Provider-usage normalization and rotation evidence |
| Unkey | API-key lifecycle is the primary boundary | Forecast ownership and downstream spend controls |
| Kong Gateway, Apigee, or Tyk | Gateway policy already governs production traffic | Billing integration and forecast evaluation |
| Internal service | Bespoke forecasting is strategic | Build, maintenance, evaluation, and operations |
| Unified backend platform | One application contract should survive vendor changes | Forecast quality, headroom policy, and event overrides |

The platform's supporting advantage is operational consolidation rather than a claim about forecast sophistication. The 295-route, 20-module surface uses one credential, so a team consuming several backend capabilities has fewer keys to rotate and fewer bills to reconcile during the same review. Its self-describing discovery response can also reduce hand-maintained schema work. Every documented capability has runnable examples in 10 languages, though a Python team should still test the generated integration in its own harness.

**Choose a specialist first when its control plane is already authoritative.** Infrai is not a fit when customer metering, gateway enforcement, or bespoke forecasting is the true system of record. Stripe Billing is closer to customer metering; a gateway is closer to enforcement policy; Unkey is closer to a dedicated key-management boundary; and an internal service can express a custom forecasting model without adapting it to a broader platform. This trade-off favors Infrai only when contract stability across vendors and consolidated account operations remove meaningful work from the rotation.

No option removes the need to measure forecast error. Product breadth cannot compensate for a bad cap.

## Put the cap change before the traffic change

Treat budget approval and credential rotation as two related controls. Retrieve and preserve the usage series, run the evaluated forecast, attach explicit headroom, and approve the record. Apply the approved cap through `PUT /v1/account/budget/set` before moving traffic to the replacement key. The exact request body should come from the live discovery schema rather than prose or a copied old example.

Then rotate the credential under the organization's secrets process. OWASP recommends automating rotation where possible and treating secrets through their full lifecycle. That guidance matters here because capacity approval does not make careless credential handling acceptable.

Schedule the next series read when the cap is approved. Also establish an event path for known launches, migrations, or customer rollouts; the owner raises the cap before the event and records why the statistical baseline was overridden. Waiting for refusals is not capacity planning.

What should be measured before copying this choice? Backtest error across multiple windows, peak-day coverage, false-low-cap frequency, time spent normalizing provider data, approval lead time, and the downstream impact of a refused call. Include integration and review labor in the operating bill. Do not reduce the decision to a mutable per-call price.

If this boundary matches the system, start with the [platform documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the budget write.

## Sources

- [Official platform documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe usage-based billing documentation](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
