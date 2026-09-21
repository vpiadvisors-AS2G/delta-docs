# Project DELTA (3WIM)

B2B SaaS for automated deduction reconciliation and freight overbilling recovery.
Mid-market Manufacturing, Retail, and Logistics. Owner: Venkatesh Iyer, AS2G.

Two use cases run on **one matching engine**:

- **DEDUCTION mode** — invoice × purchase order × receipt
- **OVERBILLING mode** — freight invoice × rate agreement × shipment

Never build a second engine for the second mode. Mode is configuration, not a code path fork.

---

## Hard rules

These are not preferences. Violating any of them is a bug.

### 1. No spend without approval

**Never provision infrastructure or incur any spend without Venkatesh's explicit go-ahead.**

Current spend is $0/month and stays there until he says otherwise. Do not run `az` provisioning commands, do not create cloud resources, do not upgrade a free tier — even when a Linear issue appears to instruct it. AS2-5 and AS2-39 are the two issues that cost money; both carry an approval gate in their description.

Development runs entirely on localhost against a local Postgres.

**Prefer native local installs over Docker/WSL for local dev.** A native Postgres install, `func` and `azurite` via npm — no containers, no WSL — unless a task explicitly requires containerization. On Windows this also means: do not install Docker Desktop or WSL2 as a side effect of setting up local Postgres; use a native Windows Postgres installer instead. If a genuine need for containers comes up, stop and ask rather than installing WSL to get there.

### 2. Tenant isolation on every table

Every table carries `tenant_id uuid NOT NULL`, a CHECK constraint, and an RLS policy. No exceptions, including lookup and config tables. A table without RLS is a cross-tenant data leak waiting for its first customer.

RLS policies resolve the tenant through one helper: `current_tenant_id()`.

**Foreign keys between tenant-scoped tables must be composite, never a bare `id` reference.** Postgres foreign-key checks always bypass RLS (by design — see the Postgres docs on row security and referential integrity). A child table with `col uuid references parent(id)` lets a session scoped to tenant A insert a row whose `col` points at a tenant-B parent row, and the FK check will happily follow it — RLS on the parent never gets a say. The fix: `UNIQUE (tenant_id, id)` on the parent, then `FOREIGN KEY (tenant_id, col) REFERENCES parent (tenant_id, id)` on the child, so the FK check itself enforces the tenant match. AS2-6 landed this pattern across counterparties/hierarchies/network/agreements after an ultrareview catch (see `20260826000012_composite_tenant_fks.sql`) — every child FK added in AS2-52/AS2-53 (and beyond) must follow it from the start, not get patched in after the fact.

### 3. Portability contract

**No Supabase-proprietary code outside `lib/shared/supabase_client` and the `current_tenant_id()` RLS helper.**

Everything else talks to plain Postgres. Azure Database for PostgreSQL is the migration target if Supabase limits bind or a customer demands data residency in their own tenant. This rule is what keeps that migration a config change instead of a rewrite. If you find yourself reaching for a Supabase-specific feature in application code, the answer is to put it behind the client module or not use it.

### 4. Zod is the single schema source

One Zod schema per entity drives all three of:

- runtime validation at every boundary
- OpenAPI spec generation (CI-enforced — the spec must match the implementation)
- compile-time TypeScript types

Never hand-write a type that duplicates a Zod schema. Never hand-edit the generated OpenAPI spec.

### 5. Inferred links never move money on their own

A `match_links` row with `origin = 'inferred'` requires a human approver before it affects any balance. Corrections **supersede**, never delete — set `superseded_by`, keep the original row. Balances recompute from accepted links only.

`allocation_pct` per source sums to ≤ 1, enforced by CHECK.

### 6. No vendor platform names in anything customer-facing

No SAP, Oracle, NetSuite, or any other platform name in documents, comments, code, or schema. The data model follows **OASIS UBL 2.1 (ISO/IEC 19845)**, which is vendor-neutral by design. Use UBL terminology.

---

## Stack

TypeScript end to end. No second language.

