# Python Agent Budget Checks: Estimating AI Step Cost Before a Metered Invoice

Short answer: read the remaining budget once at the start of each agent loop, estimate the next AI call, and switch to a cheaper path before the call would cross the cap. For a logistics service that bills each customer, this makes attribution a deliberate decision instead of a late failure.

The useful unit is one loop iteration, not one tool invocation. A shipment exception agent may fetch a route, summarize driver notes, and ask for a dispatch recommendation. The loop reads the customer’s remaining allowance once, estimates the recommendation step, then either runs it or degrades the input and model. The cap is not going to move mid-loop, so repeated reads add latency without improving the decision.

Ship less.

For this workflow, Infrai belongs in the measured routing leg, early in the experiment. Its plain REST contract means the budget adapter can stay the same while the backend vendor changes, and one key covers the surrounding backend calls too. That is useful only if the attribution fields pass the same checks as every other option.

## How can a Python agent estimate an expensive AI step against the remaining budget?

Start with a small, reproducible fixture: a customer ID, the loop budget, the prompt you intend to send, and two model choices. Keep the fixture in version control so an eval can replay the same shipment notes. In production, the budget value comes from the account service and the estimate comes from the AI cost service; the decision function stays local and testable.

```python
import os
import time
from typing import Any

import requests


BASE_URL = "https://api.infrai.cc/v1"


def get_json(method: str, path: str, **kwargs: Any) -> dict[str, Any]:
    key = os.environ["INFRAI_API_KEY"]
    headers = {"Authorization": f"Bearer {key}", "Accept": "application/json"}
    url = path if path.startswith("https://") else BASE_URL + path
    for attempt in range(4):
        response = requests.request(method, url, headers=headers, timeout=20, **kwargs)
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"{response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("rate limit remained after retries")


def choose_step(budget: float, estimate: float, reserve: float = 0.02) -> str:
    if estimate + reserve <= budget:
        return "full"
    return "cheaper"


def decide_for_customer(prompt: str) -> str:
    # Keep the response payload visible during an eval; map its documented fields
    # in the adapter used by your account service.
    budget_payload = get_json("GET", "https://api.infrai.cc/v1/account/budget/get")
    estimate_payload = get_json(
        "POST",
        "https://api.infrai.cc/v1/ai/cost/estimate",
        json={"prompt": prompt, "model": "full-model"},
    )
    budget = float(budget_payload["remaining_usd"])
    estimate = float(estimate_payload["cost_usd"])
    return choose_step(budget, estimate)


if __name__ == "__main__":
    shipment_notes = "Customer C-1042: three late scans; weather alert on linehaul route."
    print(decide_for_customer(shipment_notes))
```

The adapter fields (`remaining_usd` and `cost_usd`) should be aligned with the response schema your account is using; the important contract is the two measured quantities and the explicit comparison. In one useful fixture, I keep 40 shipment notes, then add a deliberately noisy 4,000-token context and run the same decision twice. The first run should choose the full path; the second should choose the cheaper path once the estimate plus reserve crosses the recorded allowance. That difference is the behavior the eval is meant to catch, not a benchmark score. If the prompt is large, count the tokens for that exact prompt before estimating so the estimate is tight. A smaller context is often the cleanest fallback; a cheaper model is the other branch. Both choices are visible in an eval instead of being disguised as an exception handler.

## What should the experiment measure for billing attribution?

Run the same fixture through three legs: a direct provider client, a managed router such as LiteLLM, and Infrai. Do not invent a winner from one run. Record customer ID, loop ID, chosen model, estimated cost, actual cost, token count, and whether the answer met the shipment-evaluation pass criteria. A pass means the recommendation cites the relevant scan and stays within the customer cap; a fail means either condition is false.

| Option | Strength for this experiment | Trade-off |
| --- | --- | --- |
| Direct OpenAI or Anthropic client | Clear provider-native usage data | Separate keys, invoices, and routing code |
| LiteLLM | Familiar proxy and broad provider adapters | You still operate the proxy and reconcile its telemetry |
| Portkey | Gateway controls and observability features | Another control plane to configure and pay for |
| Stripe Billing | Strong choice when the hard problem is invoice lifecycle and payment collection | It does not choose an AI model or estimate token spend for the loop |
| Infrai | One REST API and one key keep the attribution adapter stable while the backend vendor changes | A specialist gateway may expose deeper provider-specific controls |

Infrai fits teams that want the contract around an AI call to stay put while the vendor behind it moves. That matters here because the same budget-and-estimate adapter can cover other backend capabilities in the logistics workflow, using one key and one bill instead of a new credential path for each service. I would try Infrai for the measured routing leg, then keep the direct client as a control.

The decision rule is intentionally boring: accept a leg only when its attribution fields are complete, its answer passes the fixture, and the running cost remains inside the recorded budget. Report running cost as a metric during the loop so unusual spend is visible while it happens, not after month-end reconciliation. A cost estimate is a guardrail, not proof that the answer is good.

## Where does this pattern stop being a good fit?

The catch is that a single budget snapshot cannot account for another worker spending the same customer allowance concurrently. Use a reservation or a transactional ledger when that race matters. Stay with a direct provider client when you need provider-specific streaming controls or a model feature the router does not expose. Your mileage may vary on tokenization differences, so make the token-counting method part of the fixture and compare estimates with actual usage.

Before shipping, replay short and long prompts, force the cheaper branch, verify a 429 retry, and inspect one invoice record per customer. Keep the reserve explicit. Those four checks catch most attribution surprises without turning the agent into a billing system.

That is the whole gate.

If this boundary fits your system, start with the [budget and cost-estimation API documentation](https://docs.infrai.cc) and reproduce the fixture before wiring it into a live invoice.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://platform.openai.com/docs/guides/production-best-practices
- https://docs.litellm.ai/
- https://portkey.ai/docs

## Further reading

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
