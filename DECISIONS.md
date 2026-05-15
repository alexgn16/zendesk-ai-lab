# DECISIONS.md — Architecture Decision Records
# Project Argus — AI Service Desk

> This file is the single source of truth for every technical decision made during development.
> Every session reads this file before writing code. Every significant decision gets logged here.
> Format: context → decision → consequences → status.

---

## Index

| ID | Decision | Session | Status |
|---|---|---|---|
| ADR-001 | Database: argus_db (not zendesk_ai_lab) | #1 | ✅ Active |
| ADR-002 | venv location: ~/argus-venv (not HD external) | #1 | ✅ Active |
| ADR-003 | Project path: ~/Documents/zendesk-ai-lab | #1 | ✅ Active |
| ADR-004 | Vector DB: pgvector (not dedicated vector DB) | #1 | ✅ Active |
| ADR-005 | Embeddings: sentence-transformers local (not API) | #1 | ✅ Active |
| ADR-006 | HITL UI: Streamlit (not custom React for Phase 1) | #1 | ✅ Active |
| ADR-007 | pgvector query pattern: psycopg2 + ORDER BY alias | #3 | ✅ Active |
| ADR-008 | agent_loop output schema (fields returned) | #4 | ✅ Active |
| ADR-009 | Webhook auth: X-Argus-Secret header (not HMAC) | #5→#6 | ✅ Active |
| ADR-010 | BackgroundTasks: webhook returns 200 immediately | #5 | ✅ Active |
| ADR-011 | Idempotency: argus_processed tag via Zendesk API | #9 | ✅ Active |
| ADR-012 | MODEL string: claude-sonnet-4-20250514 | #4 | ✅ Active |
| ADR-013 | confidence field: NOT YET IMPLEMENTED | #10 | ⚠️ Pending |
| ADR-014 | Phase 2 blocklist criteria | #10 | 📋 Decided, not built |
| ADR-015 | QA rubric: 5 criteria, config-driven weights | #10 | 📋 Decided, not built |

---

## ADR-001 — Database name: argus_db

**Context:** README originally specified `zendesk_ai_lab` as the database name.

**Decision:** Use `argus_db` — renamed during Session #1 to separate the Argus agent from the earlier lab experiments.

**Consequences:** All connection strings, `.env`, and runbook reference `argus_db`. Never use `zendesk_ai_lab` for new code.

**Status:** ✅ Active

---

## ADR-002 — venv location: ~/argus-venv

**Context:** README originally placed the venv on the external HD (`/Volumes/TOSHIBA EXT/`).

**Decision:** venv lives at `~/argus-venv` (home directory, internal SSD).

**Reason:** exFAT filesystem on the Toshiba HD does not support symlinks — Python venvs require symlinks and will corrupt silently on exFAT.

**Consequences:** Always activate with `source ~/argus-venv/bin/activate`. Never create or move venv to the external HD.

**Status:** ✅ Active

---

## ADR-003 — Project path: ~/Documents/zendesk-ai-lab

**Context:** README originally specified `/Volumes/TOSHIBA EXT/zendesk-ai-lab`.

**Decision:** Project lives at `~/Documents/zendesk-ai-lab`.

**Reason:** Same exFAT symlink issue. Code runs faster on internal SSD.

**Consequences:** All `cd` commands and path references use `~/Documents/zendesk-ai-lab`.

**Status:** ✅ Active

---

## ADR-004 — Vector DB: pgvector (PostgreSQL extension)

**Context:** Options considered: Pinecone, Weaviate, Qdrant, pgvector.

**Decision:** pgvector via Homebrew (`brew install pgvector`), running inside the same `argus_db` PostgreSQL instance.

**Reason:** No extra vendor, no extra cost, production-ready, SOP search and ticket log share one DB connection pool.

**Consequences:** Embeddings stored as `vector(384)` column in `sop_documents`. Index type: `ivfflat` with `lists=100`.

**Status:** ✅ Active

---

## ADR-005 — Embeddings: sentence-transformers local

**Context:** Options: OpenAI `text-embedding-ada-002`, Anthropic, local sentence-transformers.

**Decision:** `all-MiniLM-L6-v2` via `sentence-transformers` library, runs entirely local.

**Reason:** Zero embedding API cost. Works offline. 384-dimension vectors are fast for small SOP corpora.

**Consequences:** Model loaded once at module level in `sop_search.py` to avoid reload on every call. Vector dimension is fixed at 384 — changing the model requires re-embedding all SOPs.

