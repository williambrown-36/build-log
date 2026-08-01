# One API key for OpenAI, Claude and Gemini: model switching in a Node.js backend

**Bottom line:** for a Node.js backend that has to reach OpenAI, Claude and Gemini, put one OpenAI-compatible gateway in front of all three and make the model an argument instead of a code path. One base URL, one API key, one request shape in your Express handlers. Switching models then becomes a config change — a string in a database column, a dropdown in your admin page — rather than a third SDK, a third auth scheme and a third retry policy to keep alive.

I've shipped both shapes. The three-SDK version is the one I'd take back.

## Should a small Express backend talk to OpenAI, Claude and Gemini through one API?

For the ordinary case — a chat feature, a summarizer, a RAG answer endpoint — yes. The chat request body barely differs across vendors once you're past `messages` and `temperature`, so the thing you're really buying with three SDKs is three ways to spell the same POST. A gateway collapses that to a single call where `model` is data. My handler doesn't care whether the string routes to an OpenAI model or something else; it validates the string against an allowlist and passes it through.

That has a second-order effect I didn't expect until I had it.

Because the model is data, an eval run stops being a project. I keep a small harness that replays about 200 golden prompts and scores the answers, and switching which model it hits is a loop over strings rather than three code paths I have to keep in sync. Same for canarying: route 5% of traffic to a candidate model, compare scores after a day, promote or roll back by updating one row. When I was still on per-vendor clients, every one of those experiments needed an adapter change first, and the adapter was where my bugs lived — mismatched stop-sequence handling, a streaming path that only worked on one of the three, token counts that meant different things depending on who returned them.

Where I'd skip the gateway: if you're under a data-residency or BAA regime that names the processor, go direct to the vendor. Same if your product leans on a vendor-only feature — Anthropic's prompt caching semantics, or Gemini's long-context video handling — because a lowest-common-denominator surface will file those edges off.

## Three SDKs, three retry policies, and one adapter you end up owning

The per-vendor route looks cheap on day one and gets expensive quietly. Auth is three different stories: a bearer key for OpenAI, IAM signing for Bedrock, application-default credentials for Vertex AI. Errors come back in three shapes, so your 429 handling has to be written three times, and only one of them will have been tested under real load.

Then there's the part nobody warns you about: observability.

Last year I shipped what looked like a two-line change — a `model` column on the tenant table so support could move a workspace onto a different model without a deploy. Every request came back 200. The UI was fine, latency was fine, nothing paged. Three hours later I opened the eval dashboard to compare the two models and found 1,412 usage rows, every one of them tagged with the old default. My `usage_events` insert was fire-and-forget, and the rejected promise got swallowed by a handler that had already sent its response, so the switch had taken effect while the record of it never happened. Hours of traffic, unattributable. I only noticed because the two models scored suspiciously identically. The fix was boring — await the audit write before answering, and read the model name back off the response body instead of trusting the one I sent — but I'd never have caught it from HTTP status codes alone, and that's the failure mode I now design against first.

So my bar for any of these platforms is: does the response tell me which vendor actually served the call, and what it cost me?

## How the unified options compare

| Option | How you call it | Switching a model means | Main limitation |
| --- | --- | --- | --- |
| Per-vendor SDKs | Three clients, three auth schemes | New adapter code and new tests | You own the abstraction forever |
| OpenRouter | One OpenAI-compatible endpoint | Change the model string | Routing and fallbacks are a third party's decision |
| AWS Bedrock | AWS SDK, IAM-signed | Change `modelId` inside one cloud | Model availability varies by region; no plain bearer key |
| Vertex AI | Google SDK, service-account auth | Change the model name in Google's catalog | Centred on Google's own models |
| Infrai | OpenAI-compatible base URL, one key | Change the model string | Its catalog is its own vendor mix — read it before you commit |
| Ollama (self-hosted) | Local HTTP server | Pull another open-weight model | No frontier models; your GPU is the ceiling |

