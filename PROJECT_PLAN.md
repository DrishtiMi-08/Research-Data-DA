# Research Data Discovery Agent — Project Plan

**One-line description:** A tool that takes a research topic, finds relevant papers, extracts the datasets those papers used, resolves each dataset to an accessible download link, lets the user download public ones (with permission), and drafts access-request emails for restricted ones (never auto-sent).

**Origin / why this exists:** During capstone research, finding relevant papers, reading them to identify which datasets they used, then locating those datasets (many not publicly linked) was a slow, fully manual process. This tool automates the entire path from "I have a research topic" to "I have the data on my machine."

---

## The core insight

Everything between "I have a research topic" and "I have the data" is currently manual. Paper *search* is a solved, commodity problem (Elicit, Semantic Scholar, etc.) — do NOT rebuild it, use it as a component. The unsolved, valuable parts are:
1. **Extracting which datasets a paper actually used** (from prose/data-availability sections).
2. **Resolving those dataset mentions to accessible URLs.**
3. **Automating access requests for restricted datasets** (draft-only, human-approved).

Build effort should concentrate on 1-3, not on search.

---

## Umbrella topics to study independently

**Major:**
- Information Extraction / Named Entity Recognition (NER)
- Entity Resolution / Record Linkage (same dataset, many names)
- Scholarly APIs (Semantic Scholar, OpenAlex, Crossref, arXiv)
- Dataset repository APIs (Kaggle, HuggingFace Datasets, Zenodo, Dataverse)
- PDF / full-text parsing (extracting data-availability sections)
- Retrieval / ranking
- Agent orchestration (chaining pipeline stages)
- Human-in-the-loop design (permission gates)
- Background job processing (task queues)
- Gmail API + OAuth 2.0

**Minor (will surface mid-build):**
Rate limiting & API quotas, caching, deduplication, fuzzy string matching, DOI resolution, corresponding-author metadata extraction, async requests, secrets management, docker-compose multi-service orchestration, database migrations, progress reporting (polling vs SSE/streaming).

---

## The pipeline (stages a query flows through)

- **Stage 0 — Input:** user enters a research topic/question.
- **Stage 1 — Paper discovery:** query Semantic Scholar / OpenAlex → top ~10 papers (title, abstract, authors, year, DOI). Do not rebuild search.
- **Stage 2 — Full-text acquisition:** fetch full text or at least abstract + data-availability statement. Open-access via arXiv/PMC; closed papers fall back to abstract only.
- **Stage 3 — Dataset extraction (THE HARD CORE):** extract dataset mentions from paper text via LLM.
- **Stage 4 — Entity resolution + URL linking (SECOND HARD PART):** collapse duplicate mentions into canonical datasets; resolve each to an accessible location (Kaggle/HuggingFace/Zenodo search or a URL in the paper).
- **Stage 5 — Ranking + presentation:** rank datasets by relevance/frequency; present a reviewable list with links. (Delivers Option 1.)
- **Stage 6 — Permissioned download:** user selects public datasets; agent downloads via repo APIs after explicit confirmation. (Delivers Option 2.)
- **Stage 7 — Restricted-data access requests:** for non-public datasets, find corresponding-author contact from paper metadata, draft an access-request email, show it to the user, create a Gmail DRAFT only — never send without the user hitting send. (Delivers Option 3 — the differentiator.)

---

## Where the hard problems actually are

1. **Stage 3 (dataset extraction)** is the real difficulty. Papers name datasets inconsistently — in prose, in data-availability statements, in footnotes, no clean structured field. Start with an LLM + focused prompt (fast to build). A fine-tuned NER model is more robust but far more work — only consider later.
2. **Stage 4 (resolution + URL linking)** is second hardest. A named dataset may not be findable by name on Kaggle. Fuzzy matching, multiple candidates, "does this match actually correspond." Show confidence in the UI; never pretend certainty.
3. **Stage 7 contact-finding degrades** — author emails go stale. Design "no current contact found" as a normal outcome, not an error.

Everything else (API calls, downloads, Gmail draft creation) is standard integration work.

---

## Why this needs a strong backend

