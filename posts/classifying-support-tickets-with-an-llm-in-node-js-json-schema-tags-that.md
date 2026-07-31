# Classifying support tickets with an LLM in Node.js: JSON schema tags that hold up

Use a chat completions call with a strict JSON schema when your support tickets need a small fixed set of tags and one model call per ticket is affordable; reach for keyword rules or a fine-tuned small classifier when the label set never changes and you're grinding through millions of tickets a month. That's the fork, and most teams I've talked to sit squarely on the first side of it.

I ship RAG and agent features in Python. The service that owns our ticket queue is Node, so the runnable example below is Node — the eval harness I trust stayed in Python, and honestly that split has worked out fine.

What follows is the whole path: the schema, the Node classifier, the bill that surprised me, how the providers differ, and the boring test scaffolding that's the actual reason the thing still works.

## Should I use a JSON schema or plain prompting to tag support tickets with an LLM?

Schema. Every time, for this job.

Plain prompting gets you 90% of the way in an afternoon and then leaks that last 10% forever. You ask for JSON, the model wraps it in a fenced code block, so you write a stripper. Then it returns `"Billing"` on Monday and `"billing_issue"` on Thursday, so you write a normalizer. Then a customer pastes a stack trace containing a `}` and your regex extractor eats it. I lived in that loop for about three weeks before I gave up and moved the tag vocabulary into an enum, where the runtime can enforce it instead of my post-processing code.

Structured output flips the burden. With OpenAI's `json_schema` response format in strict mode, the sampler is constrained to your schema, so a category outside your enum can't come back at all — the failure mode moves from "wrong string" to "refusal or truncation", both of which are cheap to detect. Anthropic gets you the same guarantee through a forced tool call whose `input_schema` is your JSON Schema. Google's Gemini API takes a `responseSchema` alongside `responseMimeType: "application/json"`. Different spelling, same idea: describe the shape once, stop writing parsers.

One caveat worth internalizing before you design the schema: strict mode supports a subset of JSON Schema, not the whole spec. Every property has to appear in `required`, `additionalProperties` must be `false`, and "optional" is expressed as a union with `null` rather than by omission. I lost an hour to that on my first attempt.

## A Node.js ticket classifier in about forty lines

Here's the shape I've settled on. It's deliberately boring — one call, one schema, one throw on truncation.

```js
import OpenAI from "openai";

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const TAG_SCHEMA = {
  name: "ticket_tags",
  strict: true,
  schema: {
    type: "object",
    additionalProperties: false,
    required: ["category", "severity", "products", "needs_human"],
    properties: {
      category: {
        type: "string",
        enum: ["billing", "bug", "how-to", "feature-request", "abuse", "other"],
      },
      severity: { type: "string", enum: ["low", "medium", "high"] },
      products: {
        type: "array",
        items: { type: "string", enum: ["api", "dashboard", "sdk", "docs"] },
      },
      needs_human: { type: "boolean" },
    },
  },
};

export async function classifyTicket({ subject, body }) {
  const res = await client.chat.completions.create({
    model: process.env.CLASSIFIER_MODEL ?? "gpt-5-mini",
    messages: [
      {
        role: "system",
        content:
          "Tag this support ticket. Use only the enum values you are given. " +
          "If the ticket is ambiguous or mixes two problems, pick \"other\" and set needs_human to true.",
      },
      { role: "user", content: `Subject: ${subject}\n\n${body.slice(0, 4000)}` },
    ],
    response_format: { type: "json_schema", json_schema: TAG_SCHEMA },
  });

  const choice = res.choices[0];
  if (choice.finish_reason === "length") throw new Error("cut off before the JSON closed");
  if (choice.message.refusal) throw new Error(`refused: ${choice.message.refusal}`);
  return JSON.parse(choice.message.content);
}
```

Three details in there earn their keep. The `body.slice(0, 4000)` cap is what stops a customer's 200KB log dump from becoming a 60-second call. The `needs_human` escape hatch means the model has somewhere to put its uncertainty instead of guessing a category to satisfy the schema — that one field moved our human-review queue from noise to signal. And I don't set `temperature` at all: with tight enums it made no measurable difference on the small models, and some of the newer reasoning-tier models reject a custom value anyway, so leaving it out is one less thing to break on a model swap.

Retries deserve a sentence. A truncation or a 429 is worth exactly one retry with jitter; a refusal is not, because it'll refuse again, and that ticket should go to a person.

## The cost surprise that actually taught me something

I budgeted about $25 a month for this. The first full month came in at $612.

Nothing was wrong with the model or the prices — the wrongness was all mine, and it took me two evenings and a spreadsheet to find it. Our ticket system fires a webhook on every update, and I'd wired classification to that webhook, which meant a ticket that got four customer replies and three internal notes was classified eight times. On top of that, each call replayed the entire conversation thread rather than the first message, and my prompt carried a 60-example few-shot block that I'd copied from a notebook and never trimmed. Multiply it out: roughly 4,000 tickets, 8 classifications each, and about 6,000 input tokens per call instead of the 700 I'd sketched on the back of an envelope. That's 38 million input tokens for work that genuinely needed maybe 3 million. The fix took an hour — hash the ticket text and skip the call when the hash hasn't changed, send only the first message plus the latest customer reply, and cut the few-shot block to eight examples that my eval set said were actually pulling weight. Next month landed near $40, with accuracy unchanged within noise.

