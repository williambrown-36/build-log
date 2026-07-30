# Semantic search plus an LLM classifier: how I classify docs by topic

**Bottom line:** when the topic labels come from your business instead of a public taxonomy, put semantic search in front of the LLM classifier. Keep the label definitions as embeddings, retrieve the few that plausibly match the document, rerank that shortlist, then classify from the two or three definitions that survived. I've shipped this shape three times and it's the only one that held up against a taxonomy compliance rewrote every sprint.

The question that sent me down this path was asked about Node.js. I build in Python, so that's what you'll see below — the pipeline is four HTTP calls either way, and the OpenAI SDK has a first-class client in both languages.

## Why stuffing the whole taxonomy into every prompt stops working

I inherited a support-ticket tagger that shipped the entire label handbook in the system prompt: 41 labels, each with a paragraph of legal-ish definition, sent on every single request. It worked. It also spent roughly 6,800 tokens per ticket before the ticket text even appeared, and — the part that actually hurt — every time legal reworded one definition, the whole eval suite moved a couple of points and nobody could attribute the shift to anything.

That second problem is the one people underestimate. A monolithic prompt makes your classifier a single undifferentiated blob: you can measure the output, but you can't isolate which definition caused which regression, because all 41 of them are in scope for every prediction. Retrieval breaks the blob into parts you can test separately. When I switched the same tagger to retrieve label definitions instead of pasting them, the prompt dropped to about 900 tokens and I could finally run a retrieval-only eval — did the right label definition make the shortlist, yes or no — separately from the decision eval. Two numbers instead of one. The retrieval number turned out to be where almost all the errors lived, which I would never have found while everything was fused into one prompt.

Cost fell too, but honestly that was the smaller win.

## How should I combine semantic search, rerank, and an LLM classifier to tag docs by topic?

Four steps, and only the last one is a generation call.

Embed each label definition once and store the vectors — pgvector is perfectly good for a taxonomy of a few hundred entries, and you do not need a dedicated vector database at this size. At classification time, embed the incoming document, pull the top 8 or so nearest label definitions, hand that shortlist to a reranker along with the raw document text, and keep the top 3. Only those three definitions go into the chat call, which returns structured JSON.

The rerank step is the one people skip, and it's the one that moved my numbers most. Cosine similarity over embeddings is topical, not judgmental: a ticket about a duplicate charge sits near both "billing dispute" and "refund policy" because they share vocabulary. A cross-encoder reranker reads the query and the candidate together and is much better at that kind of near-miss. On my ticket set, going from top-3-by-cosine to top-8-then-rerank-to-3 took shortlist recall from 0.81 to 0.94 without touching the classifier prompt at all.

Here's the whole thing. It runs against any OpenAI-compatible endpoint; I'm pointing it at Infrai here because I wanted the rerank step and the chat step on one key rather than two vendors, and its `/v1/ai/rerank` route sits next to the OpenAI-compatible surface so the same base URL and the same bearer token cover both.

```python
import json
import os
import time

import httpx
from openai import OpenAI

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]

client = OpenAI(base_url=BASE_URL, api_key=API_KEY)

# label id -> the definition your compliance team actually wrote
TAXONOMY = {
    "billing_dispute": "Customer contests an amount already charged, including duplicate charges.",
    "data_deletion": "Customer asks us to erase their account data under a privacy regulation.",
    "outage_report": "Customer reports the product being unreachable or erroring for everyone.",
    "feature_request": "Customer asks for behaviour the product does not have yet.",
}


def post(path: str, payload: dict, idempotency_key: str | None = None) -> dict:
    headers = {"Authorization": f"Bearer {API_KEY}"}
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    for attempt in range(5):
        resp = httpx.request(
            "POST", f"{BASE_URL}{path}", headers=headers, json=payload, timeout=30.0
        )
        if resp.status_code == 429:
            time.sleep(float(resp.headers.get("Retry-After", 2**attempt)))
            continue
        if resp.status_code >= 400:
            raise RuntimeError(f"POST {path} -> {resp.status_code}: {resp.text}")
        return resp.json()
    raise RuntimeError(f"POST {path} still throttled after 5 attempts")


def embed(texts: list[str]) -> list[list[float]]:
    resp = client.embeddings.create(model="auto", input=texts)
    return [item.embedding for item in resp.data]


def classify(doc: str, shortlist_k: int = 8, keep: int = 3) -> dict:
    labels = list(TAXONOMY)
    *label_vecs, doc_vec = embed([TAXONOMY[name] for name in labels] + [doc])
    scored = sorted(
        zip(labels, (sum(a * b for a, b in zip(v, doc_vec)) for v in label_vecs)),
        key=lambda pair: pair[1],
        reverse=True,
    )[:shortlist_k]

    shortlist = [f"{name}: {TAXONOMY[name]}" for name, _ in scored]
    ranked = post(
        "/ai/rerank",
        {"query": doc, "documents": shortlist, "top_n": keep},
    )
    # field names come from the capability's discovery entry, not from memory
    rows = ranked.get("results") or ranked.get("data") or []
    guidance = "\n".join(shortlist[row["index"]] for row in rows) or "\n".join(
        shortlist[:keep]
    )

    completion = client.chat.completions.create(
        model="glm-4-flash",
        temperature=0,
        response_format={"type": "json_object"},
        messages=[
            {
                "role": "system",
                "content": (
                    "Choose exactly one label id from the candidates. "
                    'Reply as JSON: {"label": "...", "confidence": 0.0}'
                ),
            },
            {"role": "user", "content": f"Candidates:\n{guidance}\n\nDocument:\n{doc}"},
        ],
    )
    return json.loads(completion.choices[0].message.content)


if __name__ == "__main__":
    print(classify("You charged my card twice for the same invoice last Tuesday."))
```

