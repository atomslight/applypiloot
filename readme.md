
# ApplyPilot Architecture: A Complete Reverse Engineering Guide

## 1. Executive Summary
ApplyPilot is an open-source, fully autonomous, AI-driven job application pipeline. It operates as a 6-stage funnel: Discovering jobs, Enriching job data, Scoring fit via AI, Tailoring resumes dynamically, Generating cover letters, and Auto-Applying via browser automation. Built in Python, it integrates scraping frameworks, LLM orchestrators (Gemini/OpenAI/Local), and the Claude Code CLI controlling Playwright via Model Context Protocol (MCP) to achieve true end-to-end autonomy.
This document serves as a complete reverse engineering of the architecture, its design decisions, logic, flow, and dependencies. It is written to be a definitive teaching guide on the codebase.

## 2. Problem the Project Solves
**What?** Job hunting is a numbers game requiring extreme personalization, leading to burnout. Applicants spend hours discovering jobs, tweaking resumes, writing cover letters, and filling out tedious Workday/Taleo forms.

**Why build it?** To completely offload the manual labor of the job search. By automating the entire process, candidates can scale their job hunt without sacrificing the quality or personalization of individual applications.

**Who uses it?** Software engineers, data scientists, and general job seekers wanting to scale their application output without sacrificing quality.

**Alternatives?**
- Manual application (slow, high effort).
- Simple bots (e.g., AIHawk) that only do LinkedIn "Easy Apply" (limited scope, highly competitive).
- Recruiting agencies (expensive, unpredictable).

**Why this implementation?** ApplyPilot stands out because it doesn't just hit API endpoints; it drives a real browser using Claude Code and Playwright to navigate complex, multi-page application forms natively. It uses advanced agentic workflows via LLMs to "think" its way through CAPTCHAs, tricky forms, and screen questions. It also utilizes a localized SQLite database as a queueing system, making the pipeline resilient, resumable, and highly parallelizable. It bridges the gap between dumb scrapers and general-purpose intelligence.

## 3. High-Level Architecture
ApplyPilot uses a **Pipeline/Dataflow Architecture** combined with a **Local Worker Model**.

- **The Database (SQLite):** Acts as the central state machine and queue. Jobs move through stages based on NULL/NOT NULL column states.
- **The Pipeline Layers:**
  1. **Discovery:** Scrapes boards using `python-jobspy` and custom Workday/Direct site scrapers. AI smart extraction is available as well.
  2. **Enrichment:** Extracts full job descriptions using CSS selectors or AI fallback.
  3. **Scoring:** LLM evaluates resume vs job description (1-10 scale).
  4. **Tailoring:** LLM rewrites the resume dynamically to emphasize relevant skills.
  5. **Cover Letter:** Generates customized letters.
  6. **Auto-Apply:** Claude Code CLI receives a prompt and an MCP configuration. It controls a Chrome instance via Playwright to fill forms.

- **Concurrency Model:** Uses Python's `threading` and `ThreadPoolExecutor` for parallel scraping and application submission. Each worker gets an isolated Chrome instance and CDP (Chrome DevTools Protocol) port, allowing scale to match system capabilities.

- **Orchestration:** Driven by a CLI interface built with Typer and Rich, users can invoke specific pipeline components manually or stream them constantly.

## 4. Complete Tech Stack Analysis

### Python 3.11+
* **What it is:** The programming language used to build the entire toolset.
* **Why it exists:** Python is the de-facto standard for AI, scripting, web scraping, and automation.
* **Is it Core?** Yes, it is the runtime language.

### SQLite
* **What it is:** An embedded database that stores the system's state machine.
* **Why it exists/Why this project needs it:** Instead of requiring users to set up Postgres or MySQL, SQLite is a file-based database built into Python. It acts as a resilient queue.
* **WAL Mode (Write-Ahead Logging):** SQLite is run in WAL mode to allow concurrent readers and writers, which is crucial since ApplyPilot uses multithreading to stream pipeline stages simultaneously.
* **Is it Core?** Yes.

### Typer & Rich
* **What it is:** `typer` builds CLI interfaces (commands like `applypilot run`), and `rich` handles terminal UI (tables, progress bars).
* **Why it exists:** Makes the application approachable and user-friendly for developers.

