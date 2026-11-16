# Expenses: Gmail Triage + Expense Tracker Routine

A Claude routine that runs daily on one Gmail account and its Google Drive. It:

1. Labels and archives Inbox mail using a fixed `Finance / Subscriptions / Travel / Work-Projects / …` taxonomy, and leaves personal correspondence untouched.
2. Extracts expenses from receipts, invoices, and bookings (email body and PDF/image attachments) into a Google Sheet, de-duplicated on Gmail Message ID.
3. Writes the results by overwriting the existing sheet `Gmail Reports/PPV Expense Tracker 2026` in place. It sends no email and no WhatsApp message.

The routine prompt is **[`ROUTINE.md`](ROUTINE.md)**. Paste it as the prompt of a scheduled task with the Gmail, Google Drive, and Google Sheets connectors attached.

## Sheet layout

| Tab | Write mode | Contents |
|---|---|---|
| `Expenses` | Append-only, keyed on Gmail Message ID | One row per expense, original currency |
| `Summary` | Cleared and rewritten each run | Totals per currency: cumulative, per Category, per month |
| `Run Report` | Cleared and rewritten each run | The run summary and flags (replaces the old email/WhatsApp report) |
| `Run Log` | Append-only | Run start (IST), watermark epoch, counts, `SUCCESS`/`FAILED` |

## Changelog

### v2: Drive-only output
- Removed Task 3A (report email to a fixed address) and Task 3B (WhatsApp Cloud API template send), along with the `WA_*` secrets, the template payload, and the Meta billing notes.
- Task 3 now stores the results by overwriting the existing Drive sheet in place, keeping the same file ID. It never creates a copy or an `.xlsx` export.
- Added a `Run Report` tab so the run summary and flags stay visible without any outbound message.
- Safety scope now forbids all sending (email, drafts, WhatsApp/SMS/webhooks), not just sending to other recipients.
- The watermark is now the run *start* time, stored as Unix epoch seconds, and only `SUCCESS` runs advance it, so a failed run is retried safely.
- Task 2 searches All Mail rather than the Inbox, because Task 1 archives expense mail before Task 2 runs.
- Summary totals are kept per currency, so INR and foreign amounts are never added together.
- Removed a stray quotation and a truncated operational note from the v1 text.

### v1
- The original routine: triage and archive, expense sheet, email and WhatsApp delivery.

## Hardening when you configure the scheduled task

The prompt forbids sending, deleting, and sharing. Prompt rules are a soft control, so where the routine's tool permissions allow it, also deny these tools (names as exposed by the Claude Google connectors):

- **Gmail:** `send_message`, `reply`, `forward`, `create_draft`, `update_draft`, `trash_message`, `trash_thread`, `mark_message_spam`, `mark_thread_spam`, `delete_label`
- **Drive:** `share_file`, `trash_file`, `copy_file`

What the routine needs: Gmail `search_threads`, `get_thread`, `get_message`, `download_gmail_attachment`, `list_labels`, `create_label`, `label_message` / `label_thread`, `unlabel_message` / `unlabel_thread` (INBOX only); Drive `search_files`, `create_file`, `get_file_metadata`; and Sheets `get_spreadsheet`, `get_values`, `append_values`, `update_values`, `batch_clear_values`, `update_spreadsheet`.