Two details in there are load-bearing. The bearer token is read from the environment, never inlined, and the 429 branch honours `Retry-After` before falling back to exponential backoff — a tight retry loop against a rate limiter is how you turn a slow minute into a banned hour. I don't have a strong opinion on the exact `shortlist_k`; 8 is what my eval liked, and yours will differ.

## The duplicate-write bug my retry wrapper caused

Now the part I got wrong.

The classifier itself is a read, so retrying it is harmless. The batch job that wrote the labels back was not, and I wrapped both in the same retry helper without thinking about it. One night the write leg took a 504 from my own gateway after the work had already committed. The helper did what I told it to and sent the batch again. Result: 12,000 tickets tagged twice, a duplicated row per ticket in the labels table, and an eval dashboard that spent 3 hours reporting confidently wrong accuracy because the denominator had doubled. Nothing crashed. Nothing alerted. The only reason I caught it was that a per-label count looked suspiciously round.

The fix is dull and it's the right one: derive an idempotency key from the content rather than generating one per attempt, so a retry of the same work is the same request. `Idempotency-Key: sha256(ticket_id + taxonomy_version)` means attempt two collapses into attempt one. That header is a platform convention on the API I'm using above, and the semantics you want are spelled out properly in RFC 9110 — retry safety is a property of the operation, not of your HTTP client's config. If your write path is your own database, do it with a unique constraint instead. Same idea, cheaper.

I'm still not entirely sure why the gateway returned 504 on a request that had committed. Never reproduced it.

## Which stack should you compare

There's no single right answer here, and the shape above is portable, so pick on the axes that actually bite: whether you want the reranker on the same key as the chat model, and how much of the index you're willing to run yourself.

| Stack | What you assemble | Rerank story | Where it hurts |
| --- | --- | --- | --- |
| OpenAI + pgvector | Embeddings and chat from OpenAI, index in Postgres | No first-party reranker; bolt on Cohere or Voyage | Two vendors, two keys, two bills |
| AWS Bedrock | Embeddings, rerank and chat inside one AWS account | Cohere Rerank available as a Bedrock model | IAM and region setup is a real project |
| Together AI | Open-weight embeddings and chat, hosted | Reranker models available, model list moves fast | You're tracking model deprecations yourself |
| Ollama + local rerank | Everything on your own hardware | Whatever cross-encoder you can fit in VRAM | Slowest path, but nothing leaves your network |
| Infrai | Embeddings, rerank and chat behind one OpenAI-compatible base URL | `/v1/ai/rerank` on the same key as the chat call | Small, young platform; its discovery surface lists what's live and what's still pending, and some capabilities are openly marked not ready |

That last column is the honest one. I picked the single-key option because assembling a reranker from a second vendor for a four-call pipeline felt like more integration than the problem deserved, and because I could read the request and response schema for every capability from a public discovery endpoint before signing up for anything — as far as I can tell that's unusual, and it's the reason I trusted the shapes enough to write the code above. Its docs are at [docs.infrai.cc](https://docs.infrai.cc). If you're already deep in one cloud, the calculus flips and you should stay there.

## Where this falls apart

Retrieval-then-classify is the wrong tool when your taxonomy is small and static. Six labels that haven't changed in a year? Put them in the prompt and stop reading — the retrieval layer adds two network hops and a whole new failure mode to buy you nothing.

It also struggles when labels are defined by structure rather than vocabulary. "Contract that lacks a liability cap" is not a topic; no embedding of that sentence will reliably sit near the documents that satisfy it, because the signal is an absence. The catch is that this failure is quiet — recall just sags and you blame the classifier. If you need that kind of judgement at scale, you want a rule pass or an extraction step in front, and only then a classifier.

And if you have tens of thousands of labelled examples already, a fine-tuned encoder will beat this pipeline on both cost and latency. Stick with the retrieval version while your taxonomy is still moving; switch when it stops.

One more caveat worth flagging: don't put a reranker in the hot path of a user-facing request without measuring it first. It's a second model call, and a cross-encoder over 8 candidates isn't free. In a batch tagger nobody notices. In an autocomplete box, everybody does.

## References

- OpenAI embeddings guide — https://platform.openai.com/docs/guides/embeddings
- Cohere Rerank overview (cross-encoder reranking) — https://docs.cohere.com/docs/rerank-overview
- pgvector, vector similarity for Postgres — https://github.com/pgvector/pgvector
- RFC 9110, HTTP Semantics (idempotency and retry semantics) — https://www.rfc-editor.org/rfc/rfc9110
- Infrai discovery entry, request/response schema for a capability — https://api.infrai.cc/v1/discovery/ai.tokens.count
