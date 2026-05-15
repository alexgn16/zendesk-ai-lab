# ARCHITECTURE.md — Project Argus
# System Design Reference

> This document describes the technical architecture of Project Argus.
> For the decision log behind each choice, see [DECISIONS.md](./DECISIONS.md).
> For setup instructions, see [SETUP.md](./SETUP.md).

---

## System Context

Project Argus is a middleware layer between Zendesk and the Anthropic Claude API.
It intercepts incoming support tickets, assembles context from multiple sources,
generates AI-drafted replies, and manages the human-in-the-loop review process.

```
┌─────────────────────────────────────────────────────────────┐
│                        ZENDESK                               │
│  Customer emails → Ticket created → Trigger fires → Webhook │
└─────────────────────────────┬───────────────────────────────┘
                              │ POST /webhook/zendesk
                              │ Header: X-Argus-Secret
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     PROJECT ARGUS                            │
│                                                              │
│  webhook_listener (FastAPI)                                  │
│         ↓ BackgroundTask (returns 200 immediately)           │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Context Assembly (parallel)              │   │
│  │  customer_profile.py  →  Zendesk Users API           │   │
│  │  ticket_history.py    →  Zendesk Tickets API         │   │
│  │  sop_search.py        →  pgvector (PostgreSQL)       │   │
│  └──────────────────────────┬───────────────────────────┘   │
│                             ↓                                │
│  agent_loop.py (Claude API — tool calling)                   │
│         ↓                                                    │
│  ticket_log INSERT (PostgreSQL — argus_db)                   │
│         ↓                                                    │
│  confidence ≥ 0.85? ──YES──→ approval_handler (auto-send)   │
│         │                                                    │
│        NO                                                    │
│         ↓                                                    │
│  review_interface.py (Streamlit — HITL)                      │
│         ↓                                                    │
│  approval_handler.py → Zendesk API (POST comment)            │
└─────────────────────────────────────────────────────────────┘
```

---

## Component Reference

### webhook_listener.py
**Path:** `04_agent/hitl/webhook_listener.py`  
**Framework:** FastAPI + uvicorn  
**Port:** 8000

Responsibilities:
- Receives POST from Zendesk trigger
- Validates `X-Argus-Secret` header (returns 403 on mismatch)
- Checks idempotency: calls Zendesk API to verify `argus_processed` tag not already present
- Applies `argus_processed` tag to prevent duplicate processing
- Enqueues `run_agent()` as a FastAPI BackgroundTask
- Returns HTTP 200 immediately (before Claude processing begins)
- Routes `event=ticket_solved` to `qa_scorer.py` (Phase 2)

Endpoints:
```
GET  /health              → {"status": "ok"}
POST /webhook/zendesk     → {"status": "accepted", "ticket_id": N}
```

---

### agent_loop.py
**Path:** `04_agent/agent/agent_loop.py`  
**Model:** claude-sonnet-4-20250514

Responsibilities:
- Receives assembled context (profile, history, SOP chunks)
- Runs Claude with tool calling (agentic loop)
- Extracts structured output from final Claude message
- Writes to `ticket_log`

**Tools available to Claude:**

| Tool | Description |
|---|---|
| `get_customer_profile` | Fetches customer tier, plan, account age |
| `get_ticket_history` | Last 3 tickets from this customer |
| `search_sop` | Vector search across SOP documents |
| `stage_draft` | Signals Claude is ready to output the draft |
| `escalate` | Marks ticket for human escalation |

**Claude output schema:**
```json
{
  "category": "billing | auth | bug_report | onboarding | cancellation | other",
  "sentiment_score": 1,
  "escalation_tag": "critical | billing_dispute | legal_threat | null",
  "confidence": 0.87,
  "claude_draft": "Full reply text."
}
```

⚠️ `confidence` field: planned for Phase 2, implementation in Session #11.

---

### sop_search.py
**Path:** `04_agent/retrieval/sop_search.py`

Performs cosine similarity search over SOP documents stored in pgvector.

```python
# Query pattern (validated Session #3)
SELECT title, content, category,
       1 - (embedding <=> %s::vector) AS sim
FROM sop_documents
ORDER BY sim DESC
LIMIT %s
```

Returns top-N chunks (default: 3, configurable via `MAX_SOP_CHUNKS`).

---

### customer_profile.py
**Path:** `04_agent/retrieval/customer_profile.py`

Fetches customer data from Zendesk Users API by email. Returns tier, plan, and account metadata for inclusion in the Claude system prompt.

---

### ticket_history.py
**Path:** `04_agent/retrieval/ticket_history.py`