### python-jobspy
* **What it is:** A third-party library to scrape Indeed, LinkedIn, Glassdoor, etc., without getting blocked easily.
* **Why it exists:** It is difficult to build scrapers for major job boards due to anti-bot measures. JobSpy provides a robust interface.
* **Where it fits:** Used in the Discovery stage (`src/applypilot/discovery/jobspy.py`).

### Playwright (via Node.js/MCP)
* **What it is:** A browser automation framework.
* **Why it exists:** Navigating modern SPA (Single Page Applications) like Workday or Greenhouse requires executing JavaScript and simulating real clicks.
* **Why this project needs it:** Used by the LLM agent to interact with the DOM during Auto-Apply.

### Claude Code CLI & MCP (Model Context Protocol)
* **What it is:** Claude Code is a CLI tool from Anthropic. MCP is a standard for giving AI models context and tools.
* **Why it exists:** We need an agent to physically control the browser. ApplyPilot dynamically creates an MCP config pointing to Playwright, and passes it to Claude Code so the LLM can command the browser directly.
* **Where it fits:** Used in the Auto-Apply stage (`src/applypilot/apply/launcher.py`).

### httpx
* **What it is:** An asynchronous HTTP client for Python.
* **Why it exists:** Used for making HTTP requests to LLM API endpoints and fetching websites during enrichment.
* **Where it fits:** Used for all direct LLM integration (`src/applypilot/llm.py`).

### LLM APIs (Gemini, OpenAI, Local models)
* **What it is:** The "brains" for scoring, tailoring, and smart extraction.
* **Why it exists:** Allows dynamic understanding of unstructured text (resumes and job descriptions).

## 5. Complete Terminology & Glossary

* **Pipeline Stage:** One of the 6 core steps (discover, enrich, score, tailor, cover, pdf).
  * *Real-world example:* An assembly line where a car moves from the chassis (discovery), to engine install (enrich), to painting (tailor).
* **MCP (Model Context Protocol):** A standardized way for AI agents to use external tools.
  * *In ApplyPilot:* Claude Code uses an MCP server to issue Playwright commands to a local Chrome instance.
* **CDP (Chrome DevTools Protocol):** The debugging protocol Chrome uses.
  * *In ApplyPilot:* Playwright connects to Chrome via CDP ports to drive the browser.
* **WAL Mode (Write-Ahead Logging):** An SQLite setting that allows concurrent readers and writers.
  * *Purpose:* Normally SQLite locks the entire database on write. WAL mode fixes this, allowing the multithreaded workers to function without crashing.
* **JobSpy:** An open-source scraper that bypasses anti-bot protections on major job boards.
* **Smart Extraction:** ApplyPilot's custom fallback scraper that loads a webpage, intercepts JSON-LD/APIs, and asks an LLM to generate CSS selectors for scraping.

## 6. Folder-by-Folder Breakdown

* `src/applypilot/` (Core Application Code)
  * `__init__.py` / `__main__.py` / `cli.py`: Entry points and CLI routing. The Typer app is defined here.
  * `pipeline.py`: Orchestrates the sequence of stages, supporting both sequential and streaming (concurrent) execution. It is the "Conveyor Belt".
  * `database.py`: SQLite connection management, schema definition, and state querying.
  * `config.py`: Environment variable loading, path management, and tier/dependency checking.
  * `llm.py`: Unified HTTP client for Gemini/OpenAI/Local models.
  * `discovery/`: Scrapers for JobSpy (`jobspy.py`), Workday (`workday.py`), and "Smart Extract" (`smartextract.py`).
  * `enrichment/`: Fetches full job descriptions from URLs (`detail.py`).
  * `scoring/`: The AI reasoning engines for fit scoring (`scorer.py`), resume tailoring (`tailor.py`), cover letters (`cover_letter.py`), and PDF generation (`pdf.py`).
  * `apply/`: The autonomous application engine. `launcher.py` manages worker threads, `chrome.py` handles headless/headful browser processes, `prompt.py` generates the strict instructions for Claude, and `dashboard.py` renders live CLI feedback.
  * `wizard/`: First-time setup CLI wizard (`init.py`).

## 7. File-by-File Breakdown (Key Files)

