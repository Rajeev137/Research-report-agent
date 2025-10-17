# Research Report Agent (Phase 1)

A mobile-ready backend that fetches company news, summarizes with a hybrid LLM pipeline, and produces a 3–5 slide sales brief — all cached in Firestore. Now includes Google Slides generation, Gmail send, and Calendar scheduling.

## ✅ Current Status (Updated: 18 Oct 2025)

- **Hybrid LLM pipeline** (rank → per-article summary → final merge) with **clean JSON** and **AJV validation**
- **Firestore**: report caching, **token usage logs** (linked to report), history-ready
- **Azure OpenAI** chat stable (`gpt-5-mini`, `max_completion_tokens` only)
- **Google Slides generation**: Cover + Rep Intro + Company Intro + 3 slides (Key Facts, Opportunities/Risks, Questions/Next Steps)
- **Gmail send**: builds a sensible default message with slide link and key points
- **Calendar scheduling**: schedules an event (default **+7 days at 12:00/noon**), invites attendees
- **Unified endpoint**: `/api/research-and-slides` runs research → slides in one call

## 🚀 What’s Next

- **Frontend (React)**:
  - Google sign-in
  - Form: Company, Domain (optional), Force (toggle)
  - Run unified flow → preview report + slide links
  - Buttons: **Send Email**, **Schedule Meeting**
  - **History view**: show last 5 reports (from Firestore)

- **(Optional) Lightweight RAG** _(paused until embeddings access)_:
  - Chunk `website_text` + top-N article bodies → embed → store vectors under the report → retrieve top-8 chunks for final merge context
  - Provider strategy: Azure embeddings if available; fallback to OpenAI `text-embedding-3-small` (requires `OPENAI_API_KEY`)

> RAG remains optional and will be added if Azure embeddings (or an OpenAI key) is available.

---

## Tech Stack

- **Node.js**, **Express**
- **Azure OpenAI** (`gpt-5-mini`) — chat completions (`max_completion_tokens`, `temperature`)
- **Firestore** via `firebase-admin` (reports, token_logs, meetings, emails)
- **NewsAPI** + **Google News** fallback (scrape)
- **AJV** for strict JSON schema validation
- **Google APIs**: Slides, Drive, Gmail, Calendar (OAuth)

---

## Folder Structure
src/
app.js                 # express app wiring (routes mounted here)
server.js              # server start + health
config/
env.js               # env + pipeline knobs
firestore/
init.js              # firebase-admin init
cache.js             # cache lookup by company/domain
tokenLog.js          # token usage logger (linked to report)
llm/
azureClient.js       # Azure chat call + JSON parse/repair
summarizer.js        # per-article summarizer (map, retries)
merger.js            # final merge (reduce) + AJV + fallback bullets
schema.js            # AJV final schema
prompts.js           # unified prompt
services/
newsService.js       # NewsAPI + Google News, ranking/filtering
webService.js        # website fetch + text extraction
slidesService.js     # Slides deck creation + Drive files
routes/
research.js          # POST /api/research
slides.js            # POST /api/slides/from-report
researchAndSlides.js # POST /api/research-and-slides (one-shot flow)
email.js             # POST /api/email/send
calendar.js          # POST /api/calendar/schedule
google/
auth.js              # OAuth client + token store (JSON on disk)

---

## Setup

### 1) Install
```bash
npm install
# if not done yet:
npm i express axios cheerio cors firebase-admin ajv dotenv googleapis

2) Environment

Create .env:
PORT=4000
FIREBASE_SERVICE_ACCOUNT_PATH=./serviceAccountKey.json

# Azure OpenAI (chat)
AZURE_ENDPOINT=https://<your-resource>.openai.azure.com
AZURE_KEY=<your-azure-key>
AZURE_DEPLOYMENT_NAME=gpt-5-mini
AZURE_API_VERSION=2025-04-01-preview

# News
NEWSAPI_KEY=<optional-but-recommended>

# Google (OAuth)
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI=http://localhost:4000/google/oauth2callback
GOOGLE_DRIVE_PARENT_FOLDER_ID=<drive-folder-id>   # folder ID, not URL

# Personalisation (fallbacks for slides cover)
FALLBACK_USER_NAME=RAJEEV SHARMA
FALLBACK_USER_TITLE=Sales Executive
FALLBACK_USER_ORG=Otsuka
FALLBACK_USER_EMAIL=<your gmail>  # used by Gmail send if "from" is "me"

# Pipeline knobs (keep names as-is)
SALES_TOP_K=3
PER_ARTICLE_MAX_TOKENS=900
PER_ARTICLE_RETRY_TOKENS=200
FINAL_MAX_COMPLETION_TOKENS=3500
FINAL_RETRY_TOKENS=600

Put your Firebase service account at ./serviceAccountKey.json (or update the path).
Important: re-run /google/auth after adding scopes (Slides/Drive/Gmail/Calendar).

3) Run
node src/server.js

API

Health
GET /health
→ { ok: true, ts: "..." }

Unified: research → slides (one call)
POST /api/research-and-slides
Body:
{
  "companyName": "Siemens AG",
  "domain": "www.siemens.com",   // optional, improves website fetch & cache key
  "force": true                  // optional; skip cache if true
}

Response (abridged):
{
  "cached": false,
  "elapsedMs": 12345,
  "report": {
    "companyName": "Siemens AG",
    "domainUsed": "www.siemens.com",
    "website": "https://www.siemens.com",
    "website_text": "....",
    "news": [{ "title":"...", "url":"...", "relevanceScore": 7.1 }],
    "perArticleSummaries": [
      { "id":"...", "title":"...", "short_summary":"...", "sales_bullet":"...", "url":"..." }
    ],
    "summary": { "company":"...", "highlights":[...], "slides":[... ] },
	"presentationId": "...",
  	"webViewLink": "...",
  	"webExportPdf": "...",
  	"fileId": "..."
  }
}

Send Email (Gmail)

POST /api/email/send
Body:
{
  "reportId": "<ID>",
  "to": "ravtiraman041@gmail.com",
  "subject": "Intro meeting re: <Company>",
  "bodyHtml": "<p>...</p>"
}

Schedule Meeting (Calendar)

POST /api/calendar/schedule
Body:
{
  "reportId": "<ID>",
  "date": "2025-10-28",          // optional; defaults to +7 days
  "durationMins": 30,
  "attendees": ["ravtiraman041@gmail.com"]
}
returns { eventId, htmlLink, start, end }

Firestore Collections
	•	reports — one doc per generated report (inputs, news, summaries, final JSON, slide links)
	•	token_logs — one doc per LLM call (reportId, stage, usage, model)
	•	emails — email sends (reportId, to, gmailMessageId, status)
	•	meetings — scheduled events (reportId, eventId, htmlLink, start/end)

Optional: RAG (Paused)
	•	Why: Reduces hallucinations by grounding final merge on retrieved chunks from the client site/news.
	•	Plan: Add rag.js (chunk → embed → store vectors → retrieve top-8).
	•	Provider: Azure embeddings if available; fallback to OpenAI if OPENAI_API_KEY present.
	•	Status: Waiting for Azure embedding deployment access (or OpenAI key).
	•	Impact: Minimal code change—just prepends a CONTEXT block to the merge prompt.

Troubleshooting
	•	/health shows HTML → You’re missing the JSON route. Fix with res.json({ok:true,...}).
	•	research_and_slides_failed → Try the two steps separately (/api/research then /api/slides/from-report) and/or use the improved error surfacing in researchAndSlides.js.
	•	Slides error does not allow text editing → You’re writing to the wrong objectId or not creating the text shape before insert. Use the slidesService helpers that create title/body shapes first.
	•	Gmail 403 → Re-run /google/auth with the gmail.send scope enabled.
	•	Calendar 403 → Re-run /google/auth with the calendar.events scope enabled.
	•	Cache misses → Pass a consistent domain; cache prefers exact domain match, then company-only.
	•	LLM fallback → Check mergeMeta.fallback and mergeMeta.mergeError. Increase FINAL_MAX_COMPLETION_TOKENS (e.g., 1600–2200) or reduce per-article payload.

License

MIT
