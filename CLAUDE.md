# Project DELTA (3WIM)

B2B SaaS for automated deduction reconciliation and freight overbilling recovery.
Mid-market Manufacturing, Retail, and Logistics. Owner: Venkatesh Iyer, AS2G.

Two use cases run on **one matching engine**:

- **DEDUCTION mode** — invoice × purchase order × receipt
- **OVERBILLING mode** — freight invoice × rate agreement × shipment

Never build a second engine for the second mode. Mode is configuration, not a code path fork.

---

## Guiding Principles

Cross-cutting, always-true principles — apply to every task in this project, not situational like the numbered rules below.

- **No guessing, ever.** Verify with real facts — read the actual file, grep the actual code, query the actual live table, run the actual test — before stating anything as fact, every time, not just when reporting a task done. Full detail: Rule 14.
- **Never seed data to paper over a logic gap.** If the real pipeline can't produce a result, fix the pipeline — never hand-insert the result it should have produced. Full detail: Rule 13.
- **Verify before reporting done.** Rebuild, restart, and check the actual live result before claiming a fix works — never from the source edit alone. Full detail: Rule 12.
- **Audit completely, report once.** When investigating a problem area, find everything before saying anything — not one issue per message as each is discovered. Full detail: Rule 15.

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
| Frontend | Next.js 14, containerised (`apps/web/Dockerfile`, Node 22), on **Azure Container Apps** (Consumption) in `rg-valoraiq-dev` / `cae-valoraiq-dev`. Not App Service, not AKS (decided 2026-11-05). TLS terminates at the Container Apps ingress (HTTPS only; TLS 1.0/1.1 refused). Azure Front Door + WAF are planned, **not provisioned today** |
| Image registry / CI | GitHub Container Registry (GHCR); GitHub Actions deploys by OIDC federated credential (no stored Azure secret) via `.github/workflows/azure-deploy.yml` |
| Identity | Microsoft Entra External ID customer tenant `ValoraIQ` (sign-in on `valoraiq.ciamlogin.com`); app roles come from `user_roles` (Admin / Analyst / Viewer), not Entra groups |
| Data | Supabase Postgres with RLS (free tier) |
| Queue / events | Azure Service Bus |
| Blob | Azure Blob Storage |
| Secrets | GitHub Actions secrets/variables and Container Apps secrets today. Azure Key Vault is planned, **not provisioned today** |
| LLM | OpenRouter (model choice is config, never hardcoded) |
| Telemetry | OpenTelemetry → OTLP → Azure Monitor |

**Node versions:** the web container image and local tooling use Node >= 22.13 because the pinned pnpm (11.24.0) needs it (Node 20 fails `pnpm install` with `ERR_UNKNOWN_BUILTIN_MODULE node:sqlite`, seen in the first azure-deploy run 2026-11-05). The Azure Functions agents stay on Node 20 as above; nothing in this table changes them. The repo `engines` field says `>=20`.

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

- `trace_id` (AS2-42) is OpenTelemetry request-tracing plumbing only: an in-flight span id propagated through Service Bus message `applicationProperties` (`packages/shared/src/telemetry/propagation.ts`) and stamped onto structured log lines (`telemetry/logger.ts`). It is not written to any database column — nothing persists it, so it cannot be queried after the fact. Corrected 2026-09-28 (AS2-151): this file previously called it "the data-lineage story for governance," which overclaimed a persisted, queryable record that doesn't exist.
- The real, persisted data-lineage chain — a reconciliation decision tracing back to source document → extraction → model → confidence score — exists independently of `trace_id`, through ordinary foreign keys: `reconciliations.invoice_id` → `invoices.source_document_id` → `documents`, and `extractions.document_id` + `extractions.model_used`/`prompt_version` (each `extraction_fields` row carries its own `confidence`). That chain is real and queryable today; only the claim that `trace_id` is *how* you query it was wrong.
- Structured logs only. No `console.log` in committed code.
- Migrations are forward-only and checked in. No manual schema edits against any database.
- Secrets come from GitHub Actions secrets, Container Apps secrets, or `.env.local` (Key Vault is planned, not yet provisioned). Never committed, never in code, never in a Linear issue.
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
- **The "separate summarization tool" is Gemini** (confirmed 2026-10-03):
  the vault root has its own `GEMINI.md` defining an "Ingest raw" command —
  when Venkatesh runs it in Gemini pointed at this vault, Gemini scans
  `01-Raw` for new/changed files, synthesizes `02-Wiki` notes (with
  frontmatter + `[[wikilink]]`s), updates `02-Wiki/index.md`, appends to
  `log.md`, and regenerates a consolidated status report in `03-Output`
  (`Project-DELTA-Execution-Plan-and-Status-Report.md`). This is a working,
  already-established division of labor, not something to replicate: Claude
  Code writes `01-Raw` (Rule 7) and may read `02-Wiki` for deeper context,
  but never writes to `02-Wiki` or `03-Output` directly — that's Gemini's
  output, triggered by Venkatesh, not by a Claude Code session.
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
- **Superseded (corrected 2026-10-03): Claude Code, not Cowork, is now the
  primary execution surface for Project DELTA.** The 2026-09-04 decision
  below (strikethrough-equivalent, kept for history) picked Cowork because
  at the time it was the only surface with direct local filesystem access.
  That's no longer true — Claude Code (this terminal/CLI) has full native
  filesystem, shell, git, test-runner, and Supabase-MCP access and has been
  the surface actually doing every real commit, migration, and test run on
  this project for weeks. Running Cowork and Claude Code on the same repo
  in parallel, hand-relaying context between them, is the thing to stop
  doing now — pick Claude Code as primary. Cowork remains fine for
  planning/chat-only work that doesn't touch the repo directly.
