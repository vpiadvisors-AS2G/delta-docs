# DELTA_DECISIONS

Durable architecture/scope decisions. Each entry: what was decided, why, and who/when. Newest first.

## 2026-09-04 — `document_references` vs `reference_numbers`: not duplicates, kept separate

Resolved by inspecting both migrations directly (`20260826000008_reference_numbers.sql`, `20260903000001_schema_part2_operational.sql`), closing the open question flagged since AS2-52 (2026-09-02).

- **`reference_numbers` (AS2-6):** a resolved, polymorphic reverse-lookup registry. `entity_table` + `entity_id` points at whatever real row already owns a given reference value, across many entity types (po/bol/pro/asn/load/container/seal/invoice/shipment/claim). Built *after* a value has been matched to something.
- **`document_references` (AS2-52):** scoped only to `invoices` (real FK, not polymorphic), and captures what an invoice *asserts* about another document — e.g. "this invoice cites PO #X" — as raw extracted text, before/independent of that citation being resolved to a real row. Narrower reference-type set (po/despatch_advice/receipt/contract/bol/other).
- **Decision: these are different pipeline stages, not the same concept.** `document_references` = unverified claim at extraction time; `reference_numbers` = verified pointer at resolution time. Keep both tables as-is, no merge.
- **Real gap identified (not resolved, needs a design call):** nothing currently promotes a `document_references` row into `reference_numbers` once it's matched, and nothing marks a `document_references` row as resolved/unresolved. See [[DELTA_PLAN]] for follow-up.

## 2026-09-04 — Fixed: `scripts/postgres.mjs` now loads `.env.local` itself

- **Context:** `MIGRATION_DATABASE_URL` had to be manually exported in the shell before running `migrate`, or the script silently fell back to `postgres://postgres@localhost:5432/delta_dev` and reported "up to date" against the wrong database. Bit us twice on 2026-09-02 (AS2-6, then AS2-53 the same day).
- **Fix:** the script now loads `.env.local` itself via `dotenv` (already a dependency, same pattern `scripts/smoke-test-openrouter.ts` uses) before reading `process.env.MIGRATION_DATABASE_URL`. `dotenv`'s default `override: false` means an already-exported shell value still takes precedence, so this is additive.
- **Fixed same session:** `scripts/smoke-test-openrouter.ts` had the same bug — bare `import "dotenv/config"` loads `.env` only, not `.env.local`, but the README states `.env.local` is the actual source of truth for `OPENROUTER_API_KEY`. Switched to the same explicit `.env.local`-first loading pattern as `postgres.mjs`. Verified the OpenRouter client (`packages/shared/src/llm/openrouter.ts`) reads `process.env` only inside functions, not at module top level, so import-order relative to `loadEnv()` doesn't matter here.

## 2026-09-04 — Exposed Supabase credentials rotated

Confirmed done by Venkatesh.

- **Context:** the `delta` role's database password and the `sb_secret_...` service role key were pasted in plaintext into a Cowork chat during the AS2-53 session (2026-09-02), flagged for rotation twice (2026-09-02, 2026-09-04) without confirmation.
- **Action taken:** new secret key generated via Supabase dashboard (Settings -> API Keys -> Secret keys), old exposed key deleted after confirming the new one worked. `delta` role password rotated via `ALTER ROLE delta PASSWORD '...'` against the live Supabase project. Both new values updated in `.env.local` (`SUPABASE_SERVICE_ROLE_KEY`, `DATABASE_URL`/`MIGRATION_DATABASE_URL`).
- **Consequences:** old exposed values are dead. Closes the outstanding item from the AS2-53 and 09-04 sessions.

## 2026-09-04 — AS2-10: Azure Service Bus confirmed over Storage Queue

Confirmed by Venkatesh, resolving the open conflict flagged in [[Three-Agent-Architecture-and-Messaging]] (vault).

- **Context:** AS2-10's ticket description said "Queue new files to `ingestion-queue` (Azure Storage Queue)," contradicting `CLAUDE.md`'s architecture section and stack table, which specify Azure Service Bus as the sole inter-agent messaging backbone for the choreographed 3-agent pattern.
- **Decision:** Azure Service Bus is the platform, full stop. The ticket's "Storage Queue" wording was a spec error, not an intentional exception — no cost/simplicity case was ever made for a one-off Storage Queue, and mixing transports would break the "any agent can subscribe with zero changes to existing agents" design goal `trace_id` lineage depends on.
- **Consequences:** AS2-10 corrected in Linear to specify `ingestion-topic` (Service Bus), not `ingestion-queue` (Storage Queue). No other tickets reference Storage Queue. Nothing in `CLAUDE.md` needs to change — this brings the one outlier ticket in line with the standing rule.

