# MGMG Digital Command Center

Central AI operating system for MGMG (Tashkent) — connects SAP Business One,
amoCRM, Verifix, Microsoft 365 and Telegram into one hub where the CEO sees
everything and agents handle the repetitive work.

**Divisions:** Armin · IMUS-Alliance · ONDRY · Service center · Properties

## Status — Month 1

| # | Deliverable | State |
| - | ----------- | ----- |
| 1 | Infrastructure (n8n + PostgreSQL + API on VPS) | built, needs credentials |
| 2 | Teams Command Center (13 channels + Planner) | script ready, not run |
| 3 | Agent 1 — CEO Daily Brief | built, needs credentials |
| 4 | Agent 2 — amoCRM Follow-up | built, write gate closed |
| 5 | Agent 3 — Receivables | built, needs credentials |
| 6 | Power BI Dashboard v1 | queries + DAX ready, report not built |

Nothing has been run against a live system yet — every `[PLACEHOLDER]` in
`.env` must be filled first. Each agent refuses to start while placeholders
remain.

## Architecture

```
SAP B1 ─┐
amoCRM ─┼─→ Python integration clients ─→ PostgreSQL ─→ Power BI (CEO dashboard)
Verifix ┤          │
Graph  ─┘          └─→ Telegram (briefs, alerts, approvals)
                        └─→ Teams / Planner (tasks)

n8n  — integration hub for external systems and scheduled workflows
Power Automate — Microsoft-internal flows only (Outlook → Planner, Teams)
```

Money is stored as integer **tiyin** (1 UZS = 100 tiyin) everywhere. Time is
stored in **UTC** and displayed in **Asia/Tashkent**. Both rules are enforced in
`integrations/common/money.py` and `integrations/common/timeutil.py`.

## Security model

| Rule | How it is enforced |
| ---- | ------------------ |
| 1. Read-only for 90 days | The SAP client has no write path at all. amoCRM/Planner/Teams writes are gated behind `AGENT_WRITES_ENABLED=false`; while closed, every intended write is audited as `dry_run` and nothing is sent. |
| 2. Everything audited | `integrations/common/db.audited()` wraps every external call; rows land in `agent_actions`. Audit failures are logged, never silently swallowed. |
| 3. No hardcoded secrets | All credentials come from `.env` through `integrations/common/config.py`, wrapped in `SecretStr` so they cannot leak into logs or tracebacks. |
| 4. Human approval | `approvals` table + Telegram inline buttons (`TelegramBot.request_approval`). Approvals are idempotent, expire in 24 h, and record who decided. |
| 5. Least privilege | One service account per system; the SAP user is read-only in SAP itself, and a separate `powerbi` Postgres role has `SELECT` only. |

## Setup

### 1. Fill in `.env`

```bash
cp .env.example .env   # already done; edit .env
```

Check what is still missing:

```bash
python -c "from integrations.common.config import settings; print(settings.missing_placeholders())"
```

### 2. Start the stack (VPS)

```bash
docker compose up -d
```

This starts PostgreSQL (schema applied automatically on first boot), n8n, and
the FastAPI webhook receiver. Everything binds to `127.0.0.1` — put a reverse
proxy with TLS in front before exposing anything.

Applying the schema to an existing database:

```bash
psql -U <user> -d mgmg -f database/schema.sql
```

### 3. Create the Teams structure

```bash
python scripts/setup/create_teams_structure.py --dry-run
python scripts/setup/create_teams_structure.py
```

Copy the printed `MS_PLANNER_GROUP_ID` and `MS_PLANNER_DEFAULT_PLAN_ID` into `.env`.

### 4. Verify before going live

```bash
python scripts/selfcheck.py                        # offline logic checks
python agents/ceo-daily-brief/agent.py --dry-run   # real data, nothing sent
python agents/receivables/agent.py --dry-run
```

### 5. Schedule

```bash
crontab scripts/setup/crontab.txt
```

## Layout

```
agents/                 scheduled and event-driven agents
  ceo-daily-brief/      Agent 1 — morning brief
  amocrm-followup/      Agent 2 — stalled-deal follow-up
  receivables/          Agent 3 — AR aging alert
integrations/
  common/               config, logging, DB + audit, retrying HTTP, money, time
  sap/                  SAP B1 Service Layer client (read-only)
  amocrm/               amoCRM client + FastAPI webhook receiver
  verifix/              HR attendance (API or CSV)
  microsoft/            Graph client (app-only) + Teams notifier
  telegram/             bot, alerts, approval buttons
database/               schema.sql + migrations
dashboard/powerbi-queries/  SQL sources, DAX measures, build guide
scripts/                selfcheck, setup scripts, crontab
docs/agent-specs/       what each agent does, and its runbook
```

## Conventions

- Python 3.11+, `httpx` (async), `pydantic`, `loguru`, `python-dotenv`
- Every external call retries 3× with exponential backoff and jitter
- Every function has a docstring stating what it does, returns, and can fail on
- Agents degrade rather than crash: one dead source never blocks the whole brief

## Before first production run

These need real-world values that cannot be guessed from here:

- [ ] `CASH_ACCOUNT_CODES` and `BANK_NAME_BY_ACCOUNT` — `integrations/sap/client.py`
- [ ] Division mappings — `integrations/common/divisions.py`
- [ ] Verifix CSV column names — `COLUMN_ALIASES` in `integrations/verifix/client.py`
- [ ] Reconcile one day of AR output against SAP's own aging report
- [ ] Confirm the Azure app has admin consent for the application permissions
      listed in `integrations/microsoft/client.py`