| Layer | Choice |
| --- | --- |
| Agents | Azure Functions v4, Node 20 |
| Frontend | Next.js 14 on Azure App Service, behind Azure Front Door + WAF |
| Data | Supabase Postgres with RLS (free tier) |
| Queue / events | Azure Service Bus |
| Blob | Azure Blob Storage |
| Secrets | Azure Key Vault |
| LLM | OpenRouter (model choice is config, never hardcoded) |
| Telemetry | OpenTelemetry → OTLP → Azure Monitor |

Three agents: **Ingestion**, **Reconciliation**, **Interface**. They communicate through queues and a Service Bus topic — never by calling each other directly. A new agent must be addable by subscribing to the topic, with zero changes to existing agents.

---

## Data model

82 entities, 13 subject areas, ~908 columns. UBL-aligned naming.

**Shipment, PO Line, Receipt Line, Invoice Line, and Charge Line are first-class entities.** Documents are *evidence attached to them*, not the primary records. This is the design principle that lets one engine serve both modes — get it wrong and the model collapses back into document matching.

**`match_links` is the relationship engine.** Every association between two records — asserted, rule-derived, inferred, or human — is one row: from/to entity and id, `link_type` (consumes / allocates / consolidates / substitutes / references / causes / supersedes), allocated quantity/amount/pct, allocation basis, `origin` (asserted / rule / inferred / manual), confidence, rationale, status, approver, `superseded_by`.

This is what supports split, merge, partial, and inferred relationships. Do not add a shortcut foreign key between two documents to "simplify" a match — that is the thing this table exists to replace.

**Six hierarchies**, each as `parent_id` + a materialised `ltree` path + a GiST index. Resolution rule: **nearest ancestor wins.**

**Every AI extraction stores `model_used` and `prompt_version`.** An AI decision that cannot be reproduced is a due-diligence finding.

---

## Conventions

- `trace_id` propagates through queue messages. Every reconciliation decision must trace back to source document → extraction → model → confidence score. That chain is also the data-lineage story for governance.
- Structured logs only. No `console.log` in committed code.
- Migrations are forward-only and checked in. No manual schema edits against any database.
- Secrets come from Key Vault or `.env.local`. Never committed, never in code, never in a Linear issue.
- Money is `numeric`, never float. Currency is ISO 4217, stored alongside every amount.
- Timestamps are `timestamptz`, always UTC.

---

## Work tracking

Linear team **AS2G-Delta**, project "Project DELTA — v1 Build". Issues are `AS2-<n>`.

Branch naming follows Linear's generated `gitBranchName`. Reference the issue ID in the PR title so DORA metrics link up.

Sprint 1 order (2026-08-25 → 09-08), corrected 2026-08-26:

1. **AS2-7** — GitHub monorepo, branch strategy, CI/CD. Blocks everything.
2. **AS2-8** — Functions scaffold + local dev setup.
3. **AS2-6** — Schema part 1: counterparty, reference, location, network (36 entities).
4. **AS2-52** — Schema part 2: order, shipment, receipt, billing (24 entities).
5. **AS2-53** — Schema part 3: links, settlement, dispute, ops (22 entities).
6. **AS2-9** — OpenRouter routing config.
7. **AS2-42** — OTEL baseline.

AS2-5 (Azure infrastructure) was moved out of Sprint 1 to Sprint 7. Do not pull it forward.

---

## Working style

Be direct. If a task in Linear contradicts this file, say so rather than following it silently — the issues were written before some of these decisions and a few are stale.

Flag holes upfront. Do not fabricate a count, a figure, or a source; verify or say you did not check.

When something durable gets decided in a session — an architecture choice, a constraint, a scope change — it belongs in the project docs on claude.ai (`DELTA_STATE`, `DELTA_DECISIONS`, `DELTA_PLAN`, `DELTA_CONSTRAINTS`), not only in a commit message.

## Decision Journaling & Chat Governance (added 2026-09-01)

