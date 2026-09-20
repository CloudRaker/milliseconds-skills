---
name: milliseconds-verify
description: |
  Check a value you already hold against a text with decision-machine-1: does the document support this invoice number, this total, this customer name? Returns matches, a probability, and found[], the raw spans the model read. Use as the check step after extract, OCR, a form submission or a CRM sync, before a value moves money or lands in a record. Also verifies a value against one image.
---

# verify

`POST https://api.milliseconds.ai/v1/decision-machine-1/verify`. You send a field, a value and the text. The model pulls what the text says for that field and compares. Use `extract` when you have no value yet, `answer` when you want the value with offsets, `yes-no` when the claim is a sentence rather than a field-value pair.

## Request

| Field | Required | Meaning |
| --- | --- | --- |
| `text` / `texts` | one | The document, or up to 32 documents |
| `field.name` | yes | The field, such as `invoice_number` |
| `field.description` | no | A sentence telling the model what to look for. Use it. |
| `value` | yes | String or number. Numbers are stringified. |

```bash
curl -s https://api.milliseconds.ai/v1/decision-machine-1/verify \
  -H 'content-type: application/json' -H "authorization: Bearer $MS_API_KEY" \
  -d '{
    "text": "INVOICE #4471 - Acme Corp. Total due $2,676.00.",
    "field": {"name": "invoice_number", "description": "The invoice number printed on the document"},
    "value": "4417"
  }'
```

```json
{"matches":false,"probability":0,"found":["INVOICE #4471"]}
```

- `matches`: `true` when `probability >= 0.5`.
- `probability`: confidence of the span that matched your value. Exactly `0` when nothing matches. It measures how sure the model is about the span it read, not how sure it is that you are right.
- `found`: every span the model pulled for that field, match or not. The debugging field.

## Read found[]

Three real outcomes for the same invoice:

| Result | Meaning | Action |
| --- | --- | --- |
| `{"matches":true,"probability":0.689,"found":["4471"]}` | Confirmed | Act above 0.9, confirm 0.5 to 0.9 |
| `{"matches":false,"probability":0,"found":["INVOICE #4471"]}` | The text says something else | Review. This is the transposed-digit catch. |
| `{"matches":false,"probability":0,"found":[]}` | The model saw no such field | Unconfirmed, not contradicted |

`total_due` against `2676` returned `{"matches":true,"probability":0.997,"found":["$2,676.00","$2,500.00","$176.00"]}`. The other money spans are noise from the same field and do not lower the number.

## The matching rule

Both sides are lowercased and stripped of every non-alphanumeric character. Then exact match, or containment when your canonical value is **3 or more characters**. So `$2,676.00` matches `2676`, and `471` matches `4471` too. A one or two character value must match exactly.

Short numeric values produce false positives: a three-digit code, a quantity or a year can sit inside a longer number. Compare `found` to your value in code when the value is short.

## Batching

`texts` up to 32 documents against one `field` and `value`, `{"results":[...]}` in input order. To check several fields, send one call per field. Each text is one inference call.

## With the SDK and dm1

```ts
const r = await dm.verify(
  'INVOICE #4471 - Acme Corp. Total due $2,676.00.',
  { name: 'invoice_number', description: 'The invoice number printed on the document' },
  '4417',
)
// r: VerifyResult -> matches, probability, found
```

```python
r = dm.verify(
    "INVOICE #4471 - Acme Corp. Total due $2,676.00.",
    {"name": "invoice_number", "description": "The invoice number printed on the document"},
    "4417",
)  # r: VerifyResult -> r.matches, r.probability, r.found
```

```sh
dm1 verify "INVOICE #4471 - Acme Corp. Total due \$2,676.00." \
  --field invoice_number="The invoice number printed on the document" \
  --value 4417 --check
```

`field` takes a bare name or `{ name, description }`; a bare name goes on the wire as `{"name": ...}`. `value` takes a string or a number. The result type is the same three fields, so read `found` in code the same way. `dm1` splits `--field name=description` on the first `=`, and `--check` exits 3 when the value does not match.

## Gotchas

- Missing `field`: 400 `field: Invalid input: expected object, received undefined`. Missing `value`: `value: Invalid input`.
- Write `field.description` in English even for non-English text.
- No offsets on this capability; use `answer` if you need them.
- Above 2,000 characters the text is split into overlapping windows and the spans merge.

## The extract-then-verify loop

1. `extract` the record.
2. For each field that carries risk (amount, id, name), `verify` the extracted value against the same text.
3. Act on `matches: true` above your bar. Queue every `matches: false` with a non-empty `found` for a person, with `found` shown next to the value.

Docs: /patterns/extract-then-verify.md, /recipes/invoice-extraction.md.

Full page: https://docs.milliseconds.ai/capabilities/verify.md

## Images

Send `image` (data URL or bare base64 of a JPEG, PNG or WebP, one per request, 5 MB decoded) in place of `text`, with optional `text` as context. `field` and `value` keep their meaning: the field describes what to look for in the image, the value is what you already hold. `detail` picks the longest edge, `low` 512 px, `medium` 768 px (default), `high` 1024 px; raise it for small print.

```json
{"image": "data:image/jpeg;base64,...", "field": "the total amount due", "value": "2676.00"}
```

The result keeps its shape: `matches`, `probability`, `found[]`. `texts` with `image` is a 400 (`image_with_texts`). Billing is provisional on images: the body without the base64, plus 196, plus 5,000 / 10,000 / 20,000 by tier. Read `x-input-tokens`. The extract-then-verify loop works the same way on a scan: extract the fields, then verify the money and identity ones against the same image.