Unlike a single fast endpoint, one query kicks off a long-running (30-90s), multi-stage pipeline with several slow external API calls. This requires:
- **Long-running background jobs** (return a job ID, poll/stream progress — don't freeze the request).
- **Orchestration with partial failure** (some papers/datasets fail; pipeline must degrade gracefully).
- **Caching** (scholarly APIs are rate-limited, LLM calls cost time/money — don't re-fetch/re-extract).
- **Persistent state** (download history, drafted/sent access requests).
- **Rate limiting + retry/backoff** across 4-5 external APIs.

---

## Technology stack

**Core backend:**
- FastAPI — API layer (async; I/O-bound on external APIs)
- Celery (or RQ to start simpler) — background job queue for the pipeline
- Redis — Celery broker + cache layer
- PostgreSQL — persistent state (cached results, download history, access requests)
- SQLAlchemy — ORM
- Alembic — DB migrations

**External integrations:**
- Semantic Scholar API + OpenAlex API — paper discovery (free, no card)
- arXiv API / PMC — open-access full text
- Crossref API — DOI resolution + corresponding-author metadata
- Kaggle API, HuggingFace `datasets`, Zenodo API — dataset search + download
- Gmail API (OAuth 2.0) — draft creation only (never auto-send)
- LLM API — Gemini Flash primary (large context for full-text parsing), Groq as fallback (reuse the fallback-chain pattern)

**Frontend:** minimal (HTML/JS or Streamlit) until backend works; real frontend (React) only in v4 if at all.

**Deployment:** Docker + docker-compose (multi-service: FastAPI, Celery worker, Redis, Postgres); Render or Railway for hosting.

---

## Build order (tiered — always keep a working version)

### v0 — thin vertical slice (abstract-only, synchronous, NO heavy backend yet)
Goal: prove topic → papers → extracted dataset names → links works end to end. Deliberately a single synchronous script — validate the concept before building infrastructure.

- **Step 1 — Paper discovery:** function: topic string → Semantic Scholar API → top ~10 papers (title, abstract, authors, year, DOI). Get it printing clean results.
- **Step 2 — Dataset extraction (DO THIS FIRST — riskiest assumption):** function: abstract → LLM with focused prompt → JSON list of dataset names. Test on 5-10 real abstracts, eyeball quality. If this is garbage, rethink the project before building anything else.
- **Step 3 — Naive dataset resolution:** function: dataset name → Kaggle/HuggingFace search → top match URL. Rough is fine.
- **Step 4 — Wire together:** one script: topic → papers → extract datasets per abstract → resolve each to a link → print consolidated (dataset, source paper, candidate link). Run on the real capstone topic. If output is even ~50% useful, concept validated.
- **Step 5 — One FastAPI endpoint:** `POST /search` takes a topic, returns JSON. Synchronous is fine — feeling the slowness here motivates the v1 async rework.

**Do NOT in v0:** job queue, caching, Postgres, real UI, Gmail.

### v1 — make extraction good + add the real backend
- Full-text / data-availability parsing for open-access papers.
- Improve extraction prompt; add entity resolution to collapse duplicates.
- Introduce Celery + Redis (background jobs), Postgres + SQLAlchemy (caching + state), progress reporting.
- **Most of the real effort lives here.**

### v2 — permissioned download
- Kaggle/HuggingFace/Zenodo download with a confirmation gate and clean selection UI.

### v3 — the differentiator: access-request drafting
- Detect restricted datasets → pull corresponding-author contact from metadata → draft email → Gmail draft creation (OAuth). Human-approval gate, DRAFT-ONLY, always.

### v4 — polish
- Caching improvements, better ranking, real UI, deployment, README + demo video.

**Cut line:** v0 + v1 alone is already a strong, defensible project. v2 + v3 make it exceptional. Do NOT start v3 until v0-v1 are solid — the differentiator is worthless if core extraction doesn't work.

---

## Critical guardrails (do not violate)

- **Gmail: draft-only, ALWAYS.** Never auto-send. Explicit user-send required. Automated cold email to researchers becomes spam and burns reputations.
- **Downloads: explicit permission gate** before pulling anything.
- **Show confidence, don't fake certainty** in dataset resolution (Stage 4) and contact-finding (Stage 7).
- **Secrets never in version control** (`.gitignore` + `.env.example`; real keys in host env vars — same pattern as prior projects).

---

## First action when building

Start with **v0 Step 2** (dataset extraction), not Step 1. Paste 5-10 real paper abstracts into the LLM with the extraction prompt and check output quality. This is the project's riskiest assumption — validate it before building anything else.
