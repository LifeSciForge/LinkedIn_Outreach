# 02 — LinkedIn Outreach Automation with AI Personalisation

> **Part of the [LifeSciForge](https://github.com/LifeSciForge) portfolio — 10 AI projects targeting 50 LPA roles in Life Sciences**

An n8n workflow that automates LinkedIn connection requests for pharma and life sciences professionals. It fetches live LinkedIn profile data, detects the prospect's functional area (Medical Affairs, HEOR, Market Access, etc.), generates a personalised 100–120 character connection note using Claude AI, validates message quality, checks connection status, sends the request with human-like random timing, and logs everything to Google Sheets.

---

## 🎯 What This Workflow Does

1. **Triggers** Monday–Friday at 11am automatically
2. **Reads** unsent contacts from Google Sheets
3. **Applies** a random daily limit of 8–12 connections (human-like behaviour)
4. **Fetches** live LinkedIn profile data via Browserflow
5. **Detects** functional area from job title (7 pharma-specific categories)
6. **Generates** a personalised AI connection note using Claude API (100–120 chars)
7. **Validates** message quality across 5 checks before sending
8. **Checks** connection status — skips if already connected or pending
9. **Sends** connection WITH personalised message (falls back to blank if unsupported)
10. **Logs** every outcome to Google Sheets: `YES + MESSAGE`, `YES (BLANK)`, or `SKIPPED`
11. **Waits** randomly between contacts (10–25 min) to mimic human behaviour

---

## 🏗️ Workflow Architecture

```
Schedule Trigger (Mon-Fri, 11am)
        │
        ▼
Generate Random Time (1 min startup delay)
        │
        ▼
Wait Until Start Time
        │
        ▼
Get Unsent Connections (Google Sheets)
        │
        ▼
Apply Random Limit (8-12 per day)
        │
        ▼
Process One by One (SplitInBatches)
        │
        ▼
Get LinkedIn Profile Data (Browserflow)
        │
        ▼
Wait After Profile (2 min)
        │
        ▼
Identify Functional Area
[medical affairs / market access / clinical / regulatory / commercial / bd / heor]
        │
        ▼
Generate AI Message (Claude Sonnet)
        │
        ▼
Validate Message Quality
[length 100-120 / has AI reference / first person / starts Hi Name, / ends with ?]
        │
   ┌────┴────┐
PASSED     FAILED
   │           │
   ▼           ▼
Check       Skip to
Connection  Next Person
Status
   │
   ├── Already connected/pending → Update Sheet (SKIPPED) → Wait → Next
   │
   ▼
Calculate Pre-Send Wait (15-35 min random)
        │
        ▼
Wait Before Send
        │
        ▼
Try Send Connection WITH Message (Browserflow)
        │
        ▼
Wait After Send (2 min)
        │
        ▼
Check Send Success
        │
   ┌────┴────┐
SUCCESS    FAILED
   │           │
   ▼           ▼
Update     Send Blank
Sheet      Connection
(YES+MSG)  Request
   │           │
   ▼           ▼
Calculate  Update Sheet
Next Wait  (YES BLANK)
   │           │
   └─────┬─────┘
         ▼
   Wait Next Person
   (10-25 min random)
         │
         ▼
   Back to: Process One by One
```

---

## 🧩 Nodes Used

| Node | Type | Purpose |
|---|---|---|
| Schedule Trigger | Trigger | Runs Mon–Fri at 11am |
| Generate Random Time | Code | Startup delay logic |
| Wait Until Start Time | Wait | Pause before processing |
| Get Unsent Connections | Google Sheets | Read contacts where status is blank |
| Apply Random Limit | Code | Filter to 8–12 unsent contacts per day |
| Process One by One | SplitInBatches | Process contacts sequentially |
| Get LinkedIn Profile Data | Browserflow | Fetch live profile name, title, company |
| Wait After Profile | Wait | 2 min pause after profile fetch |
| Identify Functional Area | Code | Map job title to pharma function |
| Generate AI Message | LangChain LLM Chain | Prompt Claude to write connection note |
| Claude AI Model | Anthropic (Claude) | claude-sonnet-4-20250514, temp 0.7 |
| Validate Message Quality | Code | 5-check quality gate |
| Message Quality Check | IF | Route pass/fail |
| Check Connection Status | Browserflow | Check is_connection and is_pending |
| Wait After Check | Wait | 1 min pause |
| Can Send Request | IF | Route send/skip |
| Calculate Pre-Send Wait | Code | Random 15–35 min wait |
| Wait Before Send | Wait | Human behaviour simulation |
| Try Send Connection WITH Message | Browserflow | Send with AI note |
| Wait After Send Attempt | Wait | 2 min pause |
| Check Send Success | Code | Detect send success or failure |
| Message Sent Successfully | IF | Route success/fallback |
| Update Sheet – Sent with Message | Google Sheets | Log YES + MESSAGE |
| Send Blank Connection Request | Browserflow | Fallback — send without note |
| Wait After Blank Send | Wait | 2 min pause |
| Update Sheet – Sent Blank | Google Sheets | Log YES (BLANK) |
| Calculate Next Person Wait | Code | Random 10–25 min inter-contact delay |
| Wait Next Person | Wait | Inter-contact pause |
| Update Sheet – Skip | Google Sheets | Log SKIPPED |
| Calculate Skip Wait | Code | Random 5–15 min skip delay |
| Wait Skip | Wait | Skip pause before next contact |

---

## 📊 Google Sheets Setup

Create a Google Sheet with the following columns in **Sheet1**:

| Column | Description | Example |
|---|---|---|
| `First Name` | Prospect first name | Rajesh |
| `Last Name` | Prospect last name | Sharma |
| `Person Linkedin Url` | Full LinkedIn profile URL | https://linkedin.com/in/rajesh-sharma |
| `Title` | Job title (used for fallback area detection) | Head of Market Access |
| `Company` | Company name | Novartis |
| `LinkedIn Connection Sent` | Status — **leave blank for new contacts** | YES + MESSAGE |
| `Connection Request Date` | Auto-filled by workflow | 2026-03-15 |
| `Message Sent` | Auto-filled — exact message sent | Hi Rajesh, we have developed... |
| `Connection Accepted Date` | Fill manually when they accept | 2026-03-18 |

**Important:**
- Leave `LinkedIn Connection Sent` blank for contacts you want to send to
- The workflow filters to blank rows only — it will not re-send to contacts already processed
- `Person Linkedin Url` is the unique key used for matching and updating rows

---

## ⚙️ Setup Instructions

### Step 1 — Prerequisites
Before importing this workflow, ensure you have:
- n8n running (local, cloud, or self-hosted)
- A [Browserflow](https://browserflow.app) account with LinkedIn connected
- An [Anthropic API key](https://console.anthropic.com) for Claude
- A Google account with Sheets API access enabled in n8n

### Step 2 — Import the workflow
1. Open n8n → click **"Workflows"** in the left sidebar
2. Click **"Import from File"**
3. Upload `linkedin-outreach-ai-personalisation.json`
4. Click **"Import"**

### Step 3 — Set up credentials
Add the following credentials in n8n (**Settings → Credentials → Add Credential**):

| Credential | Type | Where to get it |
|---|---|---|
| Google Sheets account | Google Sheets OAuth2 | Google Cloud Console |
| Browserflow account | Browserflow API | browserflow.app → Settings → API Key |
| Anthropic (Claude) account | Anthropic API | console.anthropic.com → API Keys |

### Step 4 — Replace placeholders
Search for and replace these values in the workflow after importing:

| Placeholder | Replace with |
|---|---|
| `YOUR_GOOGLE_SHEET_ID` | Your Sheet ID from the URL |
| `YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` | ID from n8n credentials |
| `YOUR_BROWSERFLOW_CREDENTIAL_ID` | ID from n8n credentials |
| `YOUR_ANTHROPIC_CREDENTIAL_ID` | ID from n8n credentials |
| `YOUR_COMPANY` | Your company name (in Generate AI Message node prompt) |
| `YOUR_PRODUCT` | Your product name (in Generate AI Message node prompt) |
| `YOUR_WEBHOOK_ID_1` through `_8` | Any unique strings (e.g. `wait-start-001`) |
| `YOUR_N8N_INSTANCE_ID` | Your n8n instance ID |
| `YOUR_WORKFLOW_ID` | Auto-assigned after import |

### Step 5 — Add contacts to Google Sheets
Add your prospect list to the Google Sheet. Required columns: `First Name`, `Last Name`, `Person Linkedin Url`, `Title`, `Company`. Leave `LinkedIn Connection Sent` blank.

### Step 6 — Test the workflow
1. Set the Schedule Trigger to **Manual** trigger temporarily
2. Run the workflow with 1–2 test contacts
3. Verify the AI message is generated and logged correctly
4. Check that Google Sheet updates with correct status

### Step 7 — Activate
Switch the trigger back to **Schedule** and toggle the workflow to **Active**.

---

## 🔒 Safety & LinkedIn Compliance

This workflow is built with multiple safety features to reduce risk of LinkedIn account restriction:

- **Daily limit**: 8–12 connections per day (well below LinkedIn's ~100/day limit)
- **Weekday only**: Monday–Friday only — no weekend activity
- **Random timing**: All waits are randomised to avoid predictable patterns
- **Duplicate check**: Checks connection status before every send
- **Message validation**: Validates quality before sending — no spammy messages
- **Sequential processing**: One contact at a time — never bulk sends

> ⚠️ **Disclaimer**: This workflow uses LinkedIn automation via Browserflow. Use responsibly and within LinkedIn's Terms of Service. The author takes no responsibility for account restrictions.

---

## 🧠 AI Prompt Logic

The workflow uses **Claude Sonnet** to generate personalised connection notes. The prompt:

- Identifies the recipient's pharma functional area from their job title
- Selects function-specific pain points and benefits
- Generates a 100–120 character message (strict limit for highest acceptance rates)
- Always starts with `Hi {FirstName},` and ends with a question (CTA)
- References AI product capability relevant to their specific role

**Supported functional areas:**
`Medical Affairs` · `Market Access` · `Clinical Development` · `Regulatory` · `Commercial` · `Business Development` · `HEOR`

---

## 📈 Expected Outcomes

Based on production usage:
- **Connection acceptance rate**: 28–35% (vs ~18% industry average for cold outreach)
- **Daily volume**: 8–12 connections per run
- **Monthly volume**: ~200–240 new connections per month
- **Time saved**: ~2–3 hours per week vs manual outreach

---

## 🛠️ Customisation Tips

- **Change daily limit**: Edit `minConnections` and `maxConnections` in the `Apply Random Limit` node
- **Change trigger time**: Edit `triggerAtHour` in the `Schedule Trigger` node
- **Add new functional areas**: Add new entries to `departmentMap` in `Identify Functional Area`
- **Adjust message tone**: Edit the prompt in `Generate AI Message` node
- **Change message length**: Adjust the `length` check in `Validate Message Quality`

---

## 📁 Repository Structure

```
LinkedIn_Outreach/
├── linkedin-outreach-ai-personalisation.json   # n8n workflow (sanitised)
├── README.md                                    # This file
└── Workflow_Documentation.docx                  # Full setup guide
```

---

## 🔗 Related Projects

- [01 — Email Outreach Automation](https://github.com/LifeSciForge/Email_Outreach)
- [03 — Pharma CI Research Agent](https://github.com/LifeSciForge) *(coming soon)*

---

## 👤 Author

**Pranjal Das** — CSO, Pienomial · Bengaluru, India

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Pranjal%20Das-0077B5?style=flat&logo=linkedin)](https://linkedin.com/in/pranjal-das1)
[![GitHub](https://img.shields.io/badge/GitHub-LifeSciForge-181717?style=flat&logo=github)](https://github.com/LifeSciForge)

---

*Part of the LifeSciForge portfolio — 10 AI projects built to demonstrate production-grade automation for life sciences roles targeting 50 LPA.*
