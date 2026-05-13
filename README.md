# Inbound Lead Qualifier & Router

An AI-powered automation that scores, tags, and routes inbound contact form submissions for marketing agencies — automatically, in under 30 seconds per lead.

When a B2B agency misses a hot lead because it sits in an inbox for six hours, that's often $3,000 to $10,000 in lost revenue. This system catches every form submission, lets an AI score it 0–100 based on fit, intent, and timing, then routes it to the right place — Slack ping for hot leads, soft auto-reply for warm and cold, silent archive for spam.

Built as a drop-in backend for any form provider — Webflow, Tally, Typeform, custom HTML, or AI website builders. Plug the webhook URL into your form's submit action and you're live.

---

## What it does

1. **Webhook** receives the form submission as JSON
2. **Code node** extracts the email domain and flags freemail addresses (gmail, yahoo, etc.)
3. **Groq AI (Llama 3.3 70B)** scores the lead 0–100 across three dimensions:
   - **Fit** — business email, B2B/SaaS/e-commerce mentioned, company size
   - **Intent** — budget specified, service requested, message specificity
   - **Timing** — start date, urgency, decision stage
4. **Parse node** converts the AI response into clean structured data
5. **Airtable** stores every lead with score, tier, and AI reasoning
6. **Branching logic** routes by tier:
   - **Hot leads** → instant Slack ping to `#leads-hot` + personalized Gmail reply
   - **Warm + Cold** → archived in `#leads-all` + soft auto-reply via Gmail
   - **Spam** → archived silently, no human notification, no email sent

---

## Tech stack

n8n (self-hosted, v2.16+) · Groq API (Llama 3.3 70B) · Airtable · Slack · Gmail

---

## Demo

Five sample lead payloads are included in `sample_payloads/` — Hot, Warm, Cold, Spam, and a second Hot variant. The screenshots below show the system handling all five.

### Test request

```powershell
$body = Get-Content sample_payloads/01_hot_lead.json -Raw
Invoke-RestMethod -Uri "http://localhost:5678/webhook/lead-intake" -Method POST -ContentType "application/json" -Body $body
```

### Screenshots

| Stage | What you see |
|-------|--------------|
| ![PowerShell test](screenshots/01_powershell.png) | Test request sent via PowerShell, 200 Acknowledged response |
| ![n8n canvas](screenshots/05_n8n_canvas.png) | Full workflow executed end-to-end, all nodes green |
| ![Airtable](screenshots/02_airtable.png) | All sample leads scored, tiered, and stored |
| ![Slack hot lead](screenshots/03_slack_hot.png) | Hot lead alert posted to `#leads-hot` |
| ![Gmail sent](screenshots/04_gmail_sent.png) | Personalized auto-replies sent to leads |

---

## Setup — step by step

You don't need to be a developer to set this up. Follow each section in order. Estimated time: **45–60 minutes**, faster if you've used these tools before.

### Step 1: Install n8n (skip if you already have it)

Easiest option — Docker on your local machine:

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

Open `http://localhost:5678` in your browser. Set up an admin account.

Alternative: use **n8n Cloud** (paid, no install needed) — works the same way.

### Step 2: Import this workflow

1. Download `workflow.json` from this repository
2. In n8n: open the workflow editor → click the **three dots menu** (top-right) → **Import from File** → select `workflow.json`
3. You'll see all 10 nodes appear on the canvas, but several will show errors. That's expected — we'll fix them in the next steps by adding your credentials.

### Step 3: Get a Groq API key (free)

Groq is the AI provider this workflow uses for lead scoring. The free tier is more than enough.

1. Go to **https://console.groq.com**
2. Sign in (Google login is fastest)
3. Left sidebar → **API Keys** → **Create API Key**
4. Name it `n8n Lead Qualifier` → Create
5. **Copy the key immediately** (starts with `gsk_...`) — you only see it once
6. In n8n: open the **AI Score Lead** node
7. In the **Headers** section, find the row where Name = `Authorization`
8. Replace `YOUR_GROQ_API_KEY_HERE` with: `Bearer ` followed by your key
   - Example: `Bearer gsk_abc123xyz...`
