# n8n Automation Portfolio — Gmail Multi-Label Router & Telegram Receipt Extractor

Two independent, production-oriented n8n workflows built for a technical hiring
assessment.

| # | Workflow | What it does |
|---|----------|---------------|
| 1 | **Gmail Multi-Label → Sheets + Drive** | Polls a Gmail inbox across several configurable labels, logs every email to Google Sheets, and files attachments into a dynamic Drive folder tree without renaming or duplicating them. |
| 2 | **Telegram Receipt → Gemini → Sheets + Drive** | Accepts a receipt photo via a Telegram bot, extracts structured data with Google Gemini, logs it to Sheets, and archives the untouched original photo in Drive. |

Workflow files (import directly into n8n):

- `workflows/workflow-1-gmail-multilabel.json`
- `workflows/sub-workflow-upload-gmail-attachments.json` (companion sub-workflow used by #1)
- `workflows/workflow-2-telegram-receipt-gemini.json`

---

## 1. Executive Summary

**What it does.** Workflow 1 turns a labeled Gmail inbox (e.g. `Invoices`,
`Receipts`, `Documents`, or any custom label) into a structured, searchable
log in Google Sheets, with every attachment automatically organized in Drive
by label → date → sender. Workflow 2 turns "send a photo to a Telegram bot"
into a structured expense-tracking row, using Gemini's vision capabilities to
read the receipt instead of a human re-typing it.

**Why this automation.** Both are realistic variants of the single most
common back-office automation request: *"stuff arrives unstructured (email,
a phone photo), someone has to read it and re-type it into a spreadsheet."*
Replacing that manual step is the actual business value; Sheets/Drive were
chosen over a database because they're what a small team or solo operator
already has open, and because the assessment specifically asked for them.

**Technologies used.** n8n (Schedule Trigger, Gmail, Google Sheets, Google
Drive, Telegram Trigger/node, HTTP Request, Code), Gmail API, Google Sheets
API, Google Drive API v3, Telegram Bot API, Google Gemini API
(`generateContent` with `response_mime_type: application/json`).

**Key technical decisions** (and why):

- **In-memory de-dup instead of per-message Sheets lookups.** A Sheets
  "lookup by Message ID" node that finds zero rows produces *zero output
  items* in n8n — which silently drops the very messages you meant to keep
  processing. Both workflows instead read the log sheet once per run, build
  an in-memory ID set, and check membership in a Code node. Cheaper and
  correct on the first-ever run (empty sheet) too.
- **Drive "find-or-create folder" done in one Code node per level**, using
  `httpRequestWithAuthentication` against the Drive REST API rather than a
  Search node + IF + Create node chain. The native-node chain has the exact
  same "0 results = branch never fires" problem as above. One node per
  folder level is easier to read, test, and reuse.
- **Attachment upload is a sub-workflow**, called once per email via
  *Execute Workflow*. This keeps "loop over N attachments and come back with
  one result" cleanly scoped per email — no cross-email item mixing, and it's
  independently testable.
- **Gemini is called with `response_mime_type: application/json`**, not
  parsed out of free-form prose. The extraction prompt and required-field
  list are both configurable (`RECEIPT_FIELDS` env var), and a malformed or
  incomplete response degrades to a logged row with
  `Processing Status = Error` rather than crashing the run.
- **Error handling is proportionate, not maximal.** Every external-API node
  (Gmail, Sheets, Drive, Telegram, the Gemini HTTP call) has
  `continueOnFail` set so one bad item doesn't halt the whole batch, and
  every row ends up with a `Processing Status` / error message. This project
  intentionally does *not* add a separate alerting stack, retry queue, or
  dead-letter sheet — that's beyond what the assessment asked for.

---

## 2. Architecture

### Workflow 1 — Gmail Multi-Label → Sheets + Drive

```
Schedule (every N min)
        │
        ▼
Config — env-driven settings (labels, sheet id, drive root, lookback)
        │
        ▼
Sheets — read existing Message IDs  ──▶  Code — build de-dup set
        │
        ▼
Code — expand configured labels into one item per label
        │
        ▼
Gmail — search messages "label:<X> newer_than:<window>"   (runs once per label)
        │
        ▼
Code — normalize email (sender/recipient/subject/date/attachments, isDuplicate flag)
        │
        ▼
IF — already processed?  ──yes──▶  Skip (NoOp)
        │ no
        ▼
IF — has attachments?
   │ yes                              │ no
   ▼                                  ▼
Execute Workflow:                Code — set empty drive fields
"Sub — Upload Gmail Attachments"       │
   │                                  │
   └──────────────┬───────────────────┘
                   ▼
        Code — merge result, set Processing Status
                   ▼
        Sheets — append row to Email Log
```

