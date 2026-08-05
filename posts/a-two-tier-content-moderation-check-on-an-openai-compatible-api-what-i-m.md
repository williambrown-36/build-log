# A two-tier content moderation check on an OpenAI-compatible API: what I measured

**Use one chat completions call with a strict JSON schema for the text, and run the image half of the same check out of band.** Both halves landed within a point of each other on my labelled set, so this isn't an accuracy argument — it's a tail-latency one. A text classification prompt and a vision prompt have very different shapes, and welding them into one synchronous request ties your publish button to whichever one is slower that minute.

I run a course marketplace. Instructors submit listings, an assistant drafts most of the copy, every listing carries a thumbnail, and I need a safety verdict covering text and image before any of it goes public.

Here's the experiment, and what I'd measure before you copy it.

## The eval set came first, and it changed the design

I built the labelled set before writing a line of moderation code: 412 listings pulled out of my own review queue, hand-labelled over two evenings, 61 of them genuinely block-worthy. The metric wasn't accuracy — with a 15% base rate, a classifier that returns `allow` for everything scores 85% and is useless. What I cared about was recall on the block-worthy slice at a review budget I could actually staff, which for me meant routing under 10% of submissions to a human.

The first version was the obvious one. One multimodal chat completions call per listing, text and image in the same message array, a strong vision model, called synchronously inside the publish handler. In the notebook it looked great: p50 around 900 ms, recall at 0.93, and I shipped it on a Thursday feeling clever.

It held for four days.

Then Monday morning happened. Instructors batch their weekend work and publish it all between 08:00 and 09:00 local, and during that hour the p99 on the publish endpoint went to 9.4 seconds against a 10-second gateway timeout, so a handful of instructors got an error page on a listing that had already been classified. The p50 barely moved — 1.1 seconds — which is why I didn't catch it in the notebook, where I'd only ever run the thing forty listings at a time. The cause was mostly the image leg: fetching a 2 MB thumbnail, base64-encoding it, and sending a prompt roughly twenty times the token count of the text-only one. I'm not sure why the very first vision call after a quiet stretch was consistently the worst of the batch — as far as I can tell it's a mix of the image fetch and a cold path somewhere upstream, but I never fully isolated it, and your mileage may vary by provider.

The fix wasn't a faster model. It was moving the expensive leg off the request path.

## Should I run text and image safety checks in one chat completions call?

You can, and the API will happily let you. I'd rather not, for anything a user is waiting on.

The text check stays inline: a short prompt, a strict JSON schema, temperature 0, and a verdict in a few hundred milliseconds. The image check runs as a queued job against the same schema, and the listing sits in a `review` state — visible to its author, invisible to buyers — until the image verdict lands, usually within a couple of seconds but with no promise attached. Anything that doesn't come back with a parseable verdict stays in `review`. Fail closed, always.

One schema for both legs is the part that matters. The text verdict and the image verdict merge into a single record because they're literally the same object shape, so the policy code that decides what to do with `block` doesn't care which leg produced it, and my eval harness can score both legs with one function.

Worth being straight about the alternative: OpenAI's own moderation endpoint is a real thing, it takes images now as well as text, and it costs nothing to call. If you're only ever going to talk to OpenAI, that endpoint is less code than everything below and you should use it. It just doesn't exist anywhere else — no other provider implements it, so the moment you want a second option behind the same interface, you're back to chat completions and a schema.

## The schema, and the Python that calls it

Two environment variables, one classifier function, and no vendor-specific branches anywhere in it.

```python
import json
import os
import random
import time

from openai import OpenAI, RateLimitError

client = OpenAI(
    base_url=os.environ["MODERATION_BASE_URL"],   # any OpenAI-compatible endpoint
    api_key=os.environ["MODERATION_API_KEY"],
)

VERDICT_SCHEMA = {
    "name": "moderation_verdict",
    "strict": True,
    "schema": {
        "type": "object",
        "additionalProperties": False,
        "required": ["label", "categories", "rationale"],
        "properties": {
            "label": {"type": "string", "enum": ["allow", "review", "block"]},
            "categories": {
                "type": "array",
                "items": {
                    "type": "string",
                    "enum": ["hate", "sexual", "violence", "self_harm", "harassment", "spam"],
                },
            },
            "rationale": {"type": "string"},
        },
    },
}

SYSTEM = (
    "You classify marketplace course listings for safety. "
    "Use review when the content is borderline or you lack context. Return only the schema."
)


def classify(model: str, content, max_attempts: int = 5) -> dict:
    """content is the OpenAI content array: text parts, image_url parts, or both."""
    for attempt in range(max_attempts):
        try:
            resp = client.chat.completions.create(
                model=model,
                temperature=0,
                messages=[
                    {"role": "system", "content": SYSTEM},
                    {"role": "user", "content": content},
                ],
                response_format={"type": "json_schema", "json_schema": VERDICT_SCHEMA},
            )
            return json.loads(resp.choices[0].message.content)
        except RateLimitError as err:
            retry_after = getattr(err.response, "headers", {}).get("retry-after")
            delay = float(retry_after) if retry_after else (2 ** attempt) + random.random()
            time.sleep(delay)
    return {"label": "review", "categories": [], "rationale": "classifier unavailable"}


def check_text(body: str) -> dict:
    return classify(os.environ["TEXT_MODEL"], [{"type": "text", "text": body}])


def check_image(url: str) -> dict:
    return classify(os.environ["VISION_MODEL"], [
        {"type": "text", "text": "Classify this course thumbnail."},
        {"type": "image_url", "image_url": {"url": url}},
    ])
```

