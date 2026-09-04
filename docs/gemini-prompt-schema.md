# Gemini Receipt-Extraction Prompt & Schema

## Request shape

`POST https://generativelanguage.googleapis.com/v1beta/models/{GEMINI_MODEL}:generateContent`

```json
{
  "contents": [{
    "parts": [
      { "text": "<prompt below>" },
      { "inline_data": { "mime_type": "image/jpeg", "data": "<base64 photo>" } }
    ]
  }],
  "generationConfig": {
    "response_mime_type": "application/json",
    "temperature": 0.1
  }
}
```

`response_mime_type: application/json` constrains Gemini's output to valid
JSON — the workflow still defensively strips ```` ```json ```` fences and
`try/catch`-parses the result, since a model can still return an empty or
truncated body on a genuine failure (e.g. rate limit, timeout).

## Prompt template (fields are read from `RECEIPT_FIELDS`)

```
You are extracting structured data from a photo of a retail receipt.
Return ONLY a single JSON object (no markdown fences, no commentary) with
exactly these fields: merchant_name, amount, currency, date, receipt_number,
tax, subtotal, payment_method, confidence.
- amount, tax and subtotal must be numbers (no currency symbols).
- date must be ISO format YYYY-MM-DD if legible, otherwise null.
- confidence is your own 0-1 estimate of extraction reliability.
- Use null for any field you cannot read.
```

## Expected response schema

```json
{
  "merchant_name": "Benjamin",
  "amount": 1250.50,
  "currency": "PHP",
  "date": "2026-09-04",
  "receipt_number": "INV-12345",
  "tax": 133.98,
  "subtotal": 1116.52,
  "payment_method": "Cash",
  "confidence": 0.94
}
```

## Validation rules applied in "Code — Validate Gemini Response"

1. The Gemini HTTP call itself must succeed (`continueOnFail` lets the
   workflow keep going and flag the row instead of crashing).
2. The response must contain a non-empty text part.
3. That text, after stripping markdown fences, must parse as JSON.
4. `merchant_name` and `amount` are treated as required — a row missing
   either is marked `Processing Status = Error` even if the JSON parsed
   successfully (covers the "missing amount" / "missing merchant" test
   cases).
5. Numeric fields (`amount`, `tax`, `subtotal`, `confidence`) are coerced
   with `Number(...)`; anything that isn't a valid number becomes `null`
   rather than silently becoming `NaN` or a string in the sheet.

Any failure at steps 1–4 still produces exactly one appended row (status
`Error`, with a human-readable `errorMessage`) and one Telegram reply asking
the user to retake the photo — it never throws the execution.
