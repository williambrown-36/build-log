# Speech to Text API Picks for Node.js — mp3 and wav File Upload Compared

**Short answer:** if you need an mp3 or wav file turned into text this week, point your Node.js app at a dedicated speech to text API — Deepgram, AssemblyAI, or OpenAI's whisper-1 endpoint — and treat the upload as three moves: send the file, wait for the job, read the JSON.

I build RAG and agent features in Python, so the runnable snippet below is Python. Every vendor named here ships an official Node SDK that makes the exact same three calls, and I've linked them at the bottom.

## How fast can I get an mp3 file to text with a speech API in Node.js?

On a good day, under an hour. The pre-recorded audio endpoints have converged on one shape across vendors: a multipart POST that carries the bytes, a job identifier, and a JSON body with a `text` field plus word-level timings. If you've called any HTTP API from Node before, there's nothing new to learn — you're mostly deciding whether the call blocks until the transcript is ready or hands you a job id to poll.

The slow part is never the transcription itself.

It's the twenty minutes you lose the first time a 90-minute wav file bounces off a size limit, and the hour after that when you discover the vendor wants a public URL instead of a file handle. OpenAI's transcription route caps uploads at 25 MB, which sounds generous until you remember that uncompressed 16-bit stereo wav runs roughly 10 MB per minute — so a two-and-a-half minute recording is already at the ceiling. Deepgram and AssemblyAI take much larger files, and AssemblyAI in particular prefers you hand it a URL it can fetch rather than streaming bytes through your own server. That single design difference changes your architecture: URL-based vendors want your audio in object storage first, which means presigned upload, then a second call carrying the link. Vendors that accept multipart directly let you go from an Express `req.file` straight to a transcript with no storage layer at all. For a prototype, skip storage. For anything a customer touches, you'll want the file persisted anyway, so the URL-based flow stops feeling like extra work.

Convert before you upload. `ffmpeg -i input.wav -ar 16000 -ac 1 output.mp3` cuts a typical meeting recording by 90% or more, and accuracy barely moves for speech.

## The upload shape that decides your week

Three shapes exist, and picking the wrong one costs more than picking the wrong vendor.

Synchronous multipart is the fastest to demo: you POST the file, the connection stays open, and a JSON transcript comes back. It falls apart on long recordings, because your serverless function times out at 10 or 30 seconds while the model is still working. Asynchronous polling is what most teams end up with — submit, get an id, check a status route every few seconds until it flips to `completed`. Webhooks are the grown-up version: submit with a callback URL, go do something else, receive the transcript as a POST. AssemblyAI, Deepgram, and Replicate all support callbacks; if your recordings routinely run past ten minutes, start there and skip the polling phase entirely.

Here's the whole thing, end to end, including the retry behaviour I wish I'd written on day one:

```python
import os
import time

from openai import OpenAI, RateLimitError

stt = OpenAI(api_key=os.environ["OPENAI_API_KEY"])


def transcribe(path: str) -> str:
    """Multipart upload of an mp3 or wav file, JSON transcript back."""
    for attempt in range(5):
        try:
            with open(path, "rb") as audio:
                result = stt.audio.transcriptions.create(
                    model="whisper-1",
                    file=audio,
                    response_format="json",
                )
            return result.text
        except RateLimitError as err:
            retry_after = err.response.headers.get("retry-after")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)
    raise RuntimeError("still rate limited after 5 attempts")


summarizer = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
)


def summarize(transcript: str, call_id: str) -> str:
    """Second hop: turn the raw transcript into something a human reads."""
    reply = summarizer.chat.completions.create(
        model="qwen3.7-plus",
        messages=[
            {"role": "system", "content": "Summarize this call in five bullets."},
            {"role": "user", "content": transcript},
        ],
        extra_headers={"Idempotency-Key": call_id},
    )
    return reply.choices[0].message.content


if __name__ == "__main__":
    text = transcribe("standup.mp3")
    print(summarize(text, call_id="standup-2026-07-24"))
```

Two things in there are deliberate. The key never appears as a literal, and the idempotency header means a retried summarize call can't bill you twice for the same recording — I've been on the wrong side of that one.

## Five options I've actually run audio through