## 2026-09-04 — TypeScript confirmed as the only backend language; Python thread killed

Confirmed by Venkatesh.

- **Context:** An unsupervised claude.ai chat session (2026-09-04, no file-bridge access, drafts never saved to real files) proposed a parallel Python/FastAPI + direct Azure Postgres Flexible Server build, tracked in Asana, at ~$20-30/month spend — contradicting the actual applied stack (TypeScript, Supabase, $0 spend, Linear). That draft leaked into `02-Wiki` notes as if it were real; no Python code was ever built or applied against `wim_dev`.
- **Decision:** TypeScript end to end, per `CLAUDE.md`'s existing "no second language" rule, is reaffirmed as final — not reopened. Reasons: the entire applied stack (Azure Functions v4/Node 20, Zod-driven validation/types/OpenAPI, Supabase RLS) is already TypeScript-native; a second language duplicates schema/tooling work for no benefit (no ML/data-science workload exists in this product); and a single consistent stack is materially better for PE/strategic-acquirer technical diligence than a stack that reads as architecture-by-accident.
- **Consequences:** The Python/FastAPI/Asana narrative is retired from all active docs and Linear. It remains visible only in the Obsidian vault's raw session logs (`01-Raw`, `02-Wiki`) as historical record of how the confusion happened — not as project history to build from.

## 2026-09-02 — AS2-53 (schema part 3: links/settlement/reconciliation/ops) built, tested, and applied for real to Supabase

Built and applied in a Cowork session, confirmed by Venkatesh.

- **`supabase/migrations/20260904000001_schema_part3_reconciliation.sql`** — **29 new tables** across 5 groups: Links/allocation/inference (`match_links`, `fulfilment_balances`, `equivalences`, `variance_conditions`, `dispute_items`, `adjustments`, `adjustment_allocations` — 7), Settlement & deductions (`payments`, `remittance_advices`, `remittance_lines`, `deductions` — 4), Reconciliation & dispute (`match_profiles`, `reconciliations`, `reconciliation_lines`, `reconciliation_evidence`, `disputes`, `dispute_events`, `recoveries` — 7), Ops & governance (`api_keys`, `webhook_subscriptions`, `webhook_deliveries`, `dq_rules`, `dq_results`, `audit_log`, `import_jobs`, `onboarding_state` — 8), Derived (`leakage_events`, `recovery_metrics`, `reconciliation_trends` — 3). `usage_events` already existed from AS2-9 (2026-08-31) and was the only table in the ticket's original scope not newly created.
  - **Correction:** initial docs and Linear comments said "21 new tables" — a miscount. The correct figure is 29, matching the group breakdown above. The AS2-53 ticket title itself ("22 entities") doesn't match its own group totals either — a pre-existing inconsistency in the ticket, not introduced this session.
