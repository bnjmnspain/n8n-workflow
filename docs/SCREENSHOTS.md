# Screenshots — Capture Checklist

A reproducible walk-through for producing the `screenshots/` folder referenced
in the main `README.md`. Each entry says **what** to capture, **where** in
the n8n UI to find it, and the **caption** to use for that image.

> **Before you start.** Set the environment variables from
> [`CONFIGURATION.md`](./CONFIGURATION.md) and trigger at least one
> successful run of each workflow so the executions log and the downstream
> Sheets/Drive folders contain real data. A screenshot of an empty sheet or
> an empty folder is not evidence.

> **Redact before sharing.** In every screenshot, blur/redact:
> - Bot tokens, OAuth client secrets, Gemini API key
> - `GMAIL_SHEET_ID`, `RECEIPTS_SHEET_ID`, `GMAIL_DRIVE_ROOT_FOLDER_ID`,
>   `RECEIPTS_DRIVE_ROOT_FOLDER_ID`
> - Personal email addresses and phone numbers in test rows
> - The account email shown in the n8n top-right corner

Save all images under `screenshots/workflow-1/` and `screenshots/workflow-2/`
with the filenames listed below.

---

## Workflow 1 — Gmail Multi-Label

### `01-canvas-main.png` — Main workflow canvas
**Where:** Workflows → open **"Gmail Multi-Label → Sheets + Drive"** → press `F` to fit → hide the minimap (⋮ menu → Hide minimap) for a cleaner shot.
**Must show:** All 14 nodes and the connections from `Schedule — Poll Gmail` through `Sheets — Log Email`. The two branches after `Check — Already Processed?` (Skip vs. Has Attachments?) and after `Check — Has Attachments?` (Run sub-workflow vs. No Attachments) must both be visible.
**Caption:** *Main workflow — Schedule → multi-label Gmail search → de-dup → attachment sub-workflow → Sheets log.*

### `02-canvas-sub.png` — Sub-workflow canvas
**Where:** Workflows → open **"Sub — Upload Gmail Attachments"** → press `F` to fit → hide minimap.
**Must show:** All 10 nodes and the chain `Trigger → Sanitize → Get-or-create Label folder → Date folder → Sender folder → Split → Download → Upload → Aggregate → Build Final Result`.
**Caption:** *Companion sub-workflow — find-or-create `<Label>/<Date>/<Sender>` Drive folders, then download each attachment and upload with its original filename.*

### `03-node-schedule-and-config.png` — Schedule + Config node settings
**Where:** On the main workflow canvas, double-click `Schedule — Poll Gmail` and `Config — Load Settings`. Take one screenshot per node and combine, or take two adjacent shots.
**Must show:**
- Schedule Trigger: the `minutesInterval` value (e.g. `10`).
- Config node: the `raw` JSON with every line beginning `=...` and referencing `$env.GMAIL_LABELS`, `$env.GMAIL_SHEET_ID`, etc.
**Caption:** *All poll/sheet/drive settings are env-driven — no workflow edits needed to change labels or destination.*

### `04-node-gmail-search.png` — Gmail search node settings
**Where:** Double-click `Gmail — Search Messages By Label`.
**Must show:** `operation: getAll`, `simple: false`, `filters.q` showing `=label:"{{$json.label}}" newer_than:{{$json.lookbackWindow}}`, and the Gmail OAuth2 credential selected.
**Caption:** *One Gmail search per label, with `simple: false` to preserve attachment metadata.*

### `05-node-execute-workflow-link.png` — Sub-workflow linkage
**Where:** Double-click `Run — Upload Gmail Attachments`.
**Must show:** `source: database`, the `workflowId` dropdown set to **"Sub — Upload Gmail Attachments"**, `mode: each`.
**Caption:** *This node calls the companion sub-workflow — the dropdown must be re-picked after import because n8n assigns internal IDs at import time.*

### `06-execution-success.png` — Single successful execution
**Where:** Workflow → ⋮ menu → **Executions** → click the most recent successful run → click each node to see its input/output panel.
**Must show:** Green ticks on every node. The output panel of `Gmail — Search Messages By Label` showing the raw message payload (with `payload.parts[].body.attachmentId`). The output panel of `Sheets — Log Email` showing the appended row.
**Caption:** *End-to-end run: Gmail → normalized email → sub-workflow → Drive uploads → Sheets row. Status `Success`.*

