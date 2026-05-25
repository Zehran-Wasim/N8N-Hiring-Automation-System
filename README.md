🤖 Multi-Role Hiring Automation System
Built with n8n · Google Gemini 2.5 Flash · Google Workspace

A fully automated, AI-powered hiring pipeline for Software Engineer (SWE) and Business Development Manager (BDM) roles — built as part of a paid internship assignment at Trilles AI.


📌 Overview
This system automates the entire hiring pipeline from candidate intake to email communication — with zero manual intervention. It reads applications from Google Forms, screens resumes using Google Gemini 2.5 Flash, stores results in a central Master Database, and sends personalised HTML emails to candidates based on their AI classification.
The system is split into two independent n8n workflows:
WorkflowScheduleResponsibilityWorkflow 1 — Hiring ScreenerEvery 1 hourReads new applications, AI screens resumes, writes to Master DBWorkflow 2 — EmailerEvery 2 hoursReads Master DB, routes by classification, sends emails + calendar invites

🗂️ System Architecture
Show Image
Data Flow
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

🧩 Workflow 1 — Hiring Screener (18 nodes)
#NodeTypePurpose1Schedule TriggerTriggerFires every hour2Read SWE SheetGoogle SheetsReads all SWE form responses3Read BDM SheetGoogle SheetsReads all BDM form responses4Standardize SWE DataEdit FieldsNormalises columns, stamps role: SWE5Standardize BDM DataEdit FieldsNormalises columns, stamps role: BDM6Merge All ApplicationsMerge (Append)Combines both role streams into one7Read Master DBGoogle SheetsReads existing screened candidates8Filter New OnlyMerge (Non-Match)Removes already-processed emails9Any New Applications?IfExits gracefully if no new candidates10Extract File IDCodeParses Google Drive URL to extract file ID11Download FileGoogle DriveDownloads resume PDF by file ID12Extract Resume TextExtract From PDFConverts PDF to plain text13Attach Resume TextEdit FieldsMerges resume text back with candidate data14Build AI PromptCodeConstructs role-specific Gemini prompt15AI Screen CandidateHTTP RequestPOSTs to Gemini 2.5 Flash API16Parse AI ResponseCodeParses structured JSON from Gemini17Write to Master DBGoogle SheetsAppends screened candidate to Master DB18Mark as ProcessedIf + 2x Google SheetsUpdates original role sheet row to processed

📧 Workflow 2 — Emailer (7 nodes)
#NodeTypePurpose1Schedule TriggerTriggerFires every 2 hours2Read Master DBGoogle SheetsReads all candidates3Filter Screened OnlyIfKeeps only status = screened4Route by ClassificationSwitch3 outputs: strong / average / weak5aSend Interview InvitationGmailHTML invite email to strong candidate5bCreate Calendar EventGoogle CalendarInterview event with candidate as attendee5cNotify Hiring ManagerGmailHTML summary email to hiring manager5dSend Rejection EmailGmailProfessional HTML rejection to weak candidate6Merge All BranchesMerge (Append)Reunites all 3 branches7Update Master DB StatusGoogle SheetsSets status=contacted, stamps contacted_at

🧠 AI Screening Criteria
The AI prompt is role-specific, built dynamically in the Build AI Prompt Code node.
Software Engineer (SWE)
Evaluated on:

Programming languages and frameworks
Years and quality of technical experience
System design and architecture knowledge
DevOps, databases, and tooling
Overall technical depth

Business Development Manager (BDM)
Evaluated on:

Sales and revenue generation track record
B2B relationship and pipeline management
Negotiation and closing ability
Strategic thinking and market knowledge
Leadership and communication skills

AI Output Schema
json{
  "score": 85,
  "classification": "strong | average | weak",
  "summary": "2-3 sentence candidate summary",
  "strengths": "Key strengths identified",
  "concerns": "Key concerns or gaps",
  "recommendation": "Interview recommendation"
}

Model: Google Gemini 2.5 Flash · Temperature: 0.3 (low, for consistent structured output)


📋 Master Database Schema
ColumnDescriptioncandidate_idUnix timestamp used as unique IDtimestampForm submission timenameCandidate full nameemailCandidate email (used for deduplication)roleSWE or BDMresume_urlGoogle Drive link to resume PDFscoreAI score (1–100)classificationstrong / average / weaksummaryAI-generated candidate summarystrengthsAI-identified strengthsconcernsAI-identified concernsrecommendationAI recommendationstatusscreened → contactedcontacted_atTimestamp when email was sent

📝 Application Forms
RoleGoogle Form LinkSoftware Engineer (SWE)Apply HereBusiness Development Manager (BDM)Apply Here

🛠️ Tech Stack
ComponentTechnologyWorkflow Enginen8n (self-hosted via Docker)AI ScreeningGoogle Gemini 2.5 Flash (via HTTP Request)Application IntakeGoogle Forms + Google SheetsCentral DatabaseGoogle Sheets (Master DB)Resume StorageGoogle Drive (public viewer links)Email DeliveryGmail (via Google OAuth)Calendar SchedulingGoogle Calendar (via Google OAuth)AuthenticationGoogle OAuth 2.0 (single credential)

⚙️ Setup & Installation
Prerequisites

n8n self-hosted via Docker
Google account with OAuth credentials configured
Google Gemini API key
Google Forms linked to Google Sheets (one per role)

Google OAuth Setup

Go to Google Cloud Console
Create a new project and enable: Gmail API, Google Sheets API, Google Drive API, Google Calendar API
Create OAuth 2.0 credentials
Add the credential in n8n under Credentials → Google OAuth2

Environment Variables
envGEMINI_API_KEY=your_gemini_api_key_here
Workflow Import

Clone this repository
Import workflow1-hiring-screener.json into n8n
Import workflow2-emailer.json into n8n
Update Google Sheets IDs and Gmail addresses in each node
Add your Gemini API key to the HTTP Request node URL
Activate both workflows


🔄 Deduplication Logic
The system has two independent deduplication layers to prevent any candidate from being processed twice:

Filter New Only node — compares incoming application emails against the Master DB using a Merge (Keep Non-Matches) operation before any AI screening occurs
Processed flag — after screening, the original role sheet row is updated to status: processed, providing a secondary guard independent of the Master DB


⚠️ Known Limitations

No retry logic if Gemini API call fails — candidate will be reprocessed on the next hourly run
Resume text is not stored in Master DB (causes Google Sheets cell length errors)
Calendar events are scheduled at a fixed time — no availability checking
Classification routing requires exact lowercase values from Gemini (strong, average, weak)
No email bounce detection — status is marked contacted immediately after send


🚀 Future Improvements

 Candidate self-scheduling via Calendly link instead of auto-created calendar event
 Slack/email alert on workflow failure
 Multi-role scalability via config-driven role definitions
 Resume text storage in Supabase linked by candidate ID
 Retry logic with exponential backoff for Gemini API failures
 Vector search for surfacing similar past candidates (Supabase pgvector)
 Audit logging table for all workflow runs and outcomes


👤 Author
Zehran Wasim
GitHub: @Zehran-Wasim
Built as part of a paid internship assignment at Trilles AI · May 2026
Zehran Wasim
GitHub: @Zehran-Wasim
Built as part of a paid internship assignment at Trilles AI · May 2026
