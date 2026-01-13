# Ambient Email Assistant (LangGraph + Gmail + LangSmith)

A personal ambient email assistant built with LangGraph that ingests Gmail messages, routes them through a triage workflow, and logs full traces to LangSmith for observability and debugging.

This repository contains a local-friendly reference implementation that includes:
- Gmail OAuth token storage and helpers
- A Gmail ingestion script that pushes emails into LangGraph threads
- A triage + response LangGraph workflow
- LangSmith tracing for run visibility (graph + node-level traces)

---

Table of contents
- [Features](#-features)
- [Architecture Overview](#-architecture-overview)
- [Project Structure](#-project-structure-high-level)
- [Prerequisites](#-prerequisites)
- [Environment Setup](#-environment-setup)
- [Install](#-install)
- [Run Locally](#-run-locally)
- [Proof it works (what to look for)](#-proof-it-works-what-to-look-for)
- [Common Issues](#-common-issues)
- [Roadmap / Improvements](#-roadmap--improvements)

---

✨ Features

- ✅ Gmail ingestion (fetch unread emails or include read)
- ✅ Creates a stable LangGraph thread per Gmail thread (deterministic mapping)
- ✅ Triage routing node to classify emails (ignore / respond / human-in-the-loop)
- ✅ Optional mark-as-read flow (tool-driven)
- ✅ LangSmith traces for every run (inspect inputs, decisions, failures)
- ✅ Runs locally using `langgraph dev`

---

🧠 Architecture Overview

Ingestion → Graph

1. `run_ingestion.py` fetches recent emails from Gmail.
2. Each Gmail thread ID is mapped to a deterministic UUID → LangGraph thread.
3. Each email becomes a run on the configured graph. Typical graph nodes:
   - `triage_router` — classify the email (ignore, respond, escalate to HITL)
   - `triage_interrupt_handler` — optional human-in-the-loop handler
   - `response_agent` — agent that drafts or sends responses
   - `mark_as_read_node` — optional tool-driven mark-as-read

Why deterministic thread IDs?
Email threads in Gmail are stable; mapping them to stable LangGraph thread IDs preserves conversation context across runs and makes retries and debugging simpler.

---

📁 Project Structure (high level)

Example tree (paths are relative to repo root):

```text
src/email_assistant/
├── agent.py                # LangGraph workflow definition
├── configuration.py       # config helpers (env, settings)
├── prompts.py             # prompts used by agents
├── schemas.py             # state/payload schemas
├── tools/
│   └── gmail/
│       ├── gmail_tools.py    # Gmail tool implementations (fetch, mark-as-read, etc.)
│       ├── run_ingestion.py  # fetch emails and create runs on LangGraph server
│       └── setup_gmail.py    # OAuth setup helper (if used)
.run_ingestion.py (examples shown in docs)
.secrets/                   # LOCAL ONLY (token.json, secrets.json) - DO NOT COMMIT
```

Important: The `.secrets/` directory is for local OAuth tokens only. Never commit tokens or secrets.

---

✅ Prerequisites

- Python 3.10+ (3.11 recommended)
- A Gmail account and OAuth consent configured
- OpenAI API key (or whichever model provider your graph uses)
- LangSmith account (optional but recommended for tracing)

---

🔐 Environment Setup

Create a `.env` file at the repo root (example):

```bash
# Model provider
OPENAI_API_KEY=your_key_here

# LangSmith tracing (recommended)
LANGSMITH_API_KEY=your_langsmith_key
LANGSMITH_TRACING=true
LANGSMITH_PROJECT="Email Tool Calling and Response Evaluation"

# Optional
EMAIL_ADDRESS=your_email@gmail.com
```


---

📦 Install

Create and activate a virtual environment:

```bash
python -m venv .venv
# Windows:
.venv\Scripts\activate
# macOS / Linux:
source .venv/bin/activate
```

Install dependencies:

```bash
pip install langgraph langgraph-cli langgraph-sdk python-dotenv google-api-python-client google-auth google-auth-oauthlib html2text
```

---

▶️ Run Locally

1) Start LangGraph dev server (run from repo root):

```bash
langgraph dev
```

If your repo uses a config file, ensure `langgraph.json` exists at root or pass `--config`.

2) Ingest Gmail emails into the graph

Ingest emails from a specified recent window (example: last 1000 minutes):

```bash
python src/email_assistant/tools/gmail/run_ingestion.py \
  --email your_email@gmail.com \
  --minutes-since 1000 \
  --url http://127.0.0.1:2024 \
  --graph-name email_assistant_hitl_memory_gmail
```

Process only a single email (good for testing):

```bash
python src/email_assistant/tools/gmail/run_ingestion.py \
  --email your_email@gmail.com \
  --minutes-since 60 \
  --early \
  --url http://127.0.0.1:2024 \
  --graph-name email_assistant_hitl_memory_gmail
```

Notes:
- `--early` often makes the ingestion stop after the first processed email — useful for debugging and to avoid rate limits.


---

🧪 Proof It Works (what to look for)
- Terminal output showing: “Run created successfully… Processed X emails…”
<img width="940" height="654" alt="image" src="https://github.com/user-attachments/assets/d80c1046-8ab0-4083-a5d8-e6af2bc79e7a" />

- LangSmith trace view showing:
  - Email input
  - `triage_router` decision
  - Response or ignore outcome
  - Node-level execution steps
<img width="940" height="631" alt="image" src="https://github.com/user-attachments/assets/cfac01e4-42fa-4860-b970-805a8e7aff29" />


---

⚠️ Common Issues

- RateLimitError (429) from model provider:
  - Caused by many runs, large email bodies (newsletters), or parallel runs.
  - Quick workarounds:
    - Run with `--early` to process a single email.
    - Lower `--minutes-since`.
    - Add throttling between runs (future improvement).

---

🗺️ Roadmap / Improvements (ideas)

- Throttle ingestion to avoid provider rate limits
- Summarize / strip long HTML newsletters before triage
- Add deduplication (skip already processed email IDs)
- Add a dashboard view and memory policy
- Add a human-in-the-loop review UI for “important” triage outcomes


---

## References & Acknowledgements

This project was inspired by and builds upon ideas from the following excellent resources:

- **LangChain – Agents from Scratch (GitHub Repository)**  
  https://github.com/langchain-ai/agents-from-scratch  
  Reference implementation for building agent workflows, tool integration, and memory using LangGraph.

- **LangChain Academy – Ambient Agents with LangGraph (Course)**  
  https://academy.langchain.com/courses/ambient-agents  
  Course material that demonstrates how to design ambient email agents with human-in-the-loop workflows and LangSmith-based observability.


All implementation decisions, integrations, and extensions in this repository reflect original development work.