### `src/applypilot/database.py`
* **Purpose:** The single source of truth. Contains `init_db()` which defines a massive flat table (`jobs`).
* **Design Decision:** Columns represent states (e.g., `detail_scraped_at`, `fit_score`). This flat design avoids complex JOINs and makes state transitions atomic (e.g., `WHERE fit_score IS NULL` means it needs scoring). It acts as a finite state machine.

### `src/applypilot/llm.py`
* **Purpose:** A custom HTTP client using `httpx`.
* **Important Logic:** It auto-detects providers from ENV vars. Crucially, it has fallback logic for Gemini—if the OpenAI-compatible endpoint returns a 403 (common for preview models), it seamlessly switches to Gemini's native API. It also handles exponential backoff for rate limits.

### `src/applypilot/pipeline.py`
* **Purpose:** The orchestrator.
* **Logic:** Contains a `run_pipeline` function. It resolves stages, handles execution order, and manages threads in `--stream` mode, utilizing a thread-safe `_StageTracker`.

### `src/applypilot/apply/launcher.py`
* **Purpose:** The heart of the auto-apply system.
* **Execution Flow:** It queries the DB for a job ready to apply, locks it (`apply_status = 'in_progress'`), spawns a local Chrome instance on a specific CDP port, generates a dynamic MCP JSON config, and spawns the `claude` CLI process as a subprocess.

### `src/applypilot/apply/prompt.py`
* **Purpose:** Generates the massive prompt given to Claude Code.
* **Important Logic:** It merges the user's profile (`profile.json`) with the tailored resume, and adds aggressive rules about Captcha bypassing, salary negotiation, and what buttons not to press.

### `src/applypilot/discovery/smartextract.py`
* **Purpose:** Discovers jobs from generic websites when JobSpy isn't enough.
* **Algorithm:** A Two-phase approach. First, it scrapes the DOM and intercepts network requests to find API responses or JSON-LD. It asks the LLM to choose a strategy. If it must use raw HTML, it extracts candidates and asks the LLM to generate CSS Selectors, which it then executes.

## 8. Module-by-Module Breakdown

### The Apply Module (`src/applypilot/apply/`)
This is the crown jewel. Unlike standard scrapers, it doesn't write hardcoded Selenium scripts (which break when UIs update). Instead, it delegates the "how to fill a form" logic to Claude 3.5 Sonnet.
1. `chrome.py` launches Chrome with remote debugging open.
2. `launcher.py` dynamically creates an `.mcp-apply-{worker_id}.json` file pointing to that Chrome instance.
3. `prompt.py` injects the user's tailored resume, profile data, and strict rules ("DO NOT hallucinate", "Use Playwright to click Submit").
4. The subprocess runs `claude --mcp-config ...`. Claude sees the browser, reads the DOM, types into fields, and clicks submit entirely on its own.

### The Scoring Module (`src/applypilot/scoring/`)
Uses LLMs to evaluate the job.
* `scorer.py`: Asks the LLM to rate the fit out of 10.
* `tailor.py`: Rewrites the user's base resume markdown to highlight keywords found in the job description.
* `pdf.py`: Uses `weasyprint` or similar to turn the markdown back into a professional PDF for uploading.

## 9. Core Engine Deep Dive
The "Engine" is the SQLite Database acting as a state machine.
**Execution Start:** User runs `applypilot run --stream`.
**Initialization:** `pipeline.py` initializes the DB and starts thread workers.
**Control Flow (Streaming Mode):**
- A thread starts the `discovery` scraper. It inserts rows into SQLite.
- The `enrich` thread polls the DB: `SELECT * FROM jobs WHERE detail_scraped_at IS NULL`. It picks up new rows, scrapes them, updates the row.
- The `score` thread polls: `WHERE fit_score IS NULL AND full_description IS NOT NULL`. It calls the LLM, updates the row.
- The `tailor` thread polls: `WHERE fit_score >= 7 AND tailored_resume_path IS NULL`.
**Data Flow:** Data flows left-to-right through the database columns. A job only moves to the next stage when its prerequisite columns are populated.

## 10. Complete Execution Flow

1. **User runs `applypilot run`:**
   - DB is initialized (`init_db()`).
   - JobSpy searches boards based on `searches.yaml`.
   - URLs and brief descriptions are saved to DB.
