---
name: milliseconds-classify
description: |
  Put a text into one label from a described set with decision-machine-1 (classify), or walk a nested label tree in one call (classify-tree). Returns the winner, its probability, a confidence number and the full score distribution. Use for intent routing, queue assignment, topic tagging, tool selection in front of an agent, taxonomy classification, and any mutually exclusive choice over text.
---

# classify and classify-tree

`POST https://api.milliseconds.ai/v1/decision-machine-1/classify` picks one label out of 2 to 64. `POST .../classify-tree` runs classify once per level of a nested tree and returns the path. Use `yes-no` instead when the two labels are a claim and its negation. Use `rate` when the labels are ordered.

## classify request

| Field | Required | Meaning | Limits |
| --- | --- | --- | --- |
| `text` / `texts` | one | The text, or a batch | 20,000 chars; 32 items |
| `labels` | yes | Array of names, or object `name: description` | array 2 to 64; object no count cap |

The API sends `"name: description"` to the model. The response returns the **name**. Always use the object form: bare `billing`/`shipping`/`account` returned a 0.5 tie on a ticket that described labels scored at 0.995.

```bash
curl -s https://api.milliseconds.ai/v1/decision-machine-1/classify \
  -H 'content-type: application/json' -H "authorization: Bearer $MS_API_KEY" \
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

- `probability`: the winner's normalized share of the label set. Add or remove a label and every number moves.
- `confidence`: `1 − normalized entropy` over `scores`. 1 is one sharp peak, 0 is flat.
- `scores`: one probability per label name, summing to 1.

## Read both numbers

`"Hi, I have a question about my order."` over the same labels returned `label: billing, probability 0.52, confidence 0.067` with shipping at 0.244 and account at 0.237. The winner looks acceptable; the confidence says it is a coin toss. Rule of thumb:

| `probability` | `confidence` | Action |
| --- | --- | --- |
| high | high | Act |
| high | low | Winner leads a crowded field. Act only on low blast radius, else ask a person |
| low | any | Escalate |

With many labels, compare the top two `scores` before you act. Pick exact cut-offs per action from a golden set (milliseconds-evaluate). The label-set change rule applies: re-tune after any edit to names or descriptions.

## Label writing

- Names short and stable (code reads them); descriptions carry the meaning.
- Mutually exclusive. Two labels that describe the same case tie whatever the text says.
- Add `other` with a description when real inputs fall outside the set.
- English descriptions, even for non-English text. A French label set scored 0.983 where English descriptions scored 0.998 on the same French sentence.

## Batching

`texts` up to 32, one label set, response `{"results":[...]}` in input order. Each text is one inference call. Label count does not change model time; extra labels only add their own length to billed input.

## classify-tree

`tree` maps label name to a description string (leaf) or a node `{description, labels}`. Each level 2 to 64 entries, at most 8 levels deep. The walk always runs to a leaf.

```bash
curl -s https://api.milliseconds.ai/v1/decision-machine-1/classify-tree \
  -H 'content-type: application/json' -H "authorization: Bearer $MS_API_KEY" \
  -d '{
    "text": "I was charged twice for my annual subscription on 3 March and the second charge has not been refunded.",
    "tree": {
      "billing": {
        "description": "payments, invoices, charges, refunds and subscriptions",
        "labels": {
          "duplicate_charge": "the customer was billed more than once for the same item",
          "refund_request": "the customer asks for money back",
          "subscription_change": "the customer wants to upgrade, downgrade or cancel a plan"
        }
      },
      "shipping": "delivery, tracking, lost or damaged parcels",
      "account": "profile, password, plan and team members"
    }
  }'
```

```json
{"path":["billing","duplicate_charge"],"label":"duplicate_charge","probability":0.992,"confidence":0.953,
 "levels":[
  {"label":"billing","probability":0.994,"confidence":0.966,"scores":{"billing":0.994,"shipping":0,"account":0.006},"input_chars":301,"input_tokens":80,"inference_ms":26},
  {"label":"duplicate_charge","probability":0.998,"confidence":0.987,"scores":{"duplicate_charge":0.998,"refund_request":0.001,"subscription_change":0.001},"input_chars":336,"input_tokens":89,"inference_ms":25}
 ]}
```

- Compound `probability` and `confidence` are products over levels, so they fall with depth even on a correct path. Threshold the level you act on, and log `levels`.
- A weak inner node hides the doubt: accept the partial path and send the leaf choice to a person.
- Inner-node descriptions need the cue words their leaves carry. A node described as `reduced blood flow to the heart` lost to `arrhythmia` on an ECG note; adding `chest pain, ST elevation, troponin rise` flipped it.
- Latency is about 0.4 s per level, sequential. Walk the tree in your own code when you need a stop rule per level; the API has none. Docs: /recipes/taxonomy-classification.md.
- Errors: `tree.billing.labels: each level needs 2 to 64 labels`, `tree is deeper than 8 levels`.

## Gotchas

- Array form with one label: 400 `labels: Too small: expected array to have >=2 items`. Object form with one label silently returns probability 1.
- `text` and `texts` together, or neither: 400 `body: provide text or texts, not both`.
- Over 2,000 characters the score is the max over chunks. A short billing sentence inside 2,889 characters of logistics prose never won a chunk; split first when you need per-part verdicts.
- `scores` keys are the exact label names you sent, spaces and punctuation included.

## Typical uses

- **Intent routing** in front of tools or agents: labels are tool descriptions, dispatch on the winner, escalate low confidence. Docs: /patterns/intent-routing.md.
- **Cascade**: act on `probability >= 0.90 && confidence >= 0.70`, send the rest to a large model. Docs: /patterns/cascade.md.
- **Support triage** with a `rate` for priority and `yes-no` flags. Docs: /recipes/support-triage.md.

Full pages: https://docs.milliseconds.ai/capabilities/classify.md, https://docs.milliseconds.ai/capabilities/classify-tree.md
