# DELTA_STATE

## 2026-09-20 — First real end-to-end dry run; Delta design applied; UI/RBAC gaps found

**The front half of the pipeline is proven, in a browser, on a real document.**
Not mocked, not unit-tested — clicked through. A freight invoice was uploaded
at `/upload` and came out the other end as `processed/11111111-1111-1111-1111-111111111111/2026-09-20/Freight Invoice1.jpeg`.

Proven this session:

1. **Login** — Supabase Auth plus a tenant-linked `users` row. No user existed
   and there is no signup flow; the account had to be created by hand in the
   Supabase dashboard plus a `users` insert with `status='active'` (the column
   defaults to `'invited'`, and the app silently bounces anything else back to
   login). Any fresh environment needs those two manual steps — a real gap.
2. **Upload (AS2-117)** — first live exercise. Writes to
   `incoming/<tenant>/`, then calls the extraction handler's HTTP trigger
   directly rather than waiting for the poll cycle.
3. **Extraction** — fields pulled correctly; classified `invoice`, so the
   `frt` trap (no transform exists for that doc type) did not fire.
4. **File lifecycle (AS2-13)** — code written 2026-09-18, never run until now.
   Success moved the blob to `processed/<tenant>/<date>/`. A failure would
   have gone to `error/` with an `ingestion_failures` row.
5. **Automatic transform (AS2-81)** — the reconciliation poller picks up
   `reconciliation_status='pending'` and runs the doc-type transform unattended.

**Where it stops, and this is the whole remaining gap.** `matchShipmentLegContract`
identifies which contract line governs a shipment and returns
`matched | lane_unresolved | no_ship_date | review_queued`. It never computes a
delta, never writes a `reconciliations` row, never assigns a reason code. The
deduction side has this piece (`match-deduction-compute.ts` + `match-deduction.ts`);
freight has no equivalent. So a freight invoice leaves the dashboard at zero and
the eight seeded overbilling reason codes unused. That is the next session's job —
see `docs/BATCH_PROMPT.md`.

**Delta design standard established.** "Delta design" now means the published
artifact canvas https://claude.ai/artifact/HutMmZyqbbML4itXqZDvkE. Applied to the
web app in `3d8440e`: sidebar shell, KPI strip with semantic colour (orange for
unresolved exposure, green for confirmed savings only), status chips, banner
bands, Ask Delta panel. Recorded in CLAUDE.md under "Delta design". **Conflict
resolved:** the `delta-ui-uxv1` skill mandated top nav plus Public Sans / IBM
Plex Mono; the canvas uses a sidebar and system fonts. Venkatesh chose the
canvas for look; the skill remains authoritative for terminology, personas,
metric definitions, colour semantics, chart types and accessibility. Do not
re-litigate.

**Environment traps found (all cost real time; documented in BATCH_PROMPT.md):**
`apps/web/.env.local` is separate from the root `.env.local` and Next.js does not
read the root one — credentials must be set in both. A leftover `func` process
silently held port 7071 and would have served stale pre-AS2-117 code. Azurite
stores blobs as GUID files with a JSON metadata db, so files copied into
`.azurite/incoming/` on disk are invisible to it. Last weekend's three test
uploads went to tenant `81abd2d8-...`, which is not seeded — the poller correctly
skipped them as an unknown tenant, silently, which is how a weekend of test
uploads went nowhere with no visible error.

**Persona/RBAC gap confirmed (AS2-101).** `users.tenant_id` is `NOT NULL` with an
FK and there is no role column of any kind. Delta Admin is therefore structurally
impossible today, not merely unbuilt — a cross-tenant user cannot be represented
even by hand-editing the database. Tenant-level personas (Company Admin / Analyst
/ Executive) have nothing to hang off either. Evidence added as a comment on
AS2-101. AS2-63 and AS2-89 look like duplicates of it.

**Not a demo blocker, noted:** the tenant was renamed from "Acme Retail Group" to
Meridian Fabrication Co — the seed name contradicted its own data (a retail group
whose customer is a retailer). AS2-17's tolerance-config CRUD remains unfinished
and uncommitted (no `page.tsx`), deliberately left out of the design commit.


Current state of the DELTA build. Update this when the state changes — not a changelog, a snapshot of "where things actually are right now."

## Schema