2. **Enrichment:**
   - Bot visits each URL.
   - Extracts full HTML/Text. Updates DB.
3. **Scoring:**
   - LLM reads resume text and job description text.
   - Returns a JSON string with `SCORE` and `REASONING`. DB is updated.
4. **Tailoring:**
   - If Score >= 7, LLM rewrites the resume markdown.
   - Saved to `~/.applypilot/tailored_resumes/`. DB updated with path.
5. **PDF Conversion:**
   - Markdown converted to PDF.
6. **User runs `applypilot apply`:**
   - `launcher.py` queries for a tailored job.
   - Launches Chrome.
   - Launches Claude CLI with MCP Playwright.
   - Claude navigates to the job URL, reads the form, types data from the user profile, uploads the PDF, and clicks Submit.
   - `launcher.py` reads Claude's stdout to detect "RESULT:APPLIED" or "RESULT:FAILED". DB updated.

## 11. Data Flow Analysis
Data originates from public internet job boards -> Transformed into Pandas DataFrames (JobSpy) -> Stored in SQLite (Normalized) -> Pushed to LLMs (String prompts) -> Augmented data (Scores/Tailored Text) saved back to SQLite/File System -> Consumed by Claude (Context Window) -> Pushed back to internet (Form Submissions).
The data state is purely managed by SQLite column updates.

## 12. Control Flow Analysis
- **Sequential Mode:** Simple blocking loops iterating over the pipeline stages. (`_run_sequential`)
- **Streaming Mode (`--stream`):** Uses a `_StageTracker` with `threading.Event`. Each stage runs in a daemon thread, polling the DB continuously until the upstream stage fires its event indicating it has finished producing work, AND the local stage's DB query returns 0 pending items. (`_run_streaming`)
- **Apply Concurrency:** Uses `ThreadPoolExecutor` in `launcher.py` to spawn multiple chrome instances.

## 13. Design Patterns & Architecture

*   **Blackboard Pattern:** The SQLite database acts as a blackboard where different specialist agents (Scraper, Scorer, Tailor, Applier) read state, do work, and write back without direct communication with each other.
*   **Dependency Injection / IoC:** LLM clients are unified behind `get_client()` which injects the correct base URL and authentication based on environment variables.
*   **Actor Model (Loose):** The `apply/launcher.py` spawns isolated Chrome + Claude pairs that act independently of each other without shared memory (communicating only via DB locks).
*   **Finite State Machine (FSM):** Jobs move linearly through states: `discovered` -> `enriched` -> `scored` -> `tailored` -> `applied`.

## 14. Algorithms & Important Logic

*   **Rate Limit Handling (`llm.py`):** Implements Exponential Backoff. If a 429/503 is hit, it parses the `Retry-After` header or falls back to `wait = 10 * (2 ** attempt)`. Crucial for Gemini's strict free-tier rate limits.
*   **Streaming Pipeline Check (`pipeline.py`):**
    `upstream_done = upstream is None or tracker.is_done(upstream)`
    This logic ensures a thread only terminates when there is absolutely no more data coming from the previous stage *and* its own queue is empty.
*   **Smart Extraction Phase 1 (Heuristics):** Playwright captures `__NEXT_DATA__`, intercepts network XHRs, and evaluates repeating DOM structures, picking the best payload to feed the LLM instead of blindly sending raw HTML.

## 15. External Dependencies & Integrations

*   **Claude Code CLI:** Vital for `apply`. It's a black-box agent provided by Anthropic. We control it via highly specific system prompts and stdin/stdout parsing.
*   **Node.js (`npx @playwright/mcp`)**: Acts as the bridge between Claude (which speaks MCP) and Chrome (which speaks CDP).
*   **sqlite3:** Core Python library, heavily optimized with `PRAGMA journal_mode=WAL` to prevent database locks during multithreading.
*   **python-jobspy:** Bypasses basic Cloudflare/bot protections on major networks.

## 16. Configuration & Build System

*   Configured via `pyproject.toml` (PEP 621 compliant) using `hatchling` backend.
*   Uses `~/.applypilot/` for all persistent data, keeping the user's filesystem clean.
*   Environment variables prioritize `.env` files for API keys.
*   **Build quirk:** Requires a two-step pip install in many cases because `python-jobspy` strictly pins numpy versions which conflicts with pip's resolver.

