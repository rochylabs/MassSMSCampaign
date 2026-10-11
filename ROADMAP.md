# ROADMAP.md — PH Mass SMS Marketing Campaign System

Architecture (data model, pipeline, aggregator choice) and UI (6 screens, API spec, guardrails) are already finalized. This document only sequences that design into phased, executable tasks for a small team. Phases are ordered by hard dependency, not by UI/backend split — Sender ID approval and NPC registration gate real sending and must start first.

---

## Phase 0 — Compliance & Registration (external, non-engineering — kick off immediately, ~1-3 weeks lead time)

These block real message sending. Start in parallel with Phase 1 engineering; do not wait for backend to be "done" first.

- [ ] (External) Gather Sender ID registration docs: DTI/SEC certificate, BIR COR, valid government ID, sample message templates
- [ ] (External) Draft and sign Semaphore aggregator terms/agreement
- [ ] (External) Submit alphanumeric Sender ID registration to Semaphore, track approval status (~3-5 business days)
- [ ] (External) Designate a Data Protection Officer (DPO) and document the designation
- [ ] (External) Draft consent capture text/version for opt-in (what the user sees/agrees to at point of collection)
- [ ] (External) Register as Personal Information Controller with the National Privacy Commission (required once processing 1,000+ records — confirm whether to register now or gate behind contact count)
- [ ] (External) Document breach notification procedure to meet the 72-hour NPC notification requirement
- [ ] (External) Set up prepaid wallet with Semaphore, initial top-up (₱1,000-₱5,000 minimum)
- [ ] (Engineering) Confirm with Semaphore: DLR webhook reliability per telco (Globe/Smart/DITO/Sun), decide webhook vs polling fallback strategy up front

---

## Phase 1 — Backend Foundation (engineering)

- [ ] Set up Postgres schema: `contacts` (phone_e164 unique, carrier, consent_status, consent_source, consent_at, opted_out_at)
- [ ] Set up Postgres schema: `templates` (name, body, sender_id)
- [ ] Set up Postgres schema: `campaigns` (template_id, status, scheduled_at, created_by, sent_by)
- [ ] Set up Postgres schema: `message_log` (campaign_id, contact_id, aggregator_msg_id, segments, status, sent_at, delivered_at, UNIQUE(campaign_id, contact_id))
- [ ] Implement phone normalization utility: raw input → +639XXXXXXXXX, reject unparseable numbers
- [ ] Implement carrier detection (Globe/Smart/DITO/Sun) from normalized number prefix
- [ ] Implement segment/encoding calculator (GSM-7 vs UCS-2, segment count, emoji detection)
- [ ] Stand up durable queue (job runner for send + import jobs)
- [ ] Implement Semaphore send API integration (single message send call, error handling, retry policy)
- [ ] Implement DLR handling: webhook receiver and/or polling fallback per Phase 0 confirmation, update `message_log` status
- [ ] Implement worker pool: consent check → dedup check (campaign_id + contact_id) → segment calc → send → log
- [ ] Implement consent-event logging as append-only table (consent granted/revoked, source, timestamp, text version)
- [ ] Wire `created_by` / `sent_by` audit fields into campaign and send actions

---

## Phase 2 — Bulk Recipient Upload (engineering)

- [ ] Build CSV import parser: expects phone, consent_source, consent_at, consent_status columns
- [ ] Apply phone normalization to every row on import
- [ ] Enforce rule: uploaded rows never default to opted_in — consent_status must be explicit in the file
- [ ] Implement upsert logic: update existing contact without overwriting an existing opt-out
- [ ] Hard-reject rows for numbers already marked opted-out (do not re-add without new consent)
- [ ] Generate rejection report (per-row reason: bad phone, missing consent fields, opted-out conflict, duplicate)
- [ ] Small-file import path: synchronous script/cron-triggered import
- [ ] Large-file import path: queue-based async import job with status polling
- [ ] API: `POST /contacts/import`, `POST /contacts/import/confirm`, `GET /contacts/import/rejections`