- **Four sum-integrity CHECK constraints**, each backed by an `AFTER INSERT OR UPDATE` trigger since Postgres CHECK can't reference sibling rows: `match_links.allocation_pct` per (from_table, from_id) sums ≤ 1 across active links — no double-counted money; `variance_conditions.contributes_amount` per reconciliation sums ≤ |delta_amount|; `adjustment_allocations.amount` per adjustment sums ≤ adjustment.amount; `deductions` has a plain CHECK `deducted_amount = invoiced_amount - paid_amount` (single-row, no trigger needed).
- **`match_links` never deletes — corrections supersede.** A partial unique index enforces one active link per (from_table, from_id, to_table, to_id) `where status = 'active'`; a correction sets the old row's `status = 'superseded'` and `superseded_by`, then inserts a new active row. Chain stays walkable via `superseded_by`.
- **`recompute_fulfilment_balance(tenant_id, order_line_id)`** — driven only by currently-active `match_links` rows (excludes superseded), upserts into `fulfilment_balances`. Not a trigger — called explicitly after a `match_links` change, same pattern as the rest of the schema (no auto-cascading recompute logic yet).
- **Polymorphic entity references** (`match_links.from_table/from_id`, `dispute_items.entity_table/entity_id`, etc.) use the same `entity_table text + entity_id uuid` pattern as `document_links`/`reference_numbers` (AS2-6/52) rather than typed FKs, since they must point at any of several line-item tables. Postgres can't enforce a polymorphic FK — same accepted tradeoff as those two tables.
- **Tested on scratch Postgres before applying:** full 25-file migration chain replayed clean, 0 errors. 6 explicit constraint-violation tests run and confirmed rejected. Supersede pattern verified end to end. `recompute_fulfilment_balance` verified to count only the active link.
- **Applied for real to `wim_dev`** via `scripts/postgres.mjs migrate` (Session Pooler, `postgres` role). Verified directly via `psql`: all 29 tables present in `public`, `delta_meta._migrations` shows the file applied, total `public` table count is 82 (53 prior + 29 new) — matches exactly.
- **Operational hiccup during this run, not a schema bug:** `MIGRATION_DATABASE_URL` was never actually added to `.env.local` (only `DATABASE_URL`, the restricted `delta` role, was present) — the first migrate/psql attempt silently fell back to local Postgres (`inet_server_addr() = ::1`), which happened to have a database also named `wim_dev` containing the old dead `DBScripts` schema (`parties`, `purchase_orders`, etc.), producing a confusing partial-match result before the mistake was caught. No data on the real Supabase project was at risk. Root cause is the same open item as before (`scripts/postgres.mjs` should load `.env.local` itself) — see [[DELTA_PLAN]].
- **Credentials were pasted in plaintext into the chat during this session** (the `delta` role's Supabase password and the `sb_secret_...` service role key, both from `.env.local`). Venkatesh was advised to rotate both. Not yet confirmed done — follow up.

## 2026-09-02 — Applied the retailer/carrier migration to the real Supabase project; accepted a Supabase-only LEAKPROOF gap

Confirmed by Venkatesh, executed in a Cowork session. Closes out the "known gap" from the entry below — AS2-6 is now genuinely done, not just scratch-tested.

- **Full migration chain (`20260826000001`–`20260902000001`) applied for real** to the live Supabase project `qikmbzrkkrkrdyuoswgy` (`wim_dev`), via `scripts/postgres.mjs migrate` over the Session Pooler connection (Direct connection needs Supabase's paid IPv4 add-on — not used, per spend rule). Verified directly via `psql`: `retailers`/`carriers` exist, `parties`/`party_roles` do not.
- **`scripts/postgres.mjs` doesn't read `.env.local` itself.** `MIGRATION_DATABASE_URL` must be exported in the shell session before running the script, or it silently falls back to local Postgres — this caused one run to falsely report "Migrations up to date" against the wrong (local) database. Worth fixing properly later (have the script load `.env.local`) so this can't happen again silently.
- **Supabase's `postgres` role doesn't own extension-created objects** the way a native Postgres superuser does — Supabase's own provisioning reassigns them (e.g. `ltree`'s functions) to an internal role. This broke the unconditional `alter function ltree_isparent/ltree_risparent ... leakproof` step in `20260826000001_extensions_and_helpers.sql`, which worked fine locally and in the scratch-instance test but failed on Supabase with "must be owner of function."
  - **Fix:** wrapped both `ALTER FUNCTION` statements in a `do $$ ... exception when insufficient_privilege then skip $$` block. Still applies on native local Postgres (where `postgres` does own the function); silently skips on Supabase, logging a `RAISE NOTICE`.
  - **Accepted tradeoff:** the GiST-index-under-RLS regression the original migration comment documented (ltree ancestor lookups falling back to Seq Scan under RLS) is now live on the real Supabase database. Not fixed — no current query volume where it matters. Revisit if `path <@ ...` lookups show up as slow in practice; the real fix is getting Supabase to grant `postgres` ownership of the extension's functions, not something this migration can force.
- **`delta` app role didn't exist on Supabase** — `scripts/setup-local-db.sql` only creates it locally, and `20260826000010_grants.sql` assumes it's already there. Created directly via `psql` (`CREATE ROLE delta LOGIN PASSWORD '...'`), not via a migration file. Password is Venkatesh's, not recorded here — needs to go into his `.env.local` as `DATABASE_URL` for the app to connect at runtime instead of the `postgres` superuser.

## 2026-09-02 — Retailer/Carrier split replaces AS2-6's parties/party_roles model

Confirmed by Venkatesh in a Cowork session, decided incrementally then executed. Supersedes part of the 2026-08-29 reconciliation entry below (that entry's "vendor/customer ... reused parties+party_roles" line no longer holds).

- **Domain has exactly three roles: Manufacturer, Retailer, Carrier.** The tenant *is* the manufacturer — no manufacturer table, no manufacturer_id column. "tenant"/"tenant_id" stay as the infra/multi-tenancy term (RLS, `current_tenant_id()`, auth claim) — explicitly not renamed. "Manufacturer" is business vocabulary layered on top of the existing tenant concept, not a schema change to it.
- **No supplier role exists in 3WIM's business.** AS2-6's `parties.party_type` included `supplier`/`third_party_logistics`/`bank`/`internal` — none of these map to a real counterparty in deduction or overbilling reconciliation. Dropped.
- **Party model (SCD-2 `parties` + `party_roles`) rejected in favor of two separate tables: `retailers` and `carriers`.** Reason: even where the same real-world company is both a retailer and a carrier to the same manufacturer, the two sides are run as operationally separate accounts — different billing departments, different contacts, different remit-to addresses. A shared party record with role tags doesn't reflect that; it would just push the same complexity into role-scoped overrides. Simpler to keep them as distinct tables.
- **`purchase_order`/`receipt`/`price_list` direction bug caught and noted for the AS2-52 rebuild** (not yet re-migrated as of this entry — see [[DELTA_PLAN]]): these were modeled as the manufacturer buying from a vendor. Correct direction for DEDUCTION mode is the retailer issuing the PO to the manufacturer.
- **`rate_agreement`/`shipment`/`freight_invoice` (AS2-52/53 concepts) should point at `carrier_id`, not a generic vendor/party id**, once those tables are migrated for real.
- **Retailers/carriers are plain current-state rows, not SCD-2 version-tracked** like `parties` was. Simpler, POC-appropriate. Revisit only if a real "retailer changed legal name/ownership" scenario needs history.
- **Known gap, deliberately deferred:** no link between a retailer row and a carrier row that are the same corporate parent (e.g. for netting a carrier's freight overbilling against a retailer's deduction for the same company). Build only if a concrete use case shows up — plan is a lightweight optional link table, not a merge back into a party model.
- **`party_aliases` dropped, not migrated.** Was unused — no seed data, no consuming code found in the reviewed migrations. Revisit if EDI-trading-partner-name fuzzy matching becomes a real need.
- **`party_identifiers` kept, renamed to `counterparty_identifiers`**, repointed to `retailer_id`/`carrier_id` (mutually exclusive). Still the home for DUNS/GLN/SCAC/VAT/LEI/Peppol identifiers.

**What shipped:** `supabase/migrations/20260902000001_retailer_carrier_split.sql` — creates `retailers`/`carriers`, backfills from existing `parties` rows (supplier-typed rows dropped, not migrated), repoints `addresses`/`contacts`/`bank_accounts`/`counterparty_identifiers`/`contracts`/`locations`/`party_hierarchy_nodes`, drops `parties`/`party_roles`/`party_aliases`. Replayed clean against the full 21-migration chain (all of `supabase/migrations/` through `20260831000003`) on a fresh local Postgres 16 instance — 0 errors, RLS/composite-FK conventions preserved, cross-tenant isolation smoke-tested under the `delta` role post-migration.

**What did NOT ship in this pass** (see [[DELTA_PLAN]] for follow-up): the `schema_part1/2/3.sql` standalone-`wim`-schema files built earlier in the same Cowork session (2026-09-02, before this decision) are now fully superseded — they predate the Manufacturer/Retailer/Carrier decision and were never reconciled into `supabase/migrations/`. Do not use them as a reference; `DELTA_STATE.md` needs its own update to stop pointing at them as current.

## 2026-08-30 — Two-stage thesis: $40-100K/mo personal income floor, then $40-50M personal wealth ceiling

Confirmed by Venkatesh, refined same day. Explicit goal is to avoid a famine-or-feast bet — sequence, not either/or:

- **Stage 1 (floor, near-term):** $40-100K/month personal income. Replaces a full-time job with room to grow — not a single fixed number like the earlier $50K MRR / $40K net income framing, but a range. At ~10+ mid-market customers and the already-decided $50-100K/year pricing, this is reachable well under the original $1-1.5M ARR / 10-customer target.
- **Stage 2 (ceiling, aspirational):** $40-50M personal wealth. Supersedes the earlier $10M-minimum framing as the actual number being aimed at once Stage 1 is secure — consistent with the architecture doc's $60-70M acquisition / 80% ownership math.

Does not change Sprint 1-7 scope or the 3-agent/API-first/OTEL architecture — those stay because they are cheap to build now and expensive to retrofit later if Stage 2 materializes. What it does change: if a build-vs-ship scope tradeoff comes up before customer 1, prioritize Stage 1 revenue-generating functionality over Stage 2 sellability polish (FedRAMP positioning, external API v2).

## 2026-08-29 — DBScripts schema reconciliation (schema_part1–3.sql)

Context: a request to generate DDL for a 78-entity schema (in a `wim` schema, standalone) turned out to conflict with the AS2-6 schema already built in `supabase/migrations/`. Reconciled instead of building a parallel schema. Confirmed by Venkatesh 2026-08-29. See [[DELTA_STATE]] for what actually got built.

**Superseded 2026-09-02** (see entry above): the "vendor/customer... reused parties+party_roles" line below no longer reflects the schema — parties/party_roles were dropped in favor of separate retailers/carriers tables. The rest of this entry's decisions (match_links, disputes/claim merge, deduction_type drop, price_list skip, freight_invoice folding, code_lists rename, AS2-6 conventions) still stand.

- **Target schema is `public`, not `wim`.** Matches the existing AS2-6 migrations, which are unqualified and land in `public`. The `wim` schema referenced in the original task exists locally but is unused.
- **`match_links` built now, not deferred to AS2-53.** It's the relationship engine CLAUDE.md's hard rule 5 describes ("do not add a shortcut foreign key between two documents"), but no migration had created it yet. Group 9 (Reconciliation) had no way to represent matches without violating that rule, so it's built in `schema_part1.sql` ahead of AS2-53.
- **`disputes` absorbs `claim`.** The original spec named both `claim` (deductions group) and `dispute` (dispute-workflow group) as separate entities describing the same workflow. Kept one table (`disputes`), dropped `claim`. A deduction opens at most one dispute (`uq_disputes_tenant_deduction`).
- **`deduction_type` dropped.** Folded into `reason_codes.category`, which already existed (dual-mode deduction/overbilling) and covered the same classification need. Avoids a redundant lookup table.
- **`price_list`/`price_list_item` skipped.** PO and invoice lines carry `unit_price` directly (UBL style) rather than resolving against a master price list. Revisit if a real pricing-master use case shows up.
- **`freight_invoice` folded into `invoices.shipment_id`.** One `invoices` table serves both procurement and freight invoicing (nullable `shipment_id` marks the freight case), rather than a parallel table — consistent with "mode is configuration, not a code path fork."
- **`reference_code` renamed to `code_lists`.** The original name collided in meaning with the existing `reference_numbers` table (a document reference-number registry — PO/BOL/PRO/etc. pointing at entities). `code_lists` is a distinct concept (UI/config lookup values) and needed a distinct name.
- ~~**vendor/customer/rate_agreement/rate_line_item are not new tables.** Reused `parties`+`party_roles` (role `supplier`/`carrier`/`customer`) and `contracts`+`contract_lines` (`contract_type = 'carrier_rate_agreement'`) respectively — both already existed in AS2-6 and covered these concepts.~~ — superseded 2026-09-02.
- **Every new table follows the existing AS2-6 conventions exactly**, not the originally-requested standalone style: `text ... check (col in (...))` instead of native Postgres `ENUM` types, composite tenant FKs (`(tenant_id, id)`, not bare `id`) per CLAUDE.md hard rule 2, and `alter table ... enable/force row level security` + `tenant_isolation` policy on every tenant-scoped table.

## Prior — AS2-6 (see git history / migration comments)

AS2-6's own decisions (SCD-2 parties, ltree hierarchies, composite tenant FKs retrofit, `nearest_ancestor_id()` resolution) are documented inline in `supabase/migrations/*.sql` comments and are not duplicated here. Start there for anything predating 2026-08-29. Note: the SCD-2 `parties` table itself was dropped 2026-09-02 (see top entry) — its ltree hierarchy and composite-FK conventions still apply schema-wide.


## 2026-08-31 — OpenRouter routing: config-driven, model choice is a Sprint-1 non-decision

Confirmed by Venkatesh. Model per route (extraction/classification/reconciliation/chat) is env-var-driven in `packages/shared/src/llm/openrouter.ts` — changing a model is a config change, never a code change. This was validated, not decided: wiring/retry/auth confirmed working against free-tier models (NVIDIA Nemotron), but free-tier output quality is not representative — extraction ignored format instructions, one chat call returned empty. Real model selection (Haiku/Sonnet vs. Gemini) is deferred until a golden-dataset accuracy pass, not before real documents run.

**Corrected 2026-09-04:** the `usage_events`/`model_pricing` migrations (`20260831000001`-`03`) *were* applied for real. Per-call token-logging inserts into `usage_events` are AS2-12's scope, not AS2-9's (see AS2-12's Linear description, "Added from AS2-9") — AS2-9's own done-when (all 4 routes return valid responses) was met and it correctly stays Done. This paragraph previously said the table was unapplied and blocking AS2-9 closure; that was stale/incorrect.
