# Job Market Intelligence — Telegram Bot (n8n + Bright Data)

A Telegram bot that searches job boards via Bright Data, scores results with AI (OpenAI), saves data to Google Sheets, and sends interactive report buttons. Built with [n8n](https://n8n.io) workflows.

---

## Description

Two n8n workflows work together:

1. **Main workflow** — User sends a search query in Telegram → bot searches job boards (Bright Data) → parses listings → loads job cards → scores with AI → deduplicates and appends to Google Sheets → sends “AI top 3” / “Code top 3” buttons. Optional: daily email of today’s jobs (schedule trigger).
2. **Callback workflow** — Handles inline button clicks (“AI top 3” / “Code top 3”) → reads cached report from Google Sheets → sends the chosen report back to the user in Telegram.

All credentials and the spreadsheet ID are placeholders in the JSON; you add your own in n8n after import.

---

## How It Works (Step-by-Step)

### Main workflow (Telegram → search → AI → Sheets → buttons)

1. **Trigger** — User sends a text message to the Telegram bot.
2. **Parse & notify** — Query is parsed; bot replies “⏳ Searching…”.
3. **Search** — Bright Data is called to fetch search results (HTML).
4. **Error handling** — If search fails (4xx/error) → “BrightData scraping failed.” and stop. If no jobs parsed → “No jobs found.” and stop.
5. **Parse results** — HTML is parsed into up to 10 jobs (title, body, tags, budget, hourly, posted_at, job_url).
6. **Normalize dates** — Relative/localized dates are converted to a standard format.
7. **Load job cards** — For each job, Bright Data fetches the full job page HTML.
8. **Parse activity** — “Activity on this job” and related fields are extracted.
9. **Scoring** — Reply attractiveness (competition, etc.) is computed in code; optional “Code top 3” branch picks top 3 by that score.
10. **AI analysis** — All 10 jobs are sent to OpenAI in one batch; model returns summary, hot_score, and “reason_why_top3” for the top 3.
11. **Merge & dedupe** — AI results are merged with job data; before appending to the sheet, existing Job URLs are read and only new jobs are appended.
12. **Sheets** — New jobs are appended to “Job Search Results”; AI top 3 and Code top 3 are written to other sheets; a cache row (ChatId, Timestamp, AiTop3Message, CodeTop3Message) is written for the callback workflow.
13. **Telegram** — User gets a message with two inline buttons: “AI top 3” and “Code top 3”.
14. **Schedule (optional)** — A daily trigger reads “Job Search Results,” filters by today’s date, and sends an email with up to 10 jobs.

### Callback workflow (button click → cached report)

1. **Trigger** — Telegram callback_query (user clicked “AI top 3” or “Code top 3”).
2. **Get Chat & Choice** — Extracts chat_id, callback_data (ai_top3 / code_top3), query_id.
3. **Google Sheets Read** — Reads the TelegramReportCache sheet and finds the latest row for this chat_id.
4. **Pick Message** — Selects AiTop3Message or CodeTop3Message based on callback_data.
5. **Split (optional)** — Code top 3 message can be split into multiple Telegram messages.
6. **Send** — Sends the report text back to the user in Telegram.

---

## Workflow Architecture (Nodes)

### Main workflow (`workflow-main.json`)

| Node | Purpose |
|------|--------|
| **Sticky Note** | Setup instructions (credentials, SHEETS_DOC_ID, activation). |
| **Incoming Request** | Telegram trigger (message). |
| **Parse User Query** | Extracts search query (main_query, filters). |
| **Notify Searching** | Sends “⏳ Searching…” to the user. |
| **Search Jobs** | HTTP request to Bright Data (job board search URL). |
| **Check Search Jobs Error** | IF: statusCode ≥ 400 or error → error branch. |
| **Telegram BrightData Failed** | Sends “BrightData scraping failed.” |
| **Parse Search Results** | Parses HTML into 10 job items. |
| **Check No Jobs** | IF: $json.error → no-jobs branch. |
| **Telegram No Jobs** | Sends “No jobs found.” |
| **Normalize Posted Date** | Converts dates to en-US format. |
| **Load Job Card** | Bright Data request per job_url. |
| **Parse Client Activity** | Extracts activity block from job HTML. |
| **Compute Reply Attractiveness** | Computes reply_attractiveness from activity. |
| **Mark High Paying** | Flags high budget/hourly jobs. |
| **Prepare Batch for AI** | Builds single text block for OpenAI. |
| **OpenAI Chat Model** | LLM (e.g. gpt-4o-mini). |
| **AI Research Agent** | Calls model for summary, hot_score, reason_why_top3. |
| **Parse AI Batch & Merge** | Merges AI output with job data. |
| **Get existing Job URLs** | Reads Job Search Results sheet. |
| **Dedup by Job URL** | Filters to jobs not already in sheet. |
| **Sheet Append 10 Jobs** | Appends new jobs to Job Search Results. |
| **Top 3 Hot** | Picks top 3 by hot_score. |
| **Summary for Hot** | Builds summary_for_hot text. |
| **Sheet Append AI Top 3** | Writes AI top 3 to sheet. |
| **Pick Code Hot 3** | Picks top 3 by reply_attractiveness (and high-paying). |
| **Code Top 3** | Writes Code top 3 to sheet. |
| **Code Top 3 Update Summary** | Updates Summary for Code top 3 rows. |
| **Filter Code Hot 3 for Summary** | Filters items for Code top 3 summary. |
| **Store Code Hot 3 URLs** | Stores URLs for merge. |
| **Merge AI and Code Top 3** | Merges both top-3 branches. |
| **Prepare Telegram Message** | Builds AI top 3 message for Telegram. |
| **Prepare Code Top 3 Message** | Builds Code top 3 message. |
| **Prepare Report Cache** | Builds cache row (chat_id, messages, timestamp). |
| **Merge for Cache** | Merges AI and Code messages. |
| **Cache Reports** | Appends cache row to TelegramReportCache sheet. |
| **Send Buttons** | Sends Telegram message with “AI top 3” / “Code top 3” buttons. |
| **Schedule Trigger** | Daily (e.g. 13:00). |
| **Get row(s) in sheet** | Reads Job Search Results. |
| **Filter Today 10 Jobs** | Keeps rows with today’s Posted At. |
| **Prepare Daily Email Body** | Builds HTML email body. |
| **Send an Email** | Sends daily digest via SMTP (e.g. Gmail). |

### Callback workflow (`workflow-callback.json`)

| Node | Purpose |
|------|--------|
| **Telegram Trigger** | Listens for callback_query. |
| **Get Chat & Choice** | Extracts chat_id, callback_data, query_id. |
| **Google Sheets Read** | Reads TelegramReportCache sheet. |
| **Pick Message** | Selects AiTop3Message or CodeTop3Message by callback_data. |
| **Split Code Top 3 for Send** | Optionally splits long Code top 3 into multiple messages. |
| **Send a text message** | Sends the report to the user in Telegram. |

---

## Prerequisites

- **n8n** (self-hosted or n8n Cloud).
- **Telegram Bot** — Create via [@BotFather](https://t.me/BotFather); get bot token.
- **Bright Data** — Account and Bearer token for job board scraping (e.g. Web Scraper API).
- **OpenAI** — API key for the AI Research Agent.
- **Google Sheets** — A spreadsheet with sheets: Job Search Results, AI top 3 sheet, Code hot 3 sheet, TelegramReportCache. Use a **Service Account** (JSON key) with Sheets (and Drive if needed) access; share the spreadsheet with the service account email.
- **SMTP (optional)** — For daily email (e.g. Gmail with App Password).

---

## Setup Instructions

1. **Import workflows**  
   In n8n: **Workflows → Import from File** and import `workflow-main.json` and `workflow-callback.json`.

2. **Create credentials in n8n**  
   For each placeholder below, create the matching credential in n8n and assign it to the nodes that reference it (n8n will show “missing credential” until you do).

3. **Spreadsheet ID**  
   - In the main workflow, nodes use `$vars.SHEETS_DOC_ID` with fallback `YOUR_SPREADSHEET_ID`.  
   - Set workflow or instance **Variables**: add `SHEETS_DOC_ID` = your Google Spreadsheet ID (the long string from the sheet URL: `https://docs.google.com/spreadsheets/d/<THIS_ID>/edit`).  
   - In the callback workflow, set the Google Sheets node’s Document ID to the same spreadsheet (or keep using the variable if you add it).

4. **Webhooks**  
   After activation, n8n will register webhooks. For Telegram you must use HTTPS and a public URL. If self-hosting, set `WEBHOOK_URL` (and optionally `N8N_PROXY_HOPS`) so the webhook URL is your domain (e.g. `https://n8n.yourdomain.com/`).

5. **Activate**  
   Turn both workflows **Active**. The main workflow listens for messages; the callback workflow listens for callback_query.

6. **Sheets**  
   Ensure the spreadsheet has sheets (and columns) expected by the workflows: Job Search Results (Title, Job Description, Tags, Budget, Hourly, Posted At, Summary, Job URL, Activity on job), TelegramReportCache (ChatId, Timestamp, AiTop3Message, CodeTop3Message), plus sheets for AI top 3 and Code hot 3 as used by the append/update nodes.

---

## Credentials Reference

| Placeholder in JSON | Where to use it in n8n | Where to get the value |
|--------------------|------------------------|-------------------------|
| **YOUR_TELEGRAM_CREDENTIAL_ID** | Telegram Trigger, all Telegram nodes (Notify, error messages, Send Buttons, callback “Send a text message”) | Create “Telegram API” credential in n8n; paste the **Bot Token** from [@BotFather](https://t.me/BotFather). |
| **YOUR_OPENAI_CREDENTIAL_ID** | OpenAI Chat Model (AI Research Agent) | Create “OpenAI API” credential; paste your **OpenAI API key** from [OpenAI API keys](https://platform.openai.com/api-keys). |
| **YOUR_GOOGLE_SHEETS_CREDENTIAL_ID** | All Google Sheets nodes (append, read, update) in both workflows | Create “Google Sheets Service Account” (or OAuth2); for Service Account use the **JSON key file** from [Google Cloud Console](https://console.cloud.google.com/) (APIs: Google Sheets API, optionally Drive). Share the spreadsheet with the service account email. |
| **YOUR_BRIGHTDATA_BEARER_CREDENTIAL_ID** | Search Jobs, Load Job Card | Create “HTTP Bearer Auth” (or similar) in n8n; set the token to your **Bright Data Bearer token** from the Bright Data dashboard. |
| **YOUR_SMTP_CREDENTIAL_ID** | Send an Email | Create “SMTP” credential (e.g. Gmail); use **App Password** if using Gmail 2FA. |
| **YOUR_SPREADSHEET_ID** | Replaced by variable or direct ID in nodes | From the Google Sheets URL: `https://docs.google.com/spreadsheets/d/<SPREADSHEET_ID>/edit`. Set as workflow/instance variable **SHEETS_DOC_ID** or in each Google Sheets node. |
| **YOUR_WEBHOOK_ID** | Automatically replaced by n8n on save/activate | No action; n8n generates new webhook IDs when you activate the workflow. |

---

## Workflow Files

| File | Role |
|------|------|
| **workflow-main.json** | Main workflow: Telegram message trigger → Bright Data search → parse → load job cards → AI scoring → dedup → Google Sheets (Job Search Results, AI/Code top 3, cache) → Telegram message with “AI top 3” / “Code top 3” buttons. Also includes Schedule Trigger → read sheet → filter today’s jobs → daily email. |
| **workflow-callback.json** | Callback workflow: Telegram callback_query trigger → read TelegramReportCache from Google Sheets → pick AI or Code report by button → send report back to the user. |

Both workflows must be active and use the same Telegram bot and the same spreadsheet (and cache sheet) for the buttons to work.

---

## License

Use and modify as you like. No warranty.
