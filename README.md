# Choice Gandalf — Slack AI Bot Setup Guide

A Slack bot that auto-replies to messages in a monitored channel, answers questions using internal Confluence documentation and historical Slack messages (RAG), and integrates with Jira. Runs 24/7 on Google Cloud Run — no laptop required.

Built by the Choice Analytics team at Delivery Hero as a template other teams can adapt.

---

## What it does

- **Auto-replies** to new top-level messages in a designated Slack channel
- **Answers @mentions and DMs** from any channel
- **Checks your Confluence docs first** before answering (configurable pages)
- **Searches past Slack history** using semantic search (RAG with ChromaDB + Vertex AI embeddings)
- **Handles Jira requests**: search issues, get details, create tickets

---

## Architecture

```
Slack (Socket Mode WebSocket)
        │
        ▼
  Cloud Run (slack-bot)
        │
        ├── Gemini 2.5 Flash (Vertex AI) ◄── System instruction: Confluence pages
        │                                      fetched at startup
        ├── ChromaDB (embedded in container)
        │   └── Slack channel history embeddings (pre-built, bundled at deploy time)
        │
        └── Jira Cloud API (atlassian-python-api)
```

**Why Cloud Run?**
Slack's Socket Mode uses a persistent WebSocket. Cloud Run with `--min-instances=1` and `--no-cpu-throttling` keeps the container alive indefinitely — no need to run the bot on a laptop or manage a VM.

---

## Prerequisites

### Accounts & access
- A **GCP project** with billing enabled
- A **Slack workspace** where you have admin access (to create a Slack app)
- A **Confluence/Jira** instance (Atlassian Cloud) with API access
- A **GCP service account** with the following roles:
  - `roles/aiplatform.user` (Vertex AI)
  - `roles/secretmanager.secretAccessor` (Secret Manager)

