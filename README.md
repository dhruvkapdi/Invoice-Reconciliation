# Automated Invoice / Vendor Payment Reconciliation System

An end-to-end automation pipeline that ingests vendor invoice emails, extracts
structured data with an LLM, cross-checks it against purchase orders, and
catches duplicate payments, amount mismatches, and unauthorized spend —
**before** money moves. Built with n8n, Supabase (Postgres), Google Gemini,
and Slack.

This started as a portfolio project to back up automation/agentic-AI skills
with a real, running system — not a tutorial clone. It's directly relevant to
the construction/infra industry, where manual PO-vs-invoice reconciliation is
a genuinely expensive, error-prone problem.

## The problem this solves

Every company that deals with vendors faces the same failure modes:
- Someone has to manually check every invoice against a purchase order
- Duplicate invoices get paid twice because nobody cross-checked invoice numbers
- Amount mismatches get caught *after* the money is already gone
- There's no early warning when a vendor bills for something with no approved PO

This system catches all of it automatically, the moment an invoice arrives —
not during a manual reconciliation session days or weeks later.

## Proof point

A synthetic test batch of 200 invoices (150 clean, 20 mismatches, 15
duplicates, 15 with no matching PO) was run through the live pipeline:

| Result | Count | Amount |
|---|---|---|
| ✅ Auto-approved | 150 | ₹3.39 Cr |
| ⚠️ Mismatch | 20 | ₹40.97 L |
| 🚫 Duplicate blocked | 15 | ₹25.8 L |
| ❓ No PO found (unauthorized spend) | 15 | ₹16.4 L |

**100% catch rate, zero false positives** — every planted issue was caught,
every clean invoice sailed through untouched. ~₹8.3 Cr (~$1M) in flagged/
blocked spend caught before payment in testing.

This was also validated with **real emails** sent through Gmail, extracted
live by Gemini, and matched by the database engine end-to-end — not just the
synthetic batch.

## Architecture

```
        Gmail (new invoice email, PDF attachment)
               │
               ▼
   ┌───────────────────────────┐
   │  1. Invoice Ingestion      │  n8n workflow, polls Gmail every minute
   │  Gmail → Gemini → Supabase │
   └─────────────┬──────────────┘
                  │ inserts row into `invoices`
                  ▼
   ┌───────────────────────────┐
   │  2. Matching Engine        │  Postgres trigger (NOT an n8n workflow) —
   │  (Supabase trigger)        │  fires instantly, same transaction, on
   │  duplicate? → PO exists?   │  every insert regardless of source
   │  → amount matches?         │
   └─────────────┬──────────────┘
                  │ writes match_status + payment_queue row
      ┌───────────┴───────────┐
      ▼                       ▼
┌──────────────┐      ┌─────────────────────┐
│ 3. Daily      │      │ 4. Circuit Breaker   │
│ Digest        │      │ every 15 min         │
│ 8am, Slack    │      │ checks failure log,   │
│ summary       │      │ alerts + flips        │
└──────────────┘      │ kill-switch off       │
                       └──────────┬────────────┘
                                  │
                                  ▼
                     [1. Ingestion] checks this
                     kill-switch flag on its
                     NEXT run and pauses itself
```

**Key design decision:** the matching engine is a Postgres trigger, not a
separate n8n workflow. This is more robust than a polling-based n8n
workflow — it fires atomically the instant a row is inserted, with no
polling gap, and works no matter what system writes the invoice row (n8n
today, a different integration tomorrow).

**Another key decision:** the four pieces are *not* wired together directly
in n8n. Each workflow runs on its own trigger/schedule and communicates only
by reading and writing shared Supabase tables. This means a Slack outage
doesn't block ingestion, and a Gemini rate limit doesn't break the digest —
each piece degrades independently instead of taking the whole system down.

## What's in this repo

```
├── README.md                          — you are here
├── n8n-workflows/
│   ├── 1-invoice-ingestion.json       — Gmail → Gemini → Supabase
│   ├── 2-daily-digest.json            — scheduled Slack summary
│   └── 3-circuit-breaker.json         — failure detection + kill switch
├── supabase/
│   ├── schema.sql                     — full schema: tables, matching
│   │                                     trigger, views, RLS
│   └── seed_test_data.sql             — the 40 POs used in testing
└── docs/
    ├── generate_test_invoices.py      — generates the 200-invoice test batch
    └── images/                        — workflow screenshots (below)
```

## Workflow 1 — Invoice Ingestion

![Invoice Ingestion workflow](docs/images/invoice-ingestion-workflow.png)

Gmail trigger (polls every minute for unread emails with PDF attachments) →

1. **Check Pipeline Enabled** — reads a kill-switch flag from Supabase; if
   the circuit breaker has paused the system, stops here
2. **Split Attachments Into Items** — one email can carry multiple invoice
   PDFs; splits them into independent items
3. **Encode PDF Base64** — prepares the PDF for the API call
4. **Extract Invoice Fields (Gemini HTTP)** — sends the PDF inline (base64)
   to Gemini's `generateContent` endpoint, asks for vendor name, invoice
   number, amount, and PO reference as strict JSON
5. **Normalize / Parse** — validates the JSON; malformed responses are
   caught and logged instead of crashing the run
6. **Insert into `invoices`** — the matching trigger fires the instant this
   commits
7. Every branch (extraction failure, parse failure, insert failure, success)
   logs to a `pipeline_runs` table — full dead-letter visibility, nothing
   silently vanishes