9. Save the node

### Step 4: Set up Airtable

#### 4a. Create the base

You need an Airtable base called "Lead Pipeline" with one table named "Leads".

1. Go to **https://airtable.com** → sign up free → click **+ Add a base** → **Start from scratch**
2. Name the base: `Lead Pipeline`
3. Rename the default table to: `Leads`
4. Create these 13 fields (click the **+** at the end of column headers):

| Field name | Type | Notes |
|---|---|---|
| Name | Single line text | Primary field |
| Email | Email | |
| Company | Single line text | |
| Company Domain | Single line text | |
| Message | Long text | |
| Lead Source | Single select | Options: Website, LinkedIn, Referral, Cold Outreach, Other |
| Score | Number | Integer, 0–100 |
| Tier | Single select | Options: Hot (red), Warm (orange), Cold (blue), Spam (gray) |
| Reasoning | Long text | AI's explanation |
| Company Size | Single select | Options: 1-10, 11-50, 51-200, 200+, Unknown |
| Status | Single select | Options: New, Contacted, Qualified, Disqualified, Closed Won, Closed Lost |
| Submitted At | Date | Toggle "Include time" ON |
| Slack Notified | Checkbox | |

#### 4b. Create an Airtable Personal Access Token

1. Go to **https://airtable.com/create/tokens** → **Create new token**
2. Name: `n8n Lead Pipeline`
3. **Scopes** — add ALL THREE:
   - `data.records:read`
   - `data.records:write`
   - `schema.bases:read`
4. **Access** — add the `Lead Pipeline` base only (do not grant access to all bases)
5. **Create token** → copy it immediately (starts with `pat...`)

#### 4c. Get your Base ID and Table ID

Open your `Lead Pipeline` base. Look at the URL in your browser:

```
https://airtable.com/appXXXXXXXXXX/tblYYYYYYYYYY/...
                    └── Base ID ──┘└── Table ID ──┘
```

Copy both the `app...` part (Base ID) and the `tbl...` part (Table ID).

#### 4d. Wire Airtable into n8n

1. In n8n: open the **Create Lead Record** node
2. Click the credential dropdown → **Create New Credential**
3. Credential type: **Airtable Personal Access Token API**
4. Paste your token → name it `Airtable - Lead Pipeline` → Save
5. Back in the node:
   - **Base:** click "By ID" → paste your Base ID (`app...`)
   - **Table:** click "By ID" → paste your Table ID (`tbl...`)
6. The field mapping should populate. If you see "No columns found", click **Retry**.

### Step 5: Set up Slack

#### 5a. Create a workspace and channels

If you don't already have a Slack workspace:

1. Go to **https://slack.com/get-started** → sign up
2. Create a new workspace (name it anything)
3. Inside the workspace, create two channels:
   - `#leads-hot`
   - `#leads-all`

#### 5b. Create a Slack App

1. Go to **https://api.slack.com/apps** → **Create New App** → **From scratch**
2. App Name: `n8n Lead Bot` → choose your workspace → **Create App**

#### 5c. Add permissions

1. Left sidebar → **OAuth & Permissions**
2. Scroll to **Scopes** → **Bot Token Scopes**
3. Click **Add an OAuth Scope** and add all 4:
   - `chat:write`
   - `chat:write.public`
   - `channels:read`
   - `groups:read`

#### 5d. Install the app

1. Scroll back to the top of the OAuth & Permissions page
2. Click **Install to Workspace** → **Allow**
3. Copy the **Bot User OAuth Token** that appears (starts with `xoxb-...`)

#### 5e. Invite the bot to both channels

In Slack, in `#leads-hot`, type:
```
/invite @n8n Lead Bot
```
Press Enter. Repeat in `#leads-all`.

#### 5f. Wire Slack into n8n

1. In n8n: open the **Slack: Hot Lead Alert** node
2. Credential dropdown → **Create New Credential**
3. Credential type: **Slack API** (NOT OAuth2)
4. **Access Token:** paste your `xoxb-...` token
5. **Signature Secret:** leave empty (not needed)
6. Save credential as `Slack - n8n Lead Bot`
7. **Channel:** type `leads-hot` (no `#`) — should appear in the dropdown
8. Now open **Slack: All Leads Log** node:
   - Use the same credential
   - **Channel:** `leads-all`

### Step 6: Set up Gmail

1. In n8n: open the **Gmail: Hot Reply** node
2. Credential dropdown → **Create New Credential**
3. Credential type: **Gmail OAuth2 API**
4. Click **Sign in with Google** → grant permissions (allow read + send)
   - If you see "Google hasn't verified this app" warning: click **Advanced** → **Go to n8n (unsafe)**. This is normal for local n8n installs.
5. Save credential as `Gmail - Personal`
6. Open the **Gmail: Auto Reply** node and select the same credential

### Step 7: Activate and test

1. Top-right corner of the workflow editor → toggle **Active** (turns green)
2. Click the **Webhook** node → copy the **Production URL** (the one without `-test`)
3. Open PowerShell on your machine and run:

```powershell
$body = '{"name":"Sarah Chen","email":"sarah@northbeam-saas.io","company":"Northbeam SaaS","source":"Website","message":"B2B SaaS at 2M ARR, need help scaling content marketing. Budget 8k per month, looking to start in 3 weeks."}'

Invoke-RestMethod -Uri "http://localhost:5678/webhook/lead-intake" -Method POST -ContentType "application/json" -Body $body
```

(Replace `localhost:5678` with your n8n host if different.)

4. Check that:
   - PowerShell shows `Acknowledged`
   - n8n executions tab shows a green run
   - Airtable has a new row for Sarah Chen tagged "Hot"
   - Slack `#leads-hot` shows the alert
   - Gmail Sent folder shows a personalized reply

If all four happen, the system is live.

---

## Form-agnostic by design

This is a backend system. Plug the webhook URL into any form provider:

- Webflow form action
- Tally.so or Typeform webhooks
- Custom HTML / React forms
- AI website builders (Lovable, Base44, v0, Bolt)
- WordPress contact form plugins
- Zapier/Make/n8n triggers from other tools

JSON body shape required:

```json
{
  "name": "string",
  "email": "string",
  "company": "string",
  "source": "string",
  "message": "string"
}
```

---

## What I'd add for a production client

This repo is a portfolio piece. For a paying client I would extend it with:

- LinkedIn enrichment via Apollo.io (company size, recent funding news, decision-maker contact)
- Auto-book qualified hot leads into the agency's Calendly
- Weekly digest of cold leads for re-engagement campaigns
- Custom scoring weights per agency vertical (DTC vs SaaS vs services)
- HMAC signing on the webhook to block bot spam
- Daily pipeline metrics report sent to the founder

---

## Troubleshooting

**Webhook returns 404 "not registered"**
The workflow is Inactive. Toggle it Active in the top-right corner. The Test URL only listens for one request — for repeated testing, use the Production URL.

**Airtable error "INVALID_REQUEST_MISSING_FIELDS"**
Field name in n8n doesn't match Airtable exactly, including capitalization. "Company Size" ≠ "company size".

**Slack message has literal `{{ $json.score }}` text instead of a number**
You typed the expression in Fixed mode. Toggle the field to Expression mode using the `fx` icon.

**Gmail OAuth fails**
Re-authenticate the credential. Gmail tokens can expire; clicking "Reconnect" usually resolves it.

**Groq returns 401 Unauthorized**
API key is missing the `Bearer ` prefix (with the space) before the actual key, or the key was copied wrong. Re-copy from console.groq.com.

---

## Why I built this

I'm an n8n + AI automation specialist focused on marketing and sales agencies. This is one of three flagship projects in my portfolio — the others tackle client onboarding and content repurposing. Together they form an end-to-end agency growth automation stack.

If you run an agency and want a custom version of this wired into your stack, I'd love to talk.

**GitHub:** [@Usman-rai](https://github.com/Usman-rai)

---

## License

MIT — use it, fork it, ship it. Attribution appreciated but not required.