**Sub-workflow — Upload Gmail Attachments** (called once per email that has
attachments):

```
Execute Workflow Trigger (receives messageId, label, sender, date, attachments[])
        ▼
Code — sanitize label / date / sender folder names
        ▼
Code — get-or-create "<Label>" folder          (Drive REST, search-then-create)
        ▼
Code — get-or-create "<YYYY-MM-DD>" folder      (parent = label folder)
        ▼
Code — get-or-create "<Sender Name>" folder     (parent = date folder)
        ▼
Split — one item per attachment (folder ids carried through)
        ▼
Gmail — download attachment (binary, original bytes)
        ▼
Drive — upload file (original filename + mimeType preserved)
        ▼
Aggregate — collect all uploaded file ids/links back into ONE item
        ▼
Code — carry email data + build { driveFileIds, driveFileLinks, uploadStatus }
```

Resulting Drive layout:

```
Gmail Attachments/
  └── Invoices/
        └── 2026-09-04/
               └── Benjamin/
                    ├── invoice.pdf
                    └── receipt.jpg
```

### Workflow 2 — Telegram Receipt → Gemini → Sheets + Drive

```
Telegram Trigger — new message
        ▼
IF — is it a photo?  ──no──▶ Telegram — reply "please send a receipt photo"
        │ yes
        ▼
Sheets — read existing (Chat ID, Telegram Message ID) pairs
        ▼
Code — build de-dup key, check membership
        ▼
IF — already processed?  ──yes──▶ Skip (NoOp)
        │ no
        ▼
Telegram — get file info (largest photo size) → download photo (binary)
        ▼
Code — build Gemini prompt + base64 image
        ▼
HTTP Request — Gemini generateContent (response_mime_type: application/json)
        ▼
Code — parse & validate JSON, fall back to Error status on bad/incomplete output
        ▼
Code — build filename (preserve if present, else receipt_YYYY-MM-DD_HHMMSS.ext)
        ▼
Code — get-or-create "<YYYY-MM-DD>" Drive folder
        ▼
Drive — upload ORIGINAL photo (never the AI-processed version)
        ▼
Code — merge final row + Processing Status
        ▼
Sheets — append row to Receipts
        ▼
Telegram — send confirmation (merchant + amount, or a "couldn't read it" note)
```

Resulting Drive layout:

```
Receipt Photos/
  └── 2026-09-04/
        ├── receipt.jpg
        ├── IMG_1023.jpg
        └── receipt_2026-09-04_121530.jpg
```

---

## 3. Quick Start — Testing Locally

Want to see the workflows run before wiring up real credentials? Here's the
fastest path:

### 3a. Run n8n locally (30 seconds)

**Option A — Docker (recommended):**
```bash
docker run -it --rm -p 5678:5678 -v ~/.n8n:/home/user/.n8n n8nio/n8n
```

**Option B — npm:**
```bash
npx n8n start
```

Open `http://localhost:5678` in a browser and create an admin account.

### 3b. Import the workflows

1. **Workflow 1** — drag-and-drop `workflows/workflow-1-gmail-multilabel.json`,
   then drag-and-drop `workflows/sub-workflow-upload-gmail-attachments.json`.
   Open the "Run — Upload Gmail Attachments" node in Workflow 1 and select
   "Sub — Upload Gmail Attachments" from the workflow dropdown. n8n assigns
   internal IDs on import — the link must be made manually.
2. **Workflow 2** — drag-and-drop `workflows/workflow-2-telegram-receipt-gemini.json`.

### 3c. Set environment variables

Copy `docs/.env.example` to a `.env` file (or set them in n8n's
**Settings → Environment Variables**). Only fill in values you have; the
defaults are safe for local testing.

### 3d. Create credentials

Use n8n's **Credentials** tab to create:
- **Gmail OAuth2** (Google Cloud project → APIs & Services → OAuth consent screen +
  OAuth 2.0 client ID with Gmail, Sheets, Drive scopes)
- **Google Sheets OAuth2** (same Google Cloud project, Sheets API enabled)
- **Google Drive OAuth2** (same project, Drive API enabled)
- **Telegram** (talk to @BotFather, `/newbot`, paste the token)

### 3e. Run and inspect

- **Workflow 1:** click the workflow, enable it, then click **Test Workflow**
  (the play button) or wait for the next Schedule Trigger fire. Watch the
  execution log — each node shows its input/output. Check your Google Sheet
  and Drive folder for results.
