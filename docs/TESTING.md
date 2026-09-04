# Testing

Reproduce by sending real test emails/messages against a sandbox Gmail
account, Sheet, Drive folder, and Telegram bot configured per
`CONFIGURATION.md`. "Actual result" cells below are left for you to fill in
during your own test run — the point is to show the expected behavior was
designed for, and to give a reviewer a checklist to re-run.

## Workflow 1 — Gmail Multi-Label

| # | Test case | How to trigger | Expected result |
|---|---|---|---|
| 1 | Email with no attachment | Send a plain-text email to a watched label | One row in Email Log, `Has Attachment = No`, `Attachment Count = 0`, no Drive folder created |
| 2 | Email with one attachment | Send an email with `invoice.pdf` attached | Row logged; `Gmail Attachments/<Label>/<Date>/<Sender>/invoice.pdf` created with the original filename |
| 3 | Email with multiple attachments | Attach 2–3 files of different types | All files uploaded under the same sender folder; `Attachment Count` matches; all links in `Drive File Links` |
| 4 | Email under `label1` | Apply `label1` to a test email | Picked up on the next poll; `Label` column = `label1` |
| 5 | Email under `label2` | Apply `label2` | Same as above, proves multi-label support without editing the workflow |
| 6 | Email under `label3` | Apply `label3` | Same as above |
| 7 | Duplicate email | Let the same message get re-scanned on the next poll (don't delete/change it) | No second row is appended; message ID already in the de-dup set built from the sheet |
| 8 | Missing sender information | Send from an address with no display name / malformed `From` header | Falls back to `Unknown Sender` folder + sheet value instead of erroring |
| 9 | Unsupported/problematic attachment | Attach a 0-byte file or an unusual MIME type | Upload node's `continueOnFail` lets the row still log with `Processing Status = Error` and the other attachments (if any) still upload |
| 10 | Google Drive upload failure | Temporarily revoke Drive access / use an invalid root folder ID | Row still appended with `Processing Status = Error`, `Error Message` populated; other emails in the same run are unaffected |

## Workflow 2 — Telegram Receipts

| # | Test case | How to trigger | Expected result |
|---|---|---|---|
| 1 | Clear receipt photo | Send a well-lit, in-focus receipt photo | Row logged with `Processing Status = Success` and all readable fields populated; bot replies with merchant + amount |
| 2 | Poor-quality receipt photo | Send a blurry/dark photo | Gemini returns partial/low-confidence data; if `merchant_name`/`amount` are unreadable, row logs as `Error` with a friendly bot reply asking for a retake |
| 3 | Multiple receipt photos | Send 2–3 receipts back to back | Each is a separate Telegram message → separate row, processed independently |
| 4 | Non-receipt image | Send a random photo (e.g. a selfie) | Still passes the "is it a photo" check (it IS a photo) — Gemini extraction will have low/absent field data, so it logs as `Error` rather than fabricating a receipt. If stricter filtering is wanted, tighten the Gemini prompt/validation rather than the Is-Photo check |
| 5 | Message without a photo | Send a text-only message | Rejected immediately by "Check — Is Photo?"; bot replies asking for a receipt photo; nothing is logged |
| 6 | Duplicate Telegram message | Re-deliver/replay the same update (same `chat_id` + `message_id`) | No second row appended |
| 7 | Gemini extraction failure | Simulate by pointing `GEMINI_MODEL` at an invalid model name or expiring the API key | HTTP node's `continueOnFail` + validation Code node produce one `Error` row rather than crashing |
| 8 | Missing amount | Send a receipt photo with the total torn off/illegible | `amount` fails required-field validation → `Processing Status = Error` |
| 9 | Missing merchant | Send a receipt with no visible store name | `merchant_name` fails required-field validation → `Processing Status = Error` |
| 10 | Google Drive upload failure | Use an invalid `RECEIPTS_DRIVE_ROOT_FOLDER_ID` | Row still appended with `Google Drive File ID` blank and `Processing Status = Error`; Gemini data (if extracted) is still preserved in the row |

## What this intentionally does not cover

Per the assessment's own note that "extensive logging, confidence scores,
and elaborate error handling are portfolio enhancements, not mandatory
requirements," this test matrix focuses on the two core workflows behaving
correctly end-to-end rather than exhaustively simulating every possible
transient API failure (rate limits, partial network failures mid-upload,
etc.). Those are reasonable follow-ups if this were going into real
production use rather than a portfolio assessment.