### Local tools
- [gcloud CLI](https://cloud.google.com/sdk/docs/install) — authenticated (`gcloud auth login`)
- [Colima](https://github.com/abiosoft/colima) — Docker runtime (recommended over Docker Desktop)
  ```bash
  brew install colima docker docker-buildx
  ln -sf /opt/homebrew/opt/docker-buildx/bin/docker-buildx \
      ~/.docker/cli-plugins/docker-buildx
  colima start
  ```
- Python 3.11+

---

## Step 1 — Create the Slack App

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → **From scratch**
2. Under **OAuth & Permissions → Bot Token Scopes**, add:
   - `app_mentions:read`
   - `channels:history`
   - `channels:read`
   - `chat:write`
   - `im:history`
   - `im:read`
   - `im:write`
   - `groups:history`
   - `users:read`
3. Under **Socket Mode**, enable it and generate an **App-Level Token** with scope `connections:write` → save as `SLACK_APP_TOKEN`
4. Install the app to your workspace → copy the **Bot User OAuth Token** → save as `SLACK_BOT_TOKEN`
5. Invite the bot to the channel you want it to monitor: `/invite @your-bot-name`

---

## Step 2 — Get your Jira API token

1. Go to [id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens)
2. Create a token → save as `JIRA_API_TOKEN`

---

## Step 3 — Set up the codebase

```bash
git clone <this-repo>
cd choice-gandalf
python3 -m venv .
source bin/activate
pip install -r requirements.txt
```

Create a `.env` file:
```env
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...

JIRA_API_TOKEN=...
JIRA_URL=https://yourcompany.atlassian.net
JIRA_EMAIL=you@yourcompany.com

GOOGLE_APPLICATION_CREDENTIALS=/path/to/your-service-account.json
GOOGLE_CLOUD_PROJECT=your-gcp-project-id
VERTEX_AI_LOCATION=us-central1
VERTEX_AI_MODEL=gemini-2.5-flash

LOG_CUSTOMER_DATA_CHANNEL_ID=C...   # Slack channel ID to auto-reply in
```

To find a Slack channel ID: open the channel in Slack → click the channel name at the top → scroll to the bottom of the popup.

---

## Step 4 — Configure your Confluence pages

Edit `slack_bot/confluence_context.py` and replace `_PAGES` with the Confluence pages relevant to your team:

```python
_PAGES = [
    ("Your Table Reference Page", "12345678"),   # page ID from the URL
    ("Your Dashboards Page",      "87654321"),
    ("Your FAQ Page",             "11223344"),
]
```

Page IDs are in the Confluence URL: `https://yourcompany.atlassian.net/wiki/spaces/XYZ/pages/12345678/Page+Title`

The bot fetches these once at startup and injects them as a system instruction to Gemini — so the model always checks them before answering.

---

## Step 5 — Build the RAG index (Slack history)

This is a one-time step (repeat periodically to refresh).

**Fetch channel history:**
```bash
python data/fetch_conversations.py
```
This saves JSON files to `data/`. The bot needs to be a member of the channel first (`/invite @bot` in Slack).

**Build the embeddings index:**
```bash
python data/create_embeddings.py
```
This embeds all messages using `text-embedding-005` via Vertex AI and stores them in `slack_bot/embeddings/chroma_db/`. The index is bundled into the Docker image at deploy time.

---

## Step 6 — Deploy to Cloud Run

Update the variables at the top of `deploy.sh`:
```bash
PROJECT_ID="your-gcp-project-id"
REGION="us-central1"
SERVICE_NAME="your-bot-name"
```

Then run:
```bash
bash deploy.sh
```

The script will:
1. Enable required GCP APIs
2. Store secrets in Secret Manager
3. Grant the Cloud Run service account access to secrets
4. Build and push a `linux/amd64` Docker image via Colima + docker buildx
5. Deploy to Cloud Run with the right flags for a persistent WebSocket bot

**Required Cloud Run flags** (already in `deploy.sh`):
- `--min-instances=1` — keeps the container alive (no cold starts dropping WebSocket)
- `--no-cpu-throttling` — prevents CPU from being throttled between requests
- `--memory=1Gi` — ChromaDB needs enough memory

---

## Customisation

### Change which channel the bot auto-replies in
Update `LOG_CUSTOMER_DATA_CHANNEL_ID` in `.env` and in `deploy.sh` (`--set-env-vars`).

The channel ID check in `bot.py`:
```python
if channel_id != os.environ.get("LOG_CUSTOMER_DATA_CHANNEL_ID"):
    return
```

### Change the Gemini model
Update `VERTEX_AI_MODEL` in `.env` and `deploy.sh`. Current options: `gemini-2.5-flash`, `gemini-2.5-pro`.

### Change what counts as a Jira request
Edit `detect_jira_intent()` in `bot.py`. Only include keywords that are unambiguously Jira-specific — generic words like "get", "find", "create" cause false positives.

---

## Maintenance

### Refresh the Slack history index
Run periodically (e.g. monthly) as the channel accumulates new messages:
```bash
python data/fetch_conversations.py
python data/create_embeddings.py
bash deploy.sh
```

### Update Confluence content
The pages are fetched live at each container startup — no action needed when Confluence content changes. The next deploy (or container restart) will pick up the latest content.

### Monitor logs
```bash
gcloud run services logs read YOUR-SERVICE-NAME --region us-central1 --project YOUR-PROJECT-ID
```

Healthy startup looks like:
```
✓ Gemini initialized
✓ Jira client initialized successfully
✓ Confluence page loaded: Your Table Reference Page (12,345 chars)
✓ Confluence page loaded: Your Dashboards Page (5,678 chars)
✓ Confluence page loaded: Your FAQ Page (9,012 chars)
🚀 Starting Slack Bot...
✅ Bot is running!
⚡️ Bolt app is running!
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `not_in_channel` error when fetching history | Bot not a member of the channel | `/invite @bot-name` in Slack |
| `exec format error` on Cloud Run | Wrong CPU architecture (ARM64 vs AMD64) | Use `docker buildx build --platform linux/amd64` |
| Bot connects then drops after a few minutes | Cloud Run throttling CPU or scaling to 0 | Ensure `--min-instances=1` and `--no-cpu-throttling` |
| Bot routes data questions to Jira | Jira keyword list too broad | Remove generic words from `detect_jira_intent()` |
| Confluence code blocks not being read | CDATA stripping missing | The `_strip_html()` function in `confluence_context.py` handles this — ensure CDATA regex is present |

---

## Tech stack

| Component | Technology |
|---|---|
| Slack integration | [slack-bolt](https://github.com/slackapi/bolt-python) (Socket Mode) |
| AI model | Gemini 2.5 Flash via Vertex AI (`google-genai`) |
| Embeddings | `text-embedding-005` via Vertex AI |
| Vector store | [ChromaDB](https://www.trychroma.com/) (persistent, embedded in container) |
| Confluence/Jira | [atlassian-python-api](https://github.com/atlassian-api/atlassian-python-api) |
| Runtime | Google Cloud Run |
| Secrets | Google Secret Manager |
| Container registry | Google Container Registry (GCR) |