**Engineering note on the AI provider:** this uses Google Gemini
(`gemini-3.6-flash`) via a direct HTTP call rather than a specialized
document-AI node, specifically to keep the project at zero ongoing cost
(free tier, no card required) while Gemini's native PDF understanding keeps
extraction quality high. This is a deliberate cost-vs-capability tradeoff,
not a limitation — documented here rather than hidden.

## Workflow 2 — Matching Engine (Postgres trigger, not pictured as a workflow)

Lives in `supabase/schema.sql` as `match_invoice()`. Runs on every insert to
`invoices`, in this order:

1. **Duplicate check** — same vendor + invoice number already processed
   (and not itself already flagged as a duplicate)? → `duplicate`, blocked
2. **PO lookup** — does the invoice's PO reference exist in
   `purchase_orders`? If not → `no_po`, flagged as unauthorized spend (no
   record that this purchase was ever pre-approved)
3. **Amount check** — does the invoice amount match the PO amount (₹1
   tolerance for rounding)? Match → `auto_approved`. Mismatch → `mismatch`,
   flagged for review

The result is written back onto the invoice row and a matching entry is
created in `payment_queue` (`approved` / `flagged` / `blocked`).

## Workflow 3 — Daily Digest

![Daily Digest workflow](docs/images/daily-digest-workflow.png)

Runs at 8am daily. Reads two Supabase views summarizing the last 24 hours
(`v_recent_digest_summary` for counts/totals by status,
`v_recent_flagged_invoices` for itemized details) and posts a formatted
summary to Slack:

> **Daily Invoice Reconciliation Digest**
> ✅ 150 invoices auto-matched and approved for payment
> ⚠️ 20 flagged for amount mismatch
> 🚫 15 duplicate invoices blocked
> ❓ 15 flagged as unauthorized spend (no matching PO)
>
> Total at-risk spend caught before payment: ₹83,17,668.87

## Workflow 4 — Circuit Breaker

![Circuit Breaker workflow](docs/images/circuit-breaker-workflow.png)

Runs every 15 minutes. Checks a `v_consecutive_failures` view (walks the
`pipeline_runs` log backward from the most recent entry until it hits a
success) and, on 3+ consecutive failures:

1. Posts an urgent Slack alert — distinguishing rate-limit failures (likely
   just hitting Gemini's free-tier cap, self-resolving) from real pipeline
   bugs
2. Flips `pipeline_config.ingestion_enabled` to `false`

Workflow 1 checks that same flag on its next run and pauses itself — no
human has to notice a runaway failure loop and manually intervene.

**Design note:** this originally called n8n's REST API to deactivate the
ingestion workflow directly. That was dropped in favor of the
Supabase-flag kill switch above, because n8n's trial/free plans commonly
block API key creation — the flag approach works on any plan and needs no
special n8n-level permissions.

## Tech stack

- **n8n** — workflow orchestration (Gmail trigger, HTTP requests, Supabase
  nodes, Slack, scheduling)
- **Supabase (Postgres)** — data store, matching engine (trigger-based),
  views, RLS
- **Google Gemini API** (`gemini-3.6-flash`) — PDF invoice field extraction
- **Slack** — daily digest + escalation alerts

## Setup

1. Run `supabase/schema.sql` against a fresh Supabase project
2. (Optional, for testing) run `supabase/seed_test_data.sql`, then generate a
   test batch with `docs/generate_test_invoices.py`
3. Import the three workflow JSON files in `n8n-workflows/` into n8n
4. Configure credentials in n8n:
   - Gmail OAuth2 (the inbox to watch)
   - Query Auth for Gemini (`key` = your Gemini API key from
     [aistudio.google.com](https://aistudio.google.com))
   - Supabase (host + **service_role** key — needed to bypass RLS)
   - Slack OAuth2
5. Pick your Slack channel in the digest and circuit-breaker workflows
6. Activate all three workflows

## Real bugs found and fixed via live testing

Worth calling out, since this is what separates a working system from a demo
that only survives the happy path — all of these were found by actually
running real invoices through the pipeline, not by code review:

1. **Strict boolean validation crash** — an IF node checking a boolean flag
   threw on an unused empty `rightValue` under n8n's strict type validation;
   switched to loose validation.
2. **Binary data loss through an intermediate read** — inserting a Supabase
   lookup node between the Gmail trigger and the attachment-processing step
   silently stripped the PDF binary (Supabase nodes don't pass binary
   through). Fixed by pulling binary directly from the trigger node via
   `itemMatching()`.
3. **Model availability** — `gemini-2.5-flash` returned 404s for newly
   created API keys ("no longer available to new users," per Gemini's own
   error message); switched to `gemini-3.6-flash`.
4. **Case-sensitive auth parameter** — a `Key` vs `key` mismatch in the
   query-auth credential caused silent auth failures.
5. **Error-routing misconfiguration** — `onError: continueErrorOutput` was
   being set in the wrong part of the node config three different ways
   before landing on the correct mechanism, so failures were crashing the
   whole run instead of routing to the dead-letter log.

## Roadmap / not yet production-ready

This is portfolio/demo-ready today. For real production use at a company,
still needed:
- Real PO data feed (currently 40 seeded test vendors)
- A review UI for flagged invoices (currently a database query)
- Paid n8n plan / self-hosted instance (currently built on a trial plan)
- A dedicated shared invoices inbox instead of a personal Gmail account
- Paid Gemini tier for real invoice volume
