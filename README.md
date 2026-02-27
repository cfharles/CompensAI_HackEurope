# CompensAI

CompensAI is a HackEurope project that automates compensation claims from inbound emails while keeping a human approval checkpoint before submission.

[DevPost](https://devpost.com/software/compensai-choakb)  
[Lovable Dashboard](https://github.com/yauhenifutryn/dispute-defender-dash)
[Demo Airline/Retail Sites](https://skill-deploy-x4cr0r1eo8-codex-agent-deploys.vercel.app/index.html)

## Why we built it

People miss legitimate compensation because claim workflows are fragmented and tedious: finding the right contact path, extracting evidence, mapping legal/policy context, drafting a claim, and chasing replies.

CompensAI focuses on a proactive workflow:
- monitor new inbound emails,
- detect claim-worthy incidents,
- generate a draft with estimated value,
- require one-click user approval before submission,
- track outcomes in an auditable timeline.

Privacy model (MVP): monitor incoming emails going forward; process older cases only when users explicitly forward those emails.

## What it does

- Detects potential claims in inbound emails (flight delay/cancellation/baggage, parcel delivery issues, train disruptions).
- Converts unstructured email text into a structured case.
- Enriches decisioning with rule/context knowledge.
- Produces a deterministic claim draft (email or form-oriented draft).
- Stops at a human-in-the-loop approval gate before sending.
- Tracks vendor responses and state transitions until resolution.
- Writes append-only `case_events` for explainability/debugging.

## Hackathon architecture

1. Agent 1 (`n8n + Gmail`) watches inboxes and forwards payloads to FastAPI.
2. Agent 2 (`FastAPI + Claude + fallback logic`) classifies, extracts, and drafts.
3. Agent 3 (`FastAPI billing step`) stores resolution + fee data when a case is resolved.
4. Dashboard (`Lovable + Supabase realtime`) reads `cases` and `case_events` directly.

Supabase is the system of record:
- `public.cases`: current case snapshot.
- `public.case_events`: append-only timeline.

## Demo setup used in the hackathon

- Monitored client inbox: `client.compensai@gmail.com`
- Simulated vendor inbox: `everydayaionx@gmail.com`
- Billing in MVP: financial data is generated/stored for dashboard usage; full payment collection automation is future work.

## API (implemented)

Base URL: `http://localhost:8000`

- `GET /health`  
  Health check.

- `POST /emails/ingest`  
  Triage-first entrypoint. Accepts all emails, classifies `trash` vs `candidate`, only creates cases for candidates.

- `POST /cases/intake`  
  Direct case intake path (skips triage gate, still runs Agent 2 processing).

- `POST /cases/{case_id}/approve`  
  Approves a draft and optionally sends it via configured Agent 1 webhook.

- `POST /cases/{case_id}/vendor_response`  
  Records vendor outcome and, if resolved, triggers billing updates/events.

- `POST /cases/{case_id}/run_agent2`  
  Re-runs Agent 2 on an existing case.

- `GET /cases/email-drafts/pending`  
  Returns drafts currently awaiting approval.

- `POST /cases/email-drafts/{case_id}/mark-sent`  
  Marks approved draft as submitted.

Auth headers (enabled when configured):
- `X-CompensAI-Webhook-Secret` for n8n/webhook endpoints.
- `X-CompensAI-Admin-Key` for admin endpoints.

## Case lifecycle

Common statuses:
- `processing`
- `awaiting_approval`
- `submitted_to_vendor`
- `vendor_replied`
- `needs_info`
- `rejected`
- `resolved`

If classification ends in unknown/unsupported category, the case can be removed from active dashboard flow in this MVP.

## Tech stack

- FastAPI (Python backend)
- n8n + Gmail (email watch/send orchestration)
- Supabase (state + event log + realtime)
- Claude (LLM extraction/drafting with deterministic fallback)
- Lovable (dashboard)
- ReportLab (PDF generation for form-style drafts)

## Repository structure

```text
app/
  main.py
  core/         # settings + auth header checks
  db/           # lightweight Supabase REST client
  repositories/ # case/event persistence helpers
  routers/      # API endpoints
  services/     # triage, agent2, billing, company-site helpers
  kb/           # rule knowledge (EU261, train)
examples/       # sample intake payloads
```

## Local development

### 1) Install

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 2) Configure environment

Create `.env` in the repo root:

```env
SUPABASE_URL=...
SUPABASE_SERVICE_ROLE_KEY=...

# Optional
N8N_WEBHOOK_SECRET=...
ADMIN_API_KEY=...
AGENT1_SEND_WEBHOOK_URL=...
ANTHROPIC_API_KEY=...
ANTHROPIC_MODEL=claude-haiku-4-5-20251001
ANTHROPIC_TIMEOUT_SECONDS=30
CORS_ORIGINS=http://localhost:3000
```

### 3) Run API

```bash
uvicorn app.main:app --reload --port 8000
```

## Quick test payloads

```bash
curl -X POST http://localhost:8000/cases/intake \
  -H "Content-Type: application/json" \
  --data @examples/intake_ryanair_delay.json
```

If webhook secret is enabled:

```bash
curl -X POST http://localhost:8000/cases/intake \
  -H "Content-Type: application/json" \
  -H "X-CompensAI-Webhook-Secret: <your-secret>" \
  --data @examples/intake_ryanair_delay.json
```

Other example payloads:
- `examples/intake_flight_cancellation.json`
- `examples/intake_delivery_late.json`

## Supabase verification queries

```sql
select id, vendor, category, estimated_value, status, updated_at
from public.cases
order by updated_at desc
limit 20;
```

```sql
select case_id, actor, event_type, details, created_at
from public.case_events
order by created_at desc
limit 50;
```

## Challenges and lessons

- Reply monitoring and thread correlation are orchestration-heavy; deterministic state transitions matter.
- Idempotency/dedup are critical when ingesting from watch-based workflows.
- Event sourcing (`case_events`) made debugging and explainability much easier.
- Human approval is a safety primitive in legal/financial automation.

## What’s next

- Production-grade user inbox integration, tenant isolation, and hardened auth/RLS.
- Vendor-specific submission connectors (Ryanair/Amazon/etc.) with robust retryable automation.
- Rejection mitigation + escalation packs.
- SLA-based follow-ups.
- Draft revision loop from user comments.
- Expanded billing flows (invoice delivery, reminders, collection automation).

## Built with

`claude` `codex` `cursor` `fastapi` `github` `javascript` `lovable` `n8n` `python` `sql` `stripe` `supabase`