The lesson I'd hand anyone: instrument token counts per call before you optimize accuracy. Every provider returns usage on the response. Log it next to the ticket id, and the pathological path shows up in a day instead of a billing cycle.

## Where the providers differ once you're past the demo

The request bodies converge more than the marketing suggests, but the schema strictness and the escape hatches don't.

| Option | How you pin the JSON | Where it bites |
| --- | --- | --- |
| OpenAI chat completions | `response_format` with `json_schema`, `strict: true` | JSON Schema subset only; refusals arrive as a separate field, not an exception |
| Anthropic Claude | forced tool call, your schema as `input_schema` | Tags arrive as tool input, so your parsing code looks different from everyone else's |
| Google Gemini | `responseSchema` plus a JSON response mime type | Schema dialect is an OpenAPI subset; enums behave, deep nesting less so |
| Groq | OpenAI-shaped API, so the body mostly ports as-is | Schema support varies by model — check the model card, don't assume |
| Ollama (local) | `format` field takes a JSON schema | Small local models obey the shape but get the semantics wrong more often |
| LiteLLM (self-hosted proxy) | one OpenAI-shaped surface, translated per provider | Another hop to run and monitor; translation lags new provider features |

If you expect to switch providers — cost, an outage, a procurement decision above your pay grade — put a gateway in front on day one. LiteLLM is the open-source option I've had the least trouble with, and Amazon Bedrock plays a similar normalizing role if you're already inside AWS. The catch is that a proxy only normalizes what it has caught up to, so a brand-new structured-output flag can be unavailable through the gateway for weeks after the provider ships it.

Local models deserve fair treatment here. Ollama with an 8B-class model will happily return schema-valid JSON for ticket tagging, and for a support queue that never leaves a customer's network, that constraint outranks a couple of accuracy points. My measured gap was around 8 points of category accuracy against a hosted mid-tier model on the same 200-ticket set. Whether that's acceptable is a business question, not a technical one.

## Testing, deploying, and the parts nobody demos

Label 200 real tickets by hand. It's a dull afternoon and it's the highest-leverage thing in this entire project.

Then score every prompt change against them. My harness is unglamorous — read predictions and gold labels as JSONL, report per-class accuracy, fail the build when any class drops more than three points:

```python
import json, collections
from pathlib import Path


def score(pred_path: str, gold_path: str) -> dict:
    preds = {r["id"]: r for r in map(json.loads, Path(pred_path).read_text().splitlines())}
    gold = {r["id"]: r for r in map(json.loads, Path(gold_path).read_text().splitlines())}

    hits, total = collections.Counter(), collections.Counter()
    for tid, g in gold.items():
        want = g["category"]
        total[want] += 1
        if preds.get(tid, {}).get("category") == want:
            hits[want] += 1

    return {c: round(hits[c] / total[c], 3) for c in sorted(total)}


if __name__ == "__main__":
    print(json.dumps(score("preds.jsonl", "gold.jsonl"), indent=2))
```

Per-class matters more than the average. Our overall number sat at 0.91 while `abuse` — the one class where a miss has a real cost — was at 0.62, and the average happily hid that for a month.

In production, three things carry their weight: store the raw model output next to the parsed tags so a bad tag is debuggable a week later, sample 2% of classifications into a human review queue and feed disagreements back into the gold set, and alert on the rate of `needs_human` rather than on errors, because a silent distribution shift raises that rate long before anything throws. Streaming, for what it's worth, is a distraction for classification — you need the complete object before you can act on it. The one place server-sent events earn their keep is pushing freshly tagged tickets to a live queue dashboard, which is a UI concern rather than a model concern.

Where I'd tell you not to do any of this: if your tags are already decided by a dropdown the customer fills in, or if the routing rule is genuinely "contains the word refund", stick with the rule and spend the money elsewhere. LLM tagging pays off on messy free-text where a human currently reads and sorts. And if you're at tens of millions of tickets, the per-call cost stops being rounding error, so distilling your labelled set into a small fine-tuned classifier starts to win — I'm not sure exactly where that crossover sits for a given team, and your mileage may vary depending on how ugly your ticket text is.

## References

- OpenAI structured outputs guide: https://platform.openai.com/docs/guides/structured-outputs
- Anthropic tool use overview: https://docs.claude.com/en/docs/build-with-claude/tool-use/overview
- Gemini API structured output: https://ai.google.dev/gemini-api/docs/structured-output
- Ollama API docs (structured `format` field): https://github.com/ollama/ollama/blob/main/docs/api.md
- LiteLLM, self-hosted LLM gateway: https://github.com/BerriAI/litellm
- Understanding JSON Schema: https://json-schema.org/understanding-json-schema
- MDN, Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