## 17. Performance & Scalability Considerations

*   **Multithreading:** Bound by Network I/O and DB locks. SQLite in WAL mode handles the concurrency well.
*   **Memory:** Claude Code processes are heavy. Running `applypilot apply -w 4` means 4 Chrome instances + 4 Node Playwright servers + 4 Claude agents. Requires significant RAM.
*   **LLM Context caching:** The prompt injected into Claude is massive. Prompt engineering uses standard structures to hit prompt caching on Anthropic's backend to save costs.

## 18. Security Considerations

*   **Credentials in Prompts:** User PII, phone numbers, and addresses are injected into prompts sent to third-party LLMs. Users must trust Anthropic/Google privacy policies.
*   **Bypass Permissions:** The CLI invokes Claude with `--permission-mode bypassPermissions` allowing it to run Playwright without human confirmation. Highly powerful but restricted to browser interaction via the specific MCP config.
*   **SQL Injection:** The DB layer uses parameterized queries (e.g., `execute("... WHERE url = ?", (url,))`) strictly.

## 19. How to Extend the Project

*   **Add a new Job Board:** Create a new scraper in `src/applypilot/discovery/`, map its output to the standard dictionary format, and hook it into `pipeline.py::_run_discover()`.
*   **Add a new LLM Provider:** Extend `llm.py::_detect_provider()` to parse the new API key, and handle its specific API format if it isn't OpenAI compatible.
*   **Improve Auto-Apply:** Modify `prompt.py` to handle edge-case forms (e.g., complex multi-select dropdowns) better by giving Claude more explicit instructions on how to use Playwright tools or solve new Captchas.

## 20. How to Build a Similar Project from Scratch

If you wanted to build an autonomous pipeline from scratch:

**Phase 1: Foundation (The State Machine)**
1. Define a robust SQLite schema. You need a state machine capable of tracking URLs across stages. Avoid normalized tables; one large flat table (`url`, `score`, `applied_at`) is much easier to manage.
2. Build the LLM Wrapper. Make a resilient HTTP client that handles rate limits.

**Phase 2: Discovery & Enrichment (Getting Data)**
3. Integrate an off-the-shelf scraper (like JobSpy) to populate the DB.
4. Build an HTML parser to grab the full text of URLs.

**Phase 3: The AI Brain (Scoring/Tailoring)**
5. Write the Prompt templates for Scoring and Tailoring. Ensure structured outputs (JSON/Regex parsable) so your python code can ingest the scores back into the DB.

**Phase 4: Agentic Automation (The Hard Part)**
6. Do NOT write custom selenium scripts. Instead, utilize MCP + LLM agents. Setup a local browser, connect an MCP server, and write strict system prompts guiding the agent on how to fill forms and handle CAPTCHAs via APIs (like CapSolver).

**Phase 5: Orchestration (Tying it together)**
7. Build the CLI and a multithreaded orchestrator (like a Blackboard pattern) to tie the database states to the python functions, enabling streaming architectures.

## 21. Learning Roadmap & Knowledge Map

To master the concepts in ApplyPilot, study the following in order:
1. **Concurrency in Python:** Understand `threading`, `ThreadPoolExecutor`, and the Global Interpreter Lock (GIL). (Used heavily in `pipeline.py` & `launcher.py`).
2. **SQLite Advanced Features:** Study `WAL` mode, busy timeouts, and thread-local connections. Understand why `threading.local()` is needed for database connections. (Used in `database.py`).
3. **Agentic Workflows:** Understand how LLMs can use tools. Study ReAct patterns and Anthropic's Model Context Protocol (MCP).
4. **Browser Automation:** Learn Playwright and Chrome DevTools Protocol (CDP).
5. **Prompt Engineering for Code Generation/Navigation:** Look at `prompt.py` to see how you constrain an LLM to follow exact rules, utilize standard formats, and not hallucinate when executing critical actions (like clicking submit).
6. **Data Scraping Intelligence:** Understand JSON-LD, __NEXT_DATA__, and intercepted API requests when dealing with modern React/NextJS frontends (`smartextract.py`).

---
*Generated by your Principal Software Architect.*