### `07-execution-error-path.png` — Error path (optional but recommended)
**Where:** Temporarily set `GMAIL_DRIVE_ROOT_FOLDER_ID` to an invalid value, click **Test Workflow**, then return to Executions.
**Must show:** Red node on `Code — Get Or Create Label Folder`, downstream green nodes still ran, and the `Sheets — Log Email` row was still appended with `Processing Status = Error` and an `Error Message`.
**Caption:** *One bad item never halts the batch — row still logs with `Processing Status = Error`.*

### `08-sheet-email-log.png` — Google Sheet "Email Log"
**Where:** Open the spreadsheet whose ID is in `GMAIL_SHEET_ID` → tab `Email Log`.
**Must show:** The header row (column names from [`google-sheets-templates.md`](./google-sheets-templates.md)) plus several data rows, including:
- At least one row with `Has Attachment = Yes` and a populated `Drive File Links` cell.
- At least one row with `Has Attachment = No`.
**Caption:** *Each email becomes one row. `Drive File Links` holds the webViewLinks for every attachment, comma-separated.*

### `09-drive-folder-tree.png` — Drive folder tree
**Where:** Drive → root folder from `GMAIL_DRIVE_ROOT_FOLDER_ID` → expand to show `Gmail Attachments/<Label>/<YYYY-MM-DD>/<Sender>/`.
**Must show:** Multiple files inside the sender folder. Verify by clicking one file that **Detail** shows the same filename as the original Gmail attachment (this is the "original filename retained" requirement).
**Caption:** *Attachments land in `<Label>/<Date>/<Sender>/` with original filenames preserved.*

### `10-credentials-and-env.png` — Credentials + Environment (optional)
**Where:** **Settings → Credentials** and **Settings → Environment Variables**.
**Must show:** The four credentials (Gmail OAuth2, Google Sheets OAuth2, Google Drive OAuth2 — no Telegram for this workflow) listed by name only, with secret values redacted. The env-var screen with `GMAIL_LABELS`, `GMAIL_SHEET_ID` (redacted), `GMAIL_DRIVE_ROOT_FOLDER_ID` (redacted), etc.
**Caption:** *No secret is ever embedded in workflow JSON.*

---

## Workflow 2 — Telegram Receipt → Gemini

### `01-canvas.png` — Main workflow canvas
**Where:** Workflows → open **"Telegram Receipt → Gemini → Sheets + Drive"** → press `F` to fit → hide minimap.
**Must show:** All 18 nodes and the full chain from `Telegram — Receive Message` through `Telegram — Send Confirmation`. Both branches after `Check — Is Photo?` (Reject vs. continue) and after `Check — Already Processed?` (Skip vs. continue) must be visible.
**Caption:** *Telegram → photo check → in-memory dedup → Gemini extraction → Drive archive → Sheets row → Telegram reply.*

### `02-node-telegram-trigger.png` — Telegram Trigger + credential
**Where:** Double-click `Telegram — Receive Message`.
**Must show:** `updates: ["message"]` and the Telegram credential dropdown populated.
**Caption:** *Bot receives every message; the next node decides whether it is a photo.*

### `03-node-gemini-config.png` — Gemini HTTP Request node
**Where:** Double-click `Gemini — Extract Receipt Data`.
**Must show:** URL `=https://generativelanguage.googleapis.com/v1beta/models/{{$env.GEMINI_MODEL || 'gemini-2.0-flash'}}:generateContent`, header `x-goog-api-key: ={{$env.GEMINI_API_KEY}}` (redact), body with `response_mime_type: 'application/json'`, `temperature: 0.1`, and the `inline_data` part referencing `$json.base64Image`.
**Caption:** *Gemini is constrained to JSON via `response_mime_type: application/json`. The API key never lives in workflow JSON — only the env-var reference.*

### `04-node-validate.png` — Validation Code node
**Where:** Double-click `Code — Validate Gemini Response`.
**Must show:** The `try/catch` around `JSON.parse(cleaned)` and the required-field guard `(parsed.merchant_name == null || num(parsed.amount) == null)`.
**Caption:** *Required fields (`merchant_name`, `amount`) gate `Processing Status`. Malformed JSON still produces a row instead of crashing.*

### `05-execution-success.png` — Single successful execution
**Where:** Executions → click the most recent run triggered by a real Telegram photo → expand each node.
**Must show:** Green ticks. The Gemini node output panel showing the parsed JSON. The `Code — Validate Gemini Response` output showing `extractionStatus: 'Success'`. The Drive upload node output panel showing the new file `id` and `webViewLink`. The Sheets append output panel.
**Caption:** *End-to-end: Telegram photo → Gemini JSON → Drive upload → Sheets row → confirmation reply.*