---

## Phase 3 — Core UI Screens (engineering)

- [ ] Build Contacts List screen: table view, filters (consent status, carrier, opt-out date), soft-delete
- [ ] Build Contacts List screen: export action (`GET /contacts/export`)
- [ ] Build Bulk Upload screen: file picker, upload progress, rejection report display/download
- [ ] Build Template Library screen: list, create, edit, delete templates
- [ ] Build Template Library screen: live segment/encoding counter with cost warning on body input
- [ ] Build Template Library screen: preview rendering (`GET /templates/{id}/preview`)
- [ ] API: `GET/POST/PATCH/DELETE /templates`
- [ ] API: `GET/POST/PATCH/DELETE /contacts`
- [ ] API: `GET /sender-ids` (list approved sender IDs for use in composer/templates)

---

## Phase 4 — Campaign Composer + Sending Guardrails (engineering)

- [ ] Build Campaign Composer screen: recipient selection, template selection, schedule or send-now
- [ ] Composer: cost/recipient estimate call (`POST /campaigns/estimate`) before confirm
- [ ] Composer: block send if selected Sender ID is not yet approved
- [ ] Composer: show opted-out exclusion count before confirm (recipients removed due to opt-out)
- [ ] Composer: "confirm send to N recipients" explicit confirmation step (this is the v1 approval guardrail — no multi-user workflow)
- [ ] API: `POST /campaigns`, `POST /campaigns/{id}/send`, `POST /campaigns/{id}/schedule`, `POST /campaigns/{id}/cancel`
- [ ] Build Campaign Detail/Status screen: status view, stats (`GET /campaigns/{id}/stats`)
- [ ] Campaign Detail: allow cancel/pause only while status is scheduled or in-progress
- [ ] API: `GET /campaigns`, `GET /campaigns/{id}`, `GET /campaigns/{id}/export`

---

## Phase 5 — Audit / Compliance UI (engineering)

- [ ] Build Consent/Opt-Out Audit screen: append-only event log view per contact
- [ ] API: `GET /contacts/{id}/consent-history`, `GET /contacts/{id}/consent-events`
- [ ] API: `GET /audit/consent-log` (global audit view)
- [ ] Ensure opt-out requests are processed promptly and reflected immediately in contacts + audit log (DPA data subject right)
- [ ] Verify audit screen enforces append-only at the API layer, not just the UI layer

---

## Phase 6 — Hardening / Launch Readiness (engineering)

- [ ] Load-test send pipeline at expected campaign volume (queue throughput, worker concurrency)
- [ ] Verify dedup constraint holds under concurrent sends (UNIQUE(campaign_id, contact_id))
- [ ] Confirm DLR status updates land correctly across all 4 telcos in a real test send
- [ ] Validate wallet balance checks / low-balance alerting before send
- [ ] End-to-end test: CSV import with mixed valid/invalid/opted-out rows → correct rejection report
- [ ] End-to-end test: composer blocks send on unapproved Sender ID
- [ ] End-to-end test: opted-out contact cannot be re-added via upload or manual add without new consent
- [ ] Run breach-notification runbook drill against the 72-hour NPC requirement
- [ ] Final review: Sender ID approved, NPC registration complete (if applicable), DPO designated, wallet funded — go/no-go for first real campaign

---

## Out of scope for v1

Explicitly deferred — do not re-propose mid-build:

- Drag-and-drop CSV column mapping
- WYSIWYG template builder
- Multi-user approval workflow (composer confirm step is the v1 guardrail instead)
- A/B testing
- In-UI Sender ID registration flow
- Live/websocket delivery dashboard
- Contact dedup/merge screen
- Complex audience builder
- In-app notifications
- Multi-aggregator failover (Movider is fallback-only, not active in v1)
- International SMS
- MMS
- Custom rate limiter
