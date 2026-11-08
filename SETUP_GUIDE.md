# Job Application Email Automation: Step-by-Step Setup Guide

This guide sets up the n8n job-application automation on a **new computer** with a **new Google account**, from zero to a first safe test. Follow the parts in order. Every link you need is in the steps, and all of them are repeated in the appendix at the end.

What the automation does: reads contacts from several Google Sheets tabs, sends a personalised cold email with your resume, follows up (up to 5 mails) in the same Gmail thread, checks for replies, records everything in a Tracker sheet, and optionally creates a calendar event when a recruiter proposes an interview time.

---

## Part 1: Install the prerequisites

1. **Git**: https://git-scm.com/downloads (install with the defaults).
2. **Docker Desktop**: https://www.docker.com/products/docker-desktop/
   - Run the installer and leave "Use WSL 2" ticked. Restart the PC if it asks.
   - Open Docker Desktop and wait until it shows **Engine running**.
3. Open a **new** terminal (an old one won't see the new install) and check both tools:

```bash
git --version
docker --version
```

Both must print a version number. If `docker` is "not recognized", Docker Desktop isn't installed or the terminal wasn't reopened.

---

## Part 2: Get the project files

```bash
git clone https://github.com/Shashwat023/n8n-server.git
cd n8n-server
```

> **Important: never copy an `n8n_data` folder from another computer.** It contains that person's n8n database, saved Google logins and the encryption key. A fresh clone has no `n8n_data`, and n8n creates a clean one on first start. If this folder already exists from a zip, delete it before Part 4.

The folder should now contain:
- `docker-compose.yml`: how n8n is started
- `Job Application Email Sequence (Multi-Tab Sheets + Gmail).json`: the workflow
- `files/`: where your resume goes
- `README.md` and this guide

---

## Part 3: Add your resume

1. Copy your resume PDF into the project's `files` folder.
2. Rename it to **`resume.pdf`** (or keep the name and change `resume_file_name` and `resume_file_path` later in Part 9).

Inside the container the workflow reads it from `/home/node/.n8n-files/resume.pdf`. That path already matches `docker-compose.yml`. The PDF is git-ignored, so it is never committed.

---

## Part 4: Start n8n

From the project folder:

```bash
docker compose up -d
```

1. Wait about 30 seconds, then open http://localhost:5678
2. On first visit n8n asks you to **create an owner account**. This is only a local login for n8n itself, not a Google account. Pick any email and password and remember them.
3. Skip any survey or welcome pop-ups (press Esc).

Useful commands:

```bash
docker compose logs -f n8n     # watch logs
docker compose down            # stop n8n (data is kept in n8n_data)
docker compose up -d           # start again
```

---

## Part 5: Create Google OAuth credentials

n8n needs permission to use Google Sheets, Gmail and Google Calendar on your account. You create these in Google Cloud once. It is free.

### 5.1 Create a project
1. Open https://console.cloud.google.com/projectcreate
2. Name it (e.g. `n8n-job-automation`) and click **Create**.
3. Make sure this project is selected in the top bar.

### 5.2 Enable the three APIs
Open each link and click **Enable**:
- Google Sheets API: https://console.cloud.google.com/apis/library/sheets.googleapis.com
- Gmail API: https://console.cloud.google.com/apis/library/gmail.googleapis.com
- Google Calendar API: https://console.cloud.google.com/apis/library/calendar-json.googleapis.com

### 5.3 Set up the consent screen
1. Open https://console.cloud.google.com/auth/overview and click **Get started**.
2. **App information**: enter an app name (e.g. `n8n server`) and your support email, then **Next**.
3. **Audience**: choose **External**, then **Next**.
4. **Contact information**: enter your email, then **Next**.
5. **Finish**: tick the agreement, click **Continue**, then **Create**.

### 5.4 Add yourself as a test user (do not skip)
1. In the left menu click **Audience**.
2. Under **Test users** click **+ Add users**.
3. Enter the exact Google account you will sign in with, then **Save**.

Without this step Google shows **"Access blocked: n8n server has not completed the Google verification process (Error 403: access_denied)"**.

### 5.5 Create the OAuth client
1. Open https://console.cloud.google.com/apis/credentials
2. Click **Create Credentials → OAuth client ID**.
3. **Application type**: **Web application**. Give it any name.
4. Leave **Authorized JavaScript origins** empty.
5. Under **Authorized redirect URIs** click **+ Add URI** and paste exactly:

```
http://localhost:5678/rest/oauth2-credential/callback
```

6. Click **Create**. Copy the **Client ID** and **Client secret** and keep them handy. You will paste the same pair three times in Part 6.

### 5.6 Stop the 7-day expiry (recommended)
In **Testing** mode Google invalidates your login every 7 days, so n8n would stop working weekly. To avoid that:
1. Go to **Audience**.
2. Click **Publish app** and confirm.

For personal use no Google verification is needed. You will just see an "unverified app" warning once when signing in (click **Advanced → Go to … (unsafe)**, which is your own app).

---

## Part 6: Create the three credentials in n8n

In n8n open **Credentials → Create credential** (or http://localhost:5678/home/credentials).

Create these three. For each one:

| Credential type to search for | Used for |
|---|---|
| **Google Sheets OAuth2 API** | reading and updating the sheets |
| **Gmail OAuth2 API** | sending mail and checking replies |
| **Google Calendar OAuth2 API** | interview events |

1. Paste the same **Client ID** and **Client Secret** from Part 5.5.
2. Click **Sign in with Google**, choose the account, and click **Allow** on every permission. For Gmail make sure the list includes reading mail, not only sending.
3. When it says connected, click **Save**.

---

## Part 7: Import the workflow

1. In n8n click **Create workflow** (top right, orange).
2. Click the **⋯ (Actions)** menu → **Import from file…** and choose `Job Application Email Sequence (Multi-Tab Sheets + Gmail).json` from the project folder.
3. The workflow appears with 49 nodes. Press **Ctrl+S** to save.

> The file in the repo can be older than the latest live version. After importing, **do the checklist in Part 10** so you get all bug fixes. If someone gives you a newer export (Actions → **Download**), import that instead.

### Attach your credentials to every Google node
The imported workflow points to the *original owner's* credentials. Open each node below and pick **your** credential in the Credential dropdown, then save.

| Credential | Nodes |
|---|---|
| **Google Sheets** | Read Master Switch (Settings!A2), List All Sheet Tabs, Read Rows From All Tabs, Read Tracker State, Update Tracker Row, Append To Log, Move To Invalid Contacts, Delete Bounced Row From Source |
| **Gmail** | Check Gmail For Reply, Fetch Reply Message, Send Follow-up (Same Thread), Send First Email (New Thread) |
| **Google Calendar** | Create Interview Calendar Event |

`List All Sheet Tabs` is an HTTP Request node: set **Authentication → Predefined Credential Type → Google Sheets OAuth2 API** and select your credential.

---

## Part 8: Prepare the Google Sheet

Create a new spreadsheet at https://sheets.new, or use your own.

### 8.1 Share it with the right account
The Google account you signed in with in Part 6 must be able to **edit** this spreadsheet. If the sheet belongs to another account, share it with that account as **Editor**. If you skip this, nodes fail with **"Forbidden"**.

### 8.2 Required tabs (names must match exactly)

**`Settings`**

| A | B | C |
|---|---|---|
| `master_switch` | `city` | `enabled` |
| `ON` | Bangalore | ☑ (tick box) |
| | Delhi | ☐ |
| | Mumbai | ☐ |
| | Hyderabad | ☐ |
| | Pune | ☐ |
| | Chennai | ☐ |
| | Remote | ☐ |

- `A2` must be `ON` for the workflow to run. `OFF` stops it.
- Use **Insert → Tick box** for column C.
- Tick every city you want to target. If **no** city is ticked, the filter is off and everyone passes. Contacts with no city value always pass.

**`Tracker`**: header row (column order doesn't matter, names must match):

```
company | recruiter_name | email | job_role | linkedin | source | city | source_tab | status | mail_count | last_mail_date | next_mail_date | reply_status | gmail_thread_id | notes | data_flags
```

**`Log`**: header row:

```
timestamp | company | email | mail_number | result | thread_id | duration_ms
```

**`Invalid Contacts`**: header row:

```
moved_at | company | recruiter_name | email | job_role | source | source_tab | status | reason
```

### 8.3 Your contact tabs
Every tab that is **not** in the excluded list (`Settings`, `Tracker`, `Log`, `Invalid Contacts`) and not hidden is read as contacts. Each needs a header row with an email column. Column names are matched flexibly (`email`, `Email ID`, `mail`, `company`, `Company Name`, `role`, `location` and so on).

- Rows with a missing or invalid email are ignored.
- **Hidden tabs are skipped automatically**, which is how you switch tabs off.

### 8.4 Copy the spreadsheet ID
From the sheet's URL `https://docs.google.com/spreadsheets/d/<THIS_PART>/edit` copy the long ID. You need it in Part 9.

---

## Part 9: Fill in Workflow Config

Open the **Workflow Config** node and set:

| Field | What to put |
|---|---|
| `spreadsheet_id` | the ID from Part 8.4 (**must be replaced, the imported one is the original owner's sheet**) |
| `sender_name` | your full name |
| `sender_headline` | one-line profile, e.g. `3D Artist \| Maya, ZBrush, Unreal Engine 5` |
| `sender_phone` | your phone number |
| `sender_portfolio` | your LinkedIn, ArtStation or portfolio URL |
| `resume_file_path` | `/home/node/.n8n-files/resume.pdf` (leave as is) |
| `resume_file_name` | `resume.pdf` (or the name you used in Part 3) |
| `calendar_id` | `primary` (your default calendar) |
| `excluded_tabs` | `Settings,Tracker,Log,Invalid Contacts` |
| `window_start_hour` / `window_end_hour` | send window in India time, default 10 and 18 |
| `max_mails` | 5 |
| `followup_gap_hours` | `{"1": 2, "2": 1, "3": 1.5, "4": 3}` (hours to wait after mail 1, 2, 3, 4) |

Save with **Ctrl+S**.

---

## Part 10: Checklist after importing (fixes the repo copy may be missing)

Do all four. They fix the problems found while testing.

### 10.1 Follow-ups: `Send Follow-up (Same Thread)`
1. Open the node.
2. **Message Snippet or ID**: hover the field, click **Expression**, and enter:
   `{{ $json.gmail_thread_id }}`
3. **Options → Add option → Reply to Recipients Only** and switch it **on**.

Without this, every follow-up fails with `Cannot read properties of undefined (reading 'headers')`. Without "Recipients Only" the follow-up is also sent to your own address.

### 10.2 Logging: `Append To Log`
1. Open the node and make sure the `Log` tab has its header row (Part 8.2).
2. In **Values to Send** click the refresh icon so n8n loads the columns. This fixes `` `columns.schema` is required when `columns.mappingMode` is `defineBelow` ``.
3. Fill each column as an expression:

| Column | Expression |
|---|---|
| timestamp | `{{ $('Finalize Row Payload').item.json.log_timestamp }}` |
| company | `{{ $('Finalize Row Payload').item.json.company \|\| '' }}` |
| email | `{{ $('Finalize Row Payload').item.json.email }}` |
| mail_number | `{{ $('Finalize Row Payload').item.json.log_mail_number }}` |
| result | `{{ $('Finalize Row Payload').item.json.log_result }}` |
| thread_id | `{{ $('Finalize Row Payload').item.json.log_thread_id \|\| '' }}` |
| duration_ms | `{{ $('Finalize Row Payload').item.json.log_duration_ms }}` |

Use `.item`, not `.first()`, so each loop iteration logs its own contact.

### 10.3 Email text: `Generate Personalized Email`
The imported code mentions the original sender's background. Replace the whole node code with the version written for **your** resume (ask for it, or edit the skills and projects in the body text). The name, phone and portfolio come from Workflow Config, so Part 9 must be filled in.

### 10.4 Checkbox reading: `Check Gates: Switch + Time Window`
Make sure the `isTicked` function in this node includes the `☑` character in its accepted list:

```js
return ['true', 'yes', 'y', '1', 'x', 'on', 'checked', '✓', '✔', '☑'].includes(s);
```

Google Sheets tick boxes are returned as ☑ or ☐. Without this, no city is ever recognised as ticked.

---

## Part 11: Validate before running anything real

Open each node and click **Execute step**. Do this on a sheet with no real contacts yet.

1. **Read Master Switch (Settings!A2)** should return your Settings rows. A "Forbidden" error means the sheet isn't shared with the credential's account (Part 8.1), or the Sheets API isn't enabled (Part 5.2).
2. **Check Gates: Switch + Time Window** should show `master_switch: "ON"`, `proceed: true` (inside 10:00 to 18:00 IST), and your ticked cities in `selected_cities`. If the output has a pin icon / "data is pinned" banner, click **Unpin** first.
3. **List All Sheet Tabs** should list your tabs.
4. **Check Gmail For Reply** should run without a permission error (it needs Gmail read access).
5. Check that the credential dropdown is filled on `Send First Email (New Thread)` and `Create Interview Calendar Event`.

---

## Part 12: Safe first test (sandbox)

Never test on your real contact list. The first full run sends real emails.

1. **Hide every real contact tab** (right-click the tab → **Hide sheet**).
2. Add one new visible tab, e.g. `Sandbox`, with 2 to 3 **of your own** addresses:

| company | recruiter_name | email | job_role | linkedin | source | city |
|---|---|---|---|---|---|---|
| Test Co A | Yourself | yourname+test1@gmail.com | 3D Artist | | Test | |
| Test Co B | Yourself | yourname+test2@gmail.com | 3D Artist | | Test | |
| Test Co C | | yourname+test3@gmail.com | | | | |

3. Make sure `Settings!A2` is `ON` and it is between 10:00 and 18:00 IST (or temporarily widen the window in Workflow Config).
4. In n8n click **Execute workflow** on the canvas (manual run, not the schedule).
5. Check:
   - The test mailbox received the email **with the resume attached**.
   - The Tracker shows `Mail 1 Sent`, `mail_count` 1, and a thread ID.
   - The Log tab got a row per contact.
6. To test follow-ups, set `last_mail_date` in the Tracker to a time more than 2 hours ago and run again. The follow-up should arrive **in the same Gmail thread**, only to the test address.
7. Test replies: reply to one test mail from that address, run again, and the Tracker should show `Replied` / `Got Reply`.
8. Test a bad address by adding a row with `not-a-valid-address`. It should end up as `Mail Not Found` and be copied to `Invalid Contacts`.

> Known gap: an address that is correctly formatted but doesn't exist (e.g. `no-such-person-xyz789@gmail.com`) is accepted by Gmail and bounces later as a separate "Mail Delivery Subsystem" email. The workflow does not yet scan the inbox for those bounces, so such rows stay `Mail 1 Sent`.

**After testing:** unhide the real tabs, remove or hide `Sandbox`, and reset any Tracker rows you changed.

---

## Part 13: Go live

1. Confirm the Tracker only contains rows you expect.
2. Remember: **a manual run sends mail to every contact that is due**, not just a few.
3. To run automatically, turn the workflow **Active** (toggle at the top right). It then runs every 30 minutes, but only sends between `window_start_hour` and `window_end_hour` (IST) and only while `Settings!A2` is `ON`.
4. To pause everything instantly, set `Settings!A2` to `OFF` or turn the workflow off.

How the sequence works: mail 1 is sent immediately, mail 2 after 2 hours, mail 3 after 1 hour, mail 4 after 1.5 hours, mail 5 after 3 hours. After mail 5 the contact becomes `Closed`. Replied, Closed, Rejected, Skipped, Error and Mail Not Found contacts are never mailed again.

---

## Part 14: Troubleshooting

| Problem | Cause and fix |
|---|---|
| `docker` is not recognized | Docker Desktop isn't installed or the terminal is old. Install it (Part 1), start it, open a new terminal. |
| `Access blocked … Error 403: access_denied` when signing in | Your Google account isn't a Test user. Part 5.4. |
| Google sign-in works but stops working after about a week | App is in Testing mode. Publish it, Part 5.6, then reconnect each credential. |
| `Forbidden - perhaps check your credentials` | The spreadsheet isn't shared with the signed-in account (Part 8.1), the wrong `spreadsheet_id`, or the API isn't enabled (Part 5.2). |
| `` `columns.schema` is required when `columns.mappingMode` is `defineBelow` `` | Open the Sheets node and refresh the columns (Part 10.2). |
| `Cannot read properties of undefined (reading 'headers')` on a follow-up | Message Snippet or ID is empty. Part 10.1. |
| `selected_cities` is empty although a box is ticked | Missing `☑` in `isTicked`, or the output is pinned. Part 10.4, and unpin. |
| `Apply City Filter` returns nothing and the run stops | A city is ticked but your contact tabs have no city column. Blank-city rows must pass through. Check that the `!rowCity` branch pushes the row instead of excluding it. |
| Log rows are mostly empty | `Append To Log` isn't mapped. Part 10.2. |
| Workflow shows green but nothing is written | Sheets nodes are set to "continue on error". Open the node in the execution view and read the error. |
| Rows stuck as `Error` | `Error` is final. Fix the cause, then set `status` back to `Mail 1 Sent`, clear `notes`, and leave `next_mail_date` empty so it is due again. |
| Logged out of n8n | Sign in again. Don't call n8n's internal `/rest/` URLs directly from the browser console. |
| Hidden tab still being read | The hidden-tab filter is missing in `List All Sheet Tabs` (URL must include `hidden`) and `Select Raw Data Tabs`. |

---

## Part 15: Security notes

- **Never share `n8n_data/`** (zip, USB, chat or git). It holds the saved Google logins and the key that decrypts them.
- Share only the repo, the workflow JSON, and this guide. Each person creates their **own** Google credentials.
- If `n8n_data` was ever shared, treat those Google logins as exposed: in n8n reconnect each credential, and revoke access at https://myaccount.google.com/permissions
- Keep your resume out of git (it's already in `.gitignore`).
- Don't put real passwords, client secrets or tokens in this repo.

---

## Appendix: all links in one place

| What | Link |
|---|---|
| Git download | https://git-scm.com/downloads |
| Docker Desktop | https://www.docker.com/products/docker-desktop/ |
| Project repository | https://github.com/Shashwat023/n8n-server |
| n8n (local) | http://localhost:5678 |
| n8n credentials page | http://localhost:5678/home/credentials |
| Google Cloud: new project | https://console.cloud.google.com/projectcreate |
| Enable Google Sheets API | https://console.cloud.google.com/apis/library/sheets.googleapis.com |
| Enable Gmail API | https://console.cloud.google.com/apis/library/gmail.googleapis.com |
| Enable Google Calendar API | https://console.cloud.google.com/apis/library/calendar-json.googleapis.com |
| OAuth consent screen / audience | https://console.cloud.google.com/auth/overview |
| Create OAuth client (credentials) | https://console.cloud.google.com/apis/credentials |
| OAuth redirect URI to paste | `http://localhost:5678/rest/oauth2-credential/callback` |
| New Google Sheet | https://sheets.new |
| Review / revoke Google app access | https://myaccount.google.com/permissions |
