# 📧 Life Sciences Email Outreach — With Follow-Up Sequence

An n8n workflow that automates personalised email outreach to life sciences contacts (pharma, biotech, CRO, HEOR) with an intelligent follow-up sequence. The workflow reads contacts and drug/indication data from Google Sheets, selects the right email template based on recipient designation and follow-up stage, sends via Gmail, and updates tracking status automatically.

---

## What this workflow does

1. **Triggers daily** at a set time (weekdays only — skips Saturday and Sunday automatically)
2. **Reads contacts** from a Google Sheet — including drug name, indication, phase, and follow-up history
3. **Routes each contact** through a Switch node based on their follow-up stage and last contact date:
   - Follow-ups 1–2 → waits 3 days between emails
   - Follow-ups 3–5 → waits 5 days between emails
   - Follow-up > 5 → contact is marked as exhausted and skipped
4. **Throttles volume** randomly to 40–50 emails per run to avoid spam triggers
5. **Fetches the right template** from a second Google Sheet, matched by designation and follow-up number
6. **Personalises** the template by injecting `{FirstName}`, `{Company}`, `{DrugName}`, `{Phase}`, `{TherapeuticArea}`, and a custom HTML signature
7. **Sends via Gmail** with a 15-second delay between each email
8. **Updates the tracker sheet** with the new status, incremented follow-up count, and today's date

---

## Workflow diagram
```
Schedule Trigger (daily)
  └─► Weekday check (If node)
        └─► Read contacts from Google Sheet
              └─► Map fields + build signature (Code)
                    └─► Route by follow-up stage (Switch)
                          ├─► [Stages 1–5] → Merge2 → Random batch (40–50)
                          │     └─► Loop Over Items
                          │           ├─► Fetch matching template (Google Sheets)
                          │           │     └─► Format HTML (Code1)
                          │           │           └─► Merge with contact data
                          │           │                 └─► Personalise subject + body (Edit Fields)
                          │           │                       ├─► Send email (Gmail)
                          │           │                       │     └─► Wait 15s → loop back
                          │           │                       └─► Merge1
                          │           │                             └─► Wait 1s → Update sheet
                          └─► [Stage > 5] → skip (no email sent)
```

---

## Nodes used

| Node | Purpose |
|---|---|
| Schedule Trigger | Fires the workflow daily via cron expression |
| If | Blocks execution on weekends (Saturday = 6, Sunday = 0) |
| Google Sheets (read contacts) | Pulls all contact rows from the outreach tracker sheet |
| Code (field mapping) | Parses dates, maps columns, builds HTML email signature |
| Switch | Routes contacts by follow-up number and days since last contact |
| Merge2 | Consolidates the 3 active follow-up routes into one stream |
| Code2 (batch throttle) | Randomly caps the batch at 40–50 contacts to protect sender reputation |
| Loop Over Items | Iterates one contact at a time |
| Google Sheets (read templates) | Looks up the correct template by `follow_up_number` + `designation` |
| Code1 (HTML formatter) | Wraps template in HTML div, converts line breaks, appends unsubscribe link |
| Merge | Combines contact data with fetched template |
| Edit Fields (Set) | Replaces all `{placeholders}` in template body and subject line |
| Gmail | Sends the personalised email |
| Wait (15s) | Adds a 15-second gap between sends |
| Merge1 | Combines send output with contact data for status update |
| Wait1 (1s) | Brief pause before writing back to sheet |
| Google Sheets (update) | Appends/updates status, follow-up number, and date in the tracker |

---

## Google Sheets setup

### Sheet 1 — Contacts tracker

| Column | Description |
|---|---|
| `First Name` | Contact's first name |
| `Last Name` | Contact's last name |
| `Title` | Job title (e.g. Senior Director, VP) |
| `Company` | Company name |
| `Company Link` | Company website URL |
| `Email` | Contact email address (used as match key) |
| `Person Linkedin Url` | LinkedIn profile URL |
| `Company Country` | Country of the company |
| `designation` | Role category used to select template (e.g. `KOL`, `Payer`, `HEOR`) |
| `drug_name` | Drug or product name to personalise the email |
| `indication` | Therapeutic area or disease indication |
| `phase` | Clinical trial phase (e.g. Phase II, Phase III) |
| `status` | Current email status (e.g. `SENT`, `INBOX`) — auto-updated by workflow |
| `folow_up_number` | Integer tracking which follow-up this contact is on (starts at 1) |
| `date` | Date of last email sent (dd/mm/yyyy format) — auto-updated |

