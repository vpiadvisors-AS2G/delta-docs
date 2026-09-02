# DELTA_PLAN

Forward-looking work items that don't fit in a single Linear issue yet, or that were surfaced as follow-ups during other work. Not a replacement for Linear — durable/cross-cutting items only.

## Open

- **Promote `DBScripts/schema_part1–3.sql` into real migrations.** They're currently ad hoc DDL applied directly via `psql`, not checked into `supabase/migrations/` and not run by `scripts/postgres.mjs`. Effectively early groundwork for AS2-52 and AS2-53, done out of migration-chain order. Needs: numbering (`20260829xxxxxx_*.sql`), a pass to confirm every new table matches the retrofit pattern used for RLS/composite FKs (spot-checked during creation, not independently reviewed), and reconciling against whatever AS2-52/AS2-53's actual issue text says once picked up — the entity groups used here came from a different, non-Linear spec and may not match those issues verbatim. See [[DELTA_DECISIONS]] for what was built and why.
- **`mv_deduction_summary`, `mv_reconciliation_kpi`, `mv_vendor_scorecard`** are plain materialized views with no refresh strategy defined yet (no `REFRESH MATERIALIZED VIEW CONCURRENTLY` cron, no unique index required for concurrent refresh). Needs a decision on refresh cadence once there's real data volume to care about.
- **`price_list`/`price_list_item`** were explicitly skipped (see [[DELTA_DECISIONS]]) on the assumption PO/invoice line prices are sufficient. Revisit if a pricing-master use case emerges.

## Sprint 1 remaining (per CLAUDE.md)

- AS2-9 — OpenRouter routing config
- AS2-42 — OTEL baseline
