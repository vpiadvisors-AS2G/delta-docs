# DELTA_PLAN

Forward-looking work items that don't fit in a single Linear issue yet, or that were surfaced as follow-ups during other work. Not a replacement for Linear — durable/cross-cutting items only.

## Open

- **Revisit the Supabase LEAKPROOF gap** (see [[DELTA_DECISIONS]] 2026-09-02) if `ltree` ancestor lookups (`path <@ ...`) show up as slow under RLS in practice. Currently accepted, not fixed.
- **Design a resolution-linkage between `document_references` and `reference_numbers`.** Confirmed 2026-09-04 these are NOT duplicates — different pipeline stages (see [[DELTA_DECISIONS]]) — but there's no mechanism today to promote an asserted `document_references` citation into a resolved `reference_numbers` row once it's matched, and no status field marking a citation resolved/unresolved. Needs: (a) a `status` or `resolved_entity_table`/`resolved_entity_id` column on `document_references`, or (b) a trigger/application step that inserts into `reference_numbers` on match and marks the source row resolved. Decision on which approach — not yet made.
- **Optional retailer↔carrier link table** — deferred. Only needed if a concrete cross-role netting use case shows up (e.g. same corporate parent is both a retailer and a carrier to the manufacturer). See [[DELTA_DECISIONS]].
- **`mv_deduction_summary`, `mv_reconciliation_kpi`, `mv_vendor_scorecard`** (if/when rebuilt against the real schema) will need renaming away from "vendor" terminology to match the carrier/retailer split, plus a refresh strategy (no `REFRESH MATERIALIZED VIEW CONCURRENTLY` cron, no unique index required for concurrent refresh decided yet).
- **`price_list`/`price_list_item`** were explicitly skipped (see [[DELTA_DECISIONS]]) on the assumption PO/invoice line prices are sufficient. Revisit if a pricing-master use case emerges.

## Sprint 1 remaining (per CLAUDE.md)

- AS2-9 — OpenRouter routing config (`usage_events` table still needs applying)
- AS2-42 — OTEL baseline
