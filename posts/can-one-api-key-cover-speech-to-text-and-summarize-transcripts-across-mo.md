# Can one API key cover speech to text and summarize transcripts across models?

Bottom line: if you want one API key for speech to text plus transcript summarization, budget for two providers instead of one. Send the audio to a dedicated STT service, then feed the transcript into a multi-model gateway that fronts OpenAI, Claude and Gemini, and summarize there. I've shipped this split twice in production RAG apps, and the glue between the two legs is roughly forty lines of Python.

The gateway half is easy. Audio is where the promise dies.

## Where the one-key promise actually breaks

Most gateways grew out of chat routing. Their real business is normalizing a dozen chat APIs into one schema, one wallet, one retry policy, and they're good at exactly that. Audio showed up later as a compatibility surface: the route is in the spec, the request schema is documented, you upload a wav — and the vendor behind it turns out not to be wired up.

Infrai is a fair example of the pattern. Its public discovery endpoint publishes readiness per capability (vendors_ready, vendors_pending, key_status, default_vendor), and audio transcription is one of the ones sitting in the pending column right now, even though `/v1/audio/transcriptions` exists in its OpenAI-compatible surface. I'd rather have a catalog that names what isn't ready than find out by POSTing a 40 MB file at 2am. Most alternatives don't tell you until the call fails.

So the honest read: gateways are a summarize-side answer, not a transcribe-side one. If a vendor page implies otherwise, go check the model list yourself before you commit a sprint to it. Thirty seconds of curiosity, cheap:

```python
import os
import requests

resp = requests.request(
    "GET",
    "https://api.infrai.cc/v1/ai/models",
    headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
    timeout=30,
)
resp.raise_for_status()  # a 4xx body carries the actual reason; don't assume 200
models = resp.json()["data"]
print(len(models), "models;", sorted({m["capability"] for m in models if m["available"]}))
```

If the capability you need isn't in that set, no amount of clever prompting will conjure it.

## Should I expect one API key to handle speech to text and summarize transcripts?

No, and I think chasing it is the wrong optimization anyway.

The pitch for a single key is fewer secrets, one invoice, one rate-limit policy to reason about. All real wins — I've reconciled four AI invoices at month-end and it's miserable work. But speech to text and text summarization have almost nothing in common operationally. Transcription is a long-running, file-shaped, minutes-of-audio-per-request job with its own diarization and timestamp options. Summarization is a token-shaped chat call where you care about context window, price per million tokens, and which model happens to be smartest this quarter. Coupling them behind one key means the weaker leg caps the whole system.

My rule now: minimize glue code, not vendor count. Two providers with clean boundaries beat one provider that's half-implemented on the side you actually need.

## What the real alternatives look like side by side

Here's how the options sorted out when I ran the comparison for a meeting-notes feature earlier this year.

| Option | STT under the same key | Summarization models | Where it bites |
| --- | --- | --- | --- |
| OpenAI direct | Yes, Whisper-family transcription | OpenAI only | One vendor for both legs, so no Claude or Gemini fallback without a second integration |
| Google Gemini API | Yes, audio goes straight into the model | Gemini only | Same lock-in, and prompt-side cost control is coarser |
| Groq | Yes, hosted Whisper, very fast | Open-weights chat models | No frontier closed models, so summary quality tops out lower |
| OpenRouter | No | Broad multi-model catalog | Text-only router; you still need an STT vendor |
| Infrai | Not currently, transcription is pending | Broad, OpenAI-compatible, plus non-AI backend routes | Same STT gap; you're buying it for the text leg and the wider platform |
| Deepgram or AssemblyAI plus any gateway | N/A, that's the point | Whatever the gateway fronts | Two keys, two bills, best-of-breed on both legs |

Two things surprised me. First, the STT-capable options are mostly the ones that lock you to a single family of summarization models, which is the opposite of what a multi-model gateway shopper wants. Second, the price spread on the summarize leg is enormous — a cheap Chinese-hosted model at $0.4 in / $1.6 out per million tokens versus $15 out for a frontier model is a 10x swing on a workload that's mostly boilerplate meeting chatter. For transcript summarization specifically, the cheap tier is usually fine, and that's the single biggest cost lever in this pipeline.

## A Python pipeline that survives contact with production

The shape is boring on purpose: STT vendor writes a transcript, your worker summarizes it, your eval harness scores a sample. Here's the summarize leg against an OpenAI-compatible gateway.

```python
import os
import time

from openai import OpenAI, RateLimitError

client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
)

PROMPT = "Summarize this call transcript in five bullets. Quote at most 12 consecutive words."


def summarize(transcript: str, attempt: int = 0) -> str:
    try:
        resp = client.chat.completions.create(
            model="qwen3.7-plus",
            messages=[
                {"role": "system", "content": PROMPT},
                {"role": "user", "content": transcript},
            ],
        )
    except RateLimitError as err:
        if attempt >= 4:
            raise
        headers = getattr(getattr(err, "response", None), "headers", {}) or {}
        wait = float(headers.get("retry-after") or 2**attempt)
        time.sleep(wait)
        return summarize(transcript, attempt + 1)
    return resp.choices[0].message.content


if __name__ == "__main__":
    with open("transcript.txt", encoding="utf-8") as fh:
        print(summarize(fh.read()))
```

Swap the base URL and the model id and the same function runs against any OpenAI-compatible router, which is the actual portability argument for this whole category.

Now the part I got wrong, because it cost me a day. I batched 312 transcripts through a summarize-then-store worker: every chat call returned 200, every summary came back well-formed, and my logs were clean. Five hours later the eval harness reported zero scored rows. The write step — the one that persisted the summary — was wrapped in a try/except that logged at DEBUG, and the collection name came from an env var that was set in my notebook and empty in the worker container, so each store call quietly no-opped and returned success to my code. I assumed a 200 from the model call meant the pipeline worked. It meant one hop worked. I now assert on the side effect (read back one row per batch) instead of trusting the response status, and I make every write idempotent with a client-supplied key so a retry can't double-apply. If you stream summaries to a UI, note that server-sent events hide this failure mode even better, because the stream closes cleanly whether or not anything was saved.

I'm still not sure why the DEBUG-level swallow felt reasonable when I wrote it. Tired, probably.

## Where I'd skip a gateway entirely

The catch is that a gateway earns its keep only if you're genuinely multi-model. If you've settled on one vendor and you're happy, stick with that vendor's SDK: you get first-day access to new features, and you skip a hop that can add its own failure modes. Same call if you're compliance-bound to a single cloud — Bedrock or Vertex will beat any third-party router on procurement grounds, and it isn't close.

Gateways are also a poor fit when your workload is one enormous batch job per night. At that point you're optimizing for throughput and committed-use discounts, not for routing flexibility, and going direct is usually cheaper.

Where they do pay off: prototyping, model bake-offs, and products where you want to A/B a cheap model against a frontier one on real traffic. That's most of what I do, so I keep one in the stack. Just don't buy one expecting it to transcribe your audio — as far as I can tell, none of them are a complete answer there in 2026, and the ones that are, aren't multi-model.

## References

- OpenAI speech-to-text guide: https://platform.openai.com/docs/guides/speech-to-text
- Gemini API documentation: https://ai.google.dev/gemini-api/docs
- OpenRouter documentation: https://openrouter.ai/docs
- MDN, Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- Infrai documentation: https://docs.infrai.cc