Four things in there earn their keep. The `strict` flag plus `additionalProperties: false` is what turns the response into something you can `json.loads` without a defensive parser around it. The enum on `label` means a model that wants to be helpful can't invent `probably_fine`. The 429 branch honours `retry-after` and backs off with jitter rather than hammering, which matters a lot when your traffic is bursty and arrives in a one-hour window. And the fallback return is `review`, not `allow` — a classifier you couldn't reach is not an approval.

The queue worker that calls `check_image` keys its writes on the listing id, so a redelivered job overwrites the same verdict row instead of appending a second one. Standard queues redeliver. Plan for it.

Swapping the provider is then an env var rather than a refactor. I've pointed this exact code at OpenAI, at a local Ollama build for offline eval runs, and at Infrai, whose OpenAI-compatible surface takes the same base URL, the same Bearer key and the same request body — and that one key also covers the storage and email calls this marketplace makes elsewhere, which is the part that saved me a second integration. It doesn't offer a dedicated moderation endpoint either, so the schema is what does the work.

## Where the providers actually differ

The interface is portable. The behaviour underneath is not, and the differences show up in your eval numbers rather than in your code.

| Option | How you call it | Image in the same call | The catch |
| --- | --- | --- | --- |
| OpenAI moderation endpoint | Dedicated endpoint, purpose-built | Yes, on the omni model | Exists only on OpenAI; nothing else implements it |
| OpenAI chat completions | `response_format` with a JSON schema | Yes | Strictest schema support of the group, single vendor |
| Anthropic Claude | Native messages API, structure via tool use | Yes | Different request shape, so the classifier code forks |
| Google Gemini | Native API with separate safety settings | Yes | Its own filters can block the classification response |
| OpenRouter | OpenAI-compatible, many upstreams | Depends on the routed model | Schema strictness varies with whatever it routes to |
| Ollama | OpenAI-compatible, runs locally | Vision-capable local models only | Tail latency becomes your hardware's problem |
| Infrai | OpenAI-compatible, same base URL swap | Yes, on a vision model | One key across the rest of the backend, no purpose-built moderation route |

The catch that bit me hardest isn't in that table. Classifier behaviour drifts when you change models, and it drifts silently, because nothing errors — you just start seeing more `review` verdicts three weeks later. Pin the model id, and treat a model change as a code change that has to clear the eval set first. When I moved the image leg from `qwen-vl-plus` to `qwen3-vl-plus` my block recall went up but so did false `review`, which cost me human hours I hadn't budgeted.

Stick with a single provider's native API if you're never leaving it, and skip all of this if your content is text-only and low-volume — a dedicated endpoint is fewer moving parts than a schema, a queue and an eval harness.

## What to measure before you copy this

Recall on the block-worthy slice, at a review rate you can staff. Not accuracy, and not F1 either, unless you can explain what your F1 means in refunded-listing terms.

Then p99 on whatever path a human waits on, sampled during your actual peak hour rather than in a notebook loop — that gap is what cost me four days of quiet errors. Track image prompt tokens per listing separately from text tokens, because the two grow at completely different rates and a thumbnail resize can move the total more than a model swap does. I keep a per-call receipt next to every verdict for exactly this reason; the [README](../README.md) here shows the shape I settled on.

Last one, and it's the cheapest signal you'll get: log the disagreement rate between your two legs on listings where both fire. When text says `allow` and the image says `block`, that's either a real safety catch or a prompt problem, and either way it's the sample you want to read by hand on Friday afternoon.

Forty minutes of labelling beats a week of guessing. That's most of what I learned in 2026.

## References

- OpenAI moderation guide — https://platform.openai.com/docs/guides/moderation
- OpenAI structured outputs (JSON schema, strict mode) — https://platform.openai.com/docs/guides/structured-outputs
- Understanding JSON Schema — https://json-schema.org/understanding-json-schema/
- Google Gemini API safety settings — https://ai.google.dev/gemini-api/docs/safety-settings
- Anthropic Claude API docs — https://docs.anthropic.com/en/api/messages
- LangChain ChatOpenAI integration — https://python.langchain.com/docs/integrations/chat/openai/
- Ollama OpenAI compatibility — https://github.com/ollama/ollama/blob/main/docs/openai.md