- ~~**Decision (2026-09-04): use Cowork exclusively for Project DELTA.**
  Running both chat and Cowork on this project was a likely root cause of
  the doc/Linear/codebase split-brain state found this session — chat
  sessions could only ever see the `delta-docs` mirror (which lags real
  commits), while Cowork sessions saw the live repo. Do not use plain
  claude.ai chat for this project going forward.~~

**Fallback mechanism (still valid, chat-only, plain claude.ai without Claude
Code or Cowork):** `docs/DELTA_*.md` and `CLAUDE.md` are mirrored to a public
repo cloneable at session start:

  https://github.com/vpiadvisors-AS2G/delta-docs

Use this mirror only if neither Claude Code nor Cowork's device bridge is
available. Verify freshness against the latest commit date before relying on
it.

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

## "Delta design" — the UI standard (superseded 2026-10-04, AS2-192)

**The 2026-09-20 canvas below is no longer current.** Venkatesh decided
(AS2-192, recorded in `docs/DELTA_DECISIONS.md`, commit `10e28eb`) to
replace it with an Apple Human Interface Guidelines look. **Status as of
this writing: the decision is committed to `main`; the implementation
(AS2-193, commit `4333ddd` — shell + design-token migration to the Apple
HIG palette) is NOT yet merged to `main`.** It lives on `demo-integration`
and on `vpiadvisors/as2-193-apple-hig-redesign` (worktree:
`.worktrees/as2-193-apple-hig-redesign/`). Until that merges, anyone
building against `main` is still looking at the old canvas-based shell in
`apps/web` — check which branch you're actually on before assuming either
standard is live.

**Where the written rules live, same split as before — only the look
changed:** read the full reasoning in `docs/DELTA_DECISIONS.md` (AS2-192)
before building or changing any screen once AS2-193 is merged.

- **Shell and typography: Apple HIG.** Left sidebar, translucent blur,
  system font stack (`system-ui, -apple-system, "Segoe UI", sans-serif` —
  same family as the 2026-09-20 canvas, no webfont change). Headline
  numbers stay plain ink (`--text-primary`), never colored — color is a
  supporting accent only (a dot, a chip, a chart line).
- **Everything else: still follow `delta-ui-uxv1`.** Terminology (never
  "party"/"counterparty" — name the role), personas, the five metric
  definitions, chart type per data shape, WCAG AA. Colour *tokens* are now
  the Apple HIG palette; the *semantics* they carry (orange = unresolved
  exposure, green = confirmed savings/matched, red = disputed) are
  unchanged.

**Superseded, do not build from:**
- The 2026-09-20 canvas (`HutMmZyqbbML4itXqZDvkE`) — the prior standard.
- The two 2026-09-13 artifacts (Freight Billing App `B4hFpraFvURYVyKEYuBprq`,
  Reconciliation Dashboard `VkuvDmZuvASn1TrrBidMhf`) — already superseded
  before this change, still superseded now.

