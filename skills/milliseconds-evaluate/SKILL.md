---
name: milliseconds-evaluate
description: |
  Measure and tune a decision-machine-1 integration: build a golden set of real labelled examples, score it in batches, sweep thresholds for precision and recall, pick one cut-off per action, and monitor drift with the response headers. Use before changing any label wording or threshold, and whenever someone wants to publish an accuracy or margin number.
---

# Evaluate and tune

A threshold you guessed is a guess. A threshold you measured is a decision. Probabilities are the model's scores, not measured accuracy: 0.94 is not a promise that 94 of 100 are right. Every wording change moves every number, so measure before and after.

## 1. Build a golden set first

50 to 200 real examples from production traffic, labelled by hand, JSON Lines:

```json
{"id":"t-001","text":"I was charged twice for my subscription this month and support has not replied.","expected":"billing"}
{"id":"t-004","text":"The invoice total looks wrong and my package is late.","expected":"billing","note":"two problems; billing is the action we want"}
```

- Under 50 is too small: one wrong example moves accuracy by more than 2 points.
- At least 10 examples per label, level or field. Keep the near misses; they decide the threshold.
- Two people labelling the same 20 rows exposes an unclear definition.
- `expected` per capability: a label name (classify), `true`/`false` (yes-no), a level index (rate), the exact span text (answer), a field value (extract).
- Keep the golden set in the repo next to the label constants. Diff it in review.

Send one row by hand before scoring 200. Check the labels and the response shape.

## 2. Score once, threshold later

Send the set in batches of 32 `texts` and keep the raw `probability` (and `confidence`, `score`) per row. Do not threshold inside the loop.

```python
import json, requests, os
URL = "https://api.milliseconds.ai/v1/decision-machine-1/yes-no"
H = {"authorization": f"Bearer {os.environ['MS_API_KEY']}"}
golden = [json.loads(l) for l in open("golden.jsonl")]
scored = []
for i in range(0, len(golden), 32):
    chunk = golden[i:i+32]
    r = requests.post(URL, headers=H, json={"texts": [g["text"] for g in chunk],
                                            "statement": "The customer needs help urgently."}, timeout=30)
    r.raise_for_status()
    for g, res in zip(chunk, r.json()["results"]):
        scored.append({**g, "probability": res["probability"]})
```

Throttle: 200 requests and 1,000,000 input tokens per minute per organization, shared across keys. A batch counts one request per item (`texts`, `statements`, `questions`). Read `x-ratelimit-remaining-*`. Retry 502 and 529 with backoff; honour `retry-after` on 429.

## 3. Sweep the cut-off

For each candidate threshold from 0.05 to 0.95, compute precision (of the cases acted on, how many were right) and recall (of the cases needing the action, how many were caught). Print the table. Then decide which error hurts more before you read it:

- **Auto-act** cares about precision. A false positive reaches a customer.
- **Screening** cares about recall. A false negative escapes the net.

Pick one number per action, not per system. Put it in the constants file with the date and the golden-set size. For classify and rate, sweep `confidence` as a second axis. For rate, sweep `score` rather than `level`.

## 4. Compare wordings the same way

Label descriptions, hints and scale text are the biggest lever. Score the set with each candidate wording and keep the one with the better table. A confident wrong answer looks exactly like a confident right one; only the golden set shows which statement was vague.

## 5. Monitor in production

Log per decision: the capability, the constants version, the full response (`scores`, `found`, `levels`), `x-input-tokens`, `x-inference-ms`, and the band you routed to. Watch:

- The share of decisions per band. A growing escalate band means drift in the inputs or a stale label set.
- `confidence` distribution on classify and rate.
- 429 and 529 counts. 529 is backpressure: lower concurrency.
- Sum of `x-input-tokens` against the console balance.

Sample the confirm band into the golden set every week. Docs: /evaluate/monitoring.md.

## Publishing numbers

Any accuracy, latency, cost or margin figure that goes on a page or in a doc must be reproduced by measurement first, with the date, the input sizes and the run count. Latency figures in the docs are medians of 7 runs from a laptop on 2026-09-18 and include network; model time alone is 36 to 54 ms per call.

Full pages: https://docs.milliseconds.ai/evaluate/golden-sets.md, /evaluate/tuning-thresholds.md, /evaluate/monitoring.md

## With the SDK and dm1

The SDK keeps the same batch of 32 and adds the usage of each call, so a scoring run reports its own cost. The result type carries the statement you sent. Scoring from a shell needs no script at all.

```ts
const chunk = golden.slice(0, 32) // one call per 32 texts
const { result, usage } = await dm
  .yesNo(chunk.map((g) => g.text), 'The customer needs help urgently.')
  .withUsage()
// result: YesNoResult<'The customer needs help urgently.'>[], one per text, in order
const scored = chunk.map((g, i) => ({ ...g, probability: result[i].probability }))
usage.inputTokens // what this chunk billed
```

```python
chunk = golden[:32]
r = dm.yes_no([g["text"] for g in chunk], "The customer needs help urgently.")
# r: Results[YesNoResult]. A list, plus r.usage.
scored = [{**g, "probability": res.probability} for g, res in zip(chunk, r)]
r.usage.input_tokens
```

```sh
dm1 yes-no --lines golden.txt "The customer needs help urgently." --jsonl > scored.ndjson
# one line per text: {"text":"…","results":[{"statement":…,"answer":true,"probability":0.91}]}
```

`--lines` sends the file in chunks of 32 in order. Each JSON line repeats the input text, so join `scored.ndjson` to your labels on `text`. The statements sit under `results`, one entry per statement.