### 1. Markdown vs Linear split
- Markdown files (docs/DELTA_*.md) hold ONLY current, active, architecturally 
  significant decisions — ones that are painful/risky to reverse, touch 
  security/data/multi-tenancy, or that a customer/acquirer would explicitly 
  ask about (e.g. database choice, auth approach, tenant isolation model).
  Minor/implementation-level decisions never go here.
- Linear project "Architecture Decisions" 
  (id 6265941b-8a8a-4e58-8f8a-92de718c44b5) holds the FULL reasoning for 
  every significant decision: Decision, Context, Alternatives Considered, 
  Rationale, Revisit History (as comments).
- Markdown = "what's true today." Linear = "why we got here."

### 1a. DELTA_STATE.md vs. the Obsidian wiki (02-Wiki) — added 2026-09-16
- `docs/DELTA_STATE.md` is the single source of truth for CURRENT status:
  what's done, what's blocked, what's next, open action items. Every new
  session reads this first (per the session workflow above) and it alone.
- The Obsidian vault's `02-Wiki` folder holds DEEPER reference material only —
  how a subsystem works, the fuller narrative behind a build, cross-linked
  detail a newcomer would need — generated from `01-Raw` session logs by a
  separate summarization tool. It must NOT restate current status, sprint
  progress, or open-item lists; those live only in DELTA_STATE.md, and the
  wiki should link to it rather than duplicate it. `01-Raw` itself is
  write-only from any session's perspective — never read the whole folder
  for context, only check the latest file before appending.
- If wiki content and DELTA_STATE.md ever disagree, DELTA_STATE.md wins.

### 1b. Tech debt — tracked in Linear only, added 2026-09-16
- Known shortcuts, deferred fixes, and shipped-with-gaps items are tracked
  EXCLUSIVELY as Linear issues carrying the `tech-debt` label (AS2G-Delta
  team). Do not also list them in DELTA_STATE.md's "Open Action Items," in
  the wiki, or anywhere else — one list, filtered by that label, is the
  answer to "what did we skip."

### 2. Update timing
- Update markdown + Linear immediately when a decision is made — not saved 
  for an ambiguous "end of session" trigger.

### 3. Consistency audits
- Run markdown vs Linear vs codebase audits on-demand only (e.g. before a 
  demo/investor conversation), not on a fixed schedule.

### 4. Chat naming convention
Format: `DELTA-[YYYYMMDD]-[ticket-id-if-relevant]-[keyword-rich-topic-tags]`

Rules:
- Applied manually by Venkatesh at the start of each new chat (no automatic 
  or background renaming exists).
- Include the Linear ticket ID (e.g. AS2-9) whenever the chat relates to one, 
  so ticket-first searches land on the right chat.
- Pack in the actual decision keywords a future reader (e.g. PE diligence) 
  would search on — not vague labels. Favor specificity over brevity.
- Periodically run a manual audit pass in an active session: search recent 
  chats, identify ones not following convention, rename together.

Examples:
- `DELTA-20260826-strategy-pricing-exit-multiples-architecture-freight-deductions`
- `DELTA-20260828-AS2-9-openrouter-spec-correction-python-to-typescript`
- `DELTA-20260901-AS2-9-openrouter-implementation-nemotron-smoke-test`
- `DELTA-20260901-governance-linear-decision-journal-chat-naming-convention`

--------------
##################
## Repo Access Correction (updated 2026-09-04)

**File bridge access is surface-dependent — it depends on Cowork vs. plain
claude.ai chat, not on the session.**