**Implementation (pending merge):** `apps/web/src/app/globals.css` carries
the design tokens; the app shell is `apps/web/src/app/(app)/layout.tsx`.
Tokens are defined once as CSS custom properties — never introduce a new
hex value for something a token already covers. Every existing live-data
query, server action, and `AskDelta`/`ask-delta.tsx` wiring must be
preserved exactly — this is a visual-layer migration, not a rebuild of any
page's logic.

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
  Claude Code session processes what it's given turn by turn; it does not
  run a literal clock in the background. The autonomous part is not needing
  a go-ahead between tickets — not skipping the judgment calls above.
- Every ticket worked this way still gets the end-of-session checklist
  (Rule 7): docs updated, Linear closed with commit SHAs, nothing left
  claiming "done" without a trace back to the commit.

### 11. Agent team — architect / developer / tester (added 2026-10-03)

Five custom subagent definitions live in `.claude/agents/`: `delta-architect`,
`delta-developer`, `delta-tester`, `delta-reviewer` (added 2026-10-03 —
security/tenant-isolation/portability specialist; exists because AS2-6
already had one real near-miss on bare-`id` cross-tenant FKs, caught by an
ultrareview, not by a generalist dev pass), and `delta-frontend` (added
2026-10-03 — owns the locked `delta-ui-uxv1` design system end to end so
`delta-developer` doesn't have to hold colors/personas/chart-type rules in
its head on top of everything else). They exist to parallelize real
implementation work and to directly address this project's single most
repeated bug class — a new document type's extraction/transform code using
field names or fallback shapes that don't match the other 4 document types',
caught only after it shipped and a human found null data live (AS2-171
through AS2-174, and AS2-176/AS2-177 found by `delta-tester`'s first real
run, were all this same mistake, one document type at a time).

**How to use them:**
- Before implementation starts on anything touching schema, the matching
  engine, or more than one `transform-*-compute.ts` file: spawn
  `delta-architect` first. It reviews scope against this file's hard rules,
  checks the new work against the *other* document types' existing
  conventions, and produces a plan + acceptance check. It does not write
  code.
- Hand that plan to one or more `delta-developer` agents. Independent
  tickets can run as parallel `Agent` calls in one message, each in its own
  worktree (`isolation: "worktree"`) to avoid collisions.
- After a developer reports done, spawn `delta-tester` independently. It
  re-runs the real tests, checks the acceptance criteria, and specifically
  audits cross-document field-name consistency — the exact class of bug
  this rule exists to stop. It does not fix what it finds; it reports back.
- This composes with Rule 10's loop engineering: an autonomous ticket queue
  can cycle architect → developer → tester per ticket instead of one
  undifferentiated pass, without changing any of Rule 10's stop conditions
  (spend, schema risk, ambiguous scope still halt it, same as before).
- This does not mean an unsupervised 24/7 background build. Spawned agents
  run within an active Claude Code session and report back into it — the
  same honest limit Rule 10 already states for "looping."

### 12. Verify before reporting done (added 2026-10-06)

Never tell Venkatesh a fix is deployed or a page renders correctly without
having actually confirmed it — not "the code change looks right," not "the
build succeeded," not "I checked the text content so it's probably fine."
Confirm the actual, current, running artifact: rebuild AND restart whatever
process serves it (editing a source file is not enough when the server is
`next start`, a production build, not `next dev`), then check the real
output — a screenshot if the tooling allows it, page content if not, a live
network/DB query if that's what the claim is about. If verification tooling
itself is broken (e.g. a stuck browser automation CDP connection), say that
explicitly rather than silently downgrading to a weaker check and reporting
success anyway.

**Why:** 2026-10-06 session, twice: (1) reported the mock-auth 6-tenant fix
as done and asked Venkatesh to refresh the login page — the server was
still serving the old build, so he saw stale content and had to catch it.
(2) Repeated the identical mistake immediately after fixing the login page's
overflow bug — edited the file, ran a content-only check, and told him it
was fixed without confirming the rebuilt server was actually serving it
first. His own words: "You do such things and every time I end up finding
these faults... You have to test before you tell me, otherwise we end up
burning double or triple the tokens and we end up wasting time."

**How to apply:** treat "I edited the file" and "it's deployed and working"
as two separate claims that each need their own evidence. Before reporting
a UI fix, a config change, or a bug fix as done: rebuild, restart the actual
serving process, and verify the live result through whatever channel most
directly confirms the specific claim being made. Asking the user to check
is a fallback for when verification tooling is genuinely unavailable, not a
substitute for trying first.

