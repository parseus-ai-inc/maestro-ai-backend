# 11. Importing the Workflows into n8n

[← Master Career Dossier](10-master-dossier.md) · [Back to README →](../README.md)

---

This guide walks you through importing Maestro's 31 workflows into your own n8n instance. It assumes you've already started n8n (see [Installation, Step 7](02-installation.md#step-7--start-the-backend-n8n)) and can open it at **http://localhost:5678**.

> ⏱️ **Time:** 20–30 minutes. Most of it is the import clicks; the wiring at the end is quick.

## What you're importing

The `workflows/` folder contains 31 workflows, organized into five subfolders by role:

```
workflows/
├── entry/        (4)  the pipelines you trigger — discovery, build, refine, cover letters
├── shared/       (5)  helpers every pipeline calls — Call LLM, Data Loader, etc.
├── agents/      (11)  the AI agents (Agent 1 … Agent 11)
├── sources/      (4)  job-board connectors — Greenhouse, Lever, Ashby, JSearch
└── recorders/    (7)  the workflows that save results to Google Sheets
```

These aren't independent — the **entry** workflows call the **agents**, **sources**, **recorders**, and **shared** helpers as sub-workflows. That dependency shapes the import order below.

## Before you start

You need n8n running and reachable at http://localhost:5678. If you haven't set up the database, Google credentials, or API keys yet, do [Installation Steps 3–9](02-installation.md) first — the workflows will import without them, but they won't *run* until credentials are connected.

---

## Step 1 — Import every workflow

n8n imports one file at a time. You'll repeat this for all 31 files. **Import order matters** because sub-workflows must exist before the workflows that reference them resolve cleanly. Import in this order:

1. `shared/` (5 files)
2. `recorders/` (7 files)
3. `agents/` (11 files)
4. `sources/` (4 files)
5. `entry/` (4 files) — **last**, since these call everything above

> 💡 If you import out of order, it's not fatal — references resolve by workflow once everything is present. But importing helpers first means fewer "missing workflow" warnings along the way.

For **each** file:

1. In n8n, click **Overview** (or **Workflows**) in the left sidebar.
2. Click the **Create Workflow** dropdown (top-right) → **Import from File…**
3. Select the `.json` file from the `workflows/` subfolder.
4. The workflow opens. Click **Save** (top-right).

!!! example "Import from File"
    In n8n, click **Create workflow ▾ → Import from File…** and select the `.json` file.

Repeat until all 31 are imported. After this, your workflow list shows all 31 by name (Agent 1 – Resume Builder, Application Orchestrator, Call LLM, and so on).

---

## Step 2 — Publish every workflow

Importing and saving is **not** enough. n8n only lets one workflow call another if the callee is **Active** (published). If you skip this, entry workflows fail at runtime with "could not find workflow" errors.

For each of the 31 workflows, open it and toggle it **Active** using the switch in the top-right.

!!! warning "Activate every workflow"
    After importing, toggle each workflow from **Inactive** to **Active** (top-right). All 31 must be Active — sub-workflows that aren't published are invisible to their callers.

> 📌 **All 31 must be Active**, not just the entry workflows. The agents, sources, recorders, and shared helpers are all called as sub-workflows, and a sub-workflow that isn't published is invisible to its caller.

A fast way to verify: in the workflow list, every row should show a green **Active** badge.

---

## Step 3 — Reconnect the Google credential

The exported workflows reference a Google service-account credential by an internal ID that won't exist in *your* n8n. After import, the Google Sheets and Google Drive nodes will show the credential as missing or unset. You fix this once, then re-select it where needed.

1. First create the credential if you haven't: **Credentials → Add Credential → Google Service Account API**, and paste your service account's `client_email` and `private_key`. (See [Installation Step 9](02-installation.md#step-9--connect-n8n-credentials).)
2. Open any workflow that has a Google Sheets or Drive node showing a red "credential" warning.
3. Click the node, open its **Credential** dropdown, and select your Google credential.
4. Save.

!!! example "A node with a missing credential"
    A node showing a ⚠ on its **Credential** dropdown (e.g. a Google Sheets node) needs you to re-select your credential. Open the node and pick your Google service account.

> 💡 You don't have to touch every node by hand. n8n shares a credential across all nodes that use it — once you select your Google credential on one Sheets node, re-open the others and they'll usually offer it as the default. Work through any that still show a warning.

**Which workflows have Google nodes?** Mainly the **recorders** (they write to Sheets), the **sources** and **Job Fetcher** (some read config), **Data Loader** and **Prompt Loader** (read config/prompts), and the **Application Recorder** (writes to Drive). Check each for a credential warning.

---

## Step 4 — Connect the AI provider credential

The agents call Anthropic / OpenAI / Gemini through HTTP nodes inside the **Call LLM** workflow, using a **Header Auth** credential.

1. **Credentials → Add Credential → Header Auth.**
2. For Anthropic: header name `x-api-key`, value = your `sk-ant-…` key.
3. Open **Call LLM**, find the **Anthropic HTTP Request** node, and select this credential.
4. If you use OpenAI or Gemini, repeat with their headers (`Authorization: Bearer …` / `x-goog-api-key`) on the respective nodes.

See [Configuration → AI provider credentials](03-configuration.md#ai-provider-credentials) for the exact header values.

---

## Step 5 — Set the webhook secret on the entry workflows

The four trigger workflows are protected by a shared secret so only your dashboard and scheduler can fire them. Each entry workflow's **Webhook** node needs a Header Auth credential whose value matches the `MAESTRO_WEBHOOK_SECRET` in your dashboard's `.env.local`.

1. **Credentials → Add Credential → Header Auth.**
2. Header name: `X-Maestro-Secret`; value: the same string you set for `MAESTRO_WEBHOOK_SECRET`.
3. Open each of these and select the credential on the **Webhook** node:
   - **Job Discovery** (`/webhook/run-discovery`)
   - **Application Orchestrator** (`/webhook/build-application`)
   - **Application Refinement** (`/webhook/refine-resume`)
   - **Cover Letter Generation & Refinement** (`/webhook/generate-cover-letter`)

> ⚠️ If this secret doesn't match what the dashboard/scheduler send, triggering returns **503** and the scheduler silently skips. This is the most common "nothing happens when I click Build" cause.

---

## Step 6 — Point the database at your Sheet

A few workflows carry a placeholder where a Google Sheet/Drive ID belongs. The most important is the **Run Error Handler**, which holds the database ID in a Set node (it can't read it from the run, by design).

1. Open **Run Error Handler**.
2. Find the **Error Handler Config** node (a Set node) holding `database_id` (value shows `YOUR_GOOGLE_SHEETS_DATABASE_ID`).
3. Replace it with your actual database Sheet ID (the long string from your Sheet's URL).
4. Save.

> 💡 Most other workflows read the database ID from config or from the data passed in at runtime, so they don't need editing. If any node shows a `YOUR_…` placeholder, replace it with your real ID the same way.

---

## Step 7 — Refresh sub-workflow schemas (if needed)

If you later edit a sub-workflow's inputs, the workflows that call it cache the old schema. After importing, if an entry workflow seems to pass empty or wrong data to a sub-workflow, open the calling node (an **Execute Workflow** node) and click the **refresh** ⟳ icon to re-pull the sub-workflow's input schema.

You usually won't need this on a fresh import, but it's the fix if data isn't flowing between workflows.

---

## Step 8 — Verify

A quick end-to-end check:

1. Open **Job Discovery** and click **Execute Workflow** (the manual-run button). It should complete without red error nodes. Check your Sheet's `jobs` tab — new rows should appear (depending on your `watchlist_companies`).
2. If discovery works, the shared helpers, sources, an agent (Agent 10 + Agent 8), and a recorder all ran — that exercises most of the system.

If a node errors, open the execution, read the red node's message, and cross-reference [Troubleshooting](07-troubleshooting.md). The usual first-import issues are: a workflow left **Inactive**, a **Google credential** not re-selected, or a missing **AI provider** key.

---

## Quick checklist

- [ ] All 31 workflows imported
- [ ] All 31 set to **Active**
- [ ] Google service-account credential created and selected on Sheets/Drive nodes
- [ ] AI provider (Header Auth) credential selected in **Call LLM**
- [ ] `X-Maestro-Secret` credential on the entry workflows' Webhook nodes
- [ ] `database_id` placeholder replaced in **Run Error Handler**
- [ ] **Job Discovery** manual run completes and writes to the `jobs` tab

Once all eight are checked, the pipeline is live and the dashboard can drive it.

---

[← Master Career Dossier](10-master-dossier.md) · [Back to README →](../README.md)
