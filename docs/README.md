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
- **Drive "find-or-create folder" done with the native Google Drive node**
  (`folder:create` + `folder:list`) plus a tiny Code node to pick the
  existing-or-just-created id. We can't use a single `folder:create`
  because Drive's API returns "folder already exists" instead of being
  idempotent, and a Search-then-IF-then-Create chain has the same
  "0 results = branch never fires" problem as the Sheets lookup above.
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

## 2. Quick Start — Testing Locally

Want to see the workflows run before wiring up real credentials? Here's the
fastest path on Windows.

### 3a. Run n8n locally (Windows, npm)

`npx n8n start` does **not** work on a fresh machine — newer npm blocks the
`prebuild-install` script that downloads the `sqlite3` native binding, and
n8n then crashes with `Initial database connection attempt … failed: SQLite
package has not been found installed`. The fix is to install n8n into a
project folder that allows the build script, then run it from there.

```powershell
# 1. Make a project folder just for n8n
mkdir C:\n8n-install
cd C:\n8n-install
npm init -y

# 2. Allow sqlite3's prebuild-install to actually run
#    (npm 7+ blocks install scripts by default)
npm config set fund false
npm install --foreground-scripts sqlite3
#    If npm still blocks the script, run this once:
npm approve-scripts sqlite3
npm rebuild sqlite3

# 3. Install n8n into the same folder
npm install n8n

# 4. Start n8n. The first run takes ~30s while it migrates the DB.
.\node_modules\.bin\n8n.cmd start
```

> **Important — required `.env` flag.** Recent n8n versions block `$env.*`
> reads in workflow expressions by default. These workflows read
> `GMAIL_LABELS`, `GMAIL_SHEET_ID`, `RECEIPTS_SHEET_ID`, `GEMINI_API_KEY`,
> etc. from env vars, so before starting n8n create
> `C:\n8n-install\.n8n\.env` with this single line:
>
> ```ini
> N8N_BLOCK_ENV_ACCESS_IN_NODE=false
> ```
>
> n8n reads it automatically on startup. Without this, every node that
> reads an env var (the `Config — Load Settings` Set node in particular)
> will throw `access to env vars denied`.

Once n8n is up, open `http://localhost:5678` and create the owner account.

> **If you see `n8n's port 5678 is already in use. Do you have another
> instance of n8n running already?`** — another n8n process is still
> holding the port. Kill it first, then start n8n again:
>
> ```powershell
> # PowerShell — kill all n8n-related node processes
> Get-Process -Name "node" -ErrorAction SilentlyContinue | Stop-Process -Force
> Start-Sleep -Seconds 3
> ```
>
> ```cmd
> :: cmd — find and kill whatever is holding port 5678
> for /f "tokens=5" %a in ('netstat -ano ^| findstr :5678') do taskkill /PID %a /F
> ```
>
> Then start n8n in the **same** terminal and **leave that terminal
> open** — closing the window kills n8n. The editor stays reachable at
> `http://localhost:5678` from any browser.

# 5. For shortcut 
cmd.exe /c "set NODE_PATH=C:\n8n-deps\node_modules&& set N8N_USER_FOLDER=C:\n8n-install\.n8n&& C:\n8n-install\node_modules\.bin\n8n.cmd start"

then press "o" for to openn browsers


> **Tip — keep n8n between sessions.** Add a PowerShell function to your
> profile (`notepad $PROFILE`) so you can re-start with one command:
>
> ```powershell
> function Start-N8n {
>   & "C:\n8n-install\node_modules\.bin\n8n.cmd" start
> }
> ```
>
> After saving, `. $PROFILE` and then `Start-N8n`.

### 3b. Import the workflows

In n8n: **Workflows → ⋮ → Import from File…** (or drag-and-drop the JSON
files onto the canvas). Import **all three**:

1. `workflows/workflow-1-gmail-multilabel.json`
2. `workflows/sub-workflow-upload-gmail-attachments.json`
3. `workflows/workflow-2-telegram-receipt-gemini.json`

After import, open **"Gmail Multi-Label → Sheets + Drive"** and
double-click the **Run — Upload Gmail Attachments** node. In the **Workflow**
dropdown, pick **"Sub — Upload Gmail Attachments"** — n8n assigns internal
IDs at import time, so this link must be re-made manually.

### 3c. Wire up the credentials

After import, every node that needs a credential will show a red "no
credential" warning. To fix:

1. **Settings → Credentials → New** and create:
   - **Gmail OAuth2** (Google Cloud project with Gmail API enabled)
   - **Google Sheets OAuth2** (Sheets API enabled)
   - **Google Drive OAuth2** (Drive API enabled)
   - **Telegram** (token from `@BotFather` → `/newbot`)
2. Open every node that shows a red badge and pick the matching credential
   from its dropdown. n8n fills in the credential ID automatically.

> The placeholder values in the JSON are deliberate — they make it obvious
> on import which nodes need a credential, instead of silently failing at
> the first run.

### 3d. Set environment variables

**Settings → Environment Variables** and paste from `docs/.env.example`.
Only fill in values you have; the defaults are safe for local testing. The
ones without defaults are required:

| Required | Default if unset |
|---|---|
| `GMAIL_SHEET_ID` | *(required)* |
| `GMAIL_DRIVE_ROOT_FOLDER_ID` | *(required)* |
| `RECEIPTS_SHEET_ID` | *(required)* |
| `RECEIPTS_DRIVE_ROOT_FOLDER_ID` | *(required)* |
| `GEMINI_API_KEY` | *(required)* |

### 3e. Create the two Sheets tabs

Use the column headers in `docs/google-sheets-templates.md`. Column names
must match exactly — the Sheets nodes map by header.

### 3f. Run and verify

- **Workflow 1:** open the workflow → click **Test Workflow** (or wait for
  the next Schedule Trigger fire). Watch the execution log, then check your
  Google Sheet + Drive folder.
- **Workflow 2:** send a photo to your Telegram bot in a private chat, then
  check the **Executions** tab in n8n.

### 3g. How to verify it's working

| Check | Where to look |
|---|---|
| Email logged to sheet | Google Sheets → "Email Log" tab → new row appears |
| Attachment saved to Drive | Google Drive → `Gmail Attachments/<Label>/<Date>/<Sender>/` |
| Receipt logged to sheet | Google Sheets → "Receipts" tab → new row appears |
| Receipt photo saved to Drive | Google Drive → `Receipt Photos/<Date>/` |
| Duplicate skipped | Re-send same email/message → no new row in sheet |
| Error path | Break a credential → check `Processing Status = Error` column |

---

## 3. Configuration

Every value called out in the assessment is environment-driven — see
`CONFIGURATION.md` for the full table and `.env.example` for a ready-to-copy
file. Nothing required to change is hard-coded inside the workflow JSON.

## 4. Duplicate Prevention

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

## 5. Error Handling

`continueOnFail` is set on every node that calls an external API (Gmail
search/download, Sheets read/append, Drive folder/upload, Telegram send,
the Gemini HTTP call). Downstream Code nodes check for `$json.error` /
malformed output and record it in the sheet's `Processing Status` and
error-message columns rather than throwing — so one bad email or one
unreadable receipt never stops the batch. See `TESTING.md` for the specific
failure cases exercised.

## 6. Testing

See `TESTING.md` for the full test matrix (attachment counts, per-label
coverage, duplicate messages, missing sender, bad/blurry receipts, non-photo
messages, Gemini/Drive failures).

## 7. Files In This Portfolio

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