### Sheet 2 — Email templates

| Column | Description |
|---|---|
| `follow_up_number` | Which follow-up this template is for (1, 2, 3 …) |
| `designation` | Role category this template targets (must match Sheet 1 `designation` values) |
| `Subject` | Email subject line — supports placeholders |
| `Template` | Full email body — supports all placeholders listed below |

### Supported placeholders

| Placeholder | Replaced with |
|---|---|
| `{FirstName}` | Contact's first name |
| `{Company}` | Contact's company |
| `{DrugName}` | Drug or product name |
| `{DrugName/TherapeuticArea}` | Drug name (alternate placeholder) |
| `{Phase}` | Clinical phase |
| `{TherapeuticArea}` | Disease indication |
| `{YourName}` | Your HTML email signature block |
| `{Email}` | Contact's email address |

---

## Follow-up logic (Switch node)

| Condition | Output | Action |
|---|---|---|
| Follow-up 3–5 AND last email > 5 days ago | Output 0 | Send next follow-up |
| Follow-up = 2 AND last email > 3 days ago | Output 1 | Send follow-up 2 |
| Follow-up = 1 AND last email > 3 days ago | Output 2 | Send follow-up 1 |
| Follow-up > 5 | Output 3 | Skip — sequence exhausted |
| Otherwise | Output 4 | Skip — too soon to follow up |

---

## Setup instructions

### Prerequisites
- n8n instance (cloud or self-hosted)
- Google account with Sheets and Gmail access
- OAuth2 credentials configured in n8n for both Google Sheets and Gmail

### Step 1 — Set up your Google Sheets
1. Create Sheet 1 (contacts tracker) with the columns listed above
2. Create Sheet 2 (email templates) with your copy for each follow-up stage and designation
3. Copy the Sheet IDs from the URL of each sheet (the long string between `/d/` and `/edit`)

### Step 2 — Import the workflow
1. In n8n, go to **Workflows → Import**
2. Upload `email-outreach-followup-sequence.json`

### Step 3 — Connect your credentials
1. Open the **Google Sheets** nodes and select your Google Sheets OAuth2 credential
2. Update the `documentId` in each Google Sheets node with your actual Sheet IDs
3. Open the **Gmail** node and connect your Gmail OAuth2 credential

### Step 4 — Update your signature
In the **Code** node, replace:
- `YOUR_NAME` → your full name
- `YOUR_TITLE` → your job title
- `YOUR_EMAIL` → your email address
- `YOUR_COMPANY` → your company name
- `YOUR_COMPANY_WEBSITE` → your company URL
- `YOUR_LINKEDIN_URL` → your LinkedIn profile URL
- `YOUR_LOGO_IMAGE_URL` → a publicly accessible URL to your logo image

### Step 5 — Update sender name and unsubscribe link
- In the **Gmail** node → Sender Name: replace `YOUR_SENDER_NAME` with your first name
- In **Code1**: replace `YOUR_UNSUBSCRIBE_URL` with your unsubscribe endpoint URL

### Step 6 — Set your send time
The Schedule Trigger is set to `0 30 22 * * *` (22:30 UTC daily). Adjust to your preferred time.

### Step 7 — Activate
Toggle the workflow to **Active**. It will fire at the next scheduled time on a weekday.

---

## Customisation tips

**Change the batch size:** In Code2, adjust the `40` and `50` values to set your desired min/max emails per run.

**Adjust follow-up intervals:** In the Switch node, change `5 * 24 * 60 * 60 * 1000` (5 days) and `3 * 24 * 60 * 60 * 1000` (3 days) to your preferred waiting periods.

**Add more designation types:** Add new rows to your templates sheet with a new `designation` value — the lookup matches automatically.

**Extend the sequence:** Add template rows for follow-up numbers 6, 7, etc. and update the Switch condition threshold.

---

## Industry use case

Built for outreach in the **life sciences industry**, targeting:
- HEOR (Health Economics and Outcomes Research) professionals
- Payers and market access teams
- KOLs (Key Opinion Leaders) in specific therapeutic areas
- Biotech and pharma business development contacts

---

## License

MIT — free to use, adapt, and share. Attribution appreciated but not required.
```

---

### After pasting — commit with this message:
```
Replace README with full workflow documentation
