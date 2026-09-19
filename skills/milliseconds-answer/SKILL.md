---
name: milliseconds-answer
description: |
  Ask a question of a text with decision-machine-1 and get back the span that answers it, with character offsets and a probability, or null when nothing fits. Use when you need one stated value plus where it came from: citations, highlighting, linking back to the source, or a few questions over the same text in one call.
---

# answer

`POST https://api.milliseconds.ai/v1/decision-machine-1/answer`. Extractive: the answer is always a span of your text, never generated words. Use `extract` when you want a whole typed record, `entities` when you want every span of a type, `yes-no` when the question is a verdict rather than a value.

## Request

| Field | Required | Meaning | Limits |
| --- | --- | --- | --- |
| `text` / `texts` | one | The text, or a batch | 20,000 chars; 32 items |
| `question` / `questions` | one | One question, or up to 32 over one text | min 1 char; 32 items |

```bash
curl -s https://api.milliseconds.ai/v1/decision-machine-1/answer \
  -H 'content-type: application/json' -H "authorization: Bearer $MS_API_KEY" \
  -d '{
    "text": "Order 88214 shipped on 3 March 2026 from the Berlin warehouse. The customer, Maria Fischer, paid 249.00 EUR by credit card.",
    "questions": ["Who is the customer?", "When did the order ship?"]
  }'
```

```json
{"results":[
  {"question":"Who is the customer?","answer":"Maria Fischer","probability":1,"start":77,"end":90},
  {"question":"When did the order ship?","answer":"3 March 2026","probability":0.998,"start":23,"end":35}
]}
```

- `answer`: the span, or `null`. `text.slice(start, end) === answer`.
- `probability`: span confidence. `0` when `answer` is `null`.
- `start`, `end`: character offsets into the text you sent. Both `null` when `answer` is `null`. Check for `null` before you slice.

## The probability is the signal, not the presence of a span

Asked `"What is the tracking number?"` of a text with no tracking number, the model returned the order number `88214` at 0.755. A `null` is not guaranteed for an unanswerable question. Treat a low probability and a `null` as the same outcome. Bands that worked on an order text: store at 0.95 and above, review 0.7 to 0.95, drop below 0.7. Raise every band when a wrong value costs money. Pass a value to `verify` before you write it anywhere.

## Ask for the role, not the type

Three questions on `"Invoice 8812 from Meridian Design. Issued 3 March 2026 by Nadia Rossi. Payment is due 2 April 2026."`: `"date"` returned the issue date; `"When is the invoice due?"` returned the due date. A bare noun matches the first thing of that kind. Write a full question about the case. Ask only for values the text states; `"Is this order late?"` has no span.

## Batching

- `questions`: up to 32 over one text, **one** inference call. Three questions measured 0.40 s, the same as one. Every result echoes its `question`.
- `texts`: one question over up to 32 texts, one call per text. Offsets index each result's own text.
- Both: `results[textIndex].results[questionIndex]`.

## With the SDK and dm1

The SDK sends the same body and keeps the wire names. A questions array returns one result per question, in order.

```ts
const [who, when] = await dm.answer(text, ['Who is the customer?', 'When did the order ship?'])
// [AnswerResult<'Who is the customer?'>, AnswerResult<'When did the order ship?'>]
if (who.answer !== null) text.slice(who.start, who.end) // start and end narrow to number
```

`AnswerResult` is a discriminated union on `answer`, so one `null` check narrows `start` and `end` for you.

```python
r = dm.answer(text, ["Who is the customer?", "When did the order ship?"])
# r: Results[AnswerResult]
span = r[0].span  # tuple[int, int] | None
if span is not None:
    text[span[0] : span[1]]
```

Python cannot narrow three fields from one check, so read the offsets through the `.span` property.

```sh
dm1 answer "$TEXT" "Who is the customer?" "When did the order ship?"
# columns: answer, probability, start-end, question
curl -s https://example.com/order.txt | dm1 answer - "Who is the customer?" --json
```

`dm1` prints the offsets as `start-end`, and a `-` for a null answer. Pass `-` as the text to read stdin.

## Gotchas

- `question` and `questions` together, or neither: 400 `body: provide question or questions, not both`.
- A body with a question but no text returns `{"results":[]}` and scores nothing.
- Over 2,000 characters the text is chunked; offsets still point into the text you sent.
- The extractor is multilingual: the span comes back in the source language, offsets stay correct across accented characters. Write the question in English. It copies `3 avril` or `3.480,00 EUR` verbatim and never converts formats. Normalise in code.

Full page: https://docs.milliseconds.ai/capabilities/answer.md