**"Complete" has a definition, added same day after a second correction:**
no task — code, UI, UX, or otherwise — gets called complete, done, or fixed
unless it has actually passed, not "should pass": the real test suite (not
skipped, not partial), a real check of UI/UX against the locked design
system, a real check that the functionality does what was asked, and a real
look for security issues in anything touching data or auth. "Looks right,"
"should work," and "the build succeeded" are not completion — a passing
test run, an actual rendered page, an actual exercised code path are. If
real uncertainty exists about scope, requirements, or what "done" means for
a given task — UI, UX, functionality, or otherwise — stop and ask Venkatesh
before proceeding, rather than guessing and finding out later. Venkatesh's
words: "No task is complete unless it passes all the tests... It has to be
solid, tested in every sense. UI, UX, functionality, security. Job complete
should mean something. It's not just you put together some piece of junk."

### 13. Never seed data to paper over a logic gap (added 2026-10-06)

If the real pipeline can't produce a result, the fix is to fix the pipeline
— never to hand-insert the result it should have produced. This applies to
any fabricated intermediate or computed record (a shipment, a carrier, a
contract, a reconciliation, anything the matching engine is supposed to
derive on its own), not to legitimate test/demo *input* data (real-shaped
documents fed through the real ingestion pipeline, which is exactly how
this project's synthetic demo datasets are meant to be loaded — see Rule on
staged document ingestion in memory/project notes). The distinction: feeding
real-shaped inputs through the real system is fine; manually writing the
system's own output by hand because the system couldn't produce it is not.