**Status:** ✅ Active

---

## ADR-006 — HITL UI: Streamlit (Phase 1)

**Context:** Options: React frontend, Flask + HTML, Streamlit.

**Decision:** Streamlit for Phase 1 HITL review interface (`review_interface.py`).

**Reason:** Fast to build, sufficient for demo and Phase 1 validation. React is planned for Phase 3 dashboard.

**Consequences:** Streamlit does NOT auto-refresh when new tickets arrive — user must click Refresh or reload the page. This is acceptable for Phase 1. Phase 2 will add `st.rerun()` with polling.

**Port:** 8501.

**Status:** ✅ Active (Phase 1 only — React replaces in Phase 3)

---

## ADR-007 — pgvector query pattern

**Context:** During Session #3, two query patterns were tested for cosine similarity search.

**Decision:** Use `psycopg2` (not `asyncpg`) with `register_vector`, `autocommit=True`, and `ORDER BY` on a named alias — not directly on the expression.

**Working pattern:**
```python
cur.execute("""
    SELECT title, content, category,
           1 - (embedding <=> %s::vector) AS sim
    FROM sop_documents
    ORDER BY sim DESC
    LIMIT %s
""", (embedding.tolist(), limit))
```

**Reason:** `asyncpg` with pgvector requires additional adapter setup that caused failures in Session #3. `psycopg2` with `register_vector` worked reliably.

**Consequences:** `sop_search.py` uses synchronous psycopg2 connection, not asyncpg. This is acceptable because SOP search runs inside the background task, not in the async request handler.

**Status:** ✅ Active

---

## ADR-008 — agent_loop.py output schema

**Context:** Claude's agentic loop must return structured data that gets written to `ticket_log`.

**Decision:** Claude is prompted to return a JSON object with these fields:

```json
{
  "category": "billing | auth | bug_report | onboarding | cancellation | other",
  "sentiment_score": 1,
  "escalation_tag": "critical | billing_dispute | legal_threat | null",
  "claude_draft": "The full reply text here."
}
```

**⚠️ MISSING FIELD — confidence NOT implemented:**
The `confidence` field (float 0.0–1.0) was planned for Phase 2 auto-send but was **never added** to the Claude prompt or the `ticket_log` schema. This must be implemented in Session #11 before any Phase 2 code is written.

**Consequences:** `ticket_log` currently has no `confidence` column. Phase 2 cannot proceed without:
1. Adding `confidence NUMERIC(3,2)` to `ticket_log` schema
2. Adding `confidence` to the Claude system prompt output spec
3. Adding `auto_sent BOOLEAN DEFAULT FALSE` to `ticket_log` schema

**Status:** ✅ Partially active — confidence field pending (ADR-013)

---

## ADR-009 — Webhook authentication: X-Argus-Secret

**Context:** Zendesk supports HMAC-SHA256 webhook signatures. Initial plan was to implement full HMAC verification.

**Decision:** Use a simpler custom header `X-Argus-Secret` with a shared secret stored in `.env` as `WEBHOOK_SECRET`.

**Reason:** HMAC-SHA256 implementation was planned in Session #5 but the Zendesk trigger configuration was done with custom header auth — this is what was validated end-to-end.

**Consequences:** `webhook_listener.py` checks `request.headers.get("X-Argus-Secret")` against `os.getenv("WEBHOOK_SECRET")`. Returns 403 if mismatch. The `WEBHOOK_SECRET` value in `.env` must match exactly (case-sensitive) with the header value configured in Zendesk Admin Center → Webhooks.

**Status:** ✅ Active

---

## ADR-010 — BackgroundTasks: webhook returns 200 immediately

**Context:** Zendesk webhooks time out if the endpoint does not respond within ~5 seconds. Claude processing takes 10–30 seconds.

**Decision:** `webhook_listener.py` uses FastAPI `BackgroundTasks`. The endpoint validates the secret, enqueues the agent task, and returns HTTP 200 immediately. Claude runs in background.

**Consequences:** Errors in the agent loop do NOT propagate to Zendesk. All errors must be caught and logged within the background task. The ticket will not appear in Streamlit if the agent crashes silently — check uvicorn logs.

**Status:** ✅ Active

---

## ADR-011 — Idempotency: argus_processed tag

