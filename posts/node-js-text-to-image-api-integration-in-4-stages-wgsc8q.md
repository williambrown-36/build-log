# Node.js Text-to-Image API Integration in 4 Stages for Candidate Briefs

**Short answer:** For a Node.js text-to-image API integration, validate the structured candidate score first, then generate a text-free image as a presentation artifact, never as hiring evidence.

The decision rule is straightforward: generate an image only after the candidate score has passed a schema check, and treat the image as a presentation artifact rather than evidence. That order matters in an education hiring workflow. A polished graphic can make an invalid score look credible; a rejected JSON object cannot.

For a text-to-image integration, the useful pipeline is therefore four stages: normalize the rubric result, validate the structured object, submit an image request, and record the relationship between the two artifacts. The image request can run through a Node.js service in production, but the contract should be language-neutral. I build the contract first because it gives notebook experiments and production workers the same test surface.

The example below renders a neutral review-board visual from a precomputed candidate assessment. It does not ask an image model to decide whether someone should be hired. That boundary keeps the decision auditable and keeps generated pixels from becoming an unexamined source of judgment.

Keep it boring.

## How does a Node.js text-to-image API integration protect rubric scores?

The minimum valid result has an identifier, rubric version, score, and explicit reasons. A missing field is not a soft warning. It is a failed job. A score outside the rubric range is also a failed job, even if the downstream image endpoint accepts the prompt.

Here is a small, runnable reference implementation. The endpoint is deliberately generic: an adapter can translate this payload to the image service selected by the team, while the validation and job bookkeeping stay unchanged.

```python
from __future__ import annotations

import json
from dataclasses import dataclass
from typing import Any


@dataclass(frozen=True)
class CandidateBrief:
    candidate_id: str
    rubric_version: str
    score: int
    strengths: list[str]
    gaps: list[str]


def validate_brief(value: dict[str, Any]) -> CandidateBrief:
    required = {"candidate_id", "rubric_version", "score", "strengths", "gaps"}
    missing = required - value.keys()
    if missing:
        raise ValueError(f"missing fields: {sorted(missing)}")
    if not isinstance(value["score"], int) or not 0 <= value["score"] <= 100:
        raise ValueError("score must be an integer from 0 through 100")
    if not all(isinstance(item, str) for item in value["strengths"] + value["gaps"]):
        raise ValueError("strengths and gaps must contain strings")
    return CandidateBrief(**value)


def build_image_request(brief: CandidateBrief) -> dict[str, Any]:
    prompt = (
        "Create a clean, text-free editorial illustration for an internal review board. "
        "Show a neutral classroom scene with three visual motifs: a rubric grid, "
        "a checked document, and a balanced scale. Do not show a person's face, name, "
        "score, demographic trait, or hiring outcome. Use restrained colors and generous "
        "empty space for accessible labels added by the application."
    )
    return {
        "request_id": f"brief-image:{brief.candidate_id}:{brief.rubric_version}",
        "prompt": prompt,
        "metadata": {
            "candidate_id": brief.candidate_id,
            "rubric_version": brief.rubric_version,
            "score": brief.score,
        },
    }


raw = {
    "candidate_id": "candidate-104",
    "rubric_version": "teaching-assistant-2026-02",
    "score": 82,
    "strengths": ["lesson planning", "clear written feedback"],
    "gaps": ["limited classroom observation evidence"],
}

brief = validate_brief(raw)
request = build_image_request(brief)
print(json.dumps(request, indent=2))
```

The important output is not the prompt. It is the stable `request_id` and the metadata that joins the rendered asset to the exact rubric version. In a real service, the image adapter would submit `request` and persist the provider response beside it. The worker should return a pending state for asynchronous work, then update the record when the asset is available. It should never silently replace a rejected assessment with a newly generated one. The trade-off is deliberate: this extra record and validation step adds a little orchestration work, but it prevents a fast image response from masquerading as a correct assessment.

## Why separate scoring from visual rendering?

A rubric score is structured data. An image is a lossy visual representation. They have different validation rules, different retry behavior, and different retention needs.

