# Google Sheets Templates

Create each spreadsheet/tab with these exact header row values (row 1) — the
Sheets nodes in both workflows map to these column names.

## Workflow 1 — "Email Log" tab

| Message ID | Thread ID | Sender Name | Sender Email | Recipient | Subject | Label | Received Date | Has Attachment | Attachment Count | Drive File Links | Processing Status | Error Message | Processed At |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 18f2a...c1 | 18f2a...00 | Benjamin | espanaflorence@gmail.com | espanaflorence@gmail.com | Invoice #4521 | Invoices | 2026-09-04T03:12:00Z | Yes | 2 | https://drive.google.com/..., https://drive.google.com/... | Success | | 2026-09-04T03:12:41Z |

- **Message ID** is the de-dup key — do not sort/hide this column.
- **Drive File Links** holds one Drive `webViewLink` per attachment, comma-separated.
- **Processing Status** is `Success` or `Error`; `Error Message` is populated only when status is `Error`.

## Workflow 2 — "Receipts" tab

| Telegram Message ID | Chat ID | Submission Date | Merchant | Amount | Currency | Receipt Date | Receipt Number | Subtotal | Tax | Payment Method | Confidence | Google Drive File ID | Processing Status | Processed At |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 5821 | 883921044 | 2026-09-04T04:20:11Z | Benjamin | 1250.5 | PHP | 2026-09-04 | INV-12345 | 1116.52 | 133.98 | Cash | 0.94 | 1AbCdEf... | Success | 2026-09-04T04:20:19Z |

- **Telegram Message ID + Chat ID together** are the de-dup key (a bot token
  can be used across many chats, so message ID alone can collide).
- Any field Gemini could not read comes through as an empty cell rather than
  a guessed value; `Processing Status = Error` flags the row for manual review.
