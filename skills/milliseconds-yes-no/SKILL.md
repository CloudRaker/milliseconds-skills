---
name: milliseconds-yes-no
description: |
  Score a statement, or up to 32 statements, against a text with decision-machine-1 and get a boolean plus a probability per statement. Use for flags (urgent, off-topic, contains PII, answers the question), LLM input and output guardrails, RAG passage filtering, and any independent true-or-false judgment over text.
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