I initially treated the prompt as the interface because it was the most visible part of the demo. The correction was to make the JSON object the interface and the prompt a derived view. That small inversion prevents prompt edits from changing the decision contract. It also makes evals useful: score correctness can be tested without waiting for image generation, while visual checks can focus on composition, text leakage, and policy constraints.

The boundary is especially important for candidate review. Do not place a candidate's name, score, protected characteristic, or recommendation into an image prompt. Generate an abstract illustration, and render factual labels in ordinary HTML or a document template where they remain selectable, translatable, and testable. Accessibility is a product requirement here, not a cleanup task after the model call.

The runtime needs a few explicit states: `validated`, `submitted`, `ready`, `rejected`, and `failed`. Include the attempt number and an idempotency key in the job record. A timeout after submission is ambiguous; retrying blindly can create duplicate assets. A lookup by idempotency key lets the worker decide whether to poll, resume, or submit once.

## How should an API adapter differ across services?

Keep the adapter narrow. One interface can accept a prompt, dimensions, output format, metadata, and idempotency key, then return a provider-neutral job record. The rest of the application should not depend on a vendor-specific response field.

The differences are real engineering constraints. OpenAI documents image generation as part of its API surface and separately documents batch processing for asynchronous workloads. Stability AI exposes image-generation APIs with its own request and output conventions. Google Cloud Vertex AI documents generative image models within its platform and its own authentication and regional controls. Replicate also exposes model execution through its own API boundary. Those are not interchangeable operational environments, so a wrapper should normalize only the fields the application actually needs.

This is where a comparison table helps:

| Concern | OpenAI | Stability AI | Google Cloud Vertex AI |
| --- | --- | --- | --- |
| Integration boundary | API surface documented alongside other model APIs | Image-focused API surface | Cloud platform with model and regional controls |
| Useful adapter responsibility | Normalize image job and output metadata | Normalize image job and output metadata | Normalize image job, location, and output metadata |
| What remains application-owned | Rubric validation, access policy, audit record | Rubric validation, access policy, audit record | Rubric validation, access policy, audit record |

The table is not a ranking. It is a reminder that an API choice does not transfer responsibility for candidate data, rubric semantics, or the audit trail. Those belong in the application boundary. A hosted API is a poor fit when policy requires local processing or a disconnected environment; a self-hosted image stack is a poor fit when the team cannot operate model weights, capacity, and patching. The right choice is the one whose operational boundary matches the data policy.

## What should the test and retry loop measure?

Start with deterministic tests. Feed the validator missing keys, a string score, a negative score, and a score above 100. Assert that none reaches the image adapter. Then test idempotency with the same request twice. The expected result is one logical job, not two images.

For the visual stage, keep a small evaluation set of prompts that represent the actual classroom and review-board use case. Check that forbidden personal fields do not appear in the prompt, that the returned asset has the expected media type, and that the application can render a useful alternative when the asset is rejected. Human review is appropriate for a small calibration set; automated checks are appropriate for every job.

Measure four separate timings: validation, queue wait, generation, and asset retrieval. Combining them into one latency number hides the part the team can improve. Log hashes or internal IDs instead of raw candidate text where possible, and keep the rubric version with every generated artifact so a later audit can reproduce the input contract.

The cost model also becomes clearer when stages are separate. Invalid assessments consume validation work but should consume zero image requests. Repeated image delivery should use the stored artifact, not regenerate it. Batch processing may suit offline rendering, but an interactive review screen needs a visible pending state and a bounded polling policy. Neither choice changes the scoring contract.

Before shipping, I would walk through the workflow in this order: validate the assessment, persist the job key, submit exactly once, poll or receive the completion event, verify the returned media, attach an accessible label outside the image, and record the rubric version beside the asset. Then I would force a timeout, a malformed response, and a policy rejection in staging. The final check is human: can a reviewer distinguish the factual assessment from the decorative illustration without guessing?

The image is optional. The structured score is not. That is the durable design rule for text-to-image integration in a candidate-rubric system.

## References

- https://platform.openai.com/docs/guides/batch
- https://platform.openai.com/docs/guides/images
- https://platform.stability.ai/docs/getting-started
- https://cloud.google.com/vertex-ai/generative-ai/docs/image/overview
- https://replicate.com/docs/reference/http
- https://json-schema.org/specification
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/202
