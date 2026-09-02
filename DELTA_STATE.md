# DELTA_STATE

Current state of the DELTA build. Update this when the state changes — not a changelog, a snapshot of "where things actually are right now."

## Schema

- **`supabase/migrations/20260826000001`–`000018`** (AS2-6): 34 tables in the `public` schema — tenants, users, parties/roles/identifiers/aliases, addresses, contacts, bank_accounts, currencies, fx_rates, uoms, item_categories, items, reason_codes, tolerance_policies, approval_policies, tax_categories, charge_codes, party_hierarchies/nodes, organisation_units, geographies, gl_accounts, calendar_periods, teams, locations, lanes, item_packaging, contracts, contract_lines, contract_terms, reference_numbers. Checked in, part of the real migration chain applied via `scripts/postgres.mjs`.
- **`DBScripts/schema_part1.sql`, `schema_part2.sql`, `schema_part3.sql`** (2026-08-29): 62 new tables + `match_links` + 3 materialized views, targeting the same `public` schema, reconciled against the AS2-6 tables above (reuse where an entity already existed, new tables only for genuine gaps — see [[DELTA_DECISIONS]]).
- **Local `wim_dev`**: all of the above applied and verified (104 base tables, 3 materialized views, 0 errors). The `wim` schema in that database exists but is empty and unused — everything lives in `public`.

## Known gap

`DBScripts/schema_part1–3.sql` are **not** numbered migrations. They were run directly via `psql`, not through `scripts/postgres.mjs`, and are not part of `supabase/migrations/`. A clean checkout that only runs the checked-in migrations will not have this schema. See [[DELTA_PLAN]] for the follow-up to promote them.

## Sprint 1 status (per CLAUDE.md)

1. AS2-7 (monorepo/CI) — done
2. AS2-8 (Functions scaffold/local dev) — done
3. AS2-6 (schema part 1) — done, migrated locally
4. AS2-52 (schema part 2: order/shipment/receipt/billing) — partially covered ad hoc by `DBScripts/schema_part1.sql`'s procurement/freight/invoicing groups, not yet a real migration
5. AS2-53 (schema part 3: links/settlement/dispute/ops) — partially covered ad hoc by `DBScripts/schema_part1.sql`'s reconciliation group (`match_links`) and `schema_part2.sql`'s dispute workflow, not yet a real migration
6. AS2-9 (OpenRouter routing) — not started
7. AS2-42 (OTEL baseline) — not started
