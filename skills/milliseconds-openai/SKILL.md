---
name: milliseconds-openai
description: |
  Point an existing OpenAI client at decision-machine-1 (base_url https://api.milliseconds.ai/v1, model decision-machine-1) for structured extraction via response_format.json_schema and function calling via tools. Use when the project already has an OpenAI SDK and wants typed decisions without a new client, and to know when to switch to the native endpoints. Also accepts a data-URL image content part.
---

# OpenAI-compatible surface

`POST https://api.milliseconds.ai/v1/chat/completions` with `model: "decision-machine-1"`. It is a compatibility layer, not a chat model. Two modes work: structured extraction and function calling. Everything else returns 400 `unsupported_request`.

```python
from openai import OpenAI
client = OpenAI(base_url="https://api.milliseconds.ai/v1", api_key=os.environ["MS_API_KEY"])
```

## Structured extraction

`response_format.json_schema` maps to `extract`. The filled object arrives as a JSON **string** in the assistant message.

```bash
curl -s https://api.milliseconds.ai/v1/chat/completions \
  -H 'content-type: application/json' -H "authorization: Bearer $MS_API_KEY" \
  -d '{
    "model": "decision-machine-1",
    "messages": [{"role": "user", "content": "Table for 4 at Nobu on Friday."}],
    "response_format": {"type": "json_schema", "json_schema": {"name": "booking", "schema": {
      "type": "object",
      "properties": {"restaurant": {"type": "string"}, "party_size": {"type": "integer"}, "day": {"type": "string"}}
    }}}
  }'
```

```json
{"id":"chatcmpl-9125a25ca46a4efb88ba0133","object":"chat.completion","created":1789826481,"model":"decision-machine-1",
 "choices":[{"index":0,"message":{"role":"assistant","content":"{\"restaurant\":\"Nobu\",\"party_size\":4,\"day\":\"Friday\"}"},"finish_reason":"stop"}],
 "usage":{"prompt_tokens":253,"completion_tokens":14,"total_tokens":267}}
```

Parse `choices[0].message.content`. The same schema rules as `extract` apply: describe every property, `null` for missing values, no probabilities.

## Function calling

`tools` runs two typed decisions: `classify` over the tool names and descriptions picks the tool, then `extract` over that tool's `parameters` fills the arguments. Exactly one tool call comes back, `finish_reason: "tool_calls"`, `content: null`, `arguments` as a JSON string. A named `tool_choice` skips the classify step. `tool_choice: "none"` without a usable `response_format` is rejected.

Write each `function.description` as a case description, the same rule as classify labels: `"The customer asks for money back for a duplicate or wrong charge."`, and describe every parameter.

## What the facade discards

- Only `user` messages are read, joined. `system`, `developer`, `assistant` and `tool` roles are dropped. Instructions in a system prompt do nothing.
- Plain chat, `response_format: text` and `json_object` return 400 `unsupported_request`.
- Any other model id returns 404 `model_not_found` (`Use "decision-machine-1"`).
- `stream: true` is served, but there is nothing to stream incrementally. Docs: /openai/streaming-and-differences.md.
- Usage is reported in `usage.prompt_tokens`; the `x-input-*` headers are absent, `x-inference-ms` is present.

## When to move to the native endpoints

The facade reaches two of eight capabilities and returns no numbers to route on.

| Chat usage | Native call | Gain |
| --- | --- | --- |
| `response_format.json_schema` | `POST /v1/decision-machine-1/extract` | Parsed `data`, `texts` batching |
| `tools` with two or more entries | `classify` | `scores` per tool, `confidence`, one inference call instead of two (0.72 s → 0.43 s) |
| One tool or named `tool_choice` | `extract` | Same arguments without the envelope |
| "Is this X, yes or no?" prompt | `yes-no` | `probability`, 32 statements per call |
| "Rate 1 to 5" prompt | `rate` | `score`, `level`, `confidence` |
| "Where does it say X?" prompt | `answer` | Span with offsets |
| "List names, dates, amounts" prompt | `entities` | Every span, typed, with offsets |
| "Check this field against the doc" prompt | `verify` | `matches`, `probability`, `found[]` |

Switch as soon as you need a probability, a confidence, an offset, a batch, or confidence routing. Docs: /openai/when-to-use-native.md.

Full pages: https://docs.milliseconds.ai/openai/overview.md, /openai/structured-extraction.md, /openai/function-calling.md

## Images

The facade accepts an `image_url` content part whose `url` is a data URL (`data:image/jpeg;base64,...`, also `png` and `webp`), under the same rules as the native `image` field: one image, 5 MB decoded, bytes only. An `http(s)` URL is a 400; the facade never fetches. The part takes `detail`: `low` and `high` pass through, `auto` and anything else become `medium`. Billing excludes the base64 and adds a fixed number per image, but the facade always bills the `extract` tier, 5,000 / 10,000 / 20,000 tokens by tier (provisional), whatever the operation. Use the native `/classify` or `/yes-no` endpoint for cheap image decisions.
