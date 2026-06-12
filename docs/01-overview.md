# 1. Overview

[← Back to README](../README.md) · [Next: Installation →](02-installation.md)

---

This page explains, in plain language, what Maestro AI does and how its parts work together. If you just want to install it, you can skim this and jump to the [Installation Guide](02-installation.md) — but five minutes here will make everything else clearer.

## What problem does it solve?

Applying for jobs is repetitive and slow: find the posting, decide whether it's worth your time, rewrite your résumé for it, write a cover letter, and keep track of where you've applied. Maestro automates the tedious parts and leaves the decisions to you.

It does five things:

1. **Discovers** open jobs from company career pages and job APIs.
2. **Scores** each job for how well it fits you — twice, in fact: once against the role you're *targeting* and once against your actual *background*.
3. **Builds** a tailored résumé and cover letter for the jobs you choose, using AI agents that draft, critique, refine, and fact-check the result.
4. **Lets you review and refine** everything through a web dashboard.
5. **Tracks** every application's status, from "not applied" through "offer" or "rejected."

## The two halves

Maestro is made of two cooperating systems that share a single database.

```
                        ┌──────────────────────────────┐
                        │   Google Sheets (database)    │
                        │   + Google Drive (files)      │
                        └──────────────┬───────────────┘
                                       │ both read & write
                ┌──────────────────────┴──────────────────────┐
                │                                              │
   ┌────────────▼─────────────┐                  ┌─────────────▼────────────┐
   │   Backend pipeline        │                  │   Maestro dashboard       │
   │   (n8n + AI agents)       │   webhooks       │   (Next.js web app)       │
   │                           │ ◄──────────────  │                           │
   │  • discovers jobs         │   "build these"  │  • review jobs            │
   │  • scores them            │   "run discovery"│  • trigger builds         │
   │  • drafts résumés         │ ──────────────►  │  • read & refine résumés  │
   │  • critiques & refines    │   results land   │  • track applications     │
   └───────────────────────────┘   in the sheet   └───────────────────────────┘
                ▲
                │ fires on a schedule you set
   ┌────────────┴─────────────┐
   │   Scheduler (Node worker) │
   └───────────────────────────┘
```

### Backend pipeline (n8n)

[n8n](https://n8n.io) is a visual automation tool. Maestro's backend is a set of n8n **workflows** — think of each as a flowchart that runs automatically. The pipeline contains ten AI **agents**, each a specialist:

| Agent | Job |
|-------|-----|
| **1 — Résumé Builder** | Drafts a tailored résumé from your master profile + the job description |
| **2 — Cover Letter Builder** | Drafts a matching cover letter |
| **3 — Résumé Verifier** | Fact-checks the résumé against your profile (no invented claims) |
| **4 — Cover Letter Verifier** | Fact-checks the cover letter |
| **5 — Critic** | Critiques the résumé like a recruiter would |
| **6 — Résumé Refiner** | Rewrites the résumé to address the critique |
| **7 — Résumé Scorer** | Scores the final résumé's fit (0–100) |
| **8 — Discovery Scorer** | Scores newly discovered jobs for fit |
| **9 — Résumé Fine-Refiner** | Applies your manual refinement instructions on later passes |
| **10 — Job Ranker** | A fast first-pass ranker that keeps the discovery funnel open |

These agents don't talk to AI providers directly. They all route through one shared module, **Call LLM**, which supports **Anthropic Claude, OpenAI, and Google Gemini** and handles errors, retries, and cost tracking in one place.

### Maestro dashboard (Next.js)

The dashboard is the human side. It reads the same Google Sheet the pipeline writes to, and it sends instructions back to the pipeline through **webhooks** (a webhook is just a URL the dashboard calls to say "please run this"). From the dashboard you can:

- Browse discovered jobs and their fit scores
- Select jobs and click **Build** to generate applications
- Read the generated résumé, see the critic's notes and verifier's checks, and request refinements
- Save a polished `.docx` résumé to Google Drive
- Track every application's status

### The scheduler

n8n can't read a schedule out of your spreadsheet on its own, so Maestro includes a tiny standalone **scheduler** — a Node program in its own container. It reads how often you want to discover jobs (a setting in your config) and fires the discovery pipeline on that cadence. You can also trigger discovery manually from the dashboard at any time.

## How the data flows

Everything is coordinated through one Google Sheets file with many tabs (the full list is in the [Database Reference](06-database-reference.md)). The most important tabs:

- **`config`** — all your settings (your name, target roles, which AI models to use, how often to run discovery).
- **`jobs`** — every job discovered or submitted, plus its scores and status.
- **`resumes`** — every résumé version generated, including refinements.
- **`agent_outputs`** — the raw critic/verifier/scorer output behind each résumé.
- **`model_usage`** — a per-call log of tokens and cost, so you always know what you're spending.

The pipeline writes; the dashboard reads and writes back; nothing is hidden.

## Two scoring axes (why fit scores come in pairs)

A subtle but important design point: every discovered job gets **two** fit scores.

- **Target fit** — does this job match the role you're *searching for*?
- **Background fit** — does this job match the experience you *actually have*?

These can legitimately differ. If you're an experienced executive applying to a more junior IC role on purpose, the target fit might be high while the background fit flags the seniority gap. Keeping them separate stops one from masking the other.

## What's automated vs. what you decide

| Maestro does automatically | You decide |
|----------------------------|------------|
| Find and score jobs | Which jobs to build applications for |
| Draft, critique, refine, verify résumés | Whether a résumé is good enough to send |
| Track cost and tokens | Which AI models and budget to use |
| Generate `.docx` files | When and where to actually apply |

Nothing is sent anywhere on your behalf. Maestro prepares; you act.

---

[← Back to README](../README.md) · [Next: Installation →](02-installation.md)
