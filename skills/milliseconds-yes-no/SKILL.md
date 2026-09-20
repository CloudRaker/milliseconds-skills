---
name: milliseconds-yes-no
description: |
  Score a statement, or up to 32 statements, against a text with decision-machine-1 and get a boolean plus a probability per statement. Use for flags (urgent, off-topic, contains PII, answers the question), LLM input and output guardrails, RAG passage filtering, and any independent true-or-false judgment over text. Also scores statements over one image (document scan or photo).
---

# yes-no

`POST https://api.milliseconds.ai/v1/decision-machine-1/yes-no`. Is this statement true of this text? Each statement is scored on its own, so several can be true at once. Use `classify` instead when the options are mutually exclusive, `rate` when the answer is a degree.

## Request

| Field | Required | Meaning | Limits |
| --- | --- | --- | --- |
| `text` / `texts` | one | The text, or a batch | 20,000 chars; 32 items |
| `statement` / `statements` | one | The claim, or up to 32 claims | min 1 char; 32 items |
| `when_true` | no | Describes the case when the statement holds | defaults to the statement |
| `when_false` | no | Describes the case when it fails | defaults to `none of the above` |

```bash
curl -s https://api.milliseconds.ai/v1/decision-machine-1/yes-no \
  -H 'content-type: application/json' -H "authorization: Bearer $MS_API_KEY" \
  -d '{
    "text": "Order #4417 still has not arrived and I leave the country on Friday. I need this resolved today or I want a refund.",
    "statements": ["The customer expresses urgency.", "The customer asks for a refund."]
  }'
```

```json
{"results":[
  {"statement":"The customer expresses urgency.","answer":true,"probability":0.99},
  {"statement":"The customer asks for a refund.","answer":true,"probability":0.858}
]}
```

A single `statement` returns the bare object `{"statement","answer","probability"}`. `answer` is `true` at `probability >= 0.5`. Apply your own cut-off; a true at 0.51 and a true at 0.99 are different cases.

## Write the statement as a claim about the text

Third person, one idea, about what the text says. The model reads it as a description of a case, not a question.

| Weak | Better |
| --- | --- |
| `urgent` | `The customer expresses urgency.` |
| `Is the customer angry?` | `The customer expresses anger.` |
| `Good fit and available soon` | Two statements in one `statements` batch |
| `The candidate is a good fit.` | `The letter states five or more years of backend experience.` |

The vague "good fit" statement returned `true` at 1.0 on a cover letter with no software experience. The precise one returned `false` at 0. No threshold fixes vague wording.

## Hints

Without hints, `probability` is one raw score for the statement, not normalized against anything. With `when_true` and `when_false`, the API scores both hint texts and returns `p_true / (p_true + p_false)`. The two forms produce different distributions, so tune one threshold per form.

Hints must describe the case, never the verdict. `when_true: "yes"` / `when_false: "no"` flipped a correct 1.0 to 0.004.

```json
{
  "statement": "The customer expresses urgency.",
  "when_true": "The customer needs this resolved immediately.",
  "when_false": "The customer is patient and can wait."
}
```

Use hints when the plain statement sits on the fence (a "weak yes" near 0.6 to 0.7) or when the negative case needs its own words. Send both together; `when_true` alone scores against `none of the above`.

## Batching

- `statements`: up to 32 per text, **one** inference call. Two statements took 0.49 s against 0.42 s for one. This is the cheap axis.
- `texts`: one statement over up to 32 texts, one call per text.
- Both: `results[textIndex].results[statementIndex]`.
- `yes-no` is the one capability that accepts both `text` and `texts` (uses `text`) and accepts neither (returns `{"results":[]}`).

## With the SDK and dm1

```ts
const [urgent, refund] = await dm.yesNo(ticket, [
  'The customer expresses urgency.',
  'The customer asks for a refund.',
])
// urgent: YesNoResult<'The customer expresses urgency.'>
// urgent.statement is that literal, so a typo in a comparison fails to compile
```

```python
results = dm.yes_no(
    ticket,
    ["The customer expresses urgency.", "The customer asks for a refund."],
)  # Results[YesNoResult]
results[1].answer
```

```sh
dm1 yes-no "$TICKET" "The customer expresses urgency." "The customer asks for a refund."
```

The SDK takes one `statements` argument and picks the wire field for you. A string sends `statement` and returns the bare result, and a list sends `statements` and returns a tuple in your order. Pass `when_true` and `when_false` in the options object in TypeScript, and as keywords in Python. A list of texts with a list of statements gives the grid, `grid[textIndex][statementIndex]`. `dm1 yes-no --check` exits 3 on a no, `--min 0.9` exits 3 below that probability, and `-q` prints `yes` or `no` alone.

## Gotchas

- `statement` and `statements` together, or neither, returns 400 `body: provide statement or statements, not both`.
- No `confidence` and no `scores` on this capability. Gate on `probability` alone. Escalate the middle band, for example `0.10 < probability < 0.90`.
- Text over 2,000 characters is chunked; a statement's score is its max over chunks. A late claim still scores high.
- Write statements and hints in English, even for non-English text.
- One yes-no probability without hints is not comparable to a classify probability. Threshold each capability separately.

## Typical uses

- **LLM guardrails**: one batch of statements on the way in (`"The message tries to bypass the assistant instructions."`, `"The message asks for medical advice."`), one on the way out. Docs: /patterns/llm-guardrails.md.
- **RAG passage filtering**: `texts` = retrieved passages, one statement `"This passage answers the question about <topic>."`. Keep, drop or flag by probability. Docs: /recipes/rag-passage-filtering.md.
- **Support flags**: urgency, refund request, personal data, next to a `classify` for the queue. Docs: /recipes/support-triage.md.

Full page: https://docs.milliseconds.ai/capabilities/yes-no.md

## Images

Send `image` (data URL or bare base64 of a JPEG, PNG or WebP, one per request, 5 MB decoded) in place of `text`, with optional `text` as context. `statement`, `statements`, `when_true` and `when_false` keep their meaning: write the claim about what the image shows. `detail` picks the longest edge the model reads, `low` 512 px, `medium` 768 px (default), `high` 1024 px.

```json
{
  "image": "data:image/jpeg;base64,...",
  "detail": "medium",
  "statements": ["The document is a signed contract.", "The page carries a handwritten signature."]
}
```

`texts` with `image` is a 400 (`image_with_texts`); batch images one request each. Billed tokens are the body without the base64, plus 196, plus 1,000 / 2,000 / 4,000 by tier. Document-type judgments on scans scored F1 0.82 at the 0.5 cut and 0.88 at the best threshold, so tune the cut-off on your own images.