Fetches the last 3 closed tickets from the same customer. Provides Claude with historical context to avoid repeating solutions or detect recurring issues.

---

### review_interface.py
**Path:** `04_agent/hitl/review_interface.py`  
**Framework:** Streamlit  
**Port:** 8501

Displays pending tickets from `ticket_log WHERE human_action = 'pending_review'`.

Actions:
- **Approve & Send** → calls `approval_handler.post_approved_reply()`
- **Edit & Send** → editable text area → `approval_handler.post_approved_reply()`
- **Escalate** → `approval_handler.post_escalation_note()` + updates `ticket_log`
- **Discard** → `approval_handler.discard_ticket()` + updates `ticket_log`

Note: Does not auto-refresh. User must click Refresh to see new tickets (Phase 1 limitation).

---

### approval_handler.py
**Path:** `04_agent/hitl/approval_handler.py`

Posts replies to Zendesk via REST API. Updates `ticket_log.human_action` and `ticket_log.final_reply` after each action.

---

### qa_scorer.py *(Phase 2 — not yet implemented)*
**Path:** `04_agent/qa/qa_scorer.py`

Triggered by `event=ticket_solved` webhook. Fetches the full interaction from `ticket_log`, calls Claude with a QA rubric prompt, and writes results to `qa_scores`.

---

## Database Schema

**Database:** `argus_db` (PostgreSQL 16 + pgvector 0.7.x)

```sql
-- SOP documents for vector search
CREATE TABLE sop_documents (
    id          SERIAL PRIMARY KEY,
    title       TEXT NOT NULL,
    content     TEXT NOT NULL,
    category    TEXT,
    embedding   vector(384),           -- all-MiniLM-L6-v2 dimensions
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE INDEX ON sop_documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Full audit log of every processed ticket
CREATE TABLE ticket_log (
    id                  SERIAL PRIMARY KEY,
    ticket_id           INTEGER NOT NULL,
    received_at         TIMESTAMP DEFAULT NOW(),
    customer_email      TEXT,
    category            TEXT,
    sentiment_score     INTEGER,
    escalation_tag      TEXT,
    confidence          NUMERIC(3,2),          -- Phase 2 (pending ADR-013)
    auto_sent           BOOLEAN DEFAULT FALSE, -- Phase 2 (pending ADR-013)
    sop_chunks_used     TEXT[],
    claude_draft        TEXT,
    system_prompt       TEXT,
    tool_calls          JSONB,
    human_action        TEXT,                  -- pending_review | approved | edited | escalated | discarded
    final_reply         TEXT,
    processing_time_ms  INTEGER,
    tokens_used         INTEGER
);

-- QA scores per ticket (Phase 2)
CREATE TABLE qa_scores (
    id              SERIAL PRIMARY KEY,
    ticket_log_id   INTEGER REFERENCES ticket_log(id),
    scored_at       TIMESTAMP DEFAULT NOW(),
    overall_score   NUMERIC(3,1),
    scorecard       JSONB,
    coaching_note   TEXT,
    disputed        BOOLEAN DEFAULT FALSE,
    dispute_note    TEXT,
    final_score     NUMERIC(3,1)
);
```

---

## Configuration

All tuneable parameters live in `config.yaml` (Phase 2) or `.env`:

```yaml
# config.yaml (Phase 2)
auto_send:
  confidence_threshold: 0.85
  blocklist:
    escalation_tag_any: true
    max_sentiment_score: 2
    blocked_categories:
      - billing
    risk_keywords:
      - lawyer
      - sue
      - lawsuit
      - refund
      - chargeback
      - fraud
      - scam

qa_rubric:
  weights:
    accuracy: 0.30
    tone: 0.20
    resolution: 0.25
    compliance: 0.15
    response_time: 0.10
```

---

## Data Flow — Timing

| Step | Expected time |
|---|---|
| Zendesk → webhook received | < 5 seconds |
| Secret validation + tag check | < 1 second |
| Context assembly (parallel) | 3–8 seconds |
| Claude draft generation | 10–30 seconds |
| ticket_log INSERT | < 1 second |
| Total to Streamlit queue | ≤ 60 seconds |

---

## Environment

| Item | Value |
|---|---|
| macOS | Apple Silicon (M-series) |
| Python | 3.11.x |
| venv | `~/argus-venv` |
| Project | `~/Documents/zendesk-ai-lab` |
| Database | `argus_db` |
| uvicorn port | 8000 |
| Streamlit port | 8501 |
| ngrok hostname | `clambake-kiwi-barber.ngrok-free.dev` |

---

*Last updated: Session #10 — 2026-05-16*