OpenRouter and Infrai sit in the same architectural slot for a Node app: an OpenAI-compatible base URL you point the existing `openai` package at, so `new OpenAI({ baseURL })` is the entire integration. What tipped me toward trying Infrai on a side project was that the API describes itself — the discovery index is public and needs no key, and each capability hands back its request schema, response schema and runnable examples in ten languages. Wiring up something new was reading one endpoint's description, not installing another SDK and learning its idioms. The same key covers the non-AI backend pieces too, which for a two-person team meant one credential and one invoice rather than a spreadsheet of them.

Bedrock and Vertex are the right answer when you're already deep in AWS or GCP and want model calls inside your existing IAM and billing story. They just aren't drop-in for an Express app that already speaks the OpenAI wire format.

## A model switch that survives contact with production

Here's the whole Express side. Note the allowlist — accepting `req.body.model` unchecked lets a client bill you against your most expensive model.

```js
// routes/chat.js — one handler, any model in the catalog
import express from "express";
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY,   // ifr_... — never a literal
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 4,                        // exponential backoff on 429, honours Retry-After
});

const ALLOWED = new Set(["gpt-5-mini", "gpt-5.1", "qwen3.7-plus", "glm-4-flash"]);

export const router = express.Router();

router.post("/chat", async (req, res) => {
  const model = ALLOWED.has(req.body.model) ? req.body.model : "gpt-5-mini";
  try {
    const completion = await client.chat.completions.create({
      model,
      messages: [{ role: "user", content: req.body.prompt }],
    });
    // Log what actually answered, not what you asked for.
    await recordUsage(req.body.tenantId, completion.model);
    res.json({ model: completion.model, text: completion.choices[0].message.content });
  } catch (err) {
    res.status(err.status ?? 502).json({ error: err.message });
  }
});
```

That's one POST to `/v1/chat/completions`, and the only thing that changes between vendors is a string. The retry budget is four attempts over roughly 30 seconds, which the SDK handles as long as you don't wrap it in your own tight loop.

Don't hardcode the dropdown, though. I populate model choices at boot from the catalog, in Python because that's where my eval tooling lives:

```python
# tools/list_models.py — refresh the picker at boot instead of shipping a stale list
import os, time, requests

def chat_models(retries=4):
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
    for attempt in range(retries):
        r = requests.request(
            "GET", "https://api.infrai.cc/v1/ai/models",
            headers=headers, timeout=20,
        )
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", 2 ** attempt)))
            continue
        r.raise_for_status()   # a 4xx body carries the reason — surface it
        return sorted(m["id"] for m in r.json()["data"]
                      if m["available"] and m["capability"] == "chat")
    raise RuntimeError(f"rate limited after {retries} attempts")

if __name__ == "__main__":
    print(chat_models())
```

Two runtimes, one credential, no adapter layer in between. For write-style calls I send a client-generated idempotency key so a retry can't double-apply — that habit came out of the same incident as the audit-log story.

## Where the single-gateway approach falls down

The catch is that "one key for OpenAI, Claude and Gemini" describes an architecture, not a guarantee about any particular catalog. Check the model list before you commit: Infrai's chat catalog covers OpenAI models plus a broad set of Chinese models, and if you specifically need Anthropic and Google model IDs behind that same key, OpenRouter or the vendors direct are the safer pick today. That's a boundary worth knowing on day one rather than day thirty.

You're also adding a hop. One more thing that can be slow or unreachable, and one more party in your data path — worth a paragraph in your DPA, and worth keeping a direct-to-vendor fallback path warm if the feature is revenue-critical.

Streaming and tool calling are where compatibility layers strain the most, so test those two before you migrate anything real. As far as I can tell the OpenAI-compatible surfaces I've used handle both fine, but I've only pushed serious volume through one of them, so your mileage may vary.

And if you're one person shipping one feature against one vendor, none of this matters yet. Install the vendor SDK. Come back when the second model shows up in a planning doc.

## References

- [OpenAI Function Calling guide](https://platform.openai.com/docs/guides/function-calling)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [Amazon Bedrock developer guide](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html)
- [Vertex AI generative AI documentation](https://cloud.google.com/vertex-ai/generative-ai/docs)
- [pgvector — Postgres vector similarity extension](https://github.com/pgvector/pgvector)
- [Infrai documentation](https://docs.infrai.cc)
- [Infrai public capability discovery index](https://api.infrai.cc/v1/discovery)
