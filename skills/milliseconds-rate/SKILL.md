---
name: milliseconds-rate
description: |
  Place a text on an ordered scale with decision-machine-1 and get a probability-weighted score, the winning level, and a confidence. Use for severity, priority, sentiment strength, harm, risk, urgency as a degree, and for composite scoring where several atomic ratings are weighted in code. Also rates one image (document scan or photo).
---

# rate

`POST https://api.milliseconds.ai/v1/decision-machine-1/rate`. You send 2 to 10 levels ordered low to high. The model scores every level; you get a continuous `score`, an argmax `level`, its `label`, a `confidence` and the full `scores` array. Use `classify` when the options have no order. Use `yes-no` when one threshold is the whole decision.

## Request

| Field | Required | Meaning | Limits |
| --- | --- | --- | --- |
| `text` / `texts` | one | The text, or a batch | 20,000 chars; 32 items |
| `scale` | yes | Level descriptions, low to high | 2 to 10 items |

```bash
curl -s https://api.milliseconds.ai/v1/decision-machine-1/rate \
  -H 'content-type: application/json' -H "authorization: Bearer $MS_API_KEY" \
  -d '{
    "text": "Our production database is down and every customer request is failing right now.",
    "scale": ["No impact", "Minor inconvenience", "Degraded service", "Major outage"]
  }'
```

```json
{"score":2.981,"level":3,"label":"Major outage","confidence":0.935,"scores":[0.001,0.002,0.013,0.984]}
```

- `score`: `Σ pᵢ · i`, from 0 to `scale.length − 1`. Not a percentage: divide by `scale.length − 1` for a 0 to 1 number.
- `level`: index of the most likely level. `label`: `scale[level]`.
- `confidence`: `1 − normalized entropy` over `scores`.
- `scores`: an **array** in scale order, summing to 1. No `probability` field on this capability.

## Route on score and confidence, not level

A near tie reads like this: `{"score":1.585,"level":2,"label":"Frustrated","confidence":0.371,"scores":[0,0.471,0.472,0.057]}`. `level` won by 0.001 and can flip between calls. `score` keeps the ordering information and moves smoothly. `confidence` 0.371 says review, not act.

| `score` | `confidence` | Action |
| --- | --- | --- |
| high | high | Act |
| high | low | Review queue |
| low | any | No action, or the low-severity path |

## Write the scale

- Descriptions, not numbers. `["1","2","3","4"]` gave confidence 0.247 on an angry cancellation threat; `["calm and factual","mildly annoyed","clearly frustrated","angry and threatening to cancel"]` ruled out the calm levels and reported 0.5 honestly.
- One axis. Equal steps between neighbours. Four or five levels is usually enough.
- Adjacent levels that overlap in meaning produce ties and low confidence.
- English levels, even for non-English text. Graded judgments are the weakest case on other languages: an angry Japanese complaint split evenly over three levels. Use a confidence floor and route the rest to a person.

## One rate call answers one question

Split a broad judgment such as "lead quality" into separate calls for budget, authority and fit, then weight the scores in your own code. One vague scale gives a weak confidence; several atomic scales give a stable number and an audit trail. Changing a weight needs no new inference. Docs: /patterns/composite-scoring.md.

## Batching

`texts` up to 32 against one scale, `{"results":[...]}` in input order, one inference call per text.

## With the SDK and dm1

```ts
const SCALE = ['No impact', 'Minor inconvenience', 'Degraded service', 'Major outage'] as const

const r = await dm.rate('Our production database is down and every customer request is failing right now.', SCALE)
// r: RateResult<typeof SCALE>
// r.level: 0 | 1 | 2 | 3. r.label: the four literals. r.scores: [number, number, number, number]
if (r.score / (SCALE.length - 1) > 0.8 && r.confidence > 0.7) page()
```

```python
from typing import Final, Literal, Sequence

Severity = Literal["No impact", "Minor inconvenience", "Degraded service", "Major outage"]
SCALE: Final[Sequence[Severity]] = [
    "No impact",
    "Minor inconvenience",
    "Degraded service",
    "Major outage",
]

r = dm.rate("Our production database is down and every customer request is failing right now.", SCALE)
# r: RateResult[Severity]. r.label is that union. r.score and r.confidence are floats.
```

```sh
dm1 rate "Our production database is down and every customer request is failing right now." \
  "No impact" "Minor inconvenience" "Degraded service" "Major outage" --min-confidence 0.7
# label, score, level, confidence, then one score per level
```

The `as const` scale gives a level literal union and a fixed-length `scores` tuple. Python solves the same union from the `Sequence[L]` annotation; without it you get `RateResult[str]`. Route on `score` and `confidence` in every language, never on `level`. `dm1` takes the levels as positionals low to high, and gates on `--min-confidence`.

## Gotchas

- Scale of one item: 400 `scale: Too small: expected array to have >=2 items`. Over ten: `Too big: expected array to have <=10 items`.
- Index 0 is the low end. An unsorted scale makes `score` meaningless.
- Over 2,000 characters a level's score is its max over chunks, so one severe paragraph lifts a long document's rating.
- Printed `scores` are rounded to 3 decimals; a printed tie resolves on the unrounded values.

## Typical uses

- **Content moderation**: rate harm on a described scale, hard-flag the clear cases, send the middle band to a reviewer. Docs: /recipes/content-moderation.md.
- **Support priority** next to a `classify` queue and `yes-no` flags. Docs: /recipes/support-triage.md.

Full page: https://docs.milliseconds.ai/capabilities/rate.md

## Images

Send `image` (data URL or bare base64 of a JPEG, PNG or WebP, one per request, 5 MB decoded) in place of `text`, with optional `text` as context. `scale` and `question` keep their meaning: order the level descriptions low to high, about what the image shows. `detail` picks the longest edge, `low` 512 px, `medium` 768 px (default), `high` 1024 px.

```json
{
  "image": "data:image/jpeg;base64,...",
  "question": "how legible this scan is",
  "scale": ["unreadable, the text is lost", "readable with effort", "clean and sharp"]
}
```

`texts` with `image` is a 400 (`image_with_texts`). Billed tokens are the body without the base64, plus 196, plus 1,000 / 2,000 / 4,000 by tier. No accuracy ground truth exists for rate on images yet; measure on a golden set before you route on `score`.
