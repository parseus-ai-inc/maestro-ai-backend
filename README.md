<div align="center">

# Maestro AI — Backend

**The n8n pipeline that powers Maestro AI.**

A product of Parseus AI · *Decode. Decide. Deliver.*

</div>

---

This repository contains the backend half of [Maestro AI](https://github.com/parseus-ai-inc/maestro-ai-backend): the n8n workflows that discover jobs, score them, and generate tailored résumés and cover letters, plus the database template they run on.

The other half — the **dashboard** (where you review, refine, and track) and the **discovery scheduler** — lives in [`maestro-ai-frontend`](https://github.com/parseus-ai-inc/maestro-ai-frontend).

## What's in here

```
maestro-ai-backend/
├── workflows/              31 n8n workflows, organized by role
│   ├── entry/              the 3 entry points + cover-letter pipeline
│   ├── agents/             the 11 AI agents
│   ├── sources/            job-board connectors (Greenhouse, Lever, Ashby, JSearch)
│   ├── recorders/          the recorders that persist results to Sheets
│   └── shared/             Call LLM, Data Loader, Prompt Loader, etc.
├── Database_Template.xlsx  the Google Sheets database template (import this)
├── docker-compose.yml      runs n8n locally
├── .env.example            copy to .env and fill in
└── docs/                   full documentation for the whole system
```

## Quick start

Full step-by-step instructions (for technical and non-technical users) are in **[docs/02-installation.md](docs/02-installation.md)**. The short version:

```bash
# 1. Configure
cp .env.example .env          # then edit .env

# 2. Start n8n
docker compose -p maestro-ai up -d

# 3. Open n8n and import the workflows
#    http://localhost:5678  →  import every file under ./workflows
```

Then import `Database_Template.xlsx` into Google Sheets, connect your Google service account and AI provider keys, and you're ready. See **[docs/02-installation.md](docs/02-installation.md)** for everything, and **[docs/11-importing-workflows.md](docs/11-importing-workflows.md)** for the workflow-import walkthrough.

## Documentation

The `docs/` folder documents the whole Maestro system (both halves):

| Guide | |
|-------|--|
| [Overview](docs/01-overview.md) | What Maestro does and how it fits together |
| [Installation](docs/02-installation.md) | Full setup from scratch |
| [Configuration](docs/03-configuration.md) | Google, API keys, and the `config` tab |
| [Running the System](docs/04-running.md) | Day-to-day use |
| [Importing Workflows](docs/11-importing-workflows.md) | How to import these 31 workflows into n8n |
| [Architecture](docs/05-architecture.md) | How the pipeline works (technical) |
| [Database Reference](docs/06-database-reference.md) | Every tab and column |
| [Troubleshooting](docs/07-troubleshooting.md) | Fixing common problems |
| [FAQ & Glossary](docs/08-faq-glossary.md) | Quick answers |
| [Prompting & Customizing](docs/09-prompting.md) | Editing the agents' prompts |
| [Master Career Dossier](docs/10-master-dossier.md) | Writing your career document |

## License

[Apache License 2.0](LICENSE) — Copyright 2026 Parseus AI. See [NOTICE](NOTICE).
