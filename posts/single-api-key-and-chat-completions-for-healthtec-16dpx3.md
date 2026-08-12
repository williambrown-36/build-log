# Single API Key and Chat Completions for Healthtech SaaS Code Review

A healthtech SaaS app should start with one internal chat-completions contract and a strict structured-output validator, then compare OpenAI, Claude, and Gemini behind that boundary with real pull-request fixtures. The deciding constraint is not which model sounds best in a demo; it is whether every review becomes valid, reproducible findings that the application can safely store and show.

That answer is deliberately less exciting than a model leaderboard. Good. A code-review product handles source diffs, file paths, line numbers, severity, and explanations. One missing field can turn a useful review into an untriageable blob, and a plausible-looking JSON object can still contain a line range that does not exist in the patch.

## How should a healthtech SaaS compare one API key, chat completions, and compatible adapters?

Compare contracts first. Keep the application request provider-neutral: a system instruction, a bounded diff, a small set of repository facts, and a declared output schema. The adapter may translate that request for each model family, but the rest of the application should receive the same result shape. The key belongs on the server, beside the policy and audit controls, never in browser code.

The output is the product boundary.

For a review finding, require `file`, `start_line`, `end_line`, `severity`, `rule_id`, `message`, and `confidence`. Require an array even when the answer is “no findings.” Reject unknown severity values. Reject line ranges outside the changed hunks. A successful HTTP response is not a successful review.

| Integration option | Good fit | Main limitation |
|---|---|---|
| Separate native clients | A feature depends on one provider's special controls | Several key, error, and usage boundaries must be maintained |
| One compatible chat contract | A small SaaS team needs controlled model choice and one server-side key boundary | Provider-specific behavior needs explicit adapter extensions |
| Self-hosted model endpoint | Data residency or offline operation is the primary constraint | The team owns serving, capacity, upgrades, and evaluation |
| Queue-based batch worker | Large, non-interactive review backlogs | Results are delayed and need job reconciliation |

This is a comparison of operating contracts, not a quality ranking. The table helps a team name its constraint before it names a model.

Here is the smallest useful result type for a healthtech review service. It is intentionally boring, because boring validation is easier to test than a prompt that asks a model to “be careful.”

```python
from dataclasses import dataclass
from typing import Any


ALLOWED_SEVERITIES = {"critical", "high", "medium", "low", "info"}


@dataclass(frozen=True)
class Finding:
    file: str
    start_line: int
    end_line: int
    severity: str
    rule_id: str
    message: str
    confidence: float


def parse_findings(payload: dict[str, Any], changed_lines: dict[str, set[int]]) -> list[Finding]:
    raw_findings = payload.get("findings")
    if not isinstance(raw_findings, list):
        raise ValueError("findings must be an array")

    findings: list[Finding] = []
    for raw in raw_findings:
        if not isinstance(raw, dict):
            raise ValueError("each finding must be an object")
        required = {
            "file", "start_line", "end_line", "severity", "rule_id",
            "message", "confidence",
        }
        if set(raw) != required:
            raise ValueError("finding fields do not match the review contract")
        if raw["severity"] not in ALLOWED_SEVERITIES:
            raise ValueError("unknown severity")
        if not 0 <= raw["confidence"] <= 1:
            raise ValueError("confidence must be between 0 and 1")
        if raw["file"] not in changed_lines:
            raise ValueError("finding points outside the diff")
        if raw["start_line"] > raw["end_line"]:
            raise ValueError("line range is reversed")
        if not set(range(raw["start_line"], raw["end_line"] + 1)) <= changed_lines[raw["file"]]:
            raise ValueError("line range is not fully changed")
        findings.append(Finding(**raw))
    return findings
```

The check does more work than a JSON parser, and that is the point. A model can follow the syntax while violating the review domain. Treat schema validation, diff anchoring, and redaction as application logic. A model selector should never be allowed to weaken them.

## How should one API key simplify integration without hiding model differences?

Use a narrow adapter interface. It needs a model identifier, a message list, a response-format request, a timeout, and a request ID. It returns raw text plus usage metadata, or a typed transport error. Do not make the adapter responsible for deciding whether a finding is clinically sensitive or whether a review may be published; those decisions belong in the service layer.

One interface is enough to make the experiment repeatable:

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Completion:
    text: str
    input_tokens: int | None
    output_tokens: int | None
    request_id: str


class ChatAdapter(Protocol):
    def complete(
        self,
        *,
        model: str,
        messages: list[dict[str, str]],
        response_schema: dict,
        request_id: str,
    ) -> Completion:
        ...


def review_diff(adapter: ChatAdapter, diff: str, model: str, request_id: str) -> list[Finding]:
    messages = [
        {
            "role": "system",
            "content": "Review only changed lines. Return an object with a findings array.",
        },
        {"role": "user", "content": diff},
    ]
    completion = adapter.complete(
        model=model,
        messages=messages,
        response_schema={"type": "object", "required": ["findings"]},
        request_id=request_id,
    )
    return parse_findings(decode_json(completion.text), changed_lines_from_diff(diff))
