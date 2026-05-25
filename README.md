# 🤖 Multi-Role Hiring Automation System
### Built with n8n · Google Gemini 2.5 Flash · Google Workspace

> A fully automated, AI-powered hiring pipeline for **Software Engineer (SWE)** and **Business Development Manager (BDM)** roles — built as part of a paid internship assignment at **Trilles AI**.

---

## 📌 Overview

This system automates the entire hiring pipeline from candidate intake to email communication — with zero manual intervention. It reads applications from Google Forms, screens resumes using Google Gemini 2.5 Flash, stores results in a central Master Database, and sends personalised HTML emails to candidates based on their AI classification.

The system is split into **two independent n8n workflows**:

| Workflow | Schedule | Responsibility |
|---|---|---|
| **Workflow 1 — Hiring Screener** | Every 1 hour | Reads new applications, AI screens resumes, writes to Master DB |
| **Workflow 2 — Emailer** | Every 2 hours | Reads Master DB, routes by classification, sends emails + calendar invites |

---

## 🗂️ System Architecture

![n8n Workflow Canvas](./assets/workflow-canvas.png)

### Data Flow

```
Google Form (SWE) ──┐
                    ├──► Merge ──► Filter New ──► Download Resume PDF
Google Form (BDM) ──┘                                    │
                                                         ▼
                                               Extract Text ──► Build Prompt
                                                         │
                                                         ▼
                                              Gemini 2.5 Flash API
                                                         │
                                                         ▼
                                               Parse AI Response
                                                         │
                                                         ▼
                                              Write to Master DB
                                                         │
                                                         ▼
                                          Mark Original Sheet as Processed


Master DB (status=screened)
          │
          ▼
   Route by Classification
    ┌─────┼──────┐
  strong average weak
    │       │      │
  Email  Notify  Reject
  +Cal   Manager Email
    │       │      │
    └───────┴──────┘
          │
          ▼
   Update status = contacted
```

---

## 🧩 Workflow 1 — Hiring Screener (18 nodes)

| # | Node | Type | Purpose |
|---|---|---|---|
| 1 | Schedule Trigger | Trigger | Fires every hour |
| 2 | Read SWE Sheet | Google Sheets | Reads all SWE form responses |
| 3 | Read BDM Sheet | Google Sheets | Reads all BDM form responses |
| 4 | Standardize SWE Data | Edit Fields | Normalises columns, stamps `role: SWE` |
| 5 | Standardize BDM Data | Edit Fields | Normalises columns, stamps `role: BDM` |
| 6 | Merge All Applications | Merge (Append) | Combines both role streams into one |
| 7 | Read Master DB | Google Sheets | Reads existing screened candidates |
| 8 | Filter New Only | Merge (Non-Match) | Removes already-processed emails |
| 9 | Any New Applications? | If | Exits gracefully if no new candidates |
| 10 | Extract File ID | Code | Parses Google Drive URL to extract file ID |
| 11 | Download File | Google Drive | Downloads resume PDF by file ID |
| 12 | Extract Resume Text | Extract From PDF | Converts PDF to plain text |
| 13 | Attach Resume Text | Edit Fields | Merges resume text back with candidate data |
| 14 | Build AI Prompt | Code | Constructs role-specific Gemini prompt |
| 15 | AI Screen Candidate | HTTP Request | POSTs to Gemini 2.5 Flash API |
| 16 | Parse AI Response | Code | Parses structured JSON from Gemini |
| 17 | Write to Master DB | Google Sheets | Appends screened candidate to Master DB |
| 18 | Mark as Processed | If + 2x Google Sheets | Updates original role sheet row to `processed` |

---

## 📧 Workflow 2 — Emailer (7 nodes)

| # | Node | Type | Purpose |
|---|---|---|---|
| 1 | Schedule Trigger | Trigger | Fires every 2 hours |
| 2 | Read Master DB | Google Sheets | Reads all candidates |
| 3 | Filter Screened Only | If | Keeps only `status = screened` |
| 4 | Route by Classification | Switch | 3 outputs: strong / average / weak |
| 5a | Send Interview Invitation | Gmail | HTML invite email to strong candidate |
| 5b | Create Calendar Event | Google Calendar | Interview event with candidate as attendee |
| 5c | Notify Hiring Manager | Gmail | HTML summary email to hiring manager |
| 5d | Send Rejection Email | Gmail | Professional HTML rejection to weak candidate |
| 6 | Merge All Branches | Merge (Append) | Reunites all 3 branches |
| 7 | Update Master DB Status | Google Sheets | Sets `status=contacted`, stamps `contacted_at` |

---

## 🧠 AI Screening Criteria

The AI prompt is role-specific, built dynamically in the **Build AI Prompt** Code node.

### Software Engineer (SWE)
Evaluated on:
- Programming languages and frameworks
- Years and quality of technical experience
- System design and architecture knowledge
- DevOps, databases, and tooling
- Overall technical depth

