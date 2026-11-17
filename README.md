# Expenses: Gmail Triage + Expense Tracker Routine

A Claude routine (scheduled task "Gmailsorting and expense tracking", daily at 06:00 IST) that works on one Gmail account and its Google Drive. It:

1. Labels and archives Inbox mail using a fixed `Finance / Subscriptions / Travel / Work-Projects / …` taxonomy, and leaves personal correspondence untouched.
2. Extracts expenses from receipts, invoices, and bookings (email body and PDF/image attachments), de-duplicated on Gmail Message ID.
3. Stores the result as one Excel file, `Gmail reports/PPV_Expense_Tracker_2026.xlsx`, on Google Drive. **It sends no email and no WhatsApp message.**

The routine prompt is **[`ROUTINE.md`](ROUTINE.md)**. Paste it over the prompt at <https://claude.ai/code/routines/trig_01QyMdASyj1gwkbkAR7yc5MU>. Agents can't edit this routine because it was created through the API.

## How the Excel file is "overwritten"

The routine only has the Gmail and Google Drive connectors. The Drive connector can create files and change their metadata, but it cannot replace a file's contents. Each run therefore:

1. Downloads the current `PPV_Expense_Tracker_2026.xlsx`.
2. Appends new rows and rewrites the `Summary` and `Run Report` tabs.
3. Uploads the result as a new `.xlsx`, then re-downloads it and checks the SHA-256 and the row count.
4. Moves the old version to Drive trash, where it stays recoverable for 30 days. If verification fails, it trashes the new upload instead and keeps the old one.

As a result there is always exactly one live tracker file, but **its file ID changes every day**. Bookmark the folder rather than the file.

## Workbook layout

| Tab | Write mode | Contents |
|---|---|---|
| `Expenses` | Append-only; an email whose Message ID is already present is skipped | One row per expense line, original currency |
| `Summary` | Rewritten each run | Totals per currency: cumulative, per Category, per month |
| `Run Report` | Rewritten each run | The run summary and flags (replaces the old email report) |
| `Run Log` | Append-only | `Run Timestamp (IST)`, `Result`, `Status`, `Watermark Epoch`. The latest `SUCCESS` row sets the next run's window. |

## Changelog

### v3: Excel on Drive only (2026-11-17)
- **Root cause of the repeated emails:** the live scheduled task still had the v1 prompt, which emailed `vino.raheesh@buildcast.in`. That address has rejected every send since at least 15 Nov (`550 5.4.1`). The fix is to paste this file into the routine.
- **Root cause of the stuck window:** the Run Log lived in a Google Sheet the routine could not edit, so every run reprocessed mail from 24 Oct and left another copy of the sheet on Drive. The Run Log now lives inside the `.xlsx`, which is replaced each run.
- Seeded Drive with the 17 Nov workbook (318 expense rows, Run Log up to 2026-11-17 06:17 IST) as `PPV_Expense_Tracker_2026.xlsx`.
- De-duplication now skips whole emails whose Message ID is already present, because one receipt can carry several line items.
- Each run's query window overlaps the previous one by one hour, and only `SUCCESS` runs advance it.

### v2: Drive-only output (design only; never deployed to the scheduled task)

### v1
- The original routine: triage and archive, a Google Sheet, an email report, and WhatsApp delivery.

## Hardening

The rule against sending lives in the prompt. Where the routine's settings allow it, also turn off these tools for the routine:
- Gmail: `send_message`, `reply`, `forward`, `create_draft`, `update_draft`, `trash_message`, `trash_thread`
- Drive: `share_file`, `copy_file`

The Google Calendar connector is attached to the task but not used, so it can be removed.
