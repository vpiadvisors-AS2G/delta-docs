# DELTA_PLAN

Forward-looking work items that don't fit in a single Linear issue yet, or that were surfaced as follow-ups during other work. Not a replacement for Linear — durable/cross-cutting items only.

## Open

- **Make `scripts/postgres.mjs` load `.env.local` itself.** Right now `MIGRATION_DATABASE_URL` must be exported in the shell before running `migrate`, or it silently falls back to local Postgres and reports "up to date" against the wrong database — bit us twice now (2026-09-02 on AS2-6, and again on AS2-53 the same day, where it fell back to a local `wim_dev` database containing the old dead schema and produced a confusing partial-match result before the mistake was caught). See [[DELTA_DECISIONS]].
- **Revisit the Supabase LEAKPROOF gap** (see [[DELTA_DECISIONS]] 2026-09-02) if `ltree` ancestor lookups (`path <@ ...`) show up as slow under RLS in practice. Currently accepted, not fixed.
- **Reconcile `document_references` (new, AS2-52) against `reference_numbers` (existing, AS2-6).** Built as two distinct tables — a typed per-invoice citation vs. a generic reverse-lookup registry — but they may turn out to be the same concept in practice. Flagged in the migration's own header comment, not resolved.
- **Optional retailer↔carrier link table** — deferred. Only needed if a concrete cross-role netting use case shows up (e.g. same corporate parent is both a retailer and a carrier to the manufacturer). See [[DELTA_DECISIONS]].
- **`mv_deduction_summary`, `mv_reconciliation_kpi`, `mv_vendor_scorecard`** (if/when rebuilt against the real schema) will need renaming away from "vendor" terminology to match the carrier/retailer split, plus a refresh strategy (no `REFRESH MATERIALIZED VIEW CONCURRENTLY` cron, no unique index required for concurrent refresh decided yet).
- **`price_list`/`price_list_item`** were explicitly skipped (see [[DELTA_DECISIONS]]) on the assumption PO/invoice line prices are sufficient. Revisit if a pricing-master use case emerges.

## Sprint 1 remaining (per CLAUDE.md)

- AS2-9 — OpenRouter routing config (`usage_events` table still needs applying)
- AS2-42 — OTEL baseline