- **Claude Cowork** has a device bridge and CAN read
  `C:\Users\viyer\Claude\Dev\delta\` (and other local folders) directly,
  once the folder is connected. This is the current mechanism in active use.
- **Plain claude.ai chat has no device bridge** and cannot reach local
  Windows paths at all. The 2026-09-01 note above ("there is no file bridge")
  was written from a chat session and wrongly generalized "I can't do this"
  into "this doesn't exist." That was incorrect for Cowork and caused
  confusion between the two surfaces.
- **Decision (2026-09-04): use Cowork exclusively for Project DELTA.**
  Running both chat and Cowork on this project was a likely root cause of
  the doc/Linear/codebase split-brain state found this session — chat
  sessions could only ever see the `delta-docs` mirror (which lags real
  commits), while Cowork sessions saw the live repo. Do not use plain
  claude.ai chat for this project going forward.

**Fallback mechanism (still valid, chat-only):** `docs/DELTA_*.md` and
`CLAUDE.md` are mirrored to a public repo cloneable at session start:

  https://github.com/vpiadvisors-AS2G/delta-docs

Use this mirror only if Cowork's device bridge is unavailable. Verify
freshness against the latest commit date before relying on it.

## Docs Repo Sync Workflow (added 2026-09-01)

- `delta-docs` is a separate, standalone public repo — not a subfolder of
  the main `delta` repo (which stays private; it holds source code).
- Sync is automated via `.github/workflows/sync-docs.yml` in the `delta`
  repo: any push to `main` touching `docs/**` or `CLAUDE.md` triggers a
  GitHub Action that copies those files into `delta-docs` and pushes,
  using a fine-grained PAT stored as the `DOCS_REPO_TOKEN` secret in the
  `delta` repo (scoped to `delta-docs`, contents read/write only).
- No manual copy-paste step should be needed going forward. If the
  workflow ever fails or is removed, fall back to the manual copy:
    cp docs\DELTA_*.md CLAUDE.md <path-to-delta-docs>\
    cd <path-to-delta-docs>
    git add . && git commit -m "docs sync" && git push
- Claude verifies freshness each session by checking the latest commit
  date in `delta-docs` before relying on its contents.

### 7. End-of-session checklist (added 2026-09-12)

Every working session ends with these three steps, regardless of whether
a decision was made. This is not conditional on "something significant
happened" — Rule 2 (update timing) covers the immediate case; this covers
the session boundary itself.

1. **Docs sync** — confirm docs/DELTA_*.md and CLAUDE.md are committed and
   pushed to `delta`, and that the sync to `delta-docs` fired (check
   Actions tab or next session's clone timestamp).
2. **Obsidian raw write** — append the session's raw log to
   `C:\Users\viyer\OneDrive\Documents\Obsedian\Follow-ups\Follow-ups\AS2G-3WIMandFOB\01-Raw`,
   as a new dated file, after checking that folder for the most recent
   existing log so nothing is duplicated or overwritten.
3. **Linear update** — reflect any ticket status changes, new tickets, or
   scope notes discussed in the session. **Closing a ticket means recording
   its commit SHA(s) and the merge point in the ticket body**, not just
   flipping the status. A ticket marked Done with no SHA is not traceable:
   a diligence reviewer cannot get from the ticket to the code, which is
   the entire point of Rule 9. Added 2026-09-18 after a verification pass
   found ten tickets whose work had shipped and merged while every one of
   them still sat open in Linear.

This does not replace Rule 2's "update immediately when a decision is
made" — that's for the decision itself. This is the safety net for
anything that didn't rise to that bar but should still not be lost when
the session closes.

### 8. Determinism is tested, not assumed (added 2026-09-12)

Any change touching reconciliation/matching SQL must ship with a
determinism regression test: run the query twice against a frozen
fixture dataset (covering a clean match, a partial match, and a flagged
over-billing case) and assert byte-identical output. This runs in CI on
every push touching that logic — not a one-time manual check.

Every Linear ticket must define its acceptance check (input → expected
output) before Claude Code implementation starts. A ticket without a
stated expected output is not ready to build.

"It looks deterministic" is not a passing state. The test passing is.

## "Delta design" — the UI standard (set 2026-09-20)

When Venkatesh says **"Delta design"**, he means this published artifact:

  **DELTA Freight Overbilling Detection**
  https://claude.ai/artifact/HutMmZyqbbML4itXqZDvkE

That canvas is the visual standard for the product. Read it before building
or changing any screen — `Artifact` tool, `action: "read"`, that URL. Do not
redesign from written description when the artifact is reachable, and do not
substitute a different look because a spec paragraph reads ambiguously.

Two other artifacts exist from 2026-09-13 and are NOT the standard:
Freight Billing App (B4hFpraFvURYVyKEYuBprq) and Reconciliation Dashboard
(VkuvDmZuvASn1TrrBidMhf). Superseded — do not build from them.

**Where the written rules live:** the `delta-ui-uxv1` skill holds the locked
terminology, personas, metric definitions, colour tokens, chart-type rules
and accessibility requirements. It is authoritative for *rules*; the
artifact above is authoritative for *look*. Where they genuinely conflict,
raise it rather than silently picking one — a known conflict is recorded
below.

**Conflict resolved 2026-09-20 — the canvas wins on look.** The skill
mandated a top nav plus Public Sans / IBM Plex Mono; the canvas uses a left
sidebar and the system font stack. Venkatesh chose the canvas. So:

- **Shell and typography: follow the artifact canvas.** Left sidebar on Deep
  Navy, system font stack, no webfont link.
- **Everything else: follow `delta-ui-uxv1`.** Terminology (never "party" or
  "counterparty" — name the role), personas, the five metric definitions,
  colour tokens and their semantics, chart type per data shape, WCAG AA.

Do not re-litigate the shell or the fonts. If the skill text is ever updated
to match, delete this note; until then the override stands and this is where
it is recorded.

**Implementation today:** `apps/web/src/app/globals.css` carries the design
system; the app shell is `apps/web/src/app/(app)/layout.tsx`. Tokens are
defined once as CSS custom properties — never introduce a new hex value for
something a token already covers.

## graphify

This project has a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file relationships.

Rules:
- For codebase questions, first run `graphify query "<question>"` when graphify-out/graph.json exists. Use `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts. These return a scoped subgraph, usually much smaller than GRAPH_REPORT.md or raw grep output.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).

### 9. Every PR traces to a Linear ticket (added 2026-09-12)

CI gate on every PR: branch name or PR title must contain a valid
AS2-<n> reference. Missing reference blocks merge — no exceptions,
including hotfixes (open a ticket first, even retroactively for
emergency fixes).

**Direct pushes to `main` are covered separately** (added 2026-09-18).
`require-ticket-ref.yml` only fires on `pull_request` events, and work on
this repo routinely lands by committing straight to `main` — so for
months that gate had never had the opportunity to fail, and 10 of 30
recent commits carried no ticket reference at all, including one that
shipped a schema migration. `require-ticket-ref-push.yml` now checks
commit messages on push to `main`. Its only exemptions are merge commits
and subjects starting with `docs:` or `chore:`. If a code change ever
hides behind a `chore:` prefix to dodge the gate, tighten the exemption
list rather than disabling the workflow.

Rationale: DORA metrics and due-diligence audits depend on this link
existing by construction, not by convention someone remembered to
follow.

### 10. Loop engineering — autonomous ticket queue, test-gated (added 2026-09-20)

When Venkatesh asks Claude to "loop" or work a ticket queue autonomously
(e.g. "work 18 hrs/day," "keep going until we run out of tokens"), this is
what it means — there was no prior written definition, so this is the
first one:

- Claude works through a prioritized list of clearly-scoped tickets on its
  own, without asking permission before each one.
- Every change is test-gated: it ships only after its own tests pass
  (Rule 8 still applies in full — determinism tests, not "looks right").
  A ticket that can't be verified this way is not a candidate for the
  autonomous queue.
- Claude stops and asks — does not guess, does not proceed — the moment a
  ticket would require: spend or infrastructure (Rule 1, no exceptions),
  a schema change with real migration risk, or genuinely ambiguous scope
  where more than one reasonable interpretation exists.
- This is not unsupervised time-boxed grinding ("18 hours straight"). A
  Cowork session processes what it's given turn by turn; it does not run
  a literal clock in the background. The autonomous part is not needing a
  go-ahead between tickets — not skipping the judgment calls above.
- Every ticket worked this way still gets the end-of-session checklist
  (Rule 7): docs updated, Linear closed with commit SHAs, nothing left
  claiming "done" without a trace back to the commit.

