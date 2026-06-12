# 2. Installation Guide

[← Overview](01-overview.md) · [Next: Configuration →](03-configuration.md)

---

This guide takes you from a fresh computer to a running Maestro AI system. It's written so that **no prior coding experience is required** — every command is copy-paste, and we explain what each one does.

> ⏱️ **Time:** 45–90 minutes. Most of the time is Google account setup, which is one-time.
>
> 💡 **Tip:** Do the steps in order. Don't skip ahead — later steps assume earlier ones are done.

## What you'll set up, in order

1. [Install the prerequisite software](#step-1--install-prerequisite-software) (Docker, Node.js, Git)
2. [Get the Maestro code](#step-2--get-the-maestro-code) (clone both repositories)
3. [Set up your Google account](#step-3--set-up-your-google-cloud-project) (project, APIs, service account)
4. [Create your database](#step-4--create-the-database-spreadsheet) (copy the Google Sheet)
5. [Get your AI provider key](#step-5--get-an-ai-provider-api-key) (Anthropic recommended)
6. [Configure environment files](#step-6--configure-environment-variables) (passwords and keys)
7. [Start the backend](#step-7--start-the-backend-n8n) (n8n in Docker)
8. [Import the workflows](#step-8--import-the-n8n-workflows)
9. [Connect n8n to Google and your AI provider](#step-9--connect-n8n-credentials)
10. [Start the dashboard](#step-10--start-the-maestro-dashboard)
11. [Start the scheduler](#step-11--start-the-scheduler-optional)
12. [Verify everything works](#step-12--verify-the-installation)

---

## Step 1 — Install prerequisite software

You need three free programs. Install whichever you don't already have.

### Docker Desktop

Docker runs the backend in a self-contained "container" so you don't have to install n8n manually.

- Download from **https://www.docker.com/products/docker-desktop/**
- Install it, then **launch it** and wait until the whale icon shows "Docker Desktop is running."
- **Windows users:** if prompted, accept the WSL 2 installation — Docker needs it.

Verify it's working. Open a terminal (**Windows:** PowerShell; **macOS/Linux:** Terminal) and run:

```bash
docker --version
```

You should see something like `Docker version 27.x.x`.

### Node.js (version 20)

Node runs the dashboard and the scheduler. **Node 20** is the intended version — it's what the scheduler's Docker image is built on (`node:20-alpine`). There's no hard version check in the code, so a newer LTS will usually work, but 20 is the tested target.

- Download the **Node 20 LTS** version from **https://nodejs.org**
- Install with all defaults.

Verify:

```bash
node --version
```

You should see `v20.x.x`.

### Git

Git downloads ("clones") the code.

- **Windows/macOS:** download from **https://git-scm.com/downloads**
- **Linux:** `sudo apt install git` (Debian/Ubuntu) or your distro's equivalent.

Verify:

```bash
git --version
```

---

## Step 2 — Get the Maestro code

Pick a folder to keep everything in. We'll use `maestro` in your home directory as an example.

```bash
# Create and enter a working folder
mkdir maestro
cd maestro

# Clone the backend (n8n workflows + scheduler)
git clone https://github.com/parseus-ai-inc/maestro-ai-backend.git

# Clone the dashboard
git clone https://github.com/parseus-ai-inc/maestro-ai-frontend.git
```

You now have two folders: `maestro-ai-backend` and `maestro-ai-frontend`.

---

## Step 3 — Set up your Google Cloud project

Maestro stores its database in Google Sheets and its files in Google Drive. To let the software read and write them automatically, you create a **service account** — a kind of robot Google user with its own key.

This is the fiddliest part. Take it slowly; you only do it once. Full screenshots and detail are in the [Configuration guide](03-configuration.md#google-cloud-setup) — the short version:

> 💡 **Consider a dedicated (burner) Google account.** The service account you create here gets **Editor** access to one Sheet and one Drive folder. That access is scoped to only the items you explicitly share with it — not your whole Drive — so Maestro can't see your personal files. Still, if you'd rather keep job-hunting fully walled off from your personal account, create a free dedicated Google account and do all of the steps below signed into it.

1. Go to **https://console.cloud.google.com** and sign in with the Google account you want to use for Maestro (a dedicated account is recommended — see the note above).
2. Create a new project — name it `maestro` (top bar → project dropdown → **New Project**).
3. Enable three APIs (search each in the top bar and click **Enable**):
   - **Google Sheets API**
   - **Google Drive API**
   - **Google Docs API**
4. Create a **service account**: **APIs & Services → Credentials → Create Credentials → Service account**. Name it `maestro-pipeline`. Skip the optional permission steps.
5. Create a **key** for it: click the new service account → **Keys → Add Key → Create new key → JSON**. A `.json` file downloads. **Keep this file safe — it's a password.**

   ```
   📷 Creating the JSON key:
   Service account: maestro-pipeline
   ┌────────────────────────────────────────────────────┐
   │  DETAILS   PERMISSIONS   [ KEYS ]   METRICS   LOGS  │
   ├────────────────────────────────────────────────────┤
   │  Keys                                                │
   │  ┌──────────────────────────────────────────────┐  │
   │  │  [ ADD KEY ▼ ]                                 │  │
   │  │     ├─ Create new key   ←── click this         │  │
   │  │     └─ Upload existing key                     │  │
   │  └──────────────────────────────────────────────┘  │
   │     Key type:   (•) JSON     ( ) P12                 │ ← choose JSON
   │                              [ Cancel ]  [ Create ]  │
   └────────────────────────────────────────────────────┘
            ↓ a .json file downloads to your computer
   ```

6. Open that JSON file in a text editor. You'll need two values from it later:
   - `client_email` (looks like `maestro-pipeline@yourproject.iam.gserviceaccount.com`)
   - `private_key` (a long block starting with `-----BEGIN PRIVATE KEY-----`)

   ```
   📷 What the JSON file looks like inside:
   {
     "type": "service_account",
     "project_id": "maestro-xxxxxx",
     "private_key": "-----BEGIN PRIVATE KEY-----\nMIIEvg...\n-----END PRIVATE KEY-----\n",  ← GOOGLE_SHEETS_PRIVATE_KEY
     "client_email": "maestro-pipeline@maestro-xxxxxx.iam.gserviceaccount.com",            ← GOOGLE_SHEETS_CLIENT_EMAIL
     "client_id": "1234567890",
     ...
   }
   ```

> ⚠️ **One service account for both halves.** The dashboard and the pipeline **must use the same service account.** If they differ, saving résumés to Drive will fail with a "403" error.

---

## Step 4 — Create the database spreadsheet

Maestro ships with a ready-made database template (`Database.xlsx` in the backend repo) containing all the required tabs and columns.

1. Go to **https://sheets.google.com** and create a blank spreadsheet (or import `Database.xlsx`: **File → Import → Upload**, choose `Database.xlsx`, **Replace spreadsheet**).
2. Name it `Maestro Database`.
3. Look at the URL. It contains the spreadsheet's ID:
   ```
   https://docs.google.com/spreadsheets/d/THIS_LONG_PART_IS_THE_ID/edit
   ```
   **Copy that ID** — you'll need it as `GOOGLE_SHEETS_DATABASE_ID`.

   ```
   📷 Where to find the ID in the URL:

   https://docs.google.com/spreadsheets/d/ 1AbC2dEfGhIjK3LmNoP4qRsTuVwXyZ /edit#gid=0
   └──────────────── ignore ─────────────┘ └───── COPY THIS PART ──────┘ └─ ignore ─┘
   ```

4. **Share the sheet with your service account.** Click **Share** (top-right), paste the service account's `client_email` from Step 3, give it **Editor** access, and send. (No email actually goes anywhere — it just grants the robot access.)

   ```
   📷 The Share dialog:
   ┌───────────────────────────────────────────────────┐
   │  Share "Maestro Database"                      [✕] │
   ├───────────────────────────────────────────────────┤
   │  ┌─────────────────────────────────────────────┐  │
   │  │ maestro-pipeline@yourproject.iam.gservice…  │  │ ← paste client_email
   │  └─────────────────────────────────────────────┘  │
   │                                    [ Editor  ▼ ]   │ ← must be Editor
   │                                                     │
   │                              [ Cancel ]  [ Send ]   │
   └───────────────────────────────────────────────────┘
   ```

Then fill in the `config` tab with your details — your name, target roles, and which AI models to use. The [Configuration guide](03-configuration.md#the-config-tab) explains every setting.

---

## Step 5 — Get an AI provider API key

Maestro needs at least one AI provider. **Anthropic (Claude) is recommended** and is the default in the shipped config.

### Anthropic (recommended)

1. Go to **https://console.anthropic.com** and sign up.
2. Add a small amount of credit (**Billing**). \$5–\$10 is plenty to start.
3. Go to **API Keys → Create Key**, name it `maestro`, and **copy the key** (starts with `sk-ant-`). You can't see it again, so save it now.

### OpenAI and Google Gemini (optional)

Maestro also supports these. Get keys from **https://platform.openai.com/api-keys** and **https://aistudio.google.com/apikey** respectively if you want to use them. You can mix providers per agent in the `config` tab.

---

## Step 6 — Configure environment variables

"Environment variables" are settings — including secrets — that live in a file the software reads at startup. Each repo has an `.env.example` template; you copy it to `.env.local` (dashboard) / `.env` (backend) and fill in your values.

> 🔒 **Never commit these files to GitHub.** They contain secrets and are already excluded by `.gitignore`.

### Backend `.env`

In the `maestro-ai-backend` folder, copy the template:

```bash
# Windows (PowerShell)
cd maestro-ai-backend
Copy-Item .env.example .env

# macOS / Linux
cd maestro-ai-backend
cp .env.example .env
```

Open `maestro-ai-backend/.env` in a text editor and fill in:

```ini
# The ID of your Google Sheet from Step 4
GOOGLE_SHEETS_DATABASE_ID=your-sheet-id-here

# The Google Drive folder ID where résumé folders are created.
# Create a folder in Drive, share it with the service account (Editor),
# and copy the ID from its URL.
ROOT_FOLDER_ID=your-drive-folder-id-here

# Lets n8n read these env vars inside its nodes — leave as false
N8N_BLOCK_ENV_ACCESS_IN_NODE=false
```

### Dashboard `.env.local`

In the `maestro-ai-frontend` folder:

```bash
# Windows (PowerShell)
cd ../maestro-ai-frontend
Copy-Item .env.example .env.local

# macOS / Linux
cd ../maestro-ai-frontend
cp .env.example .env.local
```

Open `maestro-ai-frontend/.env.local` and fill in the service-account values from Step 3:

```ini
# From your service-account JSON file (Step 3)
GOOGLE_SHEETS_CLIENT_EMAIL=maestro-pipeline@yourproject.iam.gserviceaccount.com

# The whole private key. Keep it on ONE line, with \n where the line breaks are.
# (Copy it from the JSON file — it already has the \n in it.)
GOOGLE_SHEETS_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nMIIE...\n-----END PRIVATE KEY-----\n"

# Same sheet ID as the backend
GOOGLE_SHEETS_DATABASE_ID=your-sheet-id-here

# Where the dashboard/scheduler send "run this" requests to the backend.
# Must end in /webhook (NOT /webhook-test). Default below works for local Docker.
N8N_WEBHOOK_BASE_URL=http://localhost:5678/webhook

# A shared secret you invent. Sent as the X-Maestro-Secret header on every
# webhook call. Use the SAME value here and in n8n's webhook Header Auth (Step 9).
# Without it, triggering and the scheduler will not fire.
MAESTRO_WEBHOOK_SECRET=choose-a-long-random-string

# Optional Drive folders. Leave blank to use defaults.
GOOGLE_PROMPTS_FOLDER_ID=
GOOGLE_RESUMES_PARENT_ID=
```

> 💡 **About the private key's `\n`:** the key in your JSON file contains literal `\n` characters marking line breaks. Paste it exactly as it appears in the JSON, wrapped in double quotes. Don't reformat it. (The software unescapes the `\n` at runtime.)

> 🔑 **Pick your `MAESTRO_WEBHOOK_SECRET` now and remember it.** You'll enter the exact same value into n8n in [Step 9](#step-9--connect-n8n-credentials). Any long random string works — it's a shared password between the dashboard/scheduler and the pipeline.

> ℹ️ **Don't put a space after the `=`.** Write `KEY=value`, not `KEY= value` — a leading space becomes part of the value.

---

## Step 7 — Start the backend (n8n)

The backend runs in Docker using the `docker-compose.yml` file in the `maestro-ai-backend` folder.

> ⚠️ **Compose project name matters.** Maestro pins its Docker project name to `maestro-ai` so your data survives across restarts even if you rename folders. **Always include `-p maestro-ai`** in every `docker compose` command, or add `COMPOSE_PROJECT_NAME=maestro-ai` to a `.env` file beside the compose file.

From the `maestro-ai-backend` folder:

```bash
docker compose -p maestro-ai up -d
```

This downloads and starts n8n. The first run takes a few minutes. When it's done, open **http://localhost:5678** in your browser. Create your n8n owner account (local only — these credentials stay on your machine).

To stop it later:

```bash
docker compose -p maestro-ai down
```

(Your data is kept in a named Docker volume, so stopping and starting is safe.)

---

## Step 8 — Import the n8n workflows

The backend repo contains all the workflows as `.json` files. Import them into n8n.

For **each** `.json` file in the backend's workflow folder:

1. In n8n (http://localhost:5678), click **Workflows** in the left menu.
2. Click the **⋯** (or **Add workflow** dropdown) → **Import from File**.
3. Select the `.json` file.
4. Click **Save**, then **toggle the workflow to Active / Published** (top-right switch).

```
📷 The Import menu and the Active toggle:
┌──────────────────────────────────────────────────────────────┐
│  n8n   Workflows                          [ Add workflow  ▼ ] │
│                                              ├─ Import from File │ ← (3a) import each .json
│                                              └─ Import from URL   │
├──────────────────────────────────────────────────────────────┤
│  Agent 1 - Resume Builder        [ Inactive ⚪──── ] Save        │
│                                            ▲                     │
│                            toggle to  [ Active ────⚫ ]          │ ← (3b) MUST be Active/Published
└──────────────────────────────────────────────────────────────┘
```

> 📌 **Sub-workflows must be Published, not just Saved**, or the workflows that call them will fail.

Import them in any order, but make sure **all** of them end up Published. The key entry-point workflows are:

| Workflow | Triggered by | Purpose |
|----------|-------------|---------|
| **Application Orchestrator** | dashboard "Build" | Builds résumés + cover letters for selected jobs |
| **Job Discovery** | scheduler / dashboard | Finds and scores new jobs |
| **Application Refinement** | dashboard "Refine" | Re-works a résumé with your instructions |

The rest (Agents 1–10, Call LLM, the recorders, the source connectors) are sub-workflows these three call.

> ℹ️ **Note on file naming:** one source connector file is named `Abshy_Source` — this is a known typo for **Ashby**, a job-board source. It functions correctly; the name is just misspelled.

---

## Step 9 — Connect n8n credentials

n8n needs to know your Google service account and your AI provider key. You add these as **credentials** inside n8n.

### Google service account (for Sheets & Drive)

1. In n8n: **Credentials → Add Credential → Google Service Account API**.
2. Paste the service account's `client_email` and `private_key` from your JSON file (Step 3).
3. Save. Name it something memorable like `Maestro Google SA`.

The Google Sheets and Drive nodes in the imported workflows reference this credential — open one and re-select your credential if it shows as missing.

### Anthropic / OpenAI / Gemini (for the AI agents)

The agents call the providers through HTTP nodes inside the **Call LLM** workflow using a **Header Auth** credential.

1. **Credentials → Add Credential → Header Auth**.
2. For Anthropic: header name `x-api-key`, value = your `sk-ant-...` key.
3. Save and select it in Call LLM's Anthropic HTTP node. (Repeat with the appropriate header for OpenAI/Gemini if you use them — see the [Configuration guide](03-configuration.md#ai-provider-credentials).)

```
📷 The Header Auth credential (Anthropic example):
┌────────────────────────────────────────────────┐
│  Header Auth                                     │
├────────────────────────────────────────────────┤
│  Name:   ┌──────────────────────────────────┐  │
│          │ x-api-key                        │  │ ← exactly this header name
│          └──────────────────────────────────┘  │
│  Value:  ┌──────────────────────────────────┐  │
│          │ sk-ant-•••••••••••••••••••••••••• │  │ ← your Anthropic key
│          └──────────────────────────────────┘  │
│                                       [ Save ]   │
└────────────────────────────────────────────────┘
  OpenAI → header "Authorization", value "Bearer sk-..."
  Gemini → header "x-goog-api-key", value <your key>
```

### Webhook secret (protects the trigger webhooks)

The three entry workflows (Application Orchestrator, Job Discovery, Application Refinement) are protected by a shared secret so only your dashboard and scheduler can fire them. This must match the `MAESTRO_WEBHOOK_SECRET` you set in `.env.local` ([Step 6](#step-6--configure-environment-variables)).

1. **Credentials → Add Credential → Header Auth.**
2. Header name: `X-Maestro-Secret`
3. Value: **the exact same string** you used for `MAESTRO_WEBHOOK_SECRET`.
4. Save, then open each of the three entry workflows' **Webhook** node and select this credential for authentication.

```
📷 The webhook-secret Header Auth:
┌────────────────────────────────────────────────┐
│  Header Auth                                     │
├────────────────────────────────────────────────┤
│  Name:   X-Maestro-Secret                        │ ← exactly this
│  Value:  <same as MAESTRO_WEBHOOK_SECRET>        │ ← must match .env.local
│                                       [ Save ]   │
└────────────────────────────────────────────────┘
```

> ⚠️ If this secret doesn't match on both sides, the dashboard's trigger buttons return **503** and the scheduler silently skips every run. A mismatch here is the most common reason "nothing happens when I click Build."

---

## Step 10 — Start the Maestro dashboard

From the `maestro-ai-frontend` folder, install dependencies and start it:

```bash
cd maestro-ai-frontend
npm install
npm run dev
```

When you see `ready` in the terminal, open **http://localhost:4400**.

> 📌 The dashboard runs on **port 4400**, not the usual 3000. This is set in `package.json`.

If the dashboard loads and shows your (empty) tracking and discovery pages, the connection to Google Sheets is working.

---

## Step 11 — Start the scheduler (optional)

The scheduler fires discovery automatically on a cadence you set in the `config` tab (`job_search_cron`). You can skip this and trigger discovery manually from the dashboard instead.

The scheduler runs as a Docker service defined in the dashboard repo's compose file. From the `maestro-ai-frontend` folder:

```bash
docker compose -p maestro-ai up -d scheduler
```

It reads the same `.env.local` you set up in Step 6 and posts to the discovery webhook on schedule.

> ℹ️ The scheduler and n8n share the `maestro-ai` Docker project so they're on the same network. If you put the scheduler on n8n's network, you can set `N8N_WEBHOOK_BASE_URL=http://n8n:5678/webhook` (using the service name) for a cleaner connection.

---

## Step 12 — Verify the installation

Run this quick checklist:

- [ ] **Docker** says it's running.
- [ ] **n8n** loads at http://localhost:5678 and all workflows are **Published**.
- [ ] **Dashboard** loads at http://localhost:4400.
- [ ] In n8n, open **Job Discovery** and click **Execute Workflow** (manual run). It should complete without red error nodes. Check your Google Sheet's `jobs` tab — new rows should appear (depending on your watchlist).
- [ ] In the dashboard's **Discovery** page, the new jobs appear with fit scores.
- [ ] Select a job, click **Build**, and confirm the **Application Orchestrator** runs in n8n and a résumé lands in the `resumes` tab.

If all six pass, you're done. 🎉

If something fails, see **[Troubleshooting](07-troubleshooting.md)** — the most common issues are a service account that hasn't been shared with the sheet, an unpublished sub-workflow, or a mistyped private key.

---

[← Overview](01-overview.md) · [Next: Configuration →](03-configuration.md)