- **Workflow 2:** send a photo to your Telegram bot in a private chat, then
  check the **Executions** tab in n8n for the triggered run.

### 3f. How to verify it's working

| Check | Where to look |
|---|---|
| Email logged to sheet | Google Sheets → "Email Log" tab → new row appears |
| Attachment saved to Drive | Google Drive → `Gmail Attachments/<Label>/<Date>/<Sender>/` |
| Receipt logged to sheet | Google Sheets → "Receipts" tab → new row appears |
| Receipt photo saved to Drive | Google Drive → `Receipt Photos/<Date>/` |
| Duplicate skipped | Re-send same email/message → no new row in sheet |
| Error path | Break a credential → check `Processing Status = Error` column |

---

1. **Run n8n** — self-hosted (`npx n8n` / Docker) or n8n Cloud both work; nothing here is deployment-specific.
2. **Create credentials** in n8n (Credentials → New):
   - `Gmail OAuth2` — Google Cloud project with Gmail API enabled, scopes
     `gmail.readonly` (and `gmail.modify` if you plan to auto-label/archive
     later — not required for this build).
   - `Google Sheets OAuth2` and `Google Drive OAuth2` — can reuse the same
     Google Cloud OAuth client as Gmail; just enable the Sheets and Drive
     APIs on that project too.
   - `Telegram` — message **@BotFather** on Telegram, `/newbot`, copy the
     token into the credential.
   - **Gemini** — this build calls Gemini via a plain `HTTP Request` node
     with an `x-goog-api-key` header (`GEMINI_API_KEY` env var) so the key
     never sits inside workflow JSON. If your n8n version ships a native
     Google Gemini/PaLM credential type, you can swap the HTTP node for that
     credential instead — same request shape.
3. **Import** both `workflow-1-gmail-multilabel.json` and
   `sub-workflow-upload-gmail-attachments.json`, then open the
   "Run — Upload Gmail Attachments" node in Workflow 1 and pick the sub-workflow
   from the dropdown (n8n links them by internal ID, which only exists after
   both are imported into the same instance).
4. **Import** `workflow-2-telegram-receipt-gemini.json`.
5. **Set environment variables** — see `CONFIGURATION.md` / `.env.example`.
6. **Attach credentials** to each node (Gmail nodes → Gmail OAuth2, Sheets
   nodes → Google Sheets OAuth2, Drive/Code nodes that call Drive →
   Google Drive OAuth2, Telegram nodes → Telegram credential).
7. **Create the two Sheets tabs** using the column headers in
   `google-sheets-templates.md`.
8. **Activate** both workflows.

## 4. Configuration

Every value called out in the assessment is environment-driven — see
`CONFIGURATION.md` for the full table and `.env.example` for a ready-to-copy
file. Nothing required to change is hard-coded inside the workflow JSON.

## 5. Duplicate Prevention

- **Workflow 1:** Gmail **Message ID** is the identity. The log sheet is read
  once per scheduled run; a Code node builds a `Set` of already-logged
  Message IDs and every candidate email is checked against it before any
  Sheets/Drive write happens.
- **Workflow 2:** identity is **Chat ID + Telegram Message ID** (a bot can
  serve multiple chats, so message IDs alone aren't globally unique). Same
  read-once-then-check-in-memory approach.

Both intentionally avoid the "Sheets lookup filtered by value" pattern for
the reason explained in the Executive Summary — it fails silently on the
exact case (not a duplicate) you need it to pass.

## 6. Error Handling

`continueOnFail` is set on every node that calls an external API (Gmail
search/download, Sheets read/append, Drive folder/upload, Telegram send,
the Gemini HTTP call). Downstream Code nodes check for `$json.error` /
malformed output and record it in the sheet's `Processing Status` and
error-message columns rather than throwing — so one bad email or one
unreadable receipt never stops the batch. See `TESTING.md` for the specific
failure cases exercised.

## 7. Testing

See `TESTING.md` for the full test matrix (attachment counts, per-label
coverage, duplicate messages, missing sender, bad/blurry receipts, non-photo
messages, Gemini/Drive failures).

## 8. Files In This Portfolio

```
workflows/
  workflow-1-gmail-multilabel.json
  sub-workflow-upload-gmail-attachments.json
  workflow-2-telegram-receipt-gemini.json
docs/
  README.md                      (this file)
  CONFIGURATION.md
  .env.example
  google-sheets-templates.md
  gemini-prompt-schema.md
  TESTING.md
```
