# Job Application Email Sequence — n8n Automation

Self-hosted n8n workflow that reads contacts from multiple Google Sheets tabs,
sends a personalized cold email + resume, follows up up to 5 times with reply
detection, tracks status back to a Tracker sheet, logs every outcome, and
creates a calendar event when a recruiter's reply looks like interview
scheduling.

Workflow file: `Job Application Email Sequence - FIXED.json`

## 1. Prerequisites

- Docker + Docker Compose (`docker-compose.yml` in this repo brings up n8n)
- A Google account with:
  - Google Sheets API enabled
  - Gmail API enabled
  - Google Calendar API enabled (optional — only needed for interview scheduling)
- Your resume as a PDF

## 2. First-time setup

```bash
docker compose up -d
```

n8n will be at `http://localhost:5678`. Create your owner account on first
visit.

### Import the workflow

**Workflows → ⋯ (Actions) → Import from file** → select
`Job Application Email Sequence - FIXED.json`.

### Add your resume

Drop your resume PDF at `files/resume.pdf` (this repo's `files/` directory is
bind-mounted into the container at `/home/node/.n8n-files`, per
`docker-compose.yml`). See `files/README.md`. The file is git-ignored — it
never gets committed.

### Set up credentials (3 needed)

In n8n: **Credentials → Add credential**, create all three, then open each
node listed and select the credential from the dropdown (the imported JSON
references credentials by name; if you're starting fresh you'll need to
reattach them once per node type):

| Credential | Type | Used by |
|---|---|---|
| Google Sheets | `Google Sheets OAuth2 API` | Read Master Switch, List All Sheet Tabs, Read Rows From All Tabs, Read Tracker State, Update Tracker Row, Append To Log, Move To Invalid Contacts, Delete Bounced Row From Source |
| Gmail | `Gmail OAuth2` | Check Gmail For Reply, Fetch Reply Message, Send Follow-up (Same Thread), Send First Email (New Thread) |
| Google Calendar | `Google Calendar OAuth2 API` | Create Interview Calendar Event |

**Credential validation — do this before running anything for real:**

1. Open **Read Master Switch (Settings!A2)** → click **Execute step**. If it
   returns your Settings tab's rows, Sheets auth is good. A 403/401 here means
   the OAuth scope or sharing permission is wrong — the Google account used
   for the credential must have edit access to the spreadsheet.
2. Open **Check Gmail For Reply** → **Execute step** with a real email in the
   input (e.g. pin a test item with your own email). If it errors with
   `insufficient permission` or returns nothing for a thread you know exists,
   the Gmail credential's OAuth **scope is send-only** — you need
   `gmail.readonly` or `gmail.modify` too, not just `gmail.send`. Re-authorize
   the credential and make sure the consent screen lists read access.
3. Open **Send First Email (New Thread)** — do **not** execute this one
   standalone against a real address. Instead just confirm the credential
   dropdown shows your Gmail account (not blank) — actual send testing
   happens in the sandbox run, see §4.
4. Open **Create Interview Calendar Event** → confirm the credential is
   attached and `calendar_id` in **Workflow Config** matches a calendar you
   own (`primary` = your default calendar).

### Configure Workflow Config

Open the **Workflow Config** node and set these to your own values:

| Field | Meaning |
|---|---|
| `spreadsheet_id` | The Google Sheet ID (from its URL) — **point this at your sandbox sheet while testing, see §4** |
| `resume_file_path` | Must match where you mounted the resume — default `/home/node/.n8n-files/resume.pdf` matches this repo's `docker-compose.yml` |
| `sender_name`, `sender_headline`, `sender_phone`, `sender_portfolio` | Used to personalize the email body |
| `window_start_hour` / `window_end_hour` | Send window in IST (default 10–18) |
| `max_mails` | Follow-up sequence length (default 5) |
| `calendar_id` | `primary`, or a specific calendar ID |
| `city_aliases` | JSON map used by the city filter — edit if your target cities/spellings differ |

### Set up the spreadsheet's utility tabs

Your spreadsheet needs these tabs in addition to your raw contact tabs
(everything not in `excluded_tabs` is treated as contact data):

- **Settings** — column A: `master_switch` header, row 2 = `ON`/`OFF`. Columns
  B/C: `city` / `enabled` (checkbox), one row per city — tick whichever
  city(ies) you want this run to target. Optional control row:
  `include_unknown_city` in column B with its own checkbox.
- **Tracker** — header row:
  `company | recruiter_name | email | job_role | linkedin | source | city | source_tab | status | mail_count | last_mail_date | next_mail_date | reply_status | gmail_thread_id | notes | data_flags`
- **Log** — header row:
  `timestamp | company | email | mail_number | result | thread_id | duration_ms`
- **Invalid Contacts** — header row:
  `moved_at | company | recruiter_name | email | job_role | source | source_tab | status | reason`

## 3. Known issues (not yet fixed — read before relying on this in production)

- **Reply detection only runs for follow-ups** (mail_count > 0), not the
  first mail — by design, but means you won't see any reply-checking
  activity until contacts are on their 2nd+ mail.
- **"Mail Not Found" detection is incomplete.** It only catches errors the
  Gmail *send* API throws synchronously. Real-world bounces for
  invalid/closed mailboxes usually arrive **asynchronously** as a separate
  "Mail Delivery Subsystem" email minutes later — this workflow does not yet
  scan the inbox for those. Until that's added, expect some invalid
  addresses to stay marked as sent rather than being caught and moved to
  Invalid Contacts.
- **`Append To Log` and `Move To Invalid Contacts` can log wrong/stale data
  on later loop iterations.** They read via `$('Finalize Row Payload').first()`
  / `$('Classify Send Error').first()` (a cross-node lookup), while
  `Update Tracker Row` correctly uses `$json` (the current item). Inside the
  per-contact loop this cross-node pattern can resolve to the wrong
  iteration. Symptom: only the first Log row has complete data. Tracker sheet
  is unaffected.

## 4. Test on a sandbox sheet, not the real one

Running against the real sheet (thousands of rows across all tabs) is slow —
`Read Rows From All Tabs` reads every row in every non-excluded tab on
**every execution**, so iteration time scales with your full contact list.

**Make a small sandbox copy:**

1. In Google Sheets: **File → Make a copy** of your real spreadsheet (or
   build a fresh one with the same tab structure).
2. Trim it down to just the utility tabs (Settings, Tracker, Log, Invalid
   Contacts, all empty except headers) plus **one** raw contact tab with
   5–10 rows total, using addresses you control (e.g. your own
   `+test1@gmail.com` style aliases, or a secondary mailbox) so a real send
   doesn't reach an actual recruiter.
3. Copy the sandbox sheet's ID from its URL and paste it into
   `spreadsheet_id` in **Workflow Config**.
4. Set Settings!A2 to `ON`, tick a city that matches your test rows' `city`
   column (or leave all unticked to disable the city filter entirely).
5. Click **Execute workflow** from the canvas (manual run, not the schedule
   trigger) and watch it end-to-end in the execution view.
6. Check the sandbox Tracker/Log/Invalid Contacts tabs update as expected,
   and confirm the test mailbox actually receives the email with the resume
   attached.

Once the sandbox run behaves correctly, swap `spreadsheet_id` back to your
real sheet ID before activating the schedule trigger. **Leave the workflow's
Active toggle off** until you've done at least one clean sandbox run.
