---
name: milliseconds-extract
description: |
  Fill a JSON Schema from a text with decision-machine-1 and get a typed object back, with null for every field the text does not carry. Use for invoices, orders, résumés, confirmations, form-like emails: several fields from one document in one 0.4 s call, values coerced to your types. Pair with verify before writing money or identity fields. Also extracts from one image (receipt, invoice or form scan), with a bounding box per field.
---

# extract

`POST https://api.milliseconds.ai/v1/decision-machine-1/extract`. Send `text` and a JSON Schema; get `data` in that shape. Every value is a span from the text, coerced to your type. Use `answer` for one field plus offsets, `entities` for every occurrence of a type, `verify` to check a value you already hold.

## Request

| Field | Required | Meaning | Limits |
| --- | --- | --- | --- |
| `text` / `texts` | one | The text, or a batch | 20,000 chars; 32 items |
| `schema` | yes | JSON Schema, `type: "object"` with `properties` | no size limit |

Write a `description` on every property. It is what the model searches for. Without one the API falls back to `title`, then to the property name with underscores as spaces, so `invoice_number` works and `n1` does not.

```bash
curl -s https://api.milliseconds.ai/v1/decision-machine-1/extract \
  -H 'content-type: application/json' -H "authorization: Bearer $MS_API_KEY" \
  -d '{
    "text": "INVOICE #4471\nBilled to: Acme Corp\nCurrency: USD\nPaid: yes\nTotal due: $2,676.00\nPayment terms: Net 30",
    "schema": {
      "type": "object",
      "properties": {
        "invoice_number": {"type": "string", "description": "invoice number"},
        "total_due": {"type": "number", "description": "total amount due"},
        "paid": {"type": "boolean", "description": "is the invoice paid"},
        "currency": {"enum": ["USD", "EUR", "GBP"], "description": "currency code"}
      }
    }
  }'
```

```json
{"data":{"invoice_number":"4471","total_due":2676,"paid":true,"currency":"USD"}}
```

Only `data` comes back. **No probability on this endpoint.** A missing value is `null`; the key stays. A batch wraps results: `{"results":[{"data":{...}}, ...]}`.

## Schema support

| Construct | Behaviour |
| --- | --- |
| `string`, or no `type` | The raw span |
| `number` | Non-numeric characters stripped: `$2,676.00` → `2676`. Unparseable → `null` |
| `integer` | Same, then truncated (`Net 30` → `30`) |
| `boolean` | `true`, `yes`, `y`, `1` → `true` |
| `enum` | One member; `enum` wins over `type` |
| `array` of `string` | Supported. Order is not stable between calls |
| Array of number or other scalar | Returns strings |
| Nested object | Returned nested |
| Array of objects | Accepted, returned as `[]` for now |
| `["string","null"]` | First non-null type wins |
| `{"type":"null"}` | Key dropped from `data` |
| `oneOf`, `anyOf`, `allOf`, `$ref` | Read as plain string, usually `null` |
| `required` | Parsed, not enforced. Handle `null` in code |

A root schema that is not an object with `properties` returns 400 `invalid_schema`.

## Treat the result as a draft

The model fills a described field with the closest span it finds; it does not refuse. On an invoice with no vendor block, `vendor.name` copied the billing name. For fields that move money or write a record, send the value and the source text to `verify` and check `matches` before you commit. Docs: /patterns/extract-then-verify.md, /recipes/invoice-extraction.md.

## Keep schemas small

Root scalar fields run in groups of four, because one large structure makes fields compete for the same span. Split a wide record into a few focused schemas rather than one giant one.

## Long text

Above 2,000 characters the API chunks and merges: first non-empty value per field across chunks, several records in a chunk concatenate. Keep a label and its value in one chunk: never cut between `Total due:` and the amount. Runs of two or more spaces collapse to ` | ` first, so table cells read as distinct spans.

## Batching

`texts` up to 32 with one schema, one inference call per text. Two texts measured 0.44 s against 0.41 s for one.

## With the SDK and dm1

```ts
const data = await dm.extract(invoiceText, {
  type: 'object',
  properties: {
    invoice_number: { type: 'string', description: 'invoice number' },
    total_due: { type: 'number', description: 'total amount due' },
    paid: { type: 'boolean', description: 'is the invoice paid' },
    currency: { enum: ['USD', 'EUR', 'GBP'], description: 'currency code' },
  },
})
// data: { invoice_number: string | null; total_due: number | null; paid: boolean | null;
//         currency: 'USD' | 'EUR' | 'GBP' | (string & {}) | null }
```

```python
class Invoice(TypedDict):
    invoice_number: str | None
    total_due: float | None
    paid: bool | None
    currency: Literal["USD", "EUR", "GBP"] | None

data = dm.extract(invoice_text, Invoice)  # Invoice; data["total_due"] is float | None
```

```sh
dm1 extract -f invoice.txt --schema @invoice.json --json > invoice.json
```

The TypeScript SDK reads the JSON Schema literal at the type level, so the result types itself. A zod 4.2 schema passes straight in; for valibot or older zod wrap the converter output in `typed<T>()`. The types state the four degradations. A missing value is `null`, an array of objects is `never[]`, and an array of scalars is `string[]`. An enum stays unchecked on the server. Python takes a `TypedDict`, a dataclass or a pydantic model with every field `| None`. A `TypedDict` carries no field descriptions, so pass the dict schema when you need them. `dm1` reads that same dict with `--schema @invoice.json`.

## Gotchas

- Bare field names without descriptions are the common failure. Describe each field in the words the document uses.
- Multilingual: an English schema fills from German text and returns the German spans untouched (`3.480,00 EUR`, `12.05.2026`). Normalise dates, amounts and separators in code.
- Already using an OpenAI client? `response_format.json_schema` on `/v1/chat/completions` maps to this capability but returns a JSON string and no `texts` batching. See milliseconds-openai.

Full page: https://docs.milliseconds.ai/capabilities/extract.md

## Images

Send `image` (data URL or bare base64 of a JPEG, PNG or WebP, one per request, 5 MB decoded) in place of `text`, with optional `text` as context. `schema` keeps its meaning, descriptions included. `detail` picks the longest edge, `low` 512 px, `medium` 768 px (default), `high` 1024 px; raise it for small print.

The response gains `boxes`: an object keyed by the dotted field path, each value `[x1, y1, x2, y2]` integers in the pixels of the image you uploaded. Fields the model did not locate are absent, and `boxes` can come back empty. Boxes are approximate: roughly where the value is, often missing, and a line-item box sits on the item name rather than the price. Point a reviewer at a box; never crop or redact from one without a check. `data` keeps its shape and its `null` for a missing value.

```json
{
  "data": {"invoice_number": "4471", "total_due": 2676, "items": [{"price": 12.5}]},
  "boxes": {"invoice_number": [412, 96, 520, 124]}
}
```

`texts` with `image` is a 400 (`image_with_texts`); one image per request. Billing is provisional on images: the body without the base64, plus 196, plus 5,000 / 10,000 / 20,000 by tier. Read `x-input-tokens`. Measured on receipts: 74 % on flat fields, 0.82 F1 on line items. Do not pre-parse a document to text and extract from that; the pixels carry layout that markdown drops, and the text route lost 20 F1 points on line items.