- **`supabase/migrations/20260826000001`–`20260902000001`** (AS2-6 + follow-ups, full chain including the retailer/carrier split): **applied for real to the live Supabase project `qikmbzrkkrkrdyuoswgy` (`wim_dev`)** on 2026-09-02, via `scripts/postgres.mjs migrate` against the Session Pooler connection, run by Venkatesh. Verified directly: `retailers` and `carriers` exist in `public`, `parties`/`party_roles` do not.
- **`20260826000001_extensions_and_helpers.sql` patched 2026-09-02**: the `alter function ltree_isparent/ltree_risparent ... leakproof` step now runs inside a `do $$ ... exception when insufficient_privilege then skip $$` block. Reason: Supabase reassigns extension-created objects (like `ltree`'s functions) away from the connecting `postgres` role, so the original unconditional `ALTER FUNCTION` failed there even though it worked on native local Postgres and the scratch-instance test. Applied on Supabase = skipped (logged via `RAISE NOTICE`); applied locally = still takes effect. Known, accepted tradeoff: the GiST-index-under-RLS regression the original comment described is now live on Supabase until ownership can be granted some other way. See [[DELTA_DECISIONS]].
- **`delta` app role created directly on Supabase** (2026-09-02, via `psql`, not via a migration file — `scripts/setup-local-db.sql` only covers local). Needed because `20260826000010_grants.sql` assumes the role already exists. Password set by Venkatesh, stored only in his own `.env.local` / password manager — not recorded here.
- **`supabase/migrations/20260903000001_schema_part2_operational.sql`** (AS2-52, 2026-09-02): 18 new tables — order-to-receipt (`orders`, `order_lines`, `despatch_advices`, `despatch_lines`, `receipts`, `receipt_lines`, `shipments`, `shipment_legs`, `shipment_counterparties`), billing (`invoices`, `invoice_lines`, `invoice_charges`, `invoice_taxes`, `document_references`), documents & extraction (`documents`, `extractions`, `extraction_fields`, `document_links`). Written fresh against the retailer/carrier decision — `orders.retailer_id` is NOT NULL (retailer issues the PO to the manufacturer, fixing the direction bug AS2-6 originally had), `shipments`/`shipment_legs` point at `carrier_id`, `shipment_counterparties` built with that name from the start. Replayed clean against the full chain (all prior migrations + this one) on a fresh scratch Postgres — 0 errors. 7/7 "Done When" checks verified directly: invoice-total CHECK rejects bad math, duplicate PRO-per-carrier rejected, duplicate document hash-per-tenant rejected, multi-leg shipment reconciles per leg, `orders.retailer_id` NOT NULL enforced, and a two-receipt partial/split reconciliation against one order line (60+2 then 38, against 100 ordered) resolves correctly. **Applied for real to `wim_dev` on 2026-09-02** — verified directly via `psql`: all 18 tables present in `public`.
  - Found and fixed one pre-existing gap while testing: `contract_lines` was never given `UNIQUE(tenant_id, id)` in AS2-6's composite-FK retrofit (only `contracts` itself was) — added here since `invoice_charges.contract_line_id`'s composite FK needs it.
  - Open design call, not fully resolved: `document_references` (new, per-invoice typed citation) vs. the existing `reference_numbers` (AS2-6, generic reverse-lookup registry) may turn out to be the same concept — built as distinct for now, flagged in the migration's own header comment and in [[DELTA_PLAN]] to revisit.
- **`supabase/migrations/20260904000001_schema_part3_reconciliation.sql`** (AS2-53, 2026-09-02): **29 new tables** — links/allocation/inference (`match_links`, `fulfilment_balances`, `equivalences`, `variance_conditions`, `dispute_items`, `adjustments`, `adjustment_allocations` — 7), settlement (`payments`, `remittance_advices`, `remittance_lines`, `deductions` — 4), reconciliation & dispute (`match_profiles`, `reconciliations`, `reconciliation_lines`, `reconciliation_evidence`, `disputes`, `dispute_events`, `recoveries` — 7), ops & governance (`api_keys`, `webhook_subscriptions`, `webhook_deliveries`, `dq_rules`, `dq_results`, `audit_log`, `import_jobs`, `onboarding_state` — 8), derived (`leakage_events`, `recovery_metrics`, `reconciliation_trends` — 3). `usage_events` already existed from AS2-9 and was the only table in the ticket's original scope not newly created. (Note: earlier session notes and Linear comments said "21 new tables" — that was a miscount; the correct count, matching the ticket's own L/F/G/I/J group totals, is 29. The AS2-53 ticket title itself says "22 entities," which doesn't match its own group breakdown either — a pre-existing inconsistency in the ticket, not something this session introduced.) Full 25-file chain replayed clean on scratch Postgres; all 4 sum-integrity CHECK constraints (match_links allocation, adjustment_allocations, variance_conditions, deductions math) and the supersede-not-delete pattern verified by deliberate violation. **Applied for real to `wim_dev` on 2026-09-02** — verified directly via `psql`: all 29 tables present in `public`, `delta_meta._migrations` shows the file applied, and total `public` table count is 82 (53 prior + 29 new), matching exactly. See [[DELTA_DECISIONS]].
- **`supabase/migrations/20260906000001_retailer_hq_and_bill_to_address.sql`, `20260906000002_carrier_hq_and_remit_to_address.sql`** (2026-09-06): 13 new columns each on `retailers` (`hq_*` + `bill_to_*` + `bill_to_same_as_hq`) and `carriers` (`hq_*` + `remit_to_*` + `remit_to_same_as_hq`) — addresses independent of `locations` (operational sites) and per-shipment `shipment_counterparties` roles. Both fully nullable except the same-as-hq flags. Tested against the full chain on scratch Postgres, then **applied for real to `wim_dev`** — verified directly via `psql`. See [[DELTA_DECISIONS]].
- **`supabase/migrations/20260906000003_generic_audit_trail.sql`** (2026-09-06): who/what/when history, per Venkatesh's explicit request for SCD-2-equivalent audit tracking. Not per-entity history tables and not true Kimball SCD-2 row-versioning (would require rewriting every FK across ~15+ tables that reference retailers/carriers/etc. by a stable id) — instead one generic `fn_audit_log_change()` trigger, reusing the existing `audit_log` table (AS2-53), attached to **77 tables** (every table with `tenant_id`+`id`, plus `tenants` itself; excludes `audit_log`, `usage_events`, `dq_results`, `webhook_deliveries` — already insert-only event streams). Captures full before/after row as jsonb on every INSERT/UPDATE/DELETE. "Who" reads the `app.actor_user_id` session GUC (same pattern as `current_tenant_id()`), falling back to `'system'` when unset. **Not yet wired into any app code** — every audit row will say `actor='system'` until an agent actually sets that GUC per request. Tracked in [[DELTA_PLAN]].
  - Found and fixed a second gap while building this: `retailers`/`carriers` were never wired to `trg_set_updated_at` (missed in the 2026-09-02 split) — `updated_at` had been silently stuck equal to `created_at` for both tables since creation. Fixed in the same migration.
  - Tested: 6 scenarios on scratch Postgres (actor defaults to `system`, actor captured when GUC set, INSERT/UPDATE/DELETE before/after semantics correct, `tenants` special-case correct, excluded tables produce no audit rows) all pass. **Applied for real to `wim_dev`** — verified directly: 77 distinct tables carry `trg_audit_log`. See [[DELTA_DECISIONS]].
- **`supabase/migrations/20260906000004_bol_pod_freight_docs.sql`** (2026-09-06): closes the Freight Overbilling doc-coverage gaps found when checking schema coverage of the 5 input documents (PO, Rate Contract, BOL, POD, Carrier Invoice). Adds: `shipments.order_id` (nullable FK -> `orders`, direct PO<->shipment link, was only reachable via `invoices.order_id`); `bills_of_lading` (structured BOL entity, replaces reliance on `shipments.bol_number` free text); `bill_of_lading_shipments` (junction table -- BOL<->shipment is N:M: a consolidated LTL load can share one BOL across shipments, a split shipment can span multiple BOLs); `proof_of_delivery` (previously had zero representation anywhere -- tied to `shipment_leg_id`, not `shipment_id`, since `shipment_legs.carrier_id` is where per-carrier M:N assignment actually lives; `shortage_qty`/`shortage_uom` gated by a CHECK to only apply on short/damaged/partial delivery_status). Also adds `'pod'` to `documents.doc_type` and `reference_numbers.reference_type` enums, and a clarifying comment on `shipments.carrier_id` ("primary/booking carrier only" -- `shipment_legs.carrier_id` is the real M:N source of truth). Logical model reviewed with Venkatesh via a PDF walkthrough before build (3 revisions: POD leg-level not shipment-level; BOL-shipment made N:M). 3 new tables, all auto-covered by the generic audit trigger. Tested: 6 scenarios on scratch Postgres (order_id FK, N:M BOL-shipment junction, per-leg POD, shortage constraint correctly rejected on non-exception status, pod enum value, audit trigger firing) all pass. **Applied for real to `wim_dev`** by Venkatesh via `node scripts/postgres.mjs migrate` (note: `postgres.mjs` could not be run from either the cloud session or the linked-device shell this session -- both lack raw TCP egress to Supabase's Postgres port, npm/HTTPS only -- so Venkatesh ran it directly). Verified directly via `psql`: all 3 tables present, `trg_audit_log` firing on all 3 (9 rows in `information_schema.triggers` = 3 tables x 3 events). See [[DELTA_DECISIONS]].
- **Current entity count in `public`:** 85 base tables (82 + 3 new 2026-09-06), plus `shipments.order_id`, 13 new columns each on `retailers`/`carriers`, and a generic audit trigger now on 80 tables. Verified directly, not projected.

## Superseded — do not use

- **`DBScripts/schema_part1.sql`, `schema_part2.sql`, `schema_part3.sql`** (dated 2026-08-29 in this file previously): these described a *different*, standalone-`wim`-schema effort built independently in a Cowork session on 2026-09-02, before the Manufacturer/Retailer/Carrier decision existed. They were never reconciled into `supabase/migrations/` and do not reflect the current schema. Treat `DBScripts/*.sql` as dead files pending cleanup — do not build on them.

## Architecture document

- **`AS2G/DELTA_Architecture_1.docx`** (2026-09-02): updated to match the Retailer/Carrier split — prose (§4.3, §4.4, §4.8, §4.11), all ~20 downstream appendix tables, Exhibit 12 entity counts (82→83), and Exhibits 5, 6, 11 diagrams all regenerated to show Manufacturer/Retailer/Carrier instead of the old party model. No known remaining party/parties references. Committed back to the same path.

## Freight Overbilling schema hardening — 2026-09-09

- **`supabase/migrations/20260909000001_freight_overbilling_hardening.sql`** (292 lines, drafted 2026-09-09): closes real-world gaps in the Freight Overbilling schema, found by (a) counting actual columns in the live migration files (not assuming), and (b) researching SAP TM / Oracle OTM / freight-audit industry practice. Scope explicitly limited to Overbilling mode per Venkatesh's direction ("initial focus is Freight Overbilling — get the schema to 95% real"); Deduction-mode gaps found in the same research pass are NOT included, tracked in [[DELTA_PLAN]] if needed later.
  - `fuel_surcharge_indices` (new): published index values (e.g. DOE diesel) by region/date — lets us independently verify a surcharge instead of trusting the invoice.
  - `fuel_surcharge_schedules` (new): per-contract peg + escalator-% formula, FK to `contracts`, `EXCLUDE USING GIST` to prevent overlapping effective-dated schedules on the same contract.
  - `contract_lines` widened: `rate_basis` (flat/per_mile/per_cwt/per_unit/percentage), `min_charge`, `max_charge`, mileage bands — the actual rate structure a bill gets checked against (Oracle OTM pattern).
  - `shipments` widened: `total_weight`(+uom), `total_pieces`, `declared_value`.
  - `shipment_items` (new): line-level declared weight/class/dimensions/NMFC code — the "shipper's truth" a reweigh/reclass dispute is checked against.
  - `invoice_charges` widened: `billed_rate`, `billed_quantity`(+uom), `billed_weight`(+uom), `billed_freight_class`, `calculation_basis`, FK to `fuel_surcharge_schedules`, `source_document_id`.
  - `accessorial_events` (new): `accessorial_code` (detention, reweigh, reclass, liftgate, lumper, redelivery, layover, residential, inside_delivery, other), `free_time_minutes`, `gate_in_at`/`gate_out_at`, approval fields — actual evidence for detention/lumper disputes, not just a billed line.
  - `trg_audit_log` attached to all 4 new tables. All new/altered tables carry composite `(tenant_id, id)` FKs and forced RLS per Rule 2/3.
- **Status: scratch-tested on local `delta_dev` 2026-09-09 — passed after 3 real bugs found and fixed** (not cosmetic — see [[DELTA_DECISIONS]] for detail): (1) 5 apostrophes in `COMMENT ON ... IS '...'` strings lost their SQL escaping (`''` → `'`) during file transfer to the device, breaking string literals; (2) three new tables (`fuel_surcharge_schedules`, `shipment_items`, `accessorial_events`) were missing the house-convention `unique (tenant_id, id)` constraint required for any table that's a composite-FK target — added, plus the same on `fuel_surcharge_indices` for consistency even though nothing FKs to it yet; (3) genuine pre-existing gap found: `users` never had `unique (tenant_id, id)` added by any prior migration (absent from `20260826000012_composite_tenant_fks.sql`) — this migration is the first to reference `users` via composite FK, so the constraint was added here rather than deferred. **Applied to `wim_dev` 2026-09-09** — Venkatesh ran `pnpm run db:migrate` from PowerShell (with `MIGRATION_DATABASE_URL` correctly pointed at the Supabase pooler via `.env.local`). Confirmed applied. See [[DELTA_DECISIONS]] and AS2-46 for the updated ticket scope.

## Freight Overbilling schema — day-to-day gap review, 2026-09-09

- Reviewed the 20260909000001 hardening against real day-to-day audit needs (not fraud/forensic needs). Two gaps judged worth closing now, one deliberately deferred — see [[DELTA_DECISIONS]] for the full reasoning.
- **`supabase/migrations/20260909000002_accessorial_rates_and_leg_level.sql`** (applied to `wim_dev` and scratch-tested on `delta_dev`, 2026-09-09):
  - `carrier_accessorial_rates` (new): per-contract rate + free-time allowance by accessorial code. Closes the gap where `accessorial_events.free_time_minutes` had nothing to check it against — this is the most common everyday Overbilling dispute (detention/lumper rate or free-time mismatch).
  - `accessorial_events.shipment_id` → `shipment_leg_id`: fixed to match `proof_of_delivery`'s existing correct pattern (carrier assignment is M:N at the leg level — a detention event belongs to one carrier's leg, not the whole shipment). Clean column swap, table was empty.
  - Added `accessorial_events.carrier_accessorial_rate_id` FK linking each event to the contract terms it should be checked against.
- **Deliberately NOT built:** independent evidence sourcing for gate timestamps/mileage (would need telematics/WMS/mileage-API integration — a real infra/spend decision, not a schema fix). Tracked in [[DELTA_PLAN]] as an explicit, named gap — auto-approve gates already route anything without sufficient evidence to manual review, so this isn't silently trusted.

## Credential rotation — 2026-09-08

- **All three Supabase credentials rotated and verified live**, closing out the two exposure incidents (2026-09-04, 2026-09-08). See [[Credential-Security-and-Rotation]] for full incident log.
  - `delta` role (`DATABASE_URL`): rotated via `ALTER ROLE`, verified.
  - `postgres` role (`MIGRATION_DATABASE_URL`): `ALTER ROLE` fails for this role even from an admin pooler connection ("Only superusers can alter privileged roles") — rotated via Supabase Dashboard → Database → Settings → Reset database password instead. Verified.
  - `delta_batch` role (`DELTA_BATCH_DATABASE_URL`): rotated via `ALTER ROLE`, verified. Also fixed a pre-existing bug found in the process — the value had its own key name duplicated inside it (`DELTA_BATCH_DATABASE_URL=DELTA_BATCH_DATABASE_URL=postgresql://...`).
- No credential value was written into any persisted chat transcript during the fix (generated and applied entirely within the linked-device shell / `.env.local`).
- **Unblocks**: merging PR #4 (extraction handler, AS2-12/13), which was gated on this per the wiki's own open action items.

## Sprint 1 status (per CLAUDE.md)

1. AS2-7 (monorepo/CI) — done
2. AS2-8 (Functions scaffold/local dev) — done
3. AS2-6 (schema part 1) — **done.** Full migration chain, including the retailer/carrier split, is live on the real Supabase project. Still open, but out of AS2-6's own scope: `purchase_order`/`receipt`/`price_list` direction fix and `rate_agreement`/`shipment`/`freight_invoice` → `carrier_id` repointing (both belong to AS2-52/53, called out in [[DELTA_DECISIONS]]).
4. AS2-52 (schema part 2: order/shipment/receipt/billing) — **done.** Live on the real Supabase project (see Schema section above).
5. AS2-53 (schema part 3: links/settlement/dispute/ops) — **done.** Live on the real Supabase project (see Schema section above).
6. AS2-9 (OpenRouter routing) — **done.** Wiring/retry/auth validated; `usage_events`/`model_pricing` tables applied for real via migrations `20260831000001`-`03`. The per-call insert into `usage_events` itself is AS2-12's scope (tenant-context Supabase writes), not AS2-9's — AS2-9's own done-when (all 4 routes return valid responses) was met. Real model selection (vs. free-tier) still deferred to a golden-dataset pass — tracked, not blocking.
7. AS2-42 (OTEL baseline) — **instrumentation baseline built 2026-09-04.** `packages/shared/src/telemetry/` (init.ts, logger.ts, metrics.ts, propagation.ts): config-driven OTLP trace+metric export (`OTEL_EXPORTER_OTLP_ENDPOINT`, no-ops with a warning if unset), structured JSON logger with `trace_id`/`tenant_id`/`document_id`/`duration_ms` fields, the 4 named custom metrics (`docs_ingested`, `extraction_latency`, `llm_tokens_spent`, `reconciliation_duration`), and Service-Bus trace-context inject/extract helpers ready for AS2-10/13 to use once real message-send/receive code exists. Wired into all 3 agents' startup and `health.ts` as a working example. **Not yet done:** full "single document, one connected trace across all 3 agents" — genuinely blocked on AS2-10/12/13 actually sending/receiving Service Bus messages, since there's no cross-agent call to propagate a trace through yet. `pnpm install` needs to be run to pull the new OTEL packages into the lockfile — not run this session (no `pnpm` available in the environment that wrote this).
8. AS2-10 (blob poller) — **code written 2026-09-04, not yet run against real Azure infra (none provisioned, per DELTA_CONSTRAINTS — no spend without approval).** `packages/shared/src/azure/` (blob.ts, service-bus.ts), `packages/shared/src/util/lru-set.ts`, `packages/shared/src/schemas/ingestion-message.ts`, `apps/agent-ingestion/src/functions/poller.ts` (TimerTrigger, every 5 min). Lists `incoming/<tenant_id>/*`, dedups against `processing/<tenant_id>/` contents plus known `documents.storage_path` rows, copies to `processing/<tenant_id>/`, deletes the source blob, and publishes one message per file to `ingestion-topic` with trace context injected (AS2-42's propagation helpers). Unknown top-level tenant folders are logged once via an LRU cap (max 1,000), then skipped, per spec.
   - **Design resolution, not in the original ticket text:** the poller does NOT insert into `documents` itself. `doc_type` and `file_hash` are only knowable once the file is read, which is extraction's job (AS2-12/13), not the poller's — inserting a row here would need a placeholder `doc_type` that fails the table's own CHECK constraint. The poller's handoff message (`ingestionMessageSchema`) carries what extraction needs to create the row.
   - **`delta_batch` role created 2026-09-04** directly on the live Supabase project via `psql` (same pattern as `delta` itself — not a migration file). Least-privilege: `SELECT` only on `tenants` and `documents`, nothing else — verified directly via `information_schema.role_table_grants`. Password stored only in Venkatesh's `.env.local` (`DELTA_BATCH_DATABASE_URL`), not recorded here. **Not yet wired into the poller code** — `apps/agent-ingestion/src/functions/poller.ts` still reads `tenants`/`documents` via the Supabase JS client with the service-role key (same as every other agent). Switching the poller to actually connect as `delta_batch` means adding a direct Postgres connection (the `pg` package) alongside/instead of `supabase-js` for those two reads — tracked as follow-up, not done.
   - **Not yet done:** `pnpm install` (to pull `@azure/storage-blob`/`@azure/service-bus` into the lockfile — no `pnpm` available in the environment that wrote this), `pnpm run typecheck` confirmation, and no Azure Storage account / Service Bus namespace exists yet to actually run this against — needs Venkatesh's approval to provision (~$13/mo projected, per DELTA_CONSTRAINTS) before AS2-10's own Done-When ("new file dropped in blob → appears on Service Bus topic within 5 min") can be verified for real.
9. AS2-12/13 (extraction handler — Ingestion agent, field extraction from queued documents) — **built 2026-09-06/07, not yet run against real infra** (same gap as AS2-10 — no Service Bus namespace, no reachable Postgres from an agent session; see the plan's "Known Environment Gaps"). `apps/agent-ingestion/src/functions/extraction-handler.ts` (Service Bus topic trigger), `apps/agent-ingestion/src/services/` (text extraction via `pdf-parse`, doc_type classification, tier routing, completeness, AS2-11 verification pass, Tier-1 rule-based field extraction), `packages/shared/src/llm/openrouter.ts` (+ `callOpenRouterVision`, free-tier rate/day gate), `packages/shared/src/llm/vision-model-resolver.ts` (queries OpenRouter live, never hardcodes a free vision model id), migration `20260907000001_extraction_tier_and_verification.sql` (`extractions.tier`/`tier_reason`, `extraction_fields.verified`). Full plan: `docs/superpowers/plans/2026-09-06-extraction-handler-as2-12-13.md`.
   - **Correction to the build prompt's own framing:** the prompt described `documents.doc_type` missing `'pod'` as an open gap to design around. It was already closed same-day by the (then-uncommitted) `20260906000004_bol_pod_freight_docs.sql` migration - this build treats all 7 doc types as first-class, no workaround bucket.
   - **Follow-up 1 (flagged, not built here):** Azure Document Intelligence remains the intended OCR path once real customer volume outgrows OpenRouter's free-tier rate limits (20/min, 50-1000/day) - swapping it in needs its own approval per DELTA_CONSTRAINTS and is out of scope for this ticket.
   - **Follow-up 2 (flagged, not built here):** the full DQ framework (§4.9: completeness/validity/uniqueness) is separate, larger work. This ticket's Tier-2→3 escalation only implements a narrow required-field-presence check.

## Document matching functions — 2026-09-09

- **`supabase/migrations/20260909000003_document_matching_functions.sql`** (applied to `wim_dev` 2026-09-09): SQL layer implementing Steps 0-3 of the Document Matching Algorithm (normalization, PO<->line matching, Rate Contract matching, BOL<->POD matching). Four functions: `fn_normalize_reference` (strip punctuation/whitespace, uppercase, strip leading zeros on purely-numeric values), `fn_match_order_line` (PO#/PO-Line# -> order_lines, tiered exact_line/exact_header/near_match), `fn_match_carrier_contract` (carrier+lane+mode+ship-date -> contract_lines, exact tier only), `fn_match_bol_pod` (BOL#/PRO#/Load# fallback chain, carrier-scoped, -> shipment_legs/proof_of_delivery). Each returns candidate rows tagged by `match_tier`; the calling application decides what to do — exactly one exact-tier row is a confident match, anything else (zero rows, multiple candidates, or only near_match) falls back to Step 5 AI-assisted fuzzy matching, per Venkatesh's explicit direction ("use SQL to join the doc and fallback on AI only when the deterministic join fails"). No schema changes — `invoice_lines.order_line_id`, `carriers.scac_code`, `contracts.contract_number`, `lanes`/`contract_lines.lane_id` were all already in place, verified against the live schema before writing. One new extension (`fuzzystrmatch`, for `levenshtein()` near-match support). Application-side fallback logic (Step 5) is not part of this migration.

## AS2-91 (EAV → standard entity tables) — done, merged to main — 2026-09-10

All 5 doc-type transforms (invoice, PO, BOL, POD, Rate Contract) built, unit-tested, and verified end-to-end against real `wim_dev` data via a new integration smoke test (`pnpm run test:integration:eav`, `apps/agent-reconciliation/scripts/`). Merged to `main` via `0f9ba37`. Files: `apps/agent-reconciliation/src/services/transform-{invoice,po,bol,pod,rate-contract}*.ts`, `apps/agent-reconciliation/src/functions/transform-*.ts`, `packages/shared/src/schemas/{invoice,po,bol,pod,rate-contract}-entities.ts`, `packages/shared/src/schemas/dq.ts`. Full detail and every bug found/fixed along the way in `docs/DELTA_DECISIONS.md` (multiple 2026-09-10 entries).

Real schema bugs caught by the integration test (not by unit tests, which never touch the real schema): `extractions.tier`/`tier_reason` NOT NULL (missed reading only the original `create table`, not later `alter table`s), `shipments.pro_number` wrongly NOT NULL (fixed via `20260910000001_shipments_pro_number_optional.sql`), BOL/POD `source_document_id` was pointed at `extraction_id` instead of the real `document_id`, `contract_lines.rate_basis` (added by the 2026-09-09 hardening migration) wasn't captured at all by the transform. **Lesson recorded in DELTA_DECISIONS: grep `alter table <name>` across ALL migrations before writing code against a table, not just its `create table`.**

**Still HTTP-triggered, not wired to any real event.** AS2-92 (call-site wiring) is the next real gap — nothing invokes `transformInvoice`/`transformPo`/etc. automatically after extraction completes.

**Correction to item 9 above:** AS2-12/13 (extraction handler) is confirmed merged to `main` (PR #4, commit `311306c`) — Linear had it showing "Todo" despite being live; corrected 2026-09-10 while auditing branch state (see below). The "not yet run against real infra" caveat in item 9 still stands for the Service-Bus-triggered path specifically (no Service Bus namespace provisioned) — only the EAV transform layer (AS2-91, this entry) has been exercised against real data so far, via direct function calls, not via a real Service Bus message.

## Branch audit and cleanup — 2026-09-10

Checked every branch besides `main` for real unmerged work rather than assuming:
- `worktree-agent-reconciliation-v1` — 0 commits ahead of `main` (fully merged earlier this session). Deleted (local + remote).
- `vpiadvisors/as2-7-...`, `vpiadvisors/as2-8-...` — ancient Sprint-1 scaffold branches; diffing against `main` showed ~45k net deletions if merged (they predate nearly everything now in `main`). Not real unmerged work. Deleted (already gone from origin by the time this was checked — likely cleaned up earlier and just stale in a cached branch listing).
- `worktree-extraction-handler-as2-12-13` — showed "16 commits ahead" by commit-ancestry, but a real file diff against `main` showed **zero difference** in `apps/agent-ingestion/` — the work was already on `main` via a squash-merge (PR #4), which doesn't preserve commit ancestry even though the content landed. Confirmed via `git diff --stat`, not commit count. Being deleted (local + remote) as a final step.

**Only `main` remains after this cleanup — no other branch has real unmerged work.**

## AS2-66 — NOT approved — 2026-09-11

Same-day Standard-tier approval was retracted by Venkatesh ("I am not approving Service Bus Standard tier"). No Service Bus spend is authorized. AS2-66 stays blocked/unprovisioned. AS2-92 remains blocked on the queue-triggered path; only the HTTP-triggered interim stand-in is viable until further notice. See DELTA_DECISIONS.md (2026-09-11 retraction entry).

Nothing provisioned yet — no `az` commands run this session. Next step: Venkatesh provisions the namespace + topics from his own terminal, then AS2-92 (call-site wiring, still HTTP-triggered today) can be built against the real topic instead of the interim HTTP endpoints.

## 2026-09-11 -- AS2-92 schema stress-test: rate_confirmations + po_number chain, verified

Built and verified (tests 153/153 passing, typecheck clean across all 5 workspace projects), not yet applied to Supabase:

- New `rate_confirmations` table (migration `20260911000002`) -- shipment-specific accepted carrier rate, distinct from standing `contract_lines`. Storage only; matchCarrierContract does not yet consult it (follow-up).
- `match_review_queue` (migration `20260911000001`, still unapplied) doc_type extended with `'rate_confirmation'`.
- Invoice PO-number chain closed: `po_number` added to invoice's `REQUIRED_FIELDS_BY_DOC_TYPE` and Tier-1 regex extraction; `transform-invoice.ts` now writes a `document_references` row (reference_type='po') when a PO number is found, and returns it on `TransformInvoiceResult`. New `insertDocumentReference` query, `documentReferenceInsertSchema` in `@delta/shared`.
- `classify.ts`'s `MatchResult` (ambiguous/unmatched) now carries a `reason: string`, feeding `match_review_queue.reason` once matchers write there.

Still not done: matchOrderLine/matchCarrierContract/matchBolPod/matchDeduction still have no real caller (the original AS2-92 gap) -- this closed the *data* gaps found while scoping that work, not the wiring itself. matchBolPod's dual-call-path overlap with transform-pod/bol.ts is still an open decision (paused, not resolved). po_line_number (line-level PO matching) remains unbuilt, tracked separately.

Two migrations pending your `psql`/Supabase apply: `20260911000001_match_review_queue.sql`, `20260911000002_rate_confirmations.sql`.

- **`supabase/migrations/20260911000001_match_review_queue.sql`, `20260911000002_rate_confirmations.sql`** — **applied for real to `wim_dev` on 2026-09-11** by Venkatesh via `pnpm run db:migrate`. Verified directly via `psql`: both tables present with exact expected shape (constraints, RLS `tenant_isolation`, `trg_audit_log`/`trg_set_updated_at`, FKs on `rate_confirmations` to shipments/carriers/documents all composite `(tenant_id, id)`), `delta_meta._migrations` shows 2 rows for `20260911%`. AS2-92 wiring (matchOrderLine service+endpoint, matchDeduction review-queue writes, transform-pod refactor to shared matchBolPod) now has real tables to write to — not yet smoke-tested end-to-end against live data (next step).

- **AS2-92 end-to-end verified against real `wim_dev` data (2026-09-11)**: extended `apps/agent-reconciliation/scripts/integration-test-eav-transforms.ts` (`pnpm run test:integration:eav`) with two `matchOrderLine` checks — matched (PO seeded via transformPo, invoice seeded via transformInvoice with a matching po_number, `matchOrderLine` called for real, `invoices.order_id` confirmed updated in the DB) and unmatched (invoice with a po_number that resolves to no order, confirmed a real row lands in `match_review_queue` and is read back). Both passed. `matchDeduction`'s and `transform-pod`'s review-queue writes are still only covered by unit tests with fake deps, not this real-DB script — flagged as optional follow-up if full e2e coverage of all three call sites is wanted.

- **Ingestion↔Reconciliation field-vocabulary fix verified (2026-09-11)**: `pnpm --filter agent-ingestion test` 40/40 passed, `pnpm run typecheck` clean across all 5 workspace projects. See [[DELTA_DECISIONS]] for the bug and fix detail. Next: real end-to-end run of the 10-document freight test suite through the actual Tier-1/2/3 ingestion pipeline into reconciliation.

## 2026-09-12 — AS2-95 done: Tier-3 vision 400 fixed (bad model + DQ-escalation routing bug)

Real-doc harness run surfaced this: `OPENROUTER_MODEL_EXTRACTION` (`nex-agi/nex-n2.5-mini:free`) claimed vision support in OpenRouter's catalogue but 400'd on real `image_url` payloads. Swapped to `inclusionai/ling-3.0-flash-vl:free`, confirmed 200 OK against a live POD image via a standalone script before touching `.env.local`.

That fix alone didn't close it — re-running the harness showed the POD (real image) doc now passing, but the Rate Confirmation (`text/plain`) doc still 400'd on the same, working model. Root cause was one level up: `tier-routing.ts`'s `decideTier()` escalated to Tier 3 vision on any Tier-2 completeness failure with no check that the document actually had an image — a text document that failed Tier 2 had its raw bytes base64'd and sent to `callOpenRouterVision` as "image data," which OpenRouter correctly rejects regardless of model choice.

Fixed: `decideTier()` now takes `hasVisualRepresentation` and returns a new `DQ_ESCALATION_NO_IMAGE` outcome (stays Tier 2, no vision call) when there's no image/PDF to escalate to; throws `UnroutableDocumentError` into the existing `insertFailedExtraction` path rather than calling Tier 3 on non-image bytes. Same fix applied to `text-extraction.ts`'s unrecognized-mime-type fallback, which had the identical bug (silently treated any unmapped mime type as "route to vision"). `pnpm --filter agent-ingestion test` 45/45 passing (added coverage for both new failure paths, rewrote the one existing test that had encoded the bug as expected behavior), `pnpm run typecheck` clean across all 5 workspace projects. Confirmed against the real-doc harness: Rate Confirmation now fails cleanly with `DQ_ESCALATION_NO_IMAGE`, never calls vision. Linear: AS2-95, marked Done.

Not fixed here, tracked as follow-on: the Rate Confirmation's Tier-2 failure is itself downstream of a classification bug (`classify-doc-type.ts` misclassifies it as `bol`, so completeness is checked against the wrong doc_type's required fields) — see Open Action Items below.

## 2026-09-13 — AS2-96 done: classify-doc-type fixed, all 10 real-doc harness fixtures now classify correctly

Two distinct bugs in `classify-doc-type.ts`, not one, both surfaced by the AS2-95 harness run: (1) raw pattern-match counting structurally favored doc types with more listed patterns (`bol`=4) over ones with fewer (`rate`=2) regardless of match specificity — a real Rate Confirmation's generic freight boilerplate ("Shipper", "Consignee") out-scored its own "Rate Confirmation" self-naming match, 2 raw hits beating 1. (2) Filename-only classification (used when there's no OCR text, e.g. an image) broke on `\b`-anchored regex against underscore-separated filenames — `pod_delivered.png` scored 0 against every doc_type (`_` is a regex word character, so no boundary before "delivered") and silently defaulted to whichever `DocType` is declared first in the schema (`invoice`).

Fixed: `DOC_TYPE_PATTERNS` (`extraction-config.ts`) is now `WeightedPattern[]` per doc_type — self-naming phrase (weight 3) > type-specific code prefix (weight 2) > shared vocabulary (weight 1) — scored as a normalized fraction of that doc_type's total possible weight, not a raw count. Filenames get `._-` separators replaced with spaces before matching. `CLASSIFICATION_AMBIGUITY_MARGIN` changed from an integer raw-count margin (1) to a normalized-fraction margin (0.25), and the ambiguity check only applies once a real second candidate has scored above 0 — a lone weak match with no competing candidate isn't "ambiguous" just because its own normalized score is low (this exact edge case broke 2 of the 3 `extraction-handler.test.ts` invoice tests during review; fixed before merge).

`pnpm --filter agent-ingestion test` 39/39 passing (5 new regression tests: Rate Confirmation vs BOL with shared vocabulary, POD image filename, underscore/hyphen filename normalization), `pnpm run typecheck` clean across all 5 workspace projects. Confirmed against the real-doc harness: all 10/10 documents now show `classified_doc_type` matching `expected_doc_type`, including Doc 2 (`rate`, was `bol`) and Doc 4 (`pod`, was `invoice`). Linear: AS2-96, marked Done. PR #7, squash-merged to main.

Not addressed here (flagged in the ticket as possibly separate, confirmed separate on inspection): the Tier-1 70% coverage threshold letting a clean PO (Doc 1) complete despite a missing required field (`retailer_identifier`) is a `completeness.ts`/threshold-tuning question, unrelated to classification.

## 2026-09-18 — Verification pass: all 10 queued tickets confirmed working end-to-end, merged to main

Prior session (2026-09-17) had implemented AS2-105/111/80/81/109/108/110/21/22/23 and committed locally but never verified against a real environment. This session ran the full verification chain Venkatesh required before merge, found and fixed real bugs along the way, then pushed.

**AS2-23 gap closed**: review-queue resolve action now does a real `match_links` insert (from `match_review_queue`'s stored candidates, reusing `matchCarrierContract`'s contract-line shape) instead of a stub — server re-fetches the queue row and validates the chosen candidate server-side rather than trusting client-submitted form fields.

**Real bugs found and fixed while verifying (not environment noise)**:
- `extraction-handler.test.ts`'s fake-Supabase fixture never initialized the `ingestion_failures` table (added for AS2-108), so every Tier-2/3 failure test threw on `inserted[table]!.push`.
- `retry-with-backoff.ts`'s `PermanentApiError.cause` needed an `override` modifier (TS4114, ES2022 lib).
- `document-detail.ts`: `disputes!inner(...)`'s generated type still allows an empty array (Supabase doesn't encode `!inner`'s cardinality) — non-null asserted with a comment, matching existing repo convention.
- `apps/web`: `@types/react`/`@types/react-dom` 18.3.x don't declare `useFormState`/`useFormStatus` or the form-action-as-server-action merge point in their stable channel (only in `canary.d.ts`), even though react-dom@18.3.1's runtime ships both. Added a documented ambient `.d.ts` augmentation (`apps/web/src/types/react-dom-form-hooks.d.ts`) rather than upgrading React.
- `apps/agent-reconciliation/package.json` was missing `@supabase/supabase-js` as a declared dependency — `reconciliation-poller.ts` imports it directly, only ever worked before via phantom/hoisted resolution. A real (non-hoisted) `pnpm install` on Venkatesh's machine caught it immediately via `tsc`.
- `integration-test-real-documents.ts` (the real e2e script): its `ExtractionIO` literal stubbed `publishCompletion` but not `moveToProcessed`/`moveToError`, both required on the interface. `tsx` doesn't type-check, so this ran for months(?) without ever surfacing as a build error — every document actually completed correctly, then the script threw "is not a function" afterward, logged and swallowed by `runExtraction`'s own try/catch. Easy to miss next to a wall of "[OK]" lines. Added no-op stubs with a comment (blob storage is bypassed for this script by design, so there's no real blob to move).

**All four of Venkatesh's gates verified on his own machine**, in order: `pnpm install` (clean), `pnpm run db:migrate` (all 4 pending migrations — `ingestion_failures`, `reconciliation_status`, `document_reference_line_number`, `lane_geography_mdm` — already applied), `pnpm test` (all 5 workspaces, 86+54 = matches expected counts across `@delta/shared`/`agent-ingestion`/`agent-reconciliation`), `pnpm run typecheck` (all 5 workspaces clean), and a real end-to-end run of the 10-document Gemini test set (`pnpm run test:integration:real-docs`) against real `wim_dev` — all classified doc_types matched expected, tier routing correct, doc 8's intentional missing-carrier case correctly failed rather than silently passing.

**Merged**: `git push origin main` — local main (16 commits ahead, spanning back to AS2-22) is now `origin/main` at `45d1909`. Nothing provisioned, no spend incurred.

Still open: AS2-92's queue-triggered path remains blocked on Service Bus (not approved — see DELTA_DECISIONS 2026-09-11). `integration-test-real-documents.ts` still doesn't chain into `agent-reconciliation`'s transform-*/matchOrderLine — same documented follow-up as before, now with a clean ingestion-side baseline to build it against.

## 2026-09-18 (later) — Demo re-prioritization: AS2-117 upload UI + AS2-118 dashboard rewrite built, verified in sandbox, committed. Real click-through still blocked on env config.

Demo date moved to 2026-09-24. Batch prompt rewritten mid-session around a 6-step demo script; AS2-18/AS2-15 and most infra/spend tickets explicitly cut for this week (see DELTA_PLAN). Reprioritized to: AS2-16 (already drafted) → AS2-117 (upload UI, top priority — "nothing is demonstrable without it") → AS2-118 (dashboard).

**AS2-16 (reason-code engine)**: committed `82c3dc6` earlier this session (see prior entry above — draft review caught a real sign-convention bug in the SHORT_SHIP mapping before commit). **Migration now confirmed applied** — Venkatesh ran `pnpm run db:migrate` clean. This session still cannot reach `*.supabase.co` from either the device shell or the cloud sandbox (egress allowlist permits npm registry, not Supabase — confirmed via direct DNS/HTTPS test from both), so the live schema/data was not independently re-verified; trusting Venkatesh's confirmation.

**AS2-117 (browser file upload) — commit `1e659ee`.** New `/upload` page + form (`useFormState`/`useFormStatus`) → server action writes to `incoming/<tenant_id>/<filename>` via new `uploadIncomingDocument()` blob helper → calls `postIngestionMessageHttp` synchronously (no 5-min poll wait), same HTTP-trigger path as AS2-102. Nav link added. Verified in the cloud sandbox: `@delta/shared` build clean, `@delta/web` typecheck clean, `@delta/web` build succeeds for `/upload` (same as every other authenticated page, once dummy Supabase env vars are supplied — matches the pre-existing pattern on `/`, `/documents`, `/review-queue`).

**AS2-118 (dashboard reconciliation outcomes) — commit `029a7d2`.** `getDashboardSummary` rewritten to read `reconciliations` (previously read `extractions.reconciliation_status`, which only recorded "does this need reconciling," never the outcome) — counts by status, total delta at risk (variance+disputed, unresolved). Dashboard now lists recent reconciliations with invoice number + reason code, linking to a new `/reconciliations/[id]` detail page: expected vs actual vs delta, each variance condition with its own reason code, plus supporting `reconciliation_lines`. Verified in the cloud sandbox: typecheck clean, build succeeds. **Real gap found, not fixed here, filed as AS2-120 (tech-debt)**: no confidence score exists anywhere on `reconciliations`/`variance_conditions` — only `extraction_fields.confidence` exists, which measures something different (extraction confidence, not match confidence). Shipped without one rather than fabricating a number.

**Both new pages verified only via typecheck/build/test in the cloud sandbox — NOT via an actual browser click-through.** This session cannot run long-lived local dev services (Azurite, `func start`, `next dev`) through the device-bridge's ephemeral-call model. Real environment gap, confirmed by inspection: `.env.local` (both root and `apps/web/`) has none of `AZURE_STORAGE_CONNECTION_STRING`, `MESSAGING_MODE`, `EXTRACTION_HANDLER_HTTP_URL`, `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` set. Needs Venkatesh to set these (local-http mode against Azurite) and run the upload once, on his own machine, before the code above can be called demo-verified rather than just build-verified.

### Where the 6-step demo script actually stands right now

1. Log in — pre-existing (AS2-21), not touched this session, presumed working.
2. Upload a document from the browser — **code built (AS2-117), not click-through tested.** Blocked on env config above.
3. Extract/classify in seconds — the underlying HTTP-trigger path (AS2-102/AS2-12/13) is pre-existing and was run against real docs in the 2026-09-18 verification pass; AS2-117 calls it synchronously instead of waiting on the poller. Not re-verified through the browser this session.
4. Match against invoice, reason code + confidence — matching/reason-code logic real (AS2-16, this session + earlier). **Confidence score does not exist** (AS2-120) — this part of the demo script needs to be dropped or reworded.
5. Dashboard: matched/variance counts, dollars at risk — **code built (AS2-118)**, not click-through tested.
6. Open one result, per-line detail with reason code — **code built (AS2-118's `/reconciliations/[id]`)**, not click-through tested.

Net: steps 2, 5, 6 are code-complete and sandbox-verified but need one real local run to confirm; step 4's confidence-score sub-ask isn't buildable without a schema/product decision (AS2-120); step 1/3 are pre-existing and presumed fine but not re-checked this session.

**Linear updated**: AS2-16/AS2-117/AS2-118 all carry 2026-09-18 status notes with commit SHAs; AS2-120 filed (tech-debt) for the missing confidence score.

## 2026-09-20 — AS2-123/124 built + verified: overbilling engine produces a real dollar finding with a reason code. AS2-125 checked (no change needed). AS2-126 filed.

Per `docs/BATCH_PROMPT.md`'s Step 1-3: build the freight overbilling comparison engine, wire it to auto-trigger, confirm the UI renders it.

**AS2-123 (comparison engine) — commit `e0bd3dc`.** `match-overbilling-compute.ts` (pure) + `match-overbilling.ts` (I/O orchestrator) + `functions/match-overbilling.ts` (HTTP endpoint), same split as `match-deduction-compute.ts`/`match-deduction.ts`. Compares billed freight/accessorial/detention charges against the resolved contract line and shipment evidence; five outcomes: rate_mismatch, weight_discrepancy, accessorial_unauthorized, detention_unsupported, duplicate_invoice. `contract_lines.lane_id` is still unresolved (no geography MDM — AS2-59/94), so contract-line selection is deliberately lane-agnostic (carrier + mode + freight_class + weight band); a charge with no lane-agnostic candidate returns `rate_unresolved` rather than guessing, same posture as `matchShipmentLegContract`'s `lane_unresolved`. Reason codes resolved via a new `OverbillingCategory` mapping in `reason-code.ts`, run alongside (not replacing) the existing `condition_type`-based resolver — the 8 seeded overbilling reason codes don't collapse cleanly onto the 6 DB `condition_type` values.

**Real gap found while building, not fought, filed as AS2-126 (tech-debt)**: `invoices.shipment_id` is never populated anywhere in the transform pipeline — only `carrier_id` is set. There is no freight-side equivalent of the deduction side's `po_number` → `document_references` → `matchOrderLine` chain. Worked around with a documented single-shipment-per-carrier fallback (`resolveShipmentIdForCarrier`/`findDuplicateInvoiceForCarrier` in `queries.ts`) — correct only when a carrier has exactly one shipment in the demo tenant's seed data. Real fix needs a PRO/BOL-number extraction + `document_references` row + a shipment-matching resolver built on the already-existing-but-unused `fn_match_bol_pod` SQL function.

**AS2-124 (poller auto-trigger) — commit `c76c8af`.** `reconciliation-poller.ts`'s `resolveOutcome` now calls `matchOverbilling` in the same poll cycle once a freight invoice's transform yields an `invoice_id` — same "persisted status + periodic poll" pattern as the rest of the poller (AS2-81). Never throws: not-applicable and shipment-unresolved outcomes are folded into the recorded result string, not treated as a row failure.

**AS2-125 (dashboard/detail rendering) — checked, no code change needed.** Read both `getDashboardSummary` and `getReconciliationDetail` end to end: neither has any deduction-mode-specific assumption. Both key off `reconciliations`/`reconciliation_lines`/`variance_conditions` generically (`entity_table`, `condition_type`, `reason_code_id`) and both already handle a null `invoice_id`/`order_id`. An overbilling reconciliation (null `order_id`, `entity_table='invoice_charges'`/`'invoices'`) renders through the exact same code path as a deduction one. Confirmed by inspection, not by a live click-through (see environment note below).

**Verification: `pnpm --filter @delta/shared run build`, `pnpm -r typecheck`, `pnpm -r test` all pass** — `packages/shared` 69, `agent-ingestion` 54, `agent-reconciliation` 127 (127 = 103 pre-existing + 24 new, added for AS2-123's acceptance criteria: rate match/mismatch/unresolved, weight discrepancy, accessorial authorized/unauthorized, detention supported/unsupported, duplicate invoice, multi-category variance-sum capping). Two real type errors surfaced by the actual `tsc` run (a `find()` result typed `T | undefined` returned where `T | null` was expected; a type-narrowing cast across `TransformFn`'s deliberately-erased `{status: string}` return type) — both fixed, not environment noise.

**New environment trap, added to the traps this project already knows about**: the device-bridge mount (`C:\Users\viyer\Claude\Dev\delta` reached from Cowork's cloud sandbox) is unusably slow for `node_modules`-scale filesystem operations — a `pnpm install` there did not complete inside repeated 180-second command budgets, and even a `find`/`du` over `.pnpm-store`/`.azurite`/`.postgres` timed out. Root cause isolated to those three local-service data directories (`.azurite`, `.postgres`, `.pnpm-store`) and `node_modules` itself — thousands of small files/symlinks over the mount, not the tracked source tree (a `tar` of source only, excluding those, is ~2MB and fast). Worked around this session by tarring the source tree (excluding `node_modules`/`.git`/`.azurite`/`.postgres`/`.pnpm-store`/`.next`/`graphify-out`/`graveyard`/build output), staging that single small archive into the cloud sandbox's own fast local disk, and running the real verification gates there — a scratch copy used only for running `pnpm install`/`build`/`typecheck`/`test` fast; every source edit was still made on, and is only kept in, the real repo via the device bridge. Worth remembering for any future session doing similar work: don't attempt `pnpm install`/`node_modules` operations directly over the device-bridge mount.

**Still not click-through tested**: same limitation as the 2026-09-18 entry above — this session cannot run long-lived local dev services (Azurite, `func start`, `next dev`) through the device-bridge's ephemeral-call model, so AS2-124's auto-trigger and AS2-125's rendering are sandbox/code-verified, not confirmed by an actual upload → poller → dashboard run. Needs Venkatesh to run once on his own machine.

### Demo script status update (see 2026-09-18 entry above for steps 1-3)

Step 4 ("Match against invoice, reason code + confidence") for the **overbilling** path is now code-complete: a freight invoice with a rate/weight/accessorial/detention mismatch produces a real `reconciliations` row with a dollar `delta_amount` and a headline reason code, exactly like the deduction path already did. The confidence-score sub-ask (AS2-120) is still unbuilt for both modes. Steps 5-6 (dashboard, detail view) already render whatever mode a reconciliation belongs to with no additional work — confirmed by AS2-125's check above.

**Linear updated**: AS2-123/124/125/126 all carry 2026-09-20 status notes with commit SHAs (AS2-125 notes "no change needed, verified by inspection" — no commit to reference).

## 2026-09-20 — First real dollar finding produced end to end; two upstream gaps found and filed (AS2-127)

Re-ran the same test invoice (extraction `0683ec17-3973-4d10-9d0b-b5917390c64b`, INV-908776) that AS2-124's poller had already marked `completed`/`carrier_unresolved`. Root cause was not a bug in AS2-123/124 — that code was never reached:

1. **Carrier mismatch.** The extracted `carrier_identifier` ("Apex Logistics Solutions") matched neither of the two seeded demo carriers. `findCarrierByIdentifier` correctly bailed at `carrier_unresolved` before writing anything.
2. **Zero contracts/shipments seeded** for the demo tenant at all — even a resolved carrier would have had nothing to match against.
3. **Real gap, filed as AS2-127**: the extraction itself only ever produced 7 header-level `extraction_fields` (invoice_number, carrier_identifier, subtotal, total_payable, etc.) — zero `invoice_charge[n].*` rows. `buildInvoiceInsert` only builds `invoice_charges` from that EAV pattern, so even a fully-seeded carrier/contract/shipment would have transformed into a zero-charge invoice.

**Fix applied**: `supabase/migrations/20260920000001_freight_overbilling_demo_seed.sql` (commit `9f1b6d6`) — a carrier matching the real extracted identifier, one active contract + contract_line (flat rate $4,500, no lane/class/weight restriction), one shipment/leg/item, and two `invoice_charge[n].*` extraction_fields rows reusing the real EAV convention (not a bypass insert). Applied directly via the Supabase MCP connector against the already-provisioned free-tier project — data-only, no new infra/spend — and recorded in `delta_meta._migrations` so `pnpm run db:migrate` won't try to reapply it.

**Result, confirmed on the dashboard**: a real `variance` reconciliation — expected $4,500.00, actual $6,020.50, delta **$1,520.50** — split `rate_mismatch` ($475, billed rate vs. contract rate) + `accessorial_unauthorized` ($1,045.50, no contract term). Produced through the real `transformInvoice` → `matchOverbilling` pipeline, first live confirmation that AS2-123/124/125 work end to end.

**AS2-127 filed (tech-debt)**: freight invoice extraction never emits `invoice_line[n].*`/`invoice_charge[n].*` fields — the same convention gap `docs/DELTA_DECISIONS.md` (2026-09-10) already flagged as unadopted by the extraction handler, now confirmed against real ingested data, not just by inspection. Every future freight invoice hits this same wall until AS2-12/13 are extended. AS2-126 (shipment linkage) got a status note with the same commit SHA — its workaround was exercised live for the first time and held up.

**Repo state**: AS2-123/124/126 (3 commits) pushed to `origin/main` this session.

## 2026-09-20 (later) — AS2-129/130/131: 9 more reason codes given real logic (17 of 18 total now real). WRONG_LANE deferred as tech debt.

Follow-on to the detention fix earlier today (that session had caught `DETENTION_UNSUPPORTED` checking an approval checkbox instead of the industry "past free time" definition — fixed, commit `98c883f`). Venkatesh then asked for all 17 remaining seeded reason codes to be given real logic, "including fuel-index and lane MDM." Pushed back with a verified 3-bucket schema breakdown before starting (via a clarifying question, not assumption) — he chose the maximal scope.

**AS2-131 (FUEL_SURCHARGE_ERR + CLASS_MISMATCH, overbilling) — commit `1a43082`.** `fuel_surcharge_schedules`/`fuel_surcharge_indices` turned out to already be fully-built, tenant-scoped schema with zero readers anywhere in the codebase — tenant-entered reference data (peg_amount/escalator_pct_per_unit/unit_step/base_amount_type + published index values), no extraction dependency. `computeExpectedFuelSurcharge` (`match-overbilling-compute.ts`) applies the standard step-table formula. CLASS_MISMATCH: when `billed_freight_class` diverges from the shipment's `declared_freight_class` and a distinct contract line exists for the declared class, carves the class-caused delta out before the existing rate/weight split — fixes a pre-existing quirk where the matcher trusted the carrier's billed class over the shipper's declared one when selecting the pricing contract line.

**WRONG_LANE explicitly NOT built — filed as AS2-129 (tech debt), not shipped as dead code.** Checked before building: `contract_lines.lane_id` is hardcoded `null` at insert time in `transform-rate-contract-compute.ts` ("lane_id is always left null - no lane/location resolution built"), and no shipment-leg lane is ever resolved either. The `lanes` table and both `lane_id` columns are schema-correct and unused. Building matching logic against an always-null field would never fire on real data — flagged to Venkatesh directly rather than shipping something demo-looking but inert. Needs a real lane-resolution subsystem (origin/destination → `lanes.id`, wired into both shipment ingestion and rate-contract extraction) before this is buildable.

**AS2-130 (6 deduction codes + CHARGEBACK_ASN) — commit `1ae1b74`.** DAMAGED_GOODS, DEFECTIVE_RETURN, RETURN_UNAUTHORIZED, CHARGEBACK_LABEL, COOP_ADVERTISING, VOLUME_REBATE, CHARGEBACK_ASN. **No new schema needed** — `dispute_items` (migration `20260904000001`) already existed as a generic, tenant-scoped, RLS'd polymorphic claims table (`entity_table`/`entity_id` + `claimed_amount` + `reason_code` + `status`) with zero readers anywhere in the codebase before this. Confirmed by schema search that no damage-evidence/promo-terms/rebate-tier tables exist, so an approved on-file claim is the strongest real signal available today — same approval-proxy posture `detention_unsupported` had before its own fix. `resolveDeductionClaims` (`match-deduction-compute.ts`): per invoice line, an approved `dispute_items` claim explains up to its `claimed_amount`, capped at that line's own delta. Two exceptions: an on-file return claim that was **never** approved is RETURN_UNAUTHORIZED and takes the whole line delta (fully unsupported, not partial); CHARGEBACK_ASN additionally requires no `despatch_advice` document_reference on file for the order (real evidence gate — a claim contradicted by our own ASN record isn't honored at face value).

Category resolution for both AS2-130 and AS2-131 mirrors AS2-123's `OverbillingCategory` pattern exactly: a category tag resolves straight to its seeded `reason_code_id`, bypassing the generic 6-value `condition_type` mapping that can't disambiguate this many codes.

**Known labeling slip, recorded rather than hidden**: the AS2-131 commit message and its in-code comments say "AS2-129" — reused that number by mistake (AS2-129 is actually the WRONG_LANE tech-debt ticket, filed moments earlier in the same session). Left the commit text as-is (factual about what shipped, just mislabeled) and filed AS2-131 as the correct record.

**Reason-code coverage: 17 of 18 seeded codes now have real matching logic** (was 8 at the start of today — 5 overbilling + 3 deduction via generic condition_type). Only WRONG_LANE remains unbuilt, tracked as AS2-129.

**Verification**: built+tested entirely in the fast local scratch clone (per the 2026-09-20 environment-trap note above — the device-bridge mount is unusable for `node_modules`-scale work). 34/34 → 70/70 → 147/147 full `agent-reconciliation` suite passing across both commits, `tsc --noEmit` clean on both `@delta/shared` and `agent-reconciliation`. **Not click-through tested** — same standing limitation as every prior session entry; this session cannot run long-lived local dev services through the device bridge.

**Architecture document updated.** No `markitdown` MCP tool was available this session; used `pandoc` + `python-docx` on Venkatesh's own machine instead (confirmed installed, a documented fallback). Found a sibling `DELTA_Architecture_1.md` (132KB) next to the `.docx` (3.5MB) in the same OneDrive folder — a pandoc round-trip confirmed both cover identical content start-to-finish (same title page, same closing data-dictionary table), just different table-formatting styles from two separate conversions, not a content fork. Added a new subsection **4.6a Reason-code coverage — what's computed today** to both files (matching the existing `4.9a`/`4.9b` lettered-subsection pattern), right before "4.7 Relationships": what's real (17/18), why WRONG_LANE isn't, and the dispute_items/fuel_surcharge schema-reuse story. Inserted into the live `.docx` via `python-docx` (preserves existing Word styling — no full pandoc regeneration, which would have dropped the doc's native formatting), original backed up first as `DELTA_Architecture_1.docx.bak-20260920` in the same folder. Also logged as a decision in `DELTA_DECISIONS.md` (2026-09-20 entry) since "reuse existing unused schema instead of 6 new tables" is exactly the kind of call a diligence reviewer would ask about.

**Repo state**: commits `1a43082` (AS2-131) and `1ae1b74` (AS2-130) are local on `main`, not yet pushed — this device-bridge shell has no GitHub credentials (documented limitation, every prior session too). Needs `git push origin main` from Venkatesh's own terminal, same as `98c883f` earlier today.

## 2026-09-20 (later still) — "Loop engineering" defined (CLAUDE.md Rule 10); real click-through confirmed structurally blocked from this session; AS2-127 partially fixed

Venkatesh asked to "work 18 hrs/day" following "loop engineering, rule #10" — no such rule existed anywhere in CLAUDE.md/DELTA_CONSTRAINTS/DELTA_PLAN/DELTA_STATE. Flagged that directly rather than guessing, then wrote it down once confirmed: CLAUDE.md now has **Rule 10** — an autonomous, test-gated ticket queue, not unsupervised time-boxed grinding. Stops only for spend, schema risk, or genuinely ambiguous scope.

**Verification attempt, with real findings, not just the inherited assumption:**
- `.env.local` (root and `apps/web/`) **is fully populated** — every key the 2026-09-18 entry said was missing (`AZURE_STORAGE_CONNECTION_STRING`, `MESSAGING_MODE`, `EXTRACTION_HANDLER_HTTP_URL`, `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`) is set, dated 2026-09-19. That blocker is stale — corrected here.
- Attempted to actually stand up the local dev stack (Azurite/func/next) from this session to get real click-through evidence. Two structural blockers confirmed **firsthand this session**, not just repeated from old notes: (1) `du -sh node_modules` on the mounted repo hung past 120s — the device-bridge mount really is unusable for `node_modules`-scale operations, confirmed again independently; (2) the device-bridge shell's own tool description states a service it starts is not reachable as `localhost` by other tools (browser automation included) even with full domain access — it runs in an isolated VM, not literally Venkatesh's Windows machine. Both together mean **real browser click-through cannot happen from this session, structurally** — not a "haven't tried hard enough" gap. `pnpm`/`func` also aren't on the device-bridge VM's PATH by default (worked around `pnpm` via `npx pnpm@9`; `func` never resolved).
- Conclusion given to Venkatesh plainly: this needs to run on his actual machine. Env vars are ready; next step is literally `pnpm install` (native disk, not through Cowork) + `func start` (agent-ingestion, agent-reconciliation) + Azurite + `next dev` (apps/web), then walk the 6-step demo script himself.

**AS2-127 partially fixed instead, commit `f143880`** (redirected effort here once click-through was confirmed blocked, per Rule 10 — a scoped, test-gated, no-schema, no-spend fix was available and directly serves the same "solid platform" goal). Root cause: `buildInvoiceInsert` (`transform-invoice-compute.ts`) has always correctly parsed `invoice_line[<n>].*`/`invoice_charge[<n>].*` fields — nothing on the extraction side ever produced them, because the Tier-2/3 AI prompt (`buildExtractionSystemPrompt`) only ever asked for header fields. Added `LINE_ITEM_EXTRACTION_GUIDANCE` (`extraction-config.ts`), appended to the prompt for `doc_type='invoice'` only, spelling out the exact field-name convention and the 7-value `charge_type` enum `buildInvoiceInsert` enforces. Covers both `invoice_line[n]` (deduction-mode goods lines) and `invoice_charge[n]` (freight charge lines) — same `invoice` doc_type, one prompt change. **Not fully closing AS2-127**: its third ask (DQ escalation when a freight invoice extracts zero charges) is a real design call, not built here — left open on the ticket rather than claiming more than shipped.

**Verification**: 58/58 `agent-ingestion` tests passing (was 54, 4 new — direct prompt-content assertions plus one integration-style test confirming `callOpenRouter` actually receives the new text and a mocked charge-line response flows through to `extraction_fields` unmangled), `tsc --noEmit` clean. **Not live-verified against a real OpenRouter call** — no tooling in this session reaches OpenRouter; same standing posture as every extraction-side change. Needs one real freight invoice run through the pipeline.

**Linear updated**: AS2-127 commented with the commit SHA and left deliberately open (not Done) — partial fix, said so plainly rather than closing early.

**Repo state**: 7 commits now local on `main`, unpushed — 98c883f, 1a43082, 1ae1b74, 3be73cd, 99a140b, 6180a4d, f143880. Needs `git push origin main` from Venkatesh's own terminal, same standing limitation as every session.

## 2026-09-21 — contract_lines.charge_type shipped (AS2-132): closes the accessorial-rate gap found reviewing a real Apex Ratecon

Direct follow-on to the "ACCESSORIAL_UNAUTHORIZED overstates what the engine actually checked" finding logged 2026-09-20. Venkatesh rejected "just document the gap" and asked for the real fix, same session: "this table is at the core of our thesis — freight overbilling!! ... let us fix it right away."

**Shipped:** `contract_lines.charge_type` (7-value enum), GIST exclude constraint widened to key on it, `resolve_contract_line()` given an optional `p_charge_type` param (default `'freight'`, byte-identical for existing callers), `match-overbilling-compute.ts`'s accessorial branch rewritten to actually select and price/cap an accessorial-typed contract line instead of checking a boolean pointer, `transform-rate-contract-compute.ts` parses `contract_line[n].charge_type` from extraction (defaults `'freight'`). Full writeup in `DELTA_DECISIONS.md` (2026-09-20 later entry).

**Migration applied directly to the live Supabase project** (schema-only, already-provisioned free-tier project, no new spend) — confirmed post-apply: the one seeded Apex row correctly backfilled to `charge_type='freight'`.

**Verification**: scratch-clone `packages/shared` build clean, full `agent-reconciliation` suite 149/149 passing (4 new/rewritten accessorial tests, including the exact "$75/hr, max $300, billed $500 for 6hrs → flagged for $200 over the cap" scenario Venkatesh described), `tsc --noEmit` clean on both packages. **Real-repo port verified by diff + `git status` only** — `npx pnpm@9 --filter @delta/shared build` failed against the mounted repo with `Cannot find module '.../node_modules/typescript/bin/tsc'`, consistent with the standing device-bridge/`node_modules` limitation logged repeatedly this session; not a code issue, didn't retry it.

**Linear**: AS2-132 opened and closed same session (retroactively, per Rule 10's hotfix-ticket exemption — ticket didn't exist until after the fix was built and tested) with the commit SHA in the ticket body, per Rule 9.

**Repo state**: 10 commits now local on `main`, unpushed — `98c883f, 1a43082, 1ae1b74, 3be73cd, 99a140b, 6180a4d, f143880, ef4561c, b049fbb, 452619e` (the commit for this fix, `d0beb70`, was amended in place to `452619e` to add the AS2-132 reference before push — local-only amend, nothing pushed yet, so no history was rewritten out from under anyone). Needs `git push origin main` from Venkatesh's own terminal — this is now 10 commits deep unpushed, worth doing soon.

**Still open from before, not touched this pass:** AS2-120 (confidence score, needs a product decision) and WRONG_LANE/AS2-129 (needs a real lane-resolution subsystem — schema work, bigger than today's fix).

## 2026-09-21 (continued) — AS2-133: rate-contract extraction prompt closes the last "not done here" gap from AS2-132

Same session, immediate continuation (Venkatesh: "continue with the next set of tasks"). `LINE_ITEM_EXTRACTION_GUIDANCE` gets a `rate` entry (`extraction-config.ts`) asking the Tier-2/3 AI for indexed `contract_line[<n>].charge_type/rate/rate_uom_code/rate_basis/min_charge/max_charge/mode/freight_class/currency_code/effective_from/effective_to` - exact parallel to AS2-127's invoice fix, same generic `buildExtractionSystemPrompt` append mechanism, no handler code change needed. 3 new tests (58 → 60 passing in `agent-ingestion`), `tsc --noEmit` clean. Committed `b9d8d50`, AS2-133 closed with the SHA.

**Not live-verified** — no OpenRouter access from this session, same standing posture as AS2-127/every extraction-side change this project has shipped. Needs one real Ratecon with an accessorial line run through the pipeline.

**Repo state: 12 commits now local on `main`, unpushed.** Needs `git push origin main` from Venkatesh's own terminal — worth doing soon, this is getting deep.

**Stopping here per Rule 10** — the two remaining open items both hit a Rule 10 stop condition rather than being safe to grind through solo:
- **AS2-120 (confidence score)** — needs a product decision (what should drive it, what threshold gates review) before there's anything to build. Not a code gap.
- **WRONG_LANE / AS2-129** — needs a real lane-resolution subsystem (origin/destination → `lanes.id`, wired into both shipment ingestion and rate-contract extraction). Real schema/architecture work, bigger blast radius than today's additive column - the kind of call Rule 10 says to stop and ask about, not assume.

## 2026-09-21 (continued) — AS2-134: freight charge classification taxonomy (9 categories / 74 subcategories)

Venkatesh gave his own freight-audit taxonomy (9 categories: Core transportation, Energy/market surcharges, Equipment/handling, Facility/appointment/time, Measurement/rating corrections, International origin/destination, Government/customs/tax, Financial adjustments, Non-freight pass-throughs) and asked to "review and implement these categories." Flagged the architectural tension first (74 subcategories can't each get their own pricing/matching logic before the 9/24 demo without either a multi-week rebuild or most categories silently falling into "other") and asked two clarifying questions via AskUserQuestion: scope (classification-layer-only vs. full per-category pricing rebuild) and coverage (invoice_charges only vs. both invoice_charges and contract_lines). Venkatesh: "go for it" — both recommended defaults accepted.

**Shipped:** `packages/shared/src/schemas/charge-taxonomy.ts` — single source of truth, `CHARGE_TAXONOMY` (74 entries: code, category, label, engineBucket), `chargeCategorySchema`/`chargeSubcategorySchema`, `lookupChargeSubcategory()`. Every one of the 74 subcategories maps to exactly one of the 7 pre-existing `charge_type` engine buckets — this is an additive classification/reporting layer, not a second matching engine. `invoice_charges`/`contract_lines` insert schemas extended with nullable `charge_category`/`charge_subcategory`. Derivation logic in both `transform-invoice-compute.ts`/`transform-rate-contract-compute.ts`: `charge_category` is ALWAYS derived from `charge_subcategory` (never independently trusted from AI output — the two can't drift out of sync by construction), and a valid subcategory's mapped engine bucket overrides an independently-extracted `charge_type` for that line. Invalid/unrecognized code → both fields null, don't guess (same posture as the rest of this codebase). Absent subcategory → byte-identical to pre-AS2-134 behavior.

**DB migration applied directly to the live Supabase project** (`20260921124521_as2_134_charge_taxonomy.sql`, schema-only, already-provisioned free-tier project `qikmbzrkkrkrdyuoswgy`, no new spend) — two nullable text columns + CHECK constraints on both `invoice_charges` and `contract_lines`. Deliberately no new lookup table (would need to satisfy CLAUDE.md Rule 2's tenant_id/RLS requirement despite being global reference data) — kept as compile-time TypeScript data instead.

**Extraction prompt updated** (`extraction-config.ts`'s `LINE_ITEM_EXTRACTION_GUIDANCE`, both `invoice` and `rate` doc types) to ask the AI for `charge_subcategory` from the full 74-code list, grouped by the 9 categories purely for model readability (only the leaf code is ever extracted or written).

**Verification**: `@delta/shared` build clean, `tsc --noEmit` clean on `agent-reconciliation` and `agent-ingestion`, full suites green — 155/155 (agent-reconciliation, 6 new smoke tests) and 60/60 (agent-ingestion, 2 new prompt-content assertions). **Not live-verified against a real OpenRouter call** — same standing posture as every extraction-side change this project has shipped; needs one real complex invoice run through the pipeline to confirm the model actually populates `charge_subcategory` well in practice.

**Explicitly out of scope, flagged to Venkatesh directly (not silently assumed):** overbilling/matching still operates on the 7-value `charge_type` bucket per subcategory, not per the 9 categories — several categories (Energy/market surcharges, Facility/appointment/time, Financial adjustments) split across two different buckets internally because their subcategories genuinely price differently (e.g. a diesel surcharge is fuel-index-checkable, a war-risk surcharge is not). Category-level pricing/matching would be a separate, much larger build.

**Linear**: AS2-134 created and closed same session (retroactive, per Rule 10's hotfix-ticket exemption) with the commit SHA in the ticket body, per Rule 9.

**Repo state: 13 commits now local on `main`, unpushed** — this session's commit `37979ae` added to the 12 from earlier today. Needs `git push origin main` from Venkatesh's own terminal — worth doing soon, this is getting deep.

## 2026-09-21 (continued) — Architecture document updated for AS2-134 taxonomy

Added new subsection **4.6b Charge classification taxonomy — nine categories, seventy-four subcategories (AS2-134)** to `DELTA_Architecture_1.md`/`.docx` (right after 4.6a, before 4.7 Relationships, same lettered-subsection convention as 4.6a/4.9a/4.9b), covering: classification-layer-not-a-second-engine scope, the always-derived charge_category/charge_subcategory relationship (don't-guess posture on an unrecognized code), why three of the nine categories split across two different charge_type buckets internally, and the no-new-lookup-table schema decision. `.docx` edited in place via `python-docx` (preserves existing Word styling — matched the 4.6a heading style object and body-paragraph formatting directly, not by style-name lookup, which hit a duplicate-style-name issue in this document's styles part), original backed up first as `DELTA_Architecture_1.docx.bak-20260921`. `.md` sibling kept in sync with the same content.

## 2026-09-21 (continued) — Architecture doc: full 74-row taxonomy table added

Venkatesh asked for the category/subcategory names themselves in the document, not just the scope narrative. Added **EXHIBIT 9a** — a 4-column table (Category, Subcategory code, Label, Engine bucket) covering all 74 codes across the 9 categories — to both `DELTA_Architecture_1.md` (markdown table) and `.docx` (real Word table, borders added via XML since this document has no named table styles defined to reuse), placed directly after the 4.6b narrative and before 4.7 Relationships. Verified row-by-row against `charge-taxonomy.ts` (75 rows incl. header, 74 data rows, spot-checked category boundaries).

## 2026-09-21 (continued) — AS2-135: fixed markdown-fenced JSON silently discarding extraction results

Found while walking Venkatesh through his first local click-through of the pipeline (Azurite/agent-ingestion troubleshooting this session). Live SQL check against the demo Supabase project showed an extraction marked `status: "completed"` with zero `extraction_fields` rows. Root cause: Tier 3 vision model (`inclusionai/ling-3.0-flash-vl:free`) returned a fully correct extraction wrapped in \`\`\`json fences; `parseAiExtractionResponse` (`extraction-handler.ts`) did a bare `JSON.parse`, threw on the fence, and silently treated the whole response as "nothing found." This was a known-but-unfixed gap — a 2026-09-14 comment on the same function (AS2-102) already documented discovering this exact failure mode, but only added logging, never a real fix.

**Fix:** try raw content first, then a single leading/trailing fence-stripped version, before falling back to the "no fields found" warning. Only strips a fence wrapping the entire response — a fence embedded mid-reply alongside real prose still correctly fails as malformed output.

**Verification:** `tsc --noEmit` clean, full `agent-ingestion` suite 61/61 passing (60 pre-existing + 1 new regression test using the real fenced-JSON shape). **Not yet verified against the original failing document end-to-end** — Venkatesh needs to restart agent-ingestion's Functions host (`func start` doesn't hot-reload a TS rebuild) and re-upload the same invoice.

**Linear:** AS2-135 created and closed same session (retroactive, per Rule 10's hotfix-ticket exemption) with the commit SHA in the ticket body, per Rule 9.

**Repo state: 14 commits now local on `main`, unpushed** — this session's commit `dbced1a` added to the 13 from earlier today. Needs `git push origin main` from Venkatesh's own terminal — worth doing soon, this is getting deep.

## 2026-09-21 (continued) — AS2-136: fixed maxTokens truncation, the second real bug found retesting AS2-135

Venkatesh restarted agent-ingestion and re-uploaded the same invoice to verify AS2-135. The parser fix worked correctly, but `extraction_fields` was still empty — a different failure this time. The terminal log's `raw_content_preview` showed a genuine, well-formed JSON array that stopped mid-object with no closing bracket or fence: the model hit `CallOptions.maxTokens`'s 2048 default (`openrouter.ts`) before finishing. A header-only invoice's ~6 fields fit in 2048 tokens; this real invoice (7 lines, 6 charge categories) did not. Genuine truncation, not malformed output — AS2-135's fix correctly had nothing valid to recover.

**Fix:** `extraction-handler.ts` — `EXTRACTION_MAX_TOKENS = 4096`, passed to both the Tier 2 (`callOpenRouter`) and Tier 3 (`callOpenRouterVision`) extraction calls.

**Verification:** `tsc --noEmit` clean, full `agent-ingestion` suite 61/61 passing. No new test added — this fix has no branch logic to unit test (the option is either passed or not); verification is the real end-to-end retry. **Not yet verified against the original document** — reset the same `extractions` row to `failed` again so the next re-upload retries with both fixes in place.

**Linear:** AS2-136 created and closed same session (retroactive, per Rule 10's hotfix exemption) with the commit SHA, per Rule 9.

**Repo state: 15 commits now local on `main`, unpushed** — this session added `dbced1a` (AS2-135) and `01d07e6` (AS2-136) to the 13 from earlier today. Needs `git push origin main` — 15 deep now, should not wait much longer.

## 2026-09-21 (continued) — AS2-137: fixed the timeout AS2-136 introduced, third fix in the same real-invoice retest chain

Retesting AS2-136 on the same Apex Logistics invoice, extraction now failed outright after 137665ms: "OpenRouter call timed out after 45000ms" x3. Direct consequence of AS2-136 — raising EXTRACTION_MAX_TOKENS to 4096 fixed the truncation, but the free-tier vision model (inclusionai/ling-3.0-flash-vl:free) takes proportionally longer to generate the larger response, exceeding CallOptions' 45s-per-attempt default on all 3 retries.

**Fix:** `extraction-handler.ts` — `EXTRACTION_TIMEOUT_MS = 90_000`, passed as `timeoutMs` alongside `maxTokens` on both Tier 2 and Tier 3 extraction calls.

**Verification:** `tsc --noEmit` clean, 61/61 passing. **Not yet verified end-to-end** — this is the third fix surfaced from retesting the same real invoice (AS2-135 parser → AS2-136 token cap → AS2-137 timeout). Reset the same `extractions` row to `failed` again for the next retry.

**Linear:** AS2-137 created and closed same session with the commit SHA.

**Repo state: 16 commits now local on `main`, unpushed** — `63eadbc` (AS2-137) added to the 15 from earlier. `git push origin main` is now overdue — recommend doing it before the next round of testing, not after.

## 2026-09-21 (continued) — AS2-138: DUPLICATE_INVOICE false-positive fixed, found live testing AS2-135/136/137

Uploading a second real Apex Logistics invoice (INV-448821, $4,531) to verify the AS2-135/136/137 extraction fix chain surfaced a new bug: it was flagged `DUPLICATE_INVOICE` against the existing seed invoice (INV-908776, $6,020.50) despite different amounts, dates, and charge lines entirely. Root cause: `findDuplicateInvoiceForCarrier` checked only "does any other invoice exist for this carrier at all" - guaranteed a false positive on every carrier's second-ever invoice. Confirmed live via direct query against `wim_dev`, not by inspection.

Real `invoice_number` collisions are already impossible (`uq_invoices_tenant_number`), so that was never a candidate signal - the actual pattern this reason code should catch is the same freight re-billed under a *different* invoice number, which needs a real shipment link (PRO#/BOL#), not built yet (AS2-126).

**Fix (commit `c19fd39`, AS2-138):** tightened to same carrier + same `total_payable` + `invoice_date` within a 3-day window. Still a heuristic proxy until AS2-126's real shipment resolver lands - documented as such in the code, not silently papered over. `invoiceForMatchSchema` gained `total_payable`; `loadInvoice`'s select and the `match-overbilling.ts` call site updated to match.

**Verification:** built/tested in the fast cloud scratch clone (device-bridge mount still unusable for `node_modules`-scale work, same standing trap). `@delta/shared` build clean, `tsc --noEmit` clean on `agent-reconciliation`, full suite 157/157 (2 new regression tests: same-carrier-different-amount no longer flags, same-carrier-same-amount-near-date still does).

**Live-verified 2026-09-21:** Venkatesh pushed `c19fd39` to `origin/main` and restarted all 3 local processes. `agent-reconciliation`'s reconciliation for INV-448821 was recomputed via `POST /api/match-overbilling` and the result confirmed directly against `wim_dev` (not just the API response): `DUPLICATE_INVOICE` is gone. New result is `variance` / `weight_discrepancy` (billed weight exceeds shipment's declared weight, -$5,350) - a real, different finding, not another false positive.

**Linear:** AS2-138 closed, commit SHA `c19fd39` and live-verification result recorded in the ticket, tech-debt label (real fix is still AS2-126).

**Repo state:** commit `c19fd39` pushed to `origin/main`. AS2-138 fully closed end-to-end.

**New, unrelated observation from the live-verified run (not investigated):** the recomputed INV-448821 result's `charge_outcomes` lists both `rate_unresolved` and `rate_mismatch` together - worth a look, not blocking for the demo.

## 2026-09-21 (continued) - AS2-139: reconciliations gains a supersede invariant (production-grade, not a quick patch)

The live-verification re-run above (calling `POST /api/match-overbilling` a second time for INV-448821) surfaced a second, separate bug: `reconciliations` had no notion of "current vs. historical" result for an invoice - every `matchOverbilling`/`matchDeduction` run just inserted a new row, forever. My manual re-run created a second row instead of replacing the first, and `apps/web/src/lib/dashboard.ts` sums every row for the tenant with no filter, so the dashboard showed $11,402 at risk instead of the correct $6,871 (double-counting the superseded row). The normal path (`reconciliation-poller.ts`) never re-triggers for an already-processed extraction, so this was latent until something manually re-triggered it - a real retry/webhook path later would have hit the same gap.

Venkatesh's direction: "let us not take shortcuts - let us make production grade changes" - so the fix is a real DB invariant, not a delete of the bad row.

**Fix (commit `e70586e`, AS2-139):**
- Migration `20260921231229_reconciliation_supersede.sql`: `reconciliations` gains `superseded_at`/`superseded_by` (never deleted - same "corrections supersede" convention CLAUDE.md already requires for `match_links`, applied here too), a partial unique index `ux_reconciliations_current_per_invoice` enforcing "at most one current row per (tenant_id, invoice_id)" as a real DB invariant, and `fn_insert_reconciliation` - now the only path allowed to write a new reconciliation row, atomically superseding any existing current row first (old row marked non-current before the new one is inserted, so the unique index is never transiently violated).
- Same migration backfills the two pre-existing duplicate rows for INV-448821 (superseded, not deleted) - confirmed live: the old `DUPLICATE_INVOICE` row is now `superseded_by` the current `weight_discrepancy` row.
- `queries.ts`'s `insertReconciliation` now calls the RPC instead of inserting into `reconciliations` directly.
- `apps/web/src/lib/dashboard.ts` now filters `.is("superseded_at", null)` - this is what actually fixes the dashboard double-count.
- `apps/web/src/lib/reconciliation-detail.ts` + the detail page (`reconciliations/[id]/page.tsx`) now surface a "superseded, see current result" banner rather than silently showing a stale row to anyone with an old link.

**Verified:** `@delta/shared` build clean, `tsc --noEmit` clean on both `agent-reconciliation` and `web` (no web test suite exists - `typecheck` only, so this side relies on `tsc` plus the DB-level invariant), full suite 159/159 (2 new regression tests on `insertReconciliation`'s RPC call/param mapping, on top of AS2-138's 2). Migration applied live against `wim_dev` via the Supabase MCP tool and confirmed via direct query.

**Linear:** AS2-139 filed, tech-debt label.

**Repo state:** commit `e70586e` local on `main`, not yet pushed - needs `git push origin main` from Venkatesh's own terminal, same standing limitation.

## 2026-09-21 (continued) - Venkatesh away from laptop; autonomous cleanup pass (Rule 10: test-gated, no schema/spend risk)

Venkatesh stepped away before pushing; asked to "continue with the build." Per Rule 10 (autonomous, test-gated ticket queue - stops only for spend, schema risk, or genuinely ambiguous scope), picked up small, well-scoped, already-filed tech-debt tickets that don't need his live machine to verify. Deliberately did NOT touch AS2-126 (real PRO#/BOL# shipment resolver) - well-scoped but touches the extraction pipeline, which AS2-126's own ticket text already flags as needing careful re-verification time this session doesn't have unattended; correctly left for a session where Venkatesh can sanity-check live.

**Triaged, not a bug:** the `rate_unresolved` + `rate_mismatch` combination flagged as an open question in the AS2-139 entry above is expected behavior, confirmed against `wim_dev` - `chargeOutcomes` is one entry per invoice charge line, and INV-448821 has two separate `freight` lines ($3,200 and $140); one resolves against a contract line (mismatch), the other doesn't (unresolved). No code change needed.

**DELTA_DECISIONS.md merge-conflict markers resolved** (flagged at session start, never fixed until now): `<<<<<<< Updated upstream` / `=======` / `>>>>>>> Stashed changes` around the 2026-09-14 entries - the "Updated upstream" side was empty, so the fix was mechanical (keep the "Stashed changes" content, drop all three marker lines). No content lost.

**AS2-106 + AS2-107 fixed together, commit `dea5c92`** (both small, same file, both defensive hardening on `classifyMatchRows` in `packages/shared/src/matching/classify.ts`):
- AS2-106: `.startsWith("exact")` replaced with an explicit `EXACT_MATCH_TIERS` allowlist (`exact`, `exact_line`, `exact_header`, `exact_bol`, `exact_pro`, `exact_load` - the real, closed set the three SQL match functions in `20260909000003_document_matching_functions.sql` actually emit). Guards against a future tier name that merely starts with "exact" being silently treated as confident.
- AS2-107: `classifyMatchRows` now throws a clear `TypeError` on a non-array payload instead of an obscure downstream error from `.length`/`.filter`. Every current call site already guards this via `Array.isArray(data) ? data : []` on the raw RPC result (confirmed by reading `document-matching.ts`), so this is defense-in-depth on the shared library function, not a fix to an active bug.

**Verified:** `@delta/shared` build clean, `tsc --noEmit` clean on `agent-ingestion`/`agent-reconciliation`/`agent-interface`/`web`, full suites green - `@delta/shared` 72/72 (3 new classify.ts tests), `agent-ingestion` 61/61 (unchanged), `agent-reconciliation` 159/159 (unchanged). Built/tested in the fast cloud scratch clone per the standing workflow.

**Not done, left for Venkatesh:** git push (this session still has no GitHub credentials - now 4 commits deep: `c19fd39`, `e70586e`, `b3085bf`, `dea5c92`); the unrelated uncommitted work in the tree (`tolerance.ts`, `tolerance-config.ts`, the new settings page, migration `20260918000003_invoice_source_document.sql`) - still untouched, still his call; AS2-126 real fix - deliberately deferred, see above.

## 2026-09-21 (continued) - AS2-122: unknown-tenant blobs quarantined instead of lost (still Venkatesh away, continuing Rule 10 pass)

Real defect, found 2026-09-20: three documents uploaded under a tenant_id that was never seeded sat in `incoming/` forever - the poller logged "unknown tenant_id - skipping" once (deduped) and moved on, no other trace. `ingestion_failures` structurally can't record it (`tenant_id` is `NOT NULL` with an FK).

**Fix (commit `a17eb07`, AS2-122):** `quarantineUnknownTenantBlobs` (new, exported from `poller.ts`) moves each file from `incoming/<tenant_id>/` to `error/unknown-tenant/<tenant_id>/` via the existing `moveBlob` helper (same copy+delete-on-success pattern `extraction-handler.ts` already uses for its own error/ moves - a failed copy leaves the source in place, never silently lost), logging each one at ERROR level with the destination path. A per-file failure doesn't abort the rest of the tenant's files - logged and left for the next poll cycle.

**Notable side effect:** `poller.ts` had zero test coverage before this - nothing in the file was exported, unlike `extraction-handler.ts`'s `runExtraction`/`ExtractionIO` split. Exported the new function and added `poller.test.ts` (4 tests), mocking `@delta/shared`'s `getDocumentsContainer`/`moveBlob` narrowly (spread the real module, override only those two) since this file calls real Azure/Supabase clients inline with no injectable IO - deliberately did not attempt a bigger DI refactor of the rest of the file, out of scope for this ticket.

**Verified:** `tsc --noEmit` clean across the full workspace (`pnpm -r run typecheck`), `agent-ingestion` 65/65 (4 new). **Not live-verified against real Azurite** - needs Venkatesh to create an unseeded-tenant blob on his machine and confirm it lands in `error/unknown-tenant/`. Linear AS2-122 left **In Progress**, not Done, for exactly that reason.

**Repo state:** commit `a17eb07` local on `main`, not yet pushed - 6 commits deep now (`c19fd39`, `e70586e`, `b3085bf`, `dea5c92`, `24b39ed`, `a17eb07`). Needs `git push origin main` from Venkatesh's own terminal.

**Also found, not touched:** the working tree has other uncommitted, unrelated in-progress work (`packages/shared/src/schemas/tolerance.ts` modified, a new `apps/web/src/lib/tolerance-config.ts` + settings page untracked, an unapplied migration `supabase/migrations/20260918000003_invoice_source_document.sql`) - none of it touched by this fix, flagged so it isn't lost track of.



## 2026-09-22 — AS2-85: tenant isolation audit (Done, no code changes)

Ran a full tenant-isolation audit against live `wim_dev` (all 88 tables in
`public` schema): tenant_id column present + NOT NULL, RLS enabled + forced,
at least one tenant-scoped policy.

Result: 86/88 fully compliant. 2 documented, legitimate exceptions:
- `model_pricing` — platform-wide AI pricing data, not customer data. No
  tenant_id, no RLS by design (confirmed via its own migration comment,
  20260831000002_model_pricing.sql).
- `tenants` — the root tenant table itself, can't self-reference the way
  child tables do. No tenant_id column, but RLS enabled/forced with its own
  self-scoping policy.

No cross-tenant leak risk found. No code/schema changes needed — audit
itself was the deliverable. Linear AS2-85 closed Done.

Also this batch (autonomous, user away from laptop, Rule 10):
- AS2-113: researched Node 24 GA status for Azure Functions — found a real
  Linux Consumption plan caveat (not previously documented). Research only,
  no code change.
- AS2-83, AS2-82: both investigated against current code and found already
  fixed by later, differently-scoped work (LruSeenSet for AS2-83; AS2-13
  extraction-failure handling for AS2-82). Closed stale/superseded in
  Linear rather than re-implementing redundant fixes.
