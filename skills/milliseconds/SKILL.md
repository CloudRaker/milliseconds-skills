---
name: milliseconds
description: |
  Build software on decision-machine-1, the milliseconds.ai decisions API. It turns text, or one image, into typed decisions in about 0.4 s: a boolean with a probability, one label with a score distribution, a position on a scale, a span with offsets, a filled JSON Schema, every entity of a type, or a check of a value against the text. Never prose. Use when a feature needs a small judgment over text (route, flag, rank, extract, verify, guard an LLM), when a prompt-and-parse step could become a typed decision, or when brainstorming where cheap semantic decisions replace fragile parsing, including over document scans and photos. Also use for any API question about api.milliseconds.ai.
---

# Build with decision-machine-1

`decision-machine-1` runs at `https://api.milliseconds.ai`. Every capability is one `POST` with a JSON body. It returns labels, probabilities and spans copied from your text. It never generates, summarizes or reasons. Code owns the workflow. The model supplies one narrow judgment where code needs semantic understanding.

## Read the live docs

The docs at https://docs.milliseconds.ai are the source of truth. Read the pages you need as part of the task.

- Index: https://docs.milliseconds.ai/llms.txt. Append `.md` to any page URL for clean Markdown, for example https://docs.milliseconds.ai/capabilities/classify.md.
- MCP server for the docs: `https://docs.milliseconds.ai/_mcp/server` (this plugin's `.mcp.json` registers it).
- Before you write an integration, read the capability page you chose and https://docs.milliseconds.ai/concepts/writing-good-questions.md.

## Auth and first call

Every `/v1` route needs `Authorization: Bearer sk-ms-...`. Keys come from https://console.milliseconds.ai (free plan: 125 million input tokens per month). Keep the key in `MS_API_KEY`, never in client-side code.

```bash
curl -s https://api.milliseconds.ai/v1/decision-machine-1/classify \
  -H 'content-type: application/json' \
  -H "authorization: Bearer $MS_API_KEY" \
  -d '{
    "text": "I was charged twice for my subscription this month and support has not replied.",
    "labels": {
      "billing": "payments, invoices, charges, refunds",
      "shipping": "delivery, tracking, returns of physical goods",
      "account": "login, password, profile settings"
    }
  }'
```

```json
{"label":"billing","probability":0.956,"confidence":0.833,"scores":{"billing":0.956,"shipping":0,"account":0.044}}
```

A `401 invalid_api_key` means the key is wrong or revoked. Ask the user for a key. Never guess one.

## SDKs and CLI

Official SDKs wrap every capability and keep the wire names. Prefer them over raw HTTP when the project already uses the language. Both read the key from `MS_API_KEY`. Raw `curl` and `fetch` stay valid everywhere else, and the JSON body is the same.

TypeScript, zero runtime dependencies, TypeScript 5.0 or later, Node 20 or later:

```sh
npm i @cloudraker/milliseconds
```

```ts
import { DecisionMachine } from '@cloudraker/milliseconds'

const dm = new DecisionMachine() // reads MS_API_KEY

const r = await dm.classify(ticket, {
  billing: 'payments, invoices, charges and refunds',
  shipping: 'delivery, tracking and packages',
  account: 'login, passwords and profile settings',
})
// r: ClassifyResult<'billing' | 'shipping' | 'account'>
// r.label is that union, so r.label === 'shiping' fails to compile
```

Your labels, scale levels, entity types and schema flow into the result type. A hoisted label array needs `as const` to keep the union. The package also exports `typed` for a zod or valibot schema, and `isMillisecondsError` to narrow a caught error.

Python, `httpx` only plus `typing-extensions` on Python 3.10, Python 3.10 or later:

```sh
pip install cloudraker-milliseconds
```

Python cannot read label names out of a dict display. It solves a `TypeVar` from an annotated constant, so annotate the constants file:

```python
from typing import Final, Literal, Mapping

from milliseconds import DecisionMachine

Intent = Literal["billing", "shipping", "account"]

LABELS: Final[Mapping[Intent, str]] = {
    "billing": "payments, invoices, charges and refunds",
    "shipping": "delivery, tracking and packages",
    "account": "login, passwords and profile settings",
}

dm = DecisionMachine()  # reads MS_API_KEY

r = dm.classify("I was charged twice.", LABELS)
r.label  # Intent. A match statement over it is exhaustive.
r.scores["billing"]  # r.scores["refunds"] is a type error.
```

Without the annotation you get `ClassifyResult[str]`. Nothing breaks, and you lose only the names. `AsyncDecisionMachine` has the same methods. Annotate a taxonomy constant with `Tree`. Catch `MillisecondsError`, or one subclass such as `RateLimitError`.

From a shell, the TypeScript package ships the `dm1` command:

```sh
npm i -g @cloudraker/milliseconds
dm1 classify "I was charged twice" billing shipping account
dm1 yes-no "Ship it today" "The customer expresses urgency." --check
```

`dm1 --image <path>` sends an image instead of a text, with `--detail low|medium|high`. Both SDKs take `image` and `detail` on every capability method. Python accepts a path, bytes or base64. TypeScript accepts a `Uint8Array`, an `ArrayBuffer`, a `Blob`, a data URL or bare base64; read a file with the Node-only helper `imageFile(path)` from `@cloudraker/milliseconds/node`, and pass it as the option: `dm.classify('', LABELS, { image: imageFile('receipt.jpg') })`.

`--check` works with `yes-no` and `verify`. It exits 3 when the answer is no. `--min` gates on `probability`, and `--min-confidence` on `confidence`. `--json` and `--jsonl` print machine-readable output. `dm1 --help` lists every flag.

## Choose the capability from the answer shape

| You need | Capability | Returns | Skill |
| --- | --- | --- | --- |
| True or false on a claim | `yes-no` | `answer`, `probability` | milliseconds-yes-no |
| One label from a fixed set | `classify` | `label`, `probability`, `confidence`, `scores` | milliseconds-classify |
| A path through a label tree | `classify-tree` | `path`, `label`, `levels[]` | milliseconds-classify |
| A position on an ordered scale | `rate` | `score`, `level`, `label`, `confidence`, `scores[]` | milliseconds-rate |
| One span quoted from the text | `answer` | `answer`, `start`, `end`, `probability` | milliseconds-answer |
| A typed object for your schema | `extract` | `data` (null for missing) | milliseconds-extract |
| Every mention of a type, with offsets | `entities` | `entities[]` | milliseconds-entities |
| A check of a value you already hold | `verify` | `matches`, `probability`, `found[]` | milliseconds-verify |

Endpoints: `POST /v1/decision-machine-1/<capability>`. Two questions decide most cases. Do you supply the possible answers (yes-no, classify, rate) or does the text supply them (answer, entities, extract)? Do you need character offsets (answer, entities)?

Near misses:
- **yes-no vs classify with two labels.** Yes-no scores each statement on its own; several can be true. Classify forces one winner and returns `confidence`.
- **answer vs extract.** Answer gives one span with offsets and no coercion. Extract fills several typed fields with no offsets.
- **entities vs extract.** Entities returns every occurrence. Extract fills each field once.
- **verify vs extract.** Extract asks what the value is. Verify asks whether your value agrees with the text.

An OpenAI client already in the project? See milliseconds-openai. It reaches `extract` and `classify`+`extract` only.

## Write labels that describe the case

The label text is the instruction. The model reads it literally. **Describe the case, never the verdict.** `"yes"`, `"no"`, `"A"`, `"1"` describe nothing. A `when_true: "yes"` hint measurably flipped a correct yes-no answer to wrong.

- Statements are third-person claims about the text: `"The customer expresses urgency."`, not `"urgent"` or `"Is the customer angry?"`.
- One idea per statement, label or level. Batch the rest.
- Claim what the text says, not what you conclude: `"The letter states five or more years of backend experience."` beats `"The candidate is a good fit."` (the vague one scored 1.0 on a letter with no experience at all).
- Give every label, entity type and schema field a description. `["A","B","C"]` scored 0.603 on a billing ticket; described labels scored 0.956.
- Scale levels are descriptions ordered low to high, never numbers.
- Name the role in a question, not the type: `"the date the payment is due"`, not `"date"`.
- Write labels, statements and descriptions in English even for non-English text. Keep the text in its original language.
- Add a catch-all label such as `other` when real inputs fall outside your set.

## Put the constants in one place

Keep every label set, statement, scale, schema and threshold in one file, next to the code that acts on it. Humans review those, not the HTTP calls. Expect the user to edit wording with you.

## Read the numbers, then set thresholds per action

- `probability`: support for the returned outcome. Formulas differ per capability, so a 0.8 from yes-no and a 0.8 from classify are not comparable.
- `confidence` (classify, rate): `1 − normalized entropy` over `scores`. Near 1 is one clear winner. Near 0 is a flat split. Read it with `probability`: high probability with low confidence means two labels both scored high.
- `score` (rate): probability-weighted position, 0 to `scale.length − 1`. Route on `score` and `confidence`, not `level`, which can flip on 0.001.
- Every number is rounded to 3 decimals. `extract` returns no probability at all.
- The built-in 0.5 on yes-no `answer` and verify `matches` serves the response shape, not your action. Apply your own cut-off to `probability`.

Route into three bands per action: **act** above a high bar, **confirm** in the middle (act, but show a person), **escalate** below (queue, or send to a large model). Raise the bar with the blast radius: a tag can start at 0.60, a customer-visible action at 0.90, money or deletion never auto-acts. These are starting points. Replace them with numbers from a golden set (milliseconds-evaluate). Change a label, a hint or a description and every number moves. Re-tune after any wording change.

## Batch the cheap axis

- `texts` (all capabilities): up to 32 inputs, response `{"results":[...]}` in input order. Each text is one inference call.
- `statements` (yes-no) and `questions` (answer): up to 32 per text, **one** inference call. Three questions cost 0.40 s, the same as one.
- Both axes nest: `results[textIndex].results[statementIndex]`. Offsets index the text at the same position, never a joined string.
- Cost is the request body as compact JSON at $0.04 per million input tokens, output free. One token is 3.8 characters, rounded up once per request. Read `x-input-tokens` on every response. `x-inference-ms` is the model time, summed over the calls the request made.
- Batch and throttle anything that loops over the API. Rate limits are 30 requests and 500,000 input tokens per minute for test keys, and from 1,000 requests and 1,000,000 tokens per minute for production tiers, per organization and shared by every key of that type. A batch counts one request per item: 24 `texts` are 24 requests. Watch `x-ratelimit-remaining-requests` and `x-ratelimit-remaining-tokens`.

## Long text

`text` takes up to 20,000 characters. Above 2,000 the API chunks on whitespace. Classifier scores take the max over chunks, so a late claim still scores high, but a short decisive sentence inside long off-topic prose gets diluted. Split on the structure you have (paragraphs, messages, rows) and send parts as `texts` when you need per-part verdicts. Send the whole document for `entities` and for a document-level verdict. Cut signatures and quoted history first.

## Images

Every capability also reads one image. Send `image` instead of `text`, or with `text` as extra context.

- `image`: a data URL `data:image/jpeg;base64,...` (`png` and `webp` too) or bare base64 of a JPEG, PNG or WebP. Bytes only. The API never fetches a URL, and an `http(s)` value is a 400.
- One image per request. Send `image` alone, or together with `text` as context. `texts` together with `image` is a 400 (`image_with_texts`).
- 5 MB decoded. Over that is a 400 `image_too_large`; an undecodable image is a 400 `invalid_image`. Both are rejected before the model runs.
- `detail` sets the longest edge the model reads: `low` 512 px, `medium` 768 px (the default), `high` 1024 px. Higher reads small print better and costs more.
- The rest of the body keeps its text meaning: `statement(s)`, `labels`, `tree`, `scale`, `question(s)`, `schema`, `types`, `field` and `value` describe the image instead of a text.

```bash
curl -s https://api.milliseconds.ai/v1/decision-machine-1/classify \
  -H 'content-type: application/json' -H "authorization: Bearer $MS_API_KEY" \
  -d "{\"image\":\"$(base64 scan.jpg | tr -d '\n')\",\"detail\":\"medium\",\"labels\":{
        \"invoice\":\"a bill with amounts due\",
        \"receipt\":\"proof of a completed payment\",
        \"letter\":\"correspondence in prose\"}}"
```

The base64 never enters the character count. Billed input tokens are the tokens of the body without the image, plus 196, plus a fixed number per image:

| capability | low | medium | high |
| --- | --- | --- | --- |
| `yes-no`, `classify`, `classify-tree`, `rate` (x1) | 1,000 | 2,000 | 4,000 |
| `answer` (x1.5) | 1,500 | 3,000 | 6,000 |
| `extract`, `entities`, `verify` (x2) | 2,000 | 4,000 | 8,000 |

These prices are final. The OpenAI facade bills every image at the `extract` rate, x2. `extract` with an `image` accepts at most 5 fields in the schema; a larger schema is a 400 (`image_schema_too_large`). The image bills once per request. Read `x-input-tokens` on every response for the number actually billed. Rate limits count those same tokens; an image request is one request.

### Offsets on images

An image carries no character offsets. `answer` returns `start` and `end` as `null`, and `entities` returns `start`, `end` and `probability` as `null` on every item. Every other field keeps its text meaning.

### Privacy

Images are processed in memory on the GPU machine. They are never written to disk and never logged; the runner logs the task key and the top probability only. Same zero-retention terms as text.

### Accuracy, measured

Photo classification over ten dish classes scored 100 % top-1. Document-type classification over sixteen scanned types scored 61 % top-1, so route documents through a confidence band, not a bare argmax. Short answers over document images scored 0.90 ANLS; receipt extraction scored 74 % on flat fields and 0.82 F1 on line items. Build a golden set of your own images before you set a threshold.

## Errors and retries

One envelope: `{"error":{"code":"...","message":"..."}}`.

| Status | Code | Action |
| --- | --- | --- |
| 400 | `invalid_request`, `invalid_schema`, `unsupported_request` | Fix the body. `body: provide text or texts, not both` also fires when you send neither. |
| 400 | `invalid_image`, `image_too_large`, `image_with_texts` | Fix the image: bytes of a JPEG/PNG/WebP, under 5 MB decoded, not alongside `texts`. |
| 400 | `image_schema_too_large` | An `extract` schema on an image holds more than 5 fields. Split the schema, or extract from parsed text. |
| 401 | `missing_api_key`, `invalid_api_key` | Fix the key. |
| 429 | `rate_limit_exceeded` | Wait `retry-after` seconds, then retry. |
| 429 | `insufficient_quota` | No credits. Do not retry on a timer. |
| 502 | `runner_error` | Retry once or twice. |
| 529 | `overloaded` | Backpressure, not a fault. Exponential backoff with jitter, lower concurrency. |

A batch is one request: one failure fails every item, so keep batches small enough that a retry is cheap.

## Patterns to reach for

| Problem | Pattern | Docs |
| --- | --- | --- |
| One threshold blocks cheap actions and lets risky ones through | Confidence routing: a band per action | /patterns/confidence-routing.md |
| Prompt-based safety checks cost an LLM call per message | LLM guardrails: one batched yes-no in, one out | /patterns/llm-guardrails.md |
| Extracted fields land in the database unbacked | Extract, then verify the risky fields | /patterns/extract-then-verify.md |
| Every input pays large-model latency | Cascade: decide the easy cases, escalate the rest | /patterns/cascade.md |
| One broad judgment hides why the score moved | Composite scoring: atomic rate calls, weights in code | /patterns/composite-scoring.md |
| A router prompt drifts and needs parsing | Intent routing: classify over described labels | /patterns/intent-routing.md |

Finished recipes with labels and thresholds chosen: support triage, invoice extraction, content moderation, RAG passage filtering, PII detection, taxonomy classification, under https://docs.milliseconds.ai/recipes/overview.md.

## Working method

1. Start from the behaviour the application needs: what it shows, selects, changes or hands off. Work back to the smallest judgments. Keep rules, lookups and execution in code.
2. Pick the capability from the answer shape. Read its page.
3. Write the labels, statements or schema as case descriptions. Put them in one file.
4. Run five to ten real inputs by hand and read every number, not only the winner.
5. Set thresholds per action. Build a golden set of 50 to 200 real examples before tuning anything.
6. Log `x-input-tokens`, `x-inference-ms` and the raw response for every decision you act on.

Cookbook thresholds and demo numbers are examples to measure against, not defaults. Any number you publish, reproduce by measurement first.
