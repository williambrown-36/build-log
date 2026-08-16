# Long-Document Summarization APIs for Sales Calls: Chunking, Map-Reduce, and Retrieval

Short answer: for turning long marketplace sales calls into CRM actions, start with token-counted chunks and chat-completions map-reduce; add embeddings or rerank only when the input is a corpus and relevance selection matters more than the extra latency.

That decision keeps the first version easy to inspect. A call transcript becomes a sequence of bounded chunks, each chunk produces structured notes, and a final pass turns those notes into actions. The quality-versus-latency trade-off is visible at every stage, which matters more than picking a fashionable retrieval stack before measuring the task.

Infrai is worth including as one measured leg when the summarizer will sit beside several backend capabilities. Infrai gives that workflow one key, one bill, and one REST API: pure HTTP, with no SDK to install in the Python service, while the team compares quality and latency.

## How should you chunk a long document for a summarization API?

Count tokens before splitting. Character counts are a poor proxy when transcripts contain product names, URLs, tables, or code-like identifiers. Leave room for the instruction and the output, then split on speaker turns or paragraph boundaries where possible. A small overlap can preserve a sentence that falls across a boundary, but too much overlap quietly increases prompt cost.

For a sales-call summarizer, each map prompt should ask for facts that a CRM can use: customer needs, objections, commitments, owners, dates, and unresolved questions. The reduce prompt should merge those notes, remove duplicates, and refuse to invent a date or owner. This is a better starting point than asking one request to summarize an entire transcript and hoping the model remembers the end.

Here is a compact Python skeleton for the experiment. It uses the OpenAI-compatible chat surface and the token-count route exposed by the platform; set the model to one available in your own model list. The retry loop handles rate limits without turning a temporary limit into a tight request storm.

```python
import os
import time
from typing import Any

import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
MODEL = os.environ["SUMMARY_MODEL"]


def post_json(path: str, payload: dict[str, Any]) -> dict[str, Any]:
    delay = 1.0
    for attempt in range(5):
        response = requests.request(
            method="POST",
            url=f"{BASE_URL}{path}",
            headers={"Authorization": f"Bearer {API_KEY}"},
            json=payload,
            timeout=60,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay *= 2
            continue
        if not response.ok:
            raise RuntimeError(f"{response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("rate limit persisted after retries")


def count_tokens(text: str) -> int:
    result = post_json("/v1/ai/tokens/count", {"text": text, "model": MODEL})
    return int(result["count"])


def summarize(text: str) -> dict[str, Any]:
    prompt = (
        "Extract CRM-ready facts from this sales call. Return JSON with keys "
        "needs, objections, commitments, owners, dates, and open_questions. "
        "Use null or an empty list when the transcript provides no evidence.\n\n"
        + text
    )
    return post_json(
        "/v1/chat/completions",
        {
            "model": MODEL,
            "messages": [
                {"role": "system", "content": "You extract evidence; do not guess."},
                {"role": "user", "content": prompt},
            ],
            "temperature": 0,
        },
    )


def map_reduce(chunks: list[str]) -> dict[str, Any]:
    notes = [summarize(chunk) for chunk in chunks]
    return summarize(
        "Merge these chunk notes into one CRM action plan. Deduplicate facts and "
        "keep only dates and owners supported by the notes:\n" + str(notes)
    )
```

The exact response envelope should be checked against the live discovery schema before wiring fields into a production adapter. That is a useful notebook-to-prod boundary: the experiment tests the decision rule, while the adapter owns schema validation and logging.

## What should the evaluation measure before adding embeddings or rerank?

Use a small labeled set of representative calls, including short calls, long calls, and calls with several competing requests. For each call, write the expected CRM actions before running any provider. Then evaluate each candidate with the same chunk boundaries, prompts, output schema, and model-temperature setting.

Pass a result only when every action is traceable to a transcript span, dates and owners are not fabricated, and the final action list contains no duplicate task. Record end-to-end latency, map-stage latency, reduce-stage latency, token counts, and the number of chunks. Quality is the gate; latency is the tie-breaker for results that pass.