### `06-execution-error-receipt.png` — Error path on a blurry receipt
**Where:** Send a blurry/dark photo to the bot, then open its execution.
**Must show:** Gemini output panel showing partial fields or `null`s; `Code — Validate Gemini Response` output showing `extractionStatus: 'Error'`; Sheets row appended with `Processing Status = Error`; Telegram send-message output showing the "couldn't read it" reply text.
**Caption:** *Low-confidence extraction is logged, not lost — `Processing Status = Error` flags it for review.*

### `07-execution-non-photo.png` — Non-photo rejection
**Where:** Send a plain text message to the bot, then open its execution.
**Must show:** The `Check — Is Photo?` node in red or short-circuiting into `Telegram — Reject Non-Photo`. No Sheets row in the output chain.
**Caption:** *Text messages are filtered out before any Sheets/Drive work happens.*

### `08-telegram-conversation.png` — The actual Telegram chat
**Where:** Telegram mobile or desktop client, the private chat with your bot.
**Must show:** The receipt photo you sent, then the bot's reply ("Got it! <Merchant> — <Currency> <Amount>"). Two clear shots are ideal: a successful run and an "couldn't read it" run, stacked or side-by-side.
**Caption:** *User-facing experience: send a photo → receive a confirmation with the extracted merchant + amount.*

### `09-sheet-receipts.png` — Google Sheet "Receipts"
**Where:** Open the spreadsheet whose ID is in `RECEIPTS_SHEET_ID` → tab `Receipts`.
**Must show:** Header row plus several data rows, including at least one with `Processing Status = Success` and at least one with `Processing Status = Error`. The `Google Drive File ID` column should be populated on successful rows.
**Caption:** *Each receipt becomes one row. Gemini fields come through as numbers/null; `confidence` is a 0–1 score.*

### `10-drive-photos.png` — Drive photo archive
**Where:** Drive → root folder from `RECEIPTS_DRIVE_ROOT_FOLDER_ID` → expand to show `Receipt Photos/<YYYY-MM-DD>/`.
**Must show:** Multiple original-quality photos (not AI-processed) inside the day's folder. Filenames should be `receipt_YYYY-MM-DD_HHMMSS.<ext>` or the original filename when sent as a Document.
**Caption:** *Originals are archived untouched — grouped by submission date, never renamed.*

### `11-credentials-and-env.png` — Credentials + Environment (optional)
**Where:** **Settings → Credentials** and **Settings → Environment Variables**.
**Must show:** Telegram, Google Drive OAuth2, Google Sheets OAuth2 (Gmail not needed here). Env vars: `RECEIPTS_SHEET_ID` (redacted), `RECEIPTS_DRIVE_ROOT_FOLDER_ID` (redacted), `GEMINI_API_KEY` (redacted), `GEMINI_MODEL`, `RECEIPT_FIELDS`.
**Caption:** *All secrets live in env vars or n8n Credentials — never in workflow JSON.*

---

## Final delivery

After capturing all images, the portfolio submission should look like:

```
files/
├── docs/
│   ├── README.md
│   ├── CONFIGURATION.md
│   ├── .env.example
│   ├── google-sheets-templates.md
│   ├── gemini-prompt-schema.md
│   ├── TESTING.md
│   └── SCREENSHOTS.md            ← this file
├── workflows/
│   ├── workflow-1-gmail-multilabel.json
│   ├── sub-workflow-upload-gmail-attachments.json
│   └── workflow-2-telegram-receipt-gemini.json
└── screenshots/
    ├── workflow-1/
    │   ├── 01-canvas-main.png
    │   ├── 02-canvas-sub.png
    │   ├── 03-node-schedule-and-config.png
    │   ├── 04-node-gmail-search.png
    │   ├── 05-node-execute-workflow-link.png
    │   ├── 06-execution-success.png
    │   ├── 07-execution-error-path.png        (optional)
    │   ├── 08-sheet-email-log.png
    │   ├── 09-drive-folder-tree.png
    │   └── 10-credentials-and-env.png         (optional)
    └── workflow-2/
        ├── 01-canvas.png
        ├── 02-node-telegram-trigger.png
        ├── 03-node-gemini-config.png
        ├── 04-node-validate.png
        ├── 05-execution-success.png
        ├── 06-execution-error-receipt.png
        ├── 07-execution-non-photo.png
        ├── 08-telegram-conversation.png
        ├── 09-sheet-receipts.png
        ├── 10-drive-photos.png
        └── 11-credentials-and-env.png         (optional)
```

If you want, you can also drop the actual screenshots into the repo with
these exact filenames and they will be picked up automatically by the
`docs/README.md` references.