### Business Development Manager (BDM)
Evaluated on:
- Sales and revenue generation track record
- B2B relationship and pipeline management
- Negotiation and closing ability
- Strategic thinking and market knowledge
- Leadership and communication skills

### AI Output Schema
```json
{
  "score": 85,
  "classification": "strong | average | weak",
  "summary": "2-3 sentence candidate summary",
  "strengths": "Key strengths identified",
  "concerns": "Key concerns or gaps",
  "recommendation": "Interview recommendation"
}
```

> **Model:** Google Gemini 2.5 Flash · **Temperature:** 0.3 (low, for consistent structured output)

---

## 📋 Master Database Schema

| Column | Description |
|---|---|
| `candidate_id` | Unix timestamp used as unique ID |
| `timestamp` | Form submission time |
| `name` | Candidate full name |
| `email` | Candidate email (used for deduplication) |
| `role` | SWE or BDM |
| `resume_url` | Google Drive link to resume PDF |
| `score` | AI score (1–100) |
| `classification` | strong / average / weak |
| `summary` | AI-generated candidate summary |
| `strengths` | AI-identified strengths |
| `concerns` | AI-identified concerns |
| `recommendation` | AI recommendation |
| `status` | screened → contacted |
| `contacted_at` | Timestamp when email was sent |

---

## 📝 Application Forms

| Role | Google Form Link |
|---|---|
| Software Engineer (SWE) | [Apply Here](https://docs.google.com/forms/d/e/1FAIpQLSdh-Me9MM3lkDMNpE-46mBiEy0UCe3JnxvG9MLJU1QJXdbxiA/viewform?usp=publish-editor) |
| Business Development Manager (BDM) | [Apply Here](https://docs.google.com/forms/d/e/1FAIpQLSemMg0J62Mk4p9wXf792_K250UX139DfVASeeONvjyNTmbnzQ/viewform?usp=publish-editor) |

---

## 🛠️ Tech Stack

| Component | Technology |
|---|---|
| Workflow Engine | n8n (self-hosted via Docker) |
| AI Screening | Google Gemini 2.5 Flash (via HTTP Request) |
| Application Intake | Google Forms + Google Sheets |
| Central Database | Google Sheets (Master DB) |
| Resume Storage | Google Drive (public viewer links) |
| Email Delivery | Gmail (via Google OAuth) |
| Calendar Scheduling | Google Calendar (via Google OAuth) |
| Authentication | Google OAuth 2.0 (single credential) |

---

## ⚙️ Setup & Installation

### Prerequisites
- n8n self-hosted via Docker
- Google account with OAuth credentials configured
- Google Gemini API key
- Google Forms linked to Google Sheets (one per role)

### Google OAuth Setup
1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create a new project and enable: Gmail API, Google Sheets API, Google Drive API, Google Calendar API
3. Create OAuth 2.0 credentials
4. Add the credential in n8n under **Credentials → Google OAuth2**

### Environment Variables
```env
GEMINI_API_KEY=your_gemini_api_key_here
```

### Workflow Import
1. Clone this repository
2. Import `workflow1-hiring-screener.json` into n8n
3. Import `workflow2-emailer.json` into n8n
4. Update Google Sheets IDs and Gmail addresses in each node
5. Add your Gemini API key to the HTTP Request node URL
6. Activate both workflows

---

## 🔄 Deduplication Logic

The system has **two independent deduplication layers** to prevent any candidate from being processed twice:

1. **Filter New Only node** — compares incoming application emails against the Master DB using a Merge (Keep Non-Matches) operation before any AI screening occurs
2. **Processed flag** — after screening, the original role sheet row is updated to `status: processed`, providing a secondary guard independent of the Master DB

---

## ⚠️ Known Limitations

- No retry logic if Gemini API call fails — candidate will be reprocessed on the next hourly run
- Resume text is not stored in Master DB (causes Google Sheets cell length errors)
- Calendar events are scheduled at a fixed time — no availability checking
- Classification routing requires exact lowercase values from Gemini (`strong`, `average`, `weak`)
- No email bounce detection — status is marked `contacted` immediately after send

---

## 🚀 Future Improvements

- [ ] Candidate self-scheduling via Calendly link instead of auto-created calendar event
- [ ] Slack/email alert on workflow failure
- [ ] Multi-role scalability via config-driven role definitions
- [ ] Resume text storage in Supabase linked by candidate ID
- [ ] Retry logic with exponential backoff for Gemini API failures
- [ ] Vector search for surfacing similar past candidates (Supabase pgvector)
- [ ] Audit logging table for all workflow runs and outcomes

---

## 👤 Author

**Zehran Wasim**
GitHub: [@Zehran-Wasim](https://github.com/Zehran-Wasim)

Built as part of a paid internship assignment at **Trilles AI** · May 2026