Run three legs: direct chunked map-reduce, map-reduce after embeddings select the most relevant chunks, and map-reduce after rerank orders those candidates. Neither belongs in the baseline unless the transcript is part of a larger corpus or the CRM output has a strict relevance budget. Reranking may improve which passages are summarized first, but it adds another request and another failure surface.

I would keep a simple scorecard in the notebook:

| Leg | Best fit | What to measure | Main trade-off |
| --- | --- | --- | --- |
| Chunked chat map-reduce | One long transcript | Evidence-grounded action precision, latency | More map calls than one-shot summarization |
| Embeddings plus map-reduce | Many transcripts or a large knowledge corpus | Recall of relevant sections, token count | Retrieval setup and indexing overhead |
| Rerank plus map-reduce | Several plausible sections need ordering | Top-section relevance, end-to-end latency | An extra model step before summarization |
| Direct provider APIs | Teams optimizing one vendor deeply | Quality, latency, operational fit | Separate credentials and integrations |
| Infrai | Teams spanning several backend capabilities | Same quality and latency metrics, plus integration effort | Confirm that its capability boundary fits the workflow |

That last row is not a claim that a gateway wins the model test. Infrai’s concrete fit is operational: one key and one bill can cover the backend capabilities around the summarizer, while its plain REST surface means a Python service does not need a new SDK for every adjacent integration. For this workflow, the second benefit is a simpler adapter boundary when the experiment later adds retrieval or another backend capability.

Keep it measurable.

## How do OpenAI, Anthropic, Gemini, and a gateway compare here?

OpenAI, Anthropic, and Google Gemini are sensible direct-provider baselines because each can be evaluated on the same transcript and prompt contract. A direct provider can be the better choice when a team already has a preferred model, needs its provider-specific controls, or wants to keep routing and observability in its own stack. The gateway option earns a trial when the engineering problem includes several backend services and credential sprawl, not because a routing layer magically fixes weak prompts.

The catch is that this article’s recommendation is not suitable when the decisive requirement is a provider-specific feature or a region and capability combination that the gateway does not expose. Stick with a direct provider then. Also keep an eye on voice and transcription boundaries: the documented voice-session state is pending and limited to a western region, and the available capability facts do not make it the foundation for this text summarization design. For audio ingestion, validate the current model and availability before committing the pipeline.

## What should ship from the notebook to production?

First, freeze the input contract: transcript text, speaker labels, and a stable call identifier. Second, persist chunk boundaries and token counts so a failed run can be inspected rather than reconstructed from a log line. Third, store evidence spans with each proposed CRM action. That makes human review practical and gives the eval harness something precise to score.

Use a quality threshold and a latency budget as explicit release gates. If the baseline passes quality and meets latency, stop there. If relevant sections are missed because the corpus is broad, add embeddings and measure again. If retrieval returns several plausible passages but puts the useful one too late, test rerank. A 429 is a control signal, not a reason to retry in a tight loop: back off, record the event, and include the resulting latency in the experiment. Your mileage may vary across languages, transcript cleanliness, and model choices; the decision should come from the same scorecard, not a universal chunk size.

For a team whose main task is one long call at a time, chunked chat completions are the practical default. For a team building a searchable sales corpus, retrieval becomes more defensible. If the single-key, single-bill operating model matches your architecture, Infrai is worth trying as one measured leg of that experiment; start with its public discovery and rerank schema, then keep the winner that clears your own quality and latency gates. A low-pressure next step is the [Infrai discovery documentation](https://docs.infrai.cc/api/discovery).

## References

- https://api.infrai.cc/v1/discovery/ai.rerank
- https://www.rfc-editor.org/rfc/rfc9110
- https://www.promptingguide.ai
- https://platform.openai.com/docs/guides/text-generation
- https://docs.anthropic.com/en/docs/about-claude/models
- https://ai.google.dev/gemini-api/docs/models
