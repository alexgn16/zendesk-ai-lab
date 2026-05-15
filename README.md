# Zendesk AI Service Desk — Project Argus

> An end-to-end AI-powered service desk built on Claude API, FastAPI, pgvector, and Streamlit.  
> Designed as a production-ready portfolio project demonstrating agentic AI integration with real ticketing infrastructure.

---

## What This Project Does

Project Argus turns a standard Zendesk instance into an intelligent service desk agent. When a support ticket arrives, the system automatically:

1. Retrieves the customer's profile and ticket history from Zendesk
2. Searches a vector database of SOPs (Standard Operating Procedures) for relevant context
3. Calls Claude API with all context assembled — using tool calling to gather information agentic-style
4. Generates a draft reply with category, sentiment score, and escalation tag
5. Routes the draft to a human reviewer (HITL) or auto-sends if confidence is high enough (Phase 2)
6. Logs everything to PostgreSQL for QA scoring and reporting

---

## Architecture Overview

```
Zendesk (ticket created)
        ↓
Webhook Listener (FastAPI)
        ↓ [parallel]
┌───────────────────────────────┐
│ customer_profile.py           │  → Zendesk API
│ ticket_history.py             │  → Zendesk API
│ sop_search.py                 │  → pgvector (cosine similarity)
└───────────────────────────────┘
        ↓
agent_loop.py (Claude API — tool calling)
        ↓
ticket_log INSERT (PostgreSQL)
        ↓
confidence ≥ 0.85? ──YES──→ auto-send (Phase 2)
        │
       NO
        ↓
review_interface.py (Streamlit HITL)
  ├── Approve & Send
  ├── Edit & Send
  ├── Escalate
  └── Discard
        ↓
approval_handler.py → Zendesk API (POST comment)
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| AI / LLM | Anthropic Claude API (claude-sonnet-4-20250514) |
| Backend | Python 3.11 + FastAPI + uvicorn |
| Vector DB | pgvector (PostgreSQL extension) |
| Embeddings | sentence-transformers (all-MiniLM-L6-v2, local) |
| Database | PostgreSQL 16 |
| HITL UI | Streamlit |
| Ticketing | Zendesk (webhooks + REST API) |
| Tunnel | ngrok (dev environment) |

---

## Project Phases

### Phase 1 — Shadow Mode (Complete ✅)
- Webhook listener receives Zendesk events
- Agentic loop gathers context and generates draft
- Human reviews every draft before sending
- Full audit log in PostgreSQL

### Phase 2 — Auto-Send + QA Scoring (In Progress 🔄)
- Auto-send for high-confidence drafts (≥ 0.85)
- Blocklist rules: escalation tags, negative sentiment, billing category, risk keywords
- QA scorer triggers on ticket close — Claude evaluates the interaction against a rubric
- Config-driven rules — no code deployment needed for rule changes

### Phase 3 — Reporting Dashboard (Planned)
- React + Tailwind frontend
- QA metrics, team performance, calibration dispute flow
- Supabase Auth with role-based access (agent / QA / admin)
- CSV export, multi-team scoping

---

## Repository Structure

```
zendesk-ai-lab/
├── 04_agent/
│   ├── retrieval/
│   │   ├── customer_profile.py     # Fetch customer profile via Zendesk API
│   │   ├── ticket_history.py       # Last 3 tickets for context
│   │   └── sop_search.py           # pgvector cosine similarity search
│   ├── agent/
│   │   ├── tools.py                # Claude tool definitions (5 tools)
│   │   └── agent_loop.py           # Agentic loop with tool calling
│   ├── hitl/
│   │   ├── webhook_listener.py     # FastAPI — receives Zendesk events
│   │   ├── review_interface.py     # Streamlit — human review UI
│   │   └── approval_handler.py     # Posts approved reply to Zendesk
│   └── qa/
│       ├── qa_scorer.py            # Automatic QA scoring (Phase 2)
│       └── qa_dashboard.py         # QA metrics dashboard (Phase 3)
├── scripts/
│   └── seed_sop.py                 # Embeds SOP documents into pgvector
├── data/
│   ├── sop_docs/                   # SOP source documents (markdown)
│   └── mock/                       # Mock customer and ticket data
├── config.py                       # Centralised config with validation
├── database.py                     # DB connection pool
├── ARCHITECTURE.md                 # System design and decisions
├── DECISIONS.md                    # Architecture Decision Records
├── SETUP.md                        # How to run locally
├── .env.example                    # Environment variable template
└── pytest.ini
```

---

## Key Design Decisions

- **pgvector over a dedicated vector DB** — production-ready, no extra vendor, SOP search runs in the same Postgres instance as the ticket log
- **Local embeddings** — sentence-transformers runs on-device; no embedding API cost, works offline
- **Config-driven rules** — blocklist, confidence threshold, and QA weights live in `config.yaml`, not in code
- **Idempotent webhook** — applies `argus_processed` tag via Zendesk API on first receipt; duplicate triggers are rejected before any processing
- **BackgroundTasks** — webhook returns HTTP 200 immediately; Claude processing runs async so Zendesk never times out

Full decision log in [DECISIONS.md](./DECISIONS.md).

---

## Running Locally

See [SETUP.md](./SETUP.md) for the complete setup guide.

**Quick start:**

```zsh
# 1. Activate environment
source ~/argus-venv/bin/activate
cd ~/Documents/zendesk-ai-lab

# 2. Start services (3 terminal tabs)
uvicorn 04_agent/hitl/webhook_listener:app --reload --port 8000   # Tab 1
ngrok http 8000                                                     # Tab 2
streamlit run 04_agent/hitl/review_interface.py --server.port 8501 # Tab 3
```

---

## Environment Variables

Copy `.env.example` to `.env` and fill in your values.

```dotenv
ZENDESK_SUBDOMAIN=your_subdomain
ZENDESK_EMAIL=your_email
ZENDESK_API_TOKEN=your_token
ANTHROPIC_API_KEY=sk-ant-...
MODEL=claude-sonnet-4-20250514
DATABASE_URL=postgresql://localhost/argus_db
EMBEDDING_MODEL=all-MiniLM-L6-v2
WEBHOOK_SECRET=your_secret
```

---

## License

MIT — built as a portfolio project demonstrating AI integration with enterprise ticketing systems.