**Context:** The Zendesk trigger was firing twice per ticket (Sessions #7–#9), generating duplicate `ticket_log` entries.

**Decision:** `webhook_listener.py` calls the Zendesk API to:
1. Check if the ticket already has the `argus_processed` tag
2. If yes: return `{"status": "duplicate"}` immediately, no processing
3. If no: apply the tag, then enqueue the background task

**Zendesk Trigger condition added:** "Tags — does not contain — argus_processed"

**Consequences:** Every ticket is processed exactly once. If the tag application fails (Zendesk API error), the ticket may be processed twice — acceptable edge case for Phase 1.

**Status:** ✅ Active — validated with ticket #24 (log_id=13)

---

## ADR-012 — MODEL string

**Context:** Multiple model string formats were attempted across sessions.

**Decision:** Use `claude-sonnet-4-20250514` — the exact API string.

**History:** `claude-sonnet-4-6` was used in early sessions and caused deprecation warnings. Fixed in Session #4 via:
```zsh
sed -i '' 's/MODEL=claude-sonnet-4-6/MODEL=claude-sonnet-4-20250514/' ~/.env
```

**Consequences:** `.env` must have `MODEL=claude-sonnet-4-20250514`. Any other string will cause an API error.

**Status:** ✅ Active

---

## ADR-013 — confidence field (PENDING)

**Context:** Phase 2 auto-send requires a `confidence` score (0.0–1.0) from Claude to decide whether to auto-send or route to HITL.

**Decision:** confidence will be added to the Claude prompt output spec and to `ticket_log` schema in Session #11.

**Implementation plan:**
- Add to Claude system prompt: Claude must return `"confidence": 0.85` in its JSON output
- Schema migration: `ALTER TABLE ticket_log ADD COLUMN confidence NUMERIC(3,2);`
- Schema migration: `ALTER TABLE ticket_log ADD COLUMN auto_sent BOOLEAN DEFAULT FALSE;`
- Threshold: `AUTO_SEND_CONFIDENCE_THRESHOLD=0.85` (already in `.env`)

**Status:** ⚠️ Decided, not implemented — Session #11 first task

---

## ADR-014 — Phase 2 blocklist criteria

**Context:** Even with high confidence, some tickets must always go through HITL.

**Decision:** A ticket is blocklisted (forced to HITL regardless of confidence) if ANY of the following are true:
- `escalation_tag IS NOT NULL`
- `sentiment_score <= 2`
- `category = 'billing'`
- Subject or body contains risk keywords (configurable list in `config.yaml`)

**Risk keywords (initial list):** `lawyer`, `sue`, `lawsuit`, `refund`, `chargeback`, `fraud`, `scam`, `attorney`

**Config-driven:** blocklist lives in `config.yaml`, not hardcoded. Changes take effect without redeployment.

**Status:** 📋 Decided in Session #10, not yet implemented

---

## ADR-015 — QA rubric: 5 criteria

**Context:** Phase 2 QA scorer needs a rubric to evaluate agent interactions after ticket close.

**Decision:** 5 criteria scored 1–10, weighted average = overall_score:

| Criterion | Weight | What it measures |
|---|---|---|
| accuracy | 30% | Did the reply follow the relevant SOP? |
| tone | 20% | Was the tone appropriate for the customer's sentiment? |
| resolution | 25% | Was the issue fully addressed? |
| compliance | 15% | Were any policies violated? |
| response_time | 10% | Was the SLA met? |

**Weights are config-driven** — stored in `config.yaml`, not hardcoded.

**Claude returns:**
```json
{
  "overall_score": 8.5,
  "scorecard": {
    "accuracy":      {"score": 9, "note": "SOP followed correctly"},
    "tone":          {"score": 8, "note": "Friendly but slightly formal"},
    "resolution":    {"score": 9, "note": "Issue fully addressed"},
    "compliance":    {"score": 8, "note": "No policy violations"},
    "response_time": {"score": 8, "note": "Within SLA"}
  },
  "coaching_note": "Consider a warmer opening for upset customers."
}
```

**Trigger:** Zendesk trigger fires when `ticket status = solved`, POSTs to `/webhook/zendesk` with `"event": "ticket_solved"`.

**Status:** 📋 Decided in Session #10, not yet implemented

---

## How to Add a New Decision

When a significant technical choice is made during a session, add an entry here before closing the session:

```markdown
## ADR-XXX — Short title

**Context:** What problem or question prompted this decision?

**Decision:** What was decided.

**Reason:** Why this option over alternatives.

**Consequences:** What this means for future code. What to never do.

**Status:** ✅ Active | ⚠️ Pending | 📋 Decided, not built | ❌ Superseded
```

---

*Last updated: Session #10 — 2026-05-16*
