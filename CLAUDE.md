# Kipu Support Intelligence Platform — Project Context

## Deadline: Friday April 11, 2026

## What We're Building
A Support Intelligence Platform for Kipu's EMR support team. Base44 dashboard showing Q1 2026 support metrics (Jan–Mar) across 4 lines of business. Built in **n8n** (automation) + **Base44** (frontend/app) + **Supabase** (database). Do NOT build outside these platforms unless explicitly asked.

## Infrastructure
- **Supabase**: "Equinox Supabase - Zendesk Ticket Metrics" (Postgres credential ID: `XjgfokqBkBtAIKkz`)
- **Zendesk**: "Equinox Zendesk Account" (`Nd6nYzUYdeDBS3Gj`), instance: `kipuhealth.zendesk.com`
- **n8n webhooks**:
  - `POST https://automations.kipuqa.com/webhook/zendesk-metrics-preview` — preview count
  - `POST https://automations.kipuqa.com/webhook/export-sync` — full sync
  - `POST /kb-gap-scan-001` — KB gap scan
  - `POST /kb-gap-chat-001` — KB gap chat

## 4 Lines of Business (Form IDs)
| LOB | form_id |
|-----|---------|
| EMR | `1900001563224` |
| Lab | `32729757998235` |
| CRM | `1260812421569` |
| Accounting | `5374649379867` |

## Key Supabase Tables
- `support_ticket_metrics_fact` — main metrics (Q1 fully loaded for all 4 LOBs)
- `kbg_scans` — KB gap scan metadata
- `kbg_sections` — KB gap per-section results
- `kbg_form_config` — KB gap form config (read-only by flow)
- `sync_config` — incremental sync settings *(to be created)*
- `sync_history` — incremental sync run log *(to be created)*

## Key n8n Files (GitHub: pparikh73/zendesk-metrics-sync)
- `Kipu - Zendesk Metrics.json` — metrics sync flows
- `Kipu KB Gap Finder.json` — KB gap analysis + chat flows
- `BTA - Pull and Analyze Batch.json` — not needed

---

## Master Checklist

### A. KB Gap — Fix Priority/Weighting Logic *(do first)*
- [ ] `Code - Assemble Final Output`: low link rate alone → priority capped at `medium`; ticket volume drives `high`
- [ ] `Code - Build Section Prompt` (LLM system prompt): KCS link rate is a contributing signal, NOT a standalone major gap indicator; volume + subject matter are primary drivers

### B. KB Gap — Run Monthly Q1 Scans *(after A — 12 scans total)*
| | Jan | Feb | Mar |
|---|---|---|---|
| EMR `1900001563224` | [ ] | [ ] | [ ] |
| Lab `32729757998235` | [ ] | [ ] | [ ] |
| CRM `1260812421569` | [ ] | [ ] | [ ] |
| Accounting `5374649379867` | [ ] | [ ] | [ ] |

### C. Chat to Insights — Expand Context in Both Interfaces
- [ ] Modify `Code - Build Chat Prompt` + `Postgres - Load Scan Context` in KB Gap Finder: also query `support_ticket_metrics_fact` filtered to same `form_id` + date range as active scan
- [ ] Inject aggregated metrics into LLM context: volume, CSAT, SLA breach %, resolution time, link rate
- [ ] Confirm main metrics dashboard chat is wired to a Base44 backend function — connect if not

### D. Incremental Daily Sync — New n8n Flow + Base44 Settings UI
**Supabase (new tables):**
- [ ] `sync_config`: `enabled` (bool), `scheduled_hour` (0–23), `timezone`, `last_run_at`, `last_run_status`
- [ ] `sync_history`: `run_at`, `form_id`, `tickets_synced`, `status`, `error_msg`

**n8n (new flow):**
- [ ] Hourly cron trigger — check `sync_config`: skip if `enabled=false` or hour doesn't match
- [ ] Zendesk incremental cursor API (`/api/v2/incremental/tickets/cursor.json`), cursor stored per form in Supabase
- [ ] Upsert into `support_ticket_metrics_fact` — all 4 forms
- [ ] Write each run result to `sync_history`

**Base44 (new settings page):**
- [ ] Toggle: enable/disable sync
- [ ] Time picker: set `scheduled_hour` + timezone
- [ ] Last run status indicator
- [ ] Sync history table: date, form, tickets synced, status

### E. Q1 Validation *(after A–D)*
- [ ] KB scan gap classifications look reasonable — no over-flagged link gaps
- [ ] Chat tested in both interfaces with Q1 questions (metrics + KB)
- [ ] Incremental sync test run confirms it picks up recent tickets
- [ ] Dashboard numbers spot-checked against Zendesk

---

## Build Order
**A → B → C → D → E**

## Rules / Guardrails
- **Don't build outside n8n + Base44 + Supabase** unless explicitly asked
- **Don't add features beyond the checklist** — stay in scope
- **Check this file when going down a rabbit hole** — re-anchor to the checklist
- **KB gap priority rule**: low link rate alone = max `medium` priority; volume drives `high`
- **Chat context rule**: always filter `support_ticket_metrics_fact` by the active scan's `form_id` + date range
