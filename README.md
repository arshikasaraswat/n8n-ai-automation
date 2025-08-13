# AI Automation Content Pipeline (n8n)

This repository contains the full automation to research trending topics in **AI Automation**, generate prompts, create **blog + video** content, and submit it to **Google Sheets** for human review — all orchestrated in **n8n**.

> **Artifacts included:**  
> - `n8n_ai_automation_workflow.json` – Import this into n8n  
> - `sample_content_submissions.csv` – Example Google Sheet rows  
> - `test_run_results.json` – Simulated test payloads  
> - `loom_script.md` – Script outline for your Loom recording

---

## Architecture Overview

**Agents & Flow**

1. **ContentResearch Agent**
   - **Google Trends** (HTTP Request) → parse AI-related topics
   - **YouTube Data API** (HTTP Request) → top videos for "AI Automation"
   - **Merge** → **Deduplicate**  
   Output: Canonical list of trending topics

2. **Prompt Agent**
   - **OpenAI (Chat)** → For each topic, generate:
     - Blog brief (title, outline, audience, SEO keywords)
     - 90-sec video outline (hook, beats, CTA)
     - 5 thumbnail/title ideas

3. **Content Creator Agent**
   - **OpenAI (Chat)** → 1200–1500 word blog
   - **Google VEO 3** (HTTP Request) → generate video; capture link

4. **Content Submission & Review**
   - **Google Sheets** (Append) → Topic, Prompt, Blog, Video Link, Timestamp, Status="Pending Review"
   - **Bonus:** Slack message to `#content-reviews`

---

## Prerequisites

- n8n 1.x+ (self-hosted or Cloud)
- Credentials:
  - **OpenAI API Key**
  - **YouTube Data API Key** (v3)
  - **Google Sheets** OAuth credential
  - **Google VEO 3** API credential (HTTP)
  - **Slack** App & Bot Token (optional)

### Environment Variables (recommended)
Configure these in n8n:
- `YOUTUBE_API_KEY`: YouTube Data API key
- `GSHEET_ID`: Target Google Sheet ID (create a Sheet with header row: `Topic,Prompt,Blog Content,Video Link,Timestamp,Status`)
- `SLACK_CHANNEL`: Channel (e.g., `content-reviews`)

---

## Importing the Workflow

1. In n8n, click **Workflows → Import from File**.
2. Upload `n8n_ai_automation_workflow.json`.
3. Open the workflow and:
   - Add credentials to:
     - **OpenAI** nodes (Prompt + Blog)
     - **Google Sheets** node
     - **HTTP Request** (Google VEO 3), set Authorization header
   - Set environment variables (Settings → Variables) or hardcode in the node params.

---

## Node-by-Node Configuration

### 1) Trigger
- **Schedule Trigger**: daily (edit to weekly if desired)

### 2) ContentResearch Agent
- **HTTP Request** – *Fetch Google Trends*  
  - GET `https://trends.google.com/trends/api/dailytrends?...`  
  - Response: string → parsed by next Function node
- **Function** – *Parse Trends*  
  - Strips `)]}'` prefix, extracts `trendingSearches` containing `"ai"` in query
- **HTTP Request** – *YouTube Search*  
  - GET `https://www.googleapis.com/youtube/v3/search`  
  - Query: `q=AI Automation`, `order=viewCount`, `maxResults=25`, `publishedAfter=last 7 days`
- **Function** – *Parse YouTube*
- **Merge** – *Concat*
- **Function** – *Deduplicate Topics*

### 3) Prompt Agent
- **Split In Batches** – iterate topics
- **OpenAI (Chat)** – *Generate Prompts (OpenAI)*  
  - System: prompt engineer persona  
  - User: includes topic and constraints
- **Function** – *Normalize Prompt*

### 4) Content Creator Agent
- **OpenAI (Chat)** – *Create Blog (OpenAI)*  
  - System: senior technical writer  
  - User: topic + prompt, 1200–1500 words
- **HTTP Request** – *Generate Video (Google VEO 3)*  
  - POST `https://veo.googleapis.com/v1/videos:generate`  
  - Body: `prompt`, `aspect_ratio`, `duration_seconds`  
  - Auth: Bearer token via custom credentials  
  - Response: capture `video_url`

- **Function** – *Collate Content*

### 5) Content Submission & Review
- **Google Sheets** – *Append to Google Sheet*  
  - Range: `Sheet1!A1:F1`  
  - Values: `[Topic, Prompt, Blog, VideoLink, Timestamp, Status]`
- **Slack** – *Slack Notify (Bonus)*

---

## Error Handling & Logging

- On each HTTP/OpenAI node, set **Retry On Fail** and **Max Attempts: 3**.
- Add **Error Trigger** workflow:
  - On error, push payload to a dedicated Google Sheet tab `Errors` with: `Node, Error, Input, Timestamp`.
- In **Split In Batches**, ensure **Continue On Fail** for non-blocking behavior across items.

---

## Testing

- Temporarily replace **Schedule Trigger** with **Manual Trigger** to test.
- Use 2–3 mock topics to run end-to-end.
- Verify Google Sheet row and Slack message.

---

## API Setup Notes

- **YouTube Data API**: Enable YouTube Data API v3 in Google Cloud Console; create API key (restrict by HTTP referrers/IP).
- **Google Sheets**: Share the target Sheet with your n8n service account email (if using service account).
- **OpenAI**: Set usage limits and model versions (`gpt-4o`, `gpt-4o-mini`).
- **Google VEO 3**: Use your account’s endpoint & fields; poll job status if the API is asynchronous, then store the final URL.

---

## Screenshots to Include (Guidance)

1. Schedule Trigger config
2. HTTP Request (Trends) config (URL, options)
3. HTTP Request (YouTube) config (query params)
4. OpenAI (Prompts) node config
5. OpenAI (Blog) node config
6. HTTP Request (VEO 3) node config
7. Google Sheets Append config
8. Slack node config (optional)
9. Example successful Google Sheet row

*(Take these inside n8n after importing and wiring credentials.)*

---

## Known Limits

- Google Trends endpoint may throttle; prefer official/community nodes if available.
- Video generation may be async; you may need a **Wait → Status Poll** loop before appending the final video URL.

---

## Attribution

Prepared on 2025-08-13T09:17:52.923763.

