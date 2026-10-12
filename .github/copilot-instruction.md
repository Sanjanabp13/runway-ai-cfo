# Runway: project context for Copilot

## What we are building
Runway is an autonomous collections and cash-flow agent for freelancers and small agencies, built for the PayPal AI Hackathon (deadline Nov 13, 2026) and continued afterwards as a portfolio project.

Core loop:
1. Detect overdue PayPal invoices.
2. Assess: predict client payment delay from history.
3. Plan: an LLM agent proposes an action (reminder, firm reminder, new invoice).
4. Policy check: deterministic rules validate the proposal.
5. Human approval (autonomy modes: suggest / approve / auto-under-limit).
6. Act: create and send a PayPal invoice or reminder via the PayPal Invoicing API v2 (sandbox).
7. Observe: a signed PayPal webhook marks the invoice paid and updates runway.
8. Every step is written to an audit log.

## Hard requirements (hackathon rules)
- PayPal sandbox integration must be central. AI must be meaningfully used.
- Public GitHub repo with an open-source LICENSE (MIT).
- Judges must be able to run it: complete README setup steps and a `.env.example`, plus a seed script. Aim for a hosted demo too.
- Never commit secrets. Sandbox only, never live credentials.

## Tech stack
- Backend: Python 3.12, FastAPI, SQLAlchemy 2.x, Alembic, Pydantic v2, httpx, Pandas, NumPy, pytest.
- Database: PostgreSQL 16 via Docker Compose.
- Agent: start with a plain Python tool-calling loop; migrate to LangGraph later (do not start with LangGraph).
- Frontend: Next.js (App Router, TypeScript, Tailwind), AG Grid Community for tables.
- Later: Elastic (client history search), Zapier/Slack notifications, Render deployment, Postman collection.
- Dev machine: macOS (zsh). Backend runs on port 8000 (never 5000, AirPlay uses it).

## Repo layout
backend/app/{main.py, config.py, api/, paypal/, forecast/, agent/, db/, services/}
backend/tests/
frontend/
postman/
docker-compose.yml, .env.example, LICENSE, README.md, DECISIONS.md, NOTES.md

## Architecture rules (do not violate)
1. The LLM never does arithmetic and never calls PayPal directly. It only proposes structured actions. Plain code validates them in a policy engine and executes them.
2. All PayPal HTTP goes through one `PayPalClient` class in `backend/app/paypal/client.py` so it can be mocked in tests. Cache the OAuth token until shortly before `expires_in`.
3. Webhooks: verify the signature using PayPal's verify-webhook-signature API, store every event in `webhook_events` (unique event id) BEFORE processing, and make processing idempotent (PayPal retries).
4. Every table has `business_id`, even with one seeded business (future multi-tenancy).
5. Do not store a mutable balance. Derive balance from transactions.
6. Money is `Decimal` / `NUMERIC(12,2)`, never float.
7. Any action that moves money or contacts a client must pass the policy engine and, unless in auto mode under limit, wait for approval.
8. Schema changes only through Alembic migrations.
9. Config from environment via pydantic-settings. Never hardcode secrets.
10. Write pytest tests with each module. Mock PayPal in unit tests.

## Data model (initial)
- businesses(id, name, created_at)
- clients(id, business_id, name, email, risk_score, created_at)
- invoices(id, business_id, client_id, amount, currency, status[DRAFT, SENT, PAID, OVERDUE, CANCELLED], paypal_invoice_id, payer_view_url, due_date, sent_at, paid_at, reminder_count, created_at)
- transactions(id, business_id, invoice_id nullable, amount, type[INCOME, EXPENSE], category, description, occurred_at, paypal_transaction_id unique nullable)
- webhook_events(id, event_id unique, event_type, payload JSONB, received_at, processed_at nullable, status)
- agent_actions(id, business_id, invoice_id, action_type, proposed_payload JSONB, reasoning, status[PROPOSED, APPROVED, REJECTED, EXECUTED, BLOCKED], created_at, decided_at)
- audit_logs(id, business_id, actor[agent|user|system|paypal], event, metadata JSONB, created_at)

## API contract (build stub endpoints returning fake data first)
- GET /api/invoices
- GET /api/invoices/{id}
- POST /api/invoices (create draft)
- POST /api/invoices/{id}/send
- GET /api/forecast (runway: p10, p50, p90 months, plus monthly projection series)
- POST /api/forecast/simulate (what-if inputs)
- GET /api/approvals
- POST /api/approvals/{id}/approve
- POST /api/approvals/{id}/reject
- POST /api/agent/run (trigger a detection and planning run)
- POST /api/chat
- GET /api/audit
- POST /api/paypal/webhook
- POST /api/dev/simulate-paid/{invoice_id} (dev only, so judges can see the PAID flow without ngrok)

## PayPal specifics to verify against the official docs before coding
- Invoicing API v2 (create draft, send, and get invoice; reminders). The docs may have changed, so check them and do not assume request shapes.
- Webhook event for paid invoices (expected: INVOICING.INVOICE.PAID) and the signature verification endpoint.
- Sandbox base URL: https://api-m.sandbox.paypal.com
- Negative testing via the `PayPal-Mock-Response` header on sandbox requests.

## How I want you to work
- I am learning these topics. Work in small steps, one module at a time. Before writing code, state the plan in 3-5 lines. After writing code, explain it briefly and note what could go wrong.
- Prefer simple, readable code over clever abstractions. Add type hints and short docstrings.
- Ask before making a major decision (new dependency, schema change, new service).
- Never invent API fields. If unsure about a PayPal field, say so and point me to the docs page to check.
- Keep `main` runnable. Small commits with clear messages.