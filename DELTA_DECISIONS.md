# DELTA_DECISIONS

Durable architecture/scope decisions. Each entry: what was decided, why, and who/when. Newest first.

## 2026-08-30 — Two-stage thesis: $40-100K/mo personal income floor, then $40-50M personal wealth ceiling

Confirmed by Venkatesh, refined same day. Explicit goal is to avoid a famine-or-feast bet — sequence, not either/or:

- **Stage 1 (floor, near-term):** $40-100K/month personal income. Replaces a full-time job with room to grow — not a single fixed number like the earlier $50K MRR / $40K net income framing, but a range. At ~10+ mid-market customers and the already-decided $50-100K/year pricing, this is reachable well under the original $1-1.5M ARR / 10-customer target.
- **Stage 2 (ceiling, aspirational):** $40-50M personal wealth. Supersedes the earlier $10M-minimum framing as the actual number being aimed at once Stage 1 is secure — consistent with the architecture doc's $60-70M acquisition / 80% ownership math.

Does not change Sprint 1-7 scope or the 3-agent/API-first/OTEL architecture — those stay because they are cheap to build now and expensive to retrofit later if Stage 2 materializes. What it does change: if a build-vs-ship scope tradeoff comes up before customer 1, prioritize Stage 1 revenue-generating functionality over Stage 2 sellability polish (FedRAMP positioning, external API v2).

## 2026-08-29 — DBScripts schema reconciliation (schema_part1–3.sql)

Context: a request to generate DDL for a 78-entity schema (in a `wim` schema, standalone) turned out to conflict with the AS2-6 schema already built in `supabase/migrations/`. Reconciled instead of building a parallel schema. Confirmed by Venkatesh 2026-08-29. See [[DELTA_STATE]] for what actually got built.

- **Target schema is `public`, not `wim`.** Matches the existing AS2-6 migrations, which are unqualified and land in `public`. The `wim` schema referenced in the original task exists locally but is unused.
- **`match_links` built now, not deferred to AS2-53.** It's the relationship engine CLAUDE.md's hard rule 5 describes ("do not add a shortcut foreign key between two documents"), but no migration had created it yet. Group 9 (Reconciliation) had no way to represent matches without violating that rule, so it's built in `schema_part1.sql` ahead of AS2-53.
- **`disputes` absorbs `claim`.** The original spec named both `claim` (deductions group) and `dispute` (dispute-workflow group) as separate entities describing the same workflow. Kept one table (`disputes`), dropped `claim`. A deduction opens at most one dispute (`uq_disputes_tenant_deduction`).
- **`deduction_type` dropped.** Folded into `reason_codes.category`, which already existed (dual-mode deduction/overbilling) and covered the same classification need. Avoids a redundant lookup table.
- **`price_list`/`price_list_item` skipped.** PO and invoice lines carry `unit_price` directly (UBL style) rather than resolving against a master price list. Revisit if a real pricing-master use case shows up.
- **`freight_invoice` folded into `invoices.shipment_id`.** One `invoices` table serves both procurement and freight invoicing (nullable `shipment_id` marks the freight case), rather than a parallel table — consistent with "mode is configuration, not a code path fork."
- **`reference_code` renamed to `code_lists`.** The original name collided in meaning with the existing `reference_numbers` table (a document reference-number registry — PO/BOL/PRO/etc. pointing at entities). `code_lists` is a distinct concept (UI/config lookup values) and needed a distinct name.
- **vendor/customer/rate_agreement/rate_line_item are not new tables.** Reused `parties`+`party_roles` (role `supplier`/`carrier`/`customer`) and `contracts`+`contract_lines` (`contract_type = 'carrier_rate_agreement'`) respectively — both already existed in AS2-6 and covered these concepts.
- **Every new table follows the existing AS2-6 conventions exactly**, not the originally-requested standalone style: `text ... check (col in (...))` instead of native Postgres `ENUM` types, composite tenant FKs (`(tenant_id, id)`, not bare `id`) per CLAUDE.md hard rule 2, and `alter table ... enable/force row level security` + `tenant_isolation` policy on every tenant-scoped table.

## Prior — AS2-6 (see git history / migration comments)

AS2-6's own decisions (SCD-2 parties, ltree hierarchies, composite tenant FKs retrofit, `nearest_ancestor_id()` resolution) are documented inline in `supabase/migrations/*.sql` comments and are not duplicated here. Start there for anything predating 2026-08-29.


## 2026-08-31 — OpenRouter routing: config-driven, model choice is a Sprint-1 non-decision

Confirmed by Venkatesh. Model per route (extraction/classification/reconciliation/chat) is env-var-driven in `packages/shared/src/llm/openrouter.ts` — changing a model is a config change, never a code change. This was validated, not decided: wiring/retry/auth confirmed working against free-tier models (NVIDIA Nemotron), but free-tier output quality is not representative — extraction ignored format instructions, one chat call returned empty. Real model selection (Haiku/Sonnet vs. Gemini) is deferred until a golden-dataset accuracy pass, not before real documents run.

**Outstanding, not done:** `usage_events` table (tenant-scoped, RLS via `current_tenant_id()`) drafted in `supabase/migrations/20260831000001_usage_events.sql` but not yet applied — AS2-9's Linear spec requires per-call token logging and this wasn't built in the first pass. Ticket should not be treated as fully closed until this lands.


