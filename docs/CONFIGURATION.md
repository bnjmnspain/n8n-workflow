# Configuration Reference

Every configurable value from the assessment maps to one environment
variable, read with a sane fallback default so the workflow still runs
out-of-the-box in a test n8n instance. Set these under **Settings → Environment
Variables** (self-hosted) or your n8n Cloud project's environment settings.

## Workflow 1 — Gmail Multi-Label

| Variable | Purpose | Default if unset |
|---|---|---|
| `GMAIL_LABELS` | Comma-separated list of Gmail labels to poll | `label1,label2,label3,Invoices,Receipts,Documents` |
| `GMAIL_POLL_INTERVAL_MINUTES` | How often the Schedule Trigger fires (set directly on the node — n8n's Schedule Trigger interval isn't expression-driven, so this is documented here for reference; edit the node's `minutesInterval` field) | `10` |
| `GMAIL_SHEET_ID` | Google Sheet (spreadsheet) ID for the email log | *(required)* |
| `GMAIL_SHEET_NAME` | Tab name inside that spreadsheet | `Email Log` |
| `GMAIL_DRIVE_ROOT_FOLDER_ID` | Drive folder ID under which `Gmail Attachments/<Label>/<Date>/<Sender>/` is built | *(required)* |
| `GMAIL_LOOKBACK_WINDOW` | Gmail search window, e.g. `1d`, `2d`, `12h` | `1d` |
| `WORKFLOW_TIMEZONE` | Used for date-folder formatting | `Asia/Manila` |
| `DATE_FORMAT` | Reference only — the workflow currently formats dates as `YYYY-MM-DD` internally | `YYYY-MM-DD` |

## Workflow 2 — Telegram Receipts

| Variable | Purpose | Default if unset |
|---|---|---|
| `RECEIPTS_SHEET_ID` | Google Sheet ID for receipt rows | *(required)* |
| `RECEIPTS_SHEET_NAME` | Tab name | `Receipts` |
| `RECEIPTS_DRIVE_ROOT_FOLDER_ID` | Drive folder ID under which `Receipt Photos/<Date>/` is built | *(required)* |
| `GEMINI_API_KEY` | API key for Google Gemini (`generativelanguage.googleapis.com`) | *(required)* |
| `GEMINI_MODEL` | Gemini model name | `gemini-2.0-flash` |
| `RECEIPT_FIELDS` | Comma-separated list of fields Gemini must return | `merchant_name,amount,currency,date,receipt_number,tax,subtotal,payment_method,confidence` |

## Credentials (not environment variables — set via n8n's Credentials UI)

| Credential | Used by |
|---|---|
| Gmail OAuth2 | Gmail search / attachment download nodes |
| Google Sheets OAuth2 | All Sheets read/append nodes |
| Google Drive OAuth2 | Drive upload nodes **and** the Code nodes that call the Drive REST API directly for find-or-create folder logic — attach the credential to each Code node via its "Credential" field so `httpRequestWithAuthentication` can use it |
| Telegram | Telegram Trigger + all Telegram send/file nodes |

## Security Notes

- No credential, token, or API key is ever written into workflow JSON —
  everything above is either an n8n Credential (encrypted at rest by n8n) or
  an environment variable read at execution time via `$env`.
- Do not commit a populated `.env` file. `.env.example` in this folder is a
  template only.
- When taking portfolio screenshots, redact the address bar / credential
  dropdown values and never paste `GEMINI_API_KEY` or bot tokens into
  screenshots or this documentation.