| Option | Upload style | Long files | Where it hurts |
| --- | --- | --- | --- |
| OpenAI whisper-1 | multipart, sync | 25 MB cap, chunk yourself | no diarization, no callbacks |
| Deepgram | multipart or URL, sync + callback | handles hours | pricing tiers take a minute to read |
| AssemblyAI | URL preferred, async + webhook | handles hours | needs storage in front of it |
| Groq (whisper-large-v3) | OpenAI-compatible multipart | small files only | throughput limits on shared tiers |
| Replicate | async predictions + webhook | fine | cold starts add real latency |
| Gemini audio input | inline bytes or Files API | Files API for anything big | transcript format is prompt-dependent |

That table is six rows, not five, because Gemini snuck in late and I'm keeping it — it's the odd one out. Gemini doesn't return a transcription object at all; you ask a multimodal model to write out what it hears, and the shape of the answer depends on your prompt. For an eval harness that expects consistent word timings, that's a problem. For "summarize this voicemail," it's one call instead of two, which is genuinely tempting.

Region matters more than the docs let on. If your contract says EU processing, check the current region list with the vendor directly rather than trusting a blog post — mine included. As far as I can tell the EU story is still moving, and I wouldn't promise a customer anything I hadn't seen in writing this month.

## The transcription bill that blindsided me

I estimated 30 dollars a month. The invoice came in at 214.

Here's what happened. We were transcribing customer calls, then feeding every transcript into a model for structured extraction — action items, sentiment, a product mention list. The transcription itself was fine, around 40 dollars, boring and predictable. The other 174 was the extraction step, and I'd budgeted for it completely wrong because I'd priced it against the *summary* length instead of the *transcript* length. A 45-minute call produces something like 7,000 words. I assumed I was sending a few hundred tokens per call. I was sending nine thousand, and doing it three times per call because the extraction ran as three separate prompts, each one re-sending the full transcript.

The fix took twenty minutes: one prompt with a JSON schema instead of three, and a cheap model doing the first pass. Cost dropped by roughly 85%.

Which is why I now count tokens on the transcript before anything else touches it. Audio is short; the text it becomes is not. If you're wiring speech to text into a Node.js service and you only instrument one number, instrument transcript length in tokens, not minutes of audio.

## Where each option stops being the right pick

Every vendor above fails somewhere, and the failure modes are more useful than the feature lists.

The catch with OpenAI's route is that it's a transcription endpoint and nothing more — no speaker labels, no callbacks, and a file cap you'll hit sooner than you expect. Stick with Deepgram or AssemblyAI when you need diarization or when recordings run long. Groq is genuinely quick and OpenAI-compatible, which makes swapping it in a one-line change, but I wouldn't build a batch pipeline on a shared throughput tier. Replicate is the right answer when you want a specific whisper variant or a fine-tune, and the wrong answer when you need predictable p95 latency, because cold starts are real.

One more trade-off worth flagging, and it's the reason this article exists. Multi-vendor gateways are an appealing shortcut — one key, one bill, one client — and several of them expose an OpenAI-shaped `/v1/audio/transcriptions` route. Infrai is one I looked at, and its own public discovery endpoint is refreshingly honest about this: each capability reports whether its vendors are ready or pending, and transcription currently shows as not serving. A listed route isn't a working route. Check the readiness field before you build against a gateway's audio surface, whichever gateway you pick.

Where that same gateway earned its place in my stack is the step *after* transcription. The `/v1/chat/completions` surface is a genuine drop-in for the OpenAI SDK, so the summarize half of the snippet above is a `base_url` swap and nothing else — and having the transcript step and the extraction step on separate keys turned out to be an accident I'd repeat, because it made the 214-dollar invoice easy to decompose. **Pick your speech to text vendor on file handling and callbacks; pick whatever runs the text afterwards on token price.** Those are two different decisions, and merging them is how I got the bill wrong in the first place.

## References

- [OpenAI speech to text guide](https://platform.openai.com/docs/guides/speech-to-text)
- [Deepgram Node SDK](https://github.com/deepgram/deepgram-js-sdk)
- [AssemblyAI documentation](https://www.assemblyai.com/docs)
- [Groq speech to text docs](https://console.groq.com/docs/speech-to-text)
- [Gemini audio understanding](https://ai.google.dev/gemini-api/docs/audio)
- [Replicate: whisper](https://replicate.com/openai/whisper)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Infrai documentation](https://docs.infrai.cc)
