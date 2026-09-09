# DELTA_PLAN

Forward-looking work items that don't fit in a single Linear issue yet, or that were surfaced as follow-ups during other work. Not a replacement for Linear — durable/cross-cutting items only.

## Open
- **Independent evidence sourcing for Overbilling audit (gate timestamps, mileage) — deferred, not solved.** `accessorial_events` gate-in/gate-out and `invoice_charges` mileage-based rates are currently only as trustworthy as whatever document they were extracted from (often the carrier's own BOL/invoice) — there's no independent source (telematics, WMS, a mileage-rating API) to verify against. Explicitly out of scope for now (integration/spend decision) — see [[DELTA_DECISIONS]] 2026-09-09. The auto-approve gate already routes low/no-evidence cases to manual review, so this is a known limitation, not a silent trust bug.
- **Apply `20260909000001_freight_overbilling_hardening.sql` to `wim_dev`.** Scratch-tested and passing on `delta_dev` as of 2026-09-09 (3 bugs found and fixed in the process — see [[DELTA_DECISIONS]]). Venkatesh: run `pnpm run db:migrate`, then verify against Supabase directly. See [[DELTA_DECISIONS]].
- **Deduction-mode schema hardening — deferred, not scoped yet.** The same research pass that hardened Freight Overbilling (2026-09-09) surfaced Deduction-mode gaps (promotional/off-invoice/scan-based/price-protection/damage/unsaleable-returns/slotting/co-op/logistics-penalty/admin-fee categories, each needing specific evidence fields) — explicitly out of scope per Venkatesh's direction to focus on Overbilling first. Revisit when Deduction mode becomes the priority.
- **Wire `fuel_surcharge_schedules`/`fuel_surcharge_indices` into the match/reconciliation engine** once AS2-54/55 (matching engine) starts — the tables exist but nothing computes-and-compares yet.

- **Wire `app.actor_user_id` into application request handling.** The 2026-09-06 generic audit trigger (`fn_audit_log_change()`, 77 tables) reads this session GUC for "who" in every audit_log row — until an agent's request-handling code runs `SET LOCAL app.actor_user_id = '<users.id>'` per request (same pattern as `app.tenant_id`), every audit row will say `actor='system'`. Needed before the audit trail is actually useful for "who did this." See [[DELTA_DECISIONS]] 2026-09-06.
- **Revisit the Supabase LEAKPROOF gap** (see [[DELTA_DECISIONS]] 2026-09-02) if `ltree` ancestor lookups (`path <@ ...`) show up as slow under RLS in practice. Currently accepted, not fixed.
- **Design a resolution-linkage between `document_references` and `reference_numbers`.** Confirmed 2026-09-04 these are NOT duplicates — different pipeline stages (see [[DELTA_DECISIONS]]) — but there's no mechanism today to promote an asserted `document_references` citation into a resolved `reference_numbers` row once it's matched, and no status field marking a citation resolved/unresolved. Needs: (a) a `status` or `resolved_entity_table`/`resolved_entity_id` column on `document_references`, or (b) a trigger/application step that inserts into `reference_numbers` on match and marks the source row resolved. Decision on which approach — not yet made.
- **Optional retailer↔carrier link table** — deferred. Only needed if a concrete cross-role netting use case shows up (e.g. same corporate parent is both a retailer and a carrier to the manufacturer). See [[DELTA_DECISIONS]].
- **`mv_deduction_summary`, `mv_reconciliation_kpi`, `mv_vendor_scorecard`** (if/when rebuilt against the real schema) will need renaming away from "vendor" terminology to match the carrier/retailer split, plus a refresh strategy (no `REFRESH MATERIALIZED VIEW CONCURRENTLY` cron, no unique index required for concurrent refresh decided yet).
- **`price_list`/`price_list_item`** were explicitly skipped (see [[DELTA_DECISIONS]]) on the assumption PO/invoice line prices are sufficient. Revisit if a pricing-master use case emerges.

- **Neither the cloud Cowork session nor the linked-device shell can reach Supabase's Postgres port directly** (raw TCP egress blocked both sides -- only HTTPS/npm registry allowlisted). `node scripts/postgres.mjs migrate` must be run by Venkatesh in his own terminal for every real apply going forward. Not a bug to fix -- a standing operational constraint to remember before promising a same-session apply.

## Sprint 1 remaining (per CLAUDE.md)

- AS2-9 — OpenRouter routing config (`usage_events` table still needs applying)
- AS2-42 — OTEL baseline