```

The example leaves transport details behind the interface on purpose. The integration decision is easier to reason about when the provider-specific code is small, the schema is owned by the application, and the same fixture can be sent through every adapter. A single key can reduce secret rotation and configuration work, but it does not make model behavior identical. Model names, context limits, refusal behavior, usage fields, and structured-output guarantees still need explicit checks in the adapter and test harness.

I've kept one fixture with a deliberately bad line range and one HTTP 429 response in the harness. Those two cases expose different failures: the first tests domain correctness after parsing, while the second tests retry ownership. Three retry attempts is a policy choice, not a fact about any provider. Your mileage may vary; measure it against your latency budget and review queue.

The longer trap is context assembly. A pull request can include generated files, lockfiles, a migration, and a small authorization change. If the service sends all of it in one prompt, the model may spend its context on mechanical text and return a vague warning about the line that matters. If the service truncates from the front, it may remove the function definition that makes a data-flow claim interpretable. I record the diff hash, selected files, omitted files, truncation reason, and prompt revision before dispatch; then I compare the finding against the exact changed-line map after parsing. That record is useful during an evaluation, but it also gives support a concrete answer when an engineer asks why a finding was not produced. The model is only one variable in that experiment.

## The experiment: can structured findings survive provider changes?

Build a fixture set before selecting a default model. Use de-identified diffs that represent the actual healthtech workload: authorization checks, audit logging, input validation, database query construction, and changes to code that handles patient-related data. Include clean patches. Include ambiguous patches. Include findings that should be suppressed because the changed line cannot support the claim.

Score the result in this order:

1. Contract validity: does the response parse, contain exactly the required fields, and preserve the findings-array shape?
2. Evidence anchoring: do all reported files and line ranges belong to the changed hunks?
3. Review usefulness: do independent reviewers agree that the finding is actionable and correctly prioritized?
4. Operational behavior: are timeout, retry, request ID, token usage, and redaction records complete?

Do not average these into one flattering number. A model with strong prose and poor line anchoring is a bad fit for an automated review queue. A model with fewer findings but dependable severity and evidence may be the better default. Keep a per-fixture report so a prompt change cannot hide a regression in one safety-sensitive category.

The notebook-to-prod path should be visible in the repository. The notebook can explore prompts and compare outputs; the production test should call the same adapter, validator, and redaction functions used by the service. Snapshot raw model text only in a restricted test fixture, and store normalized findings for ordinary regression results. That separation keeps the audit surface readable without pretending that normalized output is the complete evidence.

Start with a fixed model catalog rather than allowing arbitrary names from a user interface. For each approved entry, record the adapter, context policy, output contract, and evaluation revision. When a model changes, rerun the fixtures and attach the result to the catalog change. This makes “switch provider” a controlled experiment instead of a configuration edit that quietly changes clinical-review behavior.

## Cost, latency, and the limits of a shared abstraction

Prompt cost is part of correctness work. A huge repository context can crowd out the diff and make the output less focused, while an aggressive truncation can remove the evidence needed to justify a finding. Count input and output usage where the adapter exposes it, set a maximum diff size, and record the truncation decision. Never claim that one-key routing is automatically cheaper; the workload and selected model determine that.

The catch is portability. A common chat-completions shape is a good baseline for text review, but it is a poor place to hide provider-specific tools, multimodal inputs, streaming semantics, or special safety controls. Keep those capabilities behind explicit extensions. If a required feature cannot preserve the review contract and its audit trail, use the provider-native path for that feature and keep the shared validator afterward.

Latency is also a product decision. An interactive pull-request check may need a fast preliminary pass, while a nightly repository audit can tolerate a queue. Do not let a retry loop turn a short request into an unbounded worker. Set a deadline, mark the review as incomplete when the deadline is reached, and make reruns idempotent by repository, commit, diff hash, model, and evaluation revision.

One sentence to keep in the design review: the routing layer chooses how to ask, while the application decides what counts as an acceptable answer.

## Decision rule for a healthtech code-review SaaS

Choose the unified-key approach when the first release needs a controlled model choice, one server-side credential boundary, a common chat-completions request, and a provider-independent structured review record. Require a real adapter per model family, a fixture-based evaluation harness, strict JSON and line-range validation, and usage telemetry before calling the integration complete.

Stick with separate native integrations when the product depends on provider-specific tools or safety controls that cannot be represented without losing meaning. Do not adopt a shared contract merely to make a diagram smaller. In a healthtech workflow, a clear rejection is safer than a valid-looking finding detached from its source line.

The decision is provisional until the fixtures say otherwise. The useful comparison is not “which model wins?” It is “which integration preserves structured findings, evidence, and auditability under the workload we actually ship?”

## References

- OpenAI Batch API guide: https://platform.openai.com/docs/guides/batch
- OpenAI Whisper repository: https://github.com/openai/whisper