**Why:** found live, 2026-10-06 — `supabase/migrations/20260920000001_
freight_overbilling_demo_seed.sql` is a real, checked-in example of exactly
this mistake. Its own header admits the actual gap ("zero carriers
matching... zero contracts/shipments at all — so the overbilling engine had
nothing to compare against and never produced a finding") and then works
around it by hand-inserting a fabricated carrier ("Apex Logistics
Solutions" — not a real carrier, not from any real document), a fabricated
contract (`DEMO-APEX-001`, an invented $4,500 flat rate), and the shipment
row itself. That gap — the pipeline has no step that creates a `shipments`
row from a rate confirmation — was real on 2026-09-20 and is *still* real on
2026-10-06, three weeks later, because the seed hid it instead of fixing
it. Venkatesh, on learning this: "Going forward we WILL NOT SEED ANY
DATASET. If there is a gap in the logic we fix it."

**How to apply:** when a demo, a test, or a real customer's data exposes
that some step of the matching/reconciliation engine doesn't actually work
yet, say so plainly and either fix the real gap (per Rule 11's
architect-first process when it touches the matching engine or more than
one `transform-*-compute.ts` file) or leave the gap visibly unfixed and
flagged — never close it with a hand-written row that makes the UI look
like the engine did something it didn't.

### 14. No guessing, ever — verify with real facts before every answer (added 2026-10-06)

Never answer a factual question about this codebase's behavior, schema,
data, or history from memory, assumption, or general domain knowledge
alone. Read the actual file, grep the actual code, query the actual live
table, or run the actual test — every time, before stating something as
fact, not after. A claim already verified earlier in the same session can
be reused without re-checking it again; a claim never yet verified in this
conversation must be checked before it's said out loud.

**Why:** this same session, asked whether the Shipment entity comes from
the BOL document, the first answer ("BOL creates the shipment") was wrong
— said from general freight-domain knowledge, not from reading the actual
code. The correct answer (a Rate Confirmation/tender creates it; BOL only
looks one up) only came out once `transform-bol-queries.ts` was actually
read. Venkatesh: "always verify with actual facts, code, etc before
responding to me. There should be NO GUESSING or SURMISE EVER."

**How to apply:** before answering what a table contains, what a function
does, whether something is fixed/seeded/working, or why a result looks a
certain way — check it first (Read, Grep, a live SQL query, an actual test
run). If verification genuinely isn't possible in the moment (a tool is
down, the information doesn't exist to check), say that plainly — "I can't
verify this right now" — rather than filling the gap with a plausible-
sounding guess. This is Rule 12's "verify before reporting done" applied
to every factual claim, not only "is this task complete."

### 15. Audit completely, report once (added 2026-10-06)

When investigating a problem area — a bug, a tenant's broken data, a
merge, anything with more than one thing potentially wrong — find
everything first, then report it all together. Never surface issues one
at a time across multiple messages as each is separately noticed, making
Venkatesh ask follow-up questions to drag out findings that were already
discoverable in the same investigation.

**Why:** 2026-10-06 session, Beacon Manufacturing Co's broken data was
reported in at least five separate passes across many messages — the
currency/subtotal bug, then the shipment-creation architecture gap, then
a BOL duplicate-key error (self-inflicted), then two documents with zero
extracted fields, then duplicate `match_review_queue` rows for PODs — each
surfacing only when Venkatesh asked a follow-up question or happened to
look in the right place, not because a complete audit was done and
reported up front. His words: "You keep nibbling on issues instead of
giving all of them at the same time. Very Bad!"

**How to apply:** before reporting findings on any investigation, ask: have
I actually checked everything relevant, or am I reporting the first thing
I tripped over? Run the full set of checks a thorough audit would include
(all affected rows, not just the one that triggered the question; all
related tables, not just the first one queried; all files a change
touches, not just the one that errored) — then write one consolidated
report. A genuinely new issue discovered later (e.g., the user asks a
different question that surfaces something unrelated) is fine to report
when found; an issue that a complete first pass would have already caught
is not.

### 16. Agent Coordination — Muse ↔ Claude Code Protocol (added 2026-10-07)

Claude Code coordinates with Muse (strategic Lead Architect) through
Linear, team **AS2G-Delta** — not direct chat, not ad hoc instruction.
Venkatesh does not broker every ticket; the two agents hand off through
the channel below.

- **Channel:** Linear project "Agent Channel — Muse ↔ Claude Code".
- **Protocol first:** read issue **AS2-210** at the start of every
  session — it defines handoff format, hard rules, and stop conditions.
  Follow it exactly.
- **Work queue:** issue **AS2-211**, priority order. Pull the topmost
  Todo ticket; move it to In Progress when starting.
- **Ticket readiness:** a ticket is self-contained — objective,
  requirements, edge cases, exception handling, acceptance stated as
  input → expected output, TDD notes (same bar as Rule 8's "acceptance
  check before implementation starts"). A ticket missing acceptance
  criteria is not ready — comment asking for it rather than guessing scope.
- **Reporting back:** move to Done AND comment with commit SHA(s), branch
  name, test results (what ran, pass/fail), deviations from spec, anything
  that could not be verified. A Done without SHAs is not done — same
  standard as Rule 7's end-of-session checklist. Run `graphify update .`
  after code changes.
- **Blocked:** comment `BLOCKED: <reason> / Need: <what>`, keep the ticket
  In Progress. Never guess through ambiguity — verify against the repo
  first (Rule 14).
- **Hard stops — ask via issue comment, do not proceed:** any spend or
  infrastructure (Rule 1, no exceptions), a schema change with real
  migration risk, genuinely ambiguous scope. Never push to `main` without
  Venkatesh's go-ahead — branches and PRs only. Never touch the shared live
  Supabase database. Never seed data to paper over a logic gap — fix the
  pipeline instead (Rule 13).
- **Never pull these tickets:** AS2-166 (blocked on schema snapshot),
  AS2-39 (needs spend approval), anything deduction/3-way-match (deferred
  per the v1 scope decision). **AS2-199 is listed in the source protocol
  doc as carrying a "do-not-ship verdict" — flagging an unresolved
  conflict rather than silently picking a side:** the 2026-10-07 session
  fixed AS2-199's critical shipment-duplication bug end to end (tests,
  review-queue fixes, tenant/carrier context on errors) and that fix is
  merged and pushed on `demo-integration` (commits `f057be1`..`d84b666`,
  follow-ups through `9b889dd`). Whoever owns the do-not-ship verdict needs
  to confirm whether it's stale (superseded by the fix that shipped) or
  whether something about that fix still doesn't meet it — do not treat
  either claim as authoritative until that's resolved.
- **Quality loop — "do it once, do it right," non-negotiable, not
  performative:**
  - *Context loop:* re-read the ticket and every file to be touched before
    coding; state the plan in a ticket comment; after each step, write back
    what changed and what the tests said. Never act on memory of an earlier
    step — stale context is how "done" bugs ship.
  - *Graph:* `graphify query "<what>"` / `graphify path` before planning,
    to map blast radius; `graphify update .` after changes. If the graph is
    missing or stale, say so on the ticket — don't work blind.
  - *Harness:* acceptance criteria become automated tests FIRST, then
    code. Done means the full gate green — lint, types, unit/integration
    tests, determinism test for matching logic (Rule 8), coverage
    thresholds, security scan. "Looks right" is not done (Rule 12). Name
    explicitly on the ticket anything that cannot be verified automatically.
