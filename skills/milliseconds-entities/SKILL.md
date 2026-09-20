---
name: milliseconds-entities
description: |
  Find every span of the types you name in a text with decision-machine-1, each with a probability and character offsets, sorted by position. You define the types (person, email, case_number, drug_name...); there is no fixed taxonomy. Use for PII detection and redaction, highlighting, counting mentions, and any "all occurrences" need. Also finds entities in one image, where offsets and probabilities come back null.
---

# entities

`POST https://api.milliseconds.ai/v1/decision-machine-1/entities`. Every matching span for every type, sorted by `start`. Use `extract` when you want one typed record, `answer` when you want one span per question.

## Request

| Field | Required | Meaning | Limits |
| --- | --- | --- | --- |
| `text` / `texts` | one | The text, or a batch | 20,000 chars; 32 items |
| `types` | yes | Array of type names, or object `type: description` | array 1 to 64; object no count cap |

Prefer the object form. Describe any type whose name alone is ambiguous (`number` matches dates, quantities and ids).

```bash
curl -s https://api.milliseconds.ai/v1/decision-machine-1/entities \
  -H 'content-type: application/json' -H "authorization: Bearer $MS_API_KEY" \
  -d '{
    "text": "Contact Maria Alvarez at maria.alvarez@northwind.example or +1 415 555 0132.",
    "types": {
      "email": "an email address",
      "phone": "a telephone number",
      "person": "the full name of a person"
    }
  }'
```

```json
{"entities":[
  {"type":"person","text":"Maria Alvarez","probability":0.999,"start":8,"end":21},
  {"type":"email","text":"maria.alvarez@northwind.example","probability":0.991,"start":25,"end":56},
  {"type":"phone","text":"+1 415 555 0132","probability":0.996,"start":60,"end":75}
]}
```

- `type`: your type name, copied back. Only requested types come back; an unrequested person name is ignored.
- `text`: the span. `text.slice(start, end)` equals it. Use offsets, not string search: the same string can occur many times.
- `probability`: raw span confidence, not normalized across types. Each entity stands alone. No `confidence` field.
- `start`, `end`: never `null` here.
- No match for any type returns `{"entities": []}`. That is a normal result.

## Threshold per action

| Action | Threshold | Why |
| --- | --- | --- |
| Redact before storage | Keep everything | A missed span leaks data; a wrong redaction costs little |
| Highlight in a UI | 0.5 and above | The reader corrects the rest |
| Write to a database field | 0.9 and above | Route the rest to a person |

## Redaction loop

Sort by `start` descending and replace slices from the end, so earlier offsets stay valid. Duplicate mentions each return their own entity; deduplicate on `text` when you want a set. Docs: /recipes/pii-detection.md.

## Long text

Send the whole document. Above 2,000 characters the extractor scans overlapping windows (384 tokens, 64 overlap) and remaps offsets back to your original string. A 2,469-character capture returned an entity at `start: 2432` that sliced correctly.

## Batching

`texts` up to 32, `{"results":[{"entities":[...]}, ...]}` in input order, offsets per text, one inference call per text.

## With the SDK and dm1

```ts
const found = await dm.entities(text, {
  email: 'an email address',
  phone: 'a telephone number',
  person: 'the full name of a person',
})
// found: Entity<'email' | 'phone' | 'person'>[]
found[0]?.type // that union, not string
```

```python
Kind = Literal["email", "phone", "person"]
TYPES: Final[Mapping[Kind, str]] = {
    "email": "an email address",
    "phone": "a telephone number",
    "person": "the full name of a person",
}
found = dm.entities(text, TYPES)  # Results[Entity[Kind]]
found.usage.input_tokens
```

```sh
dm1 entities "$TEXT" email="an email address" phone="a telephone number" \
  person="the full name of a person" --json | jq -r '.[] | select(.type=="person") | .text'
```

The call returns a plain array of entities, not an envelope. Your type names become the `type` union, so a typo fails to compile. The SDK accepts 1 to 64 types in the array form, where `classify` needs 2 labels. Python returns `Results[Entity[Kind]]`, a list that also carries `.usage`; `dm1 --json` prints the same array for `jq`.

## Gotchas

- Empty `types` array: 400 `types: Too small: expected array to have >=1 items`.
- Multilingual: English type descriptions work on French, German and other text; offsets stay correct across accented characters.
- Typical latency 0.42 s for 3 types on a short text.

Full page: https://docs.milliseconds.ai/capabilities/entities.md

## Images

Send `image` (data URL or bare base64 of a JPEG, PNG or WebP, one per request, 5 MB decoded) in place of `text`, with optional `text` as context. `types` keeps its meaning, descriptions included. `detail` picks the longest edge, `low` 512 px, `medium` 768 px (default), `high` 1024 px.

The result keeps its shape, the same flat array of items. Character offsets do not apply on an image, so `start` and `end` are `null`. `probability` is `null` too: the generation path gives no per-span logprob. Filter image entities on `type` and `text`, or on your own rules, never on a probability threshold. The threshold table above applies to text only.

```json
{"entities":[
  {"type":"person","text":"Dana Whitfield","probability":null,"start":null,"end":null}
]}
```

`texts` with `image` is a 400 (`image_with_texts`). Billed tokens are the body without the base64, plus 196, plus 1,000 / 2,000 / 4,000 by tier, once per request. Read `x-input-tokens`.
