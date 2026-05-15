# SETUP_RUNBOOK — Project Argus
*Última atualização: 2026-05-15 — Sessão #7*

> **Este runbook reflete o ambiente real validado nas sessões #1–#6.**
> Não use o README original — os paths, nomes de venv e banco foram alterados.

---

## 1. Ambiente de Desenvolvimento

| Item | Valor real (validado) |
|---|---|
| macOS | Apple Silicon (M-series) |
| Python | 3.11.x |
| PostgreSQL | 16 (via Homebrew) |
| pgvector | 0.7.x (via Homebrew) |
| venv | `~/argus-venv` |
| Projeto | `~/Documents/zendesk-ai-lab` |
| Banco | `argus_db` |
| Porta uvicorn | 8000 |
| Porta Streamlit | 8501 |
| ngrok hostname | `your-hostname.ngrok-free.dev` |

> ⚠️ venv e projeto ficam em `~` (home), nunca no HD externo — exFAT não suporta symlinks.

---

## 2. PostgreSQL 16 + pgvector

```zsh
psql --version
brew services list | grep postgresql
brew services start postgresql@16
```

---

## 3. Banco de Dados argus_db

```zsh
psql -l | grep argus_db
createdb argus_db
psql argus_db -c "CREATE EXTENSION IF NOT EXISTS vector;"
psql argus_db -c "\dt"
```

---

## 4. Python venv (argus-venv)

```zsh
source ~/argus-venv/bin/activate
cd ~/Documents/zendesk-ai-lab
python -c "import anthropic, fastapi, uvicorn, psycopg2, streamlit, httpx; print('ALL OK')"
```

### Instalar dependências (somente se necessário)

```zsh
pip install anthropic fastapi "uvicorn[standard]" psycopg2-binary sqlalchemy \
    asyncpg pgvector sentence-transformers numpy tiktoken streamlit httpx python-dotenv
```

---

## 5. Iniciar o Sistema (Ordem Correta)

Abrir 3 abas de terminal:

```zsh
# Aba 1 — Webhook listener
source ~/argus-venv/bin/activate
cd ~/Documents/zendesk-ai-lab/04_agent/hitl
uvicorn webhook_listener:app --reload --port 8000

# Aba 2 — ngrok
ngrok http --hostname=your-hostname.ngrok-free.dev 8000

# Aba 3 — Streamlit HITL
source ~/argus-venv/bin/activate
cd ~/Documents/zendesk-ai-lab/04_agent/hitl
streamlit run review_interface.py --server.port 8501
```

---

## 6. Variáveis de Ambiente (.env)

```dotenv
ZENDESK_SUBDOMAIN=your_subdomain
ZENDESK_EMAIL=your_email@company.com
ZENDESK_API_TOKEN=your_token
ANTHROPIC_API_KEY=sk-ant-xxxx
MODEL=claude-sonnet-4-20250514
DATABASE_URL=postgresql://localhost/argus_db
EMBEDDING_MODEL=all-MiniLM-L6-v2
MAX_SOP_CHUNKS=3
DRAFT_TIMEOUT_SECONDS=180
AUTO_SEND_CONFIDENCE_THRESHOLD=0.85
WEBHOOK_SECRET=your_secret
WEBHOOK_PORT=8000
LOG_LEVEL=INFO
```

---

## 7. Zendesk — Trigger e Webhook

**Webhook:**
- Endpoint: `https://your-hostname.ngrok-free.dev/webhook/zendesk`
- Method: POST
- Auth header: `X-Argus-Secret: [valor do WEBHOOK_SECRET]`

**Trigger "Argus — New Ticket":**
- Condition: Ticket is created + Tags does not contain `argus_processed`
- Action: Notify webhook → JSON body:

```json
{
  "ticket_id": "{{ticket.id}}",
  "subject": "{{ticket.title}}",
  "description": "{{ticket.description}}",
  "requester_email": "{{ticket.requester.email}}",
  "requester_name": "{{ticket.requester.name}}",
  "priority": "{{ticket.priority}}",
  "status": "{{ticket.status}}"
}
```

---

## 8. Verificações de Sanidade

```zsh
# PostgreSQL
psql argus_db -c "SELECT COUNT(*) FROM ticket_log;"

# Python
source ~/argus-venv/bin/activate
python -c "import anthropic, fastapi, streamlit, psycopg2, httpx; print('ALL OK')"

# Webhook
curl http://localhost:8000/health

# Teste manual
curl -X POST http://localhost:8000/webhook/zendesk \
  -H "Content-Type: application/json" \
  -H "X-Argus-Secret: SEU_SECRET_AQUI" \
  -d '{"ticket_id": 9999, "subject": "Test", "description": "Test ticket",
       "requester_email": "test@example.com", "requester_name": "Test User",
       "priority": "normal", "status": "new"}'
```

---

## 9. Troubleshooting Rápido

| Erro | Causa | Fix |
|---|---|---|
| 403 Forbidden | X-Argus-Secret não bate | Comparar .env com Zendesk Admin Center |
| Ticket duplicado no banco | Trigger duplicado ativo | Desativar trigger extra no Zendesk |
| `uvicorn: command not found` | venv não ativado | `source ~/argus-venv/bin/activate` |
| `could not connect to server` | PostgreSQL parado | `brew services start postgresql@16` |
| ngrok 502/504 | uvicorn não está rodando | Iniciar uvicorn na porta 8000 |
| Streamlit não atualiza | Comportamento esperado (Phase 1) | Clicar em Refresh manualmente |

---

## 10. Comandos de Referência Rápida

```zsh
source ~/argus-venv/bin/activate
cd ~/Documents/zendesk-ai-lab
psql argus_db -c "SELECT id, ticket_id, category, human_action FROM ticket_log ORDER BY id DESC LIMIT 10;"
brew services start postgresql@16
curl http://localhost:8000/health
```

---

*SETUP_RUNBOOK.md — Project Argus — Sessão #10 — 2026-05-16*
