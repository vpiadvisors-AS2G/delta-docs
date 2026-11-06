---
name: delta-ui-uxv1
description: "Design/build UI for the Delta/3WIM Freight Overbilling Detection platform (finance-team queue, three-way match invoice detail, leadership analytics, executive overview, carrier performance). Use whenever the user asks for screens, dashboards, mockups, or front-end code for this product — even if they just say 'the dashboard' or 'the invoice review screen' without naming the skill."
---

# Delta / 3WIM Freight Overbilling Detection — UI/UX system

## What this product is
A three-way match system that catches freight overbilling for mid-market US/Canada manufacturers: it compares the rate contract, the rate confirmation, and the carrier invoice, and flags discrepancies before payment.

## Terminology — "party" and "counterparty" are banned words
Never use "party" or "counterparty" anywhere in UI copy, code, or docs for this product (Venkatesh, 2026-09-14 — the schema itself moved away from a generic party model on 2026-09-02 for the same reason). Always name the specific role: Manufacturer, Carrier, Retailer, Raw-Material Supplier. The one grandfathered exception is the existing `counterparty_identifiers` database table name — don't invent new UI copy that echoes it.

## Personas — confirmed 2026-09-14
Design every screen with a specific persona in mind, not a generic "user." Use these names exactly — they are locked, not placeholders:

**Platform-level (cross-tenant, internal):**
- **Delta Admin** — global admin, sees across all tenants, provisions accounts. Never mixed into tenant-scoped screens.

**Tenant-level (the manufacturer's own staff — the users of the 5 core pages):**
- **Company Admin** — manages the tenant's users, carriers, retailers, suppliers, integrations, billing. Needs a Settings/Users area beyond the 5 core pages.
- **Analyst** — works the exception queue day to day. This is who Invoice Review Queue, Invoice Detail, and Carriers are built for.
- **Executive** — Overview and Analytics & Savings only, read-only, no queue access. When building persona-gated navigation, Executive's nav shows ONLY these two tabs — hide Queue/Detail/Carriers entirely, don't just disable them.

**External (non-tenant, scoped to only their own records — not yet built, but name them correctly when asked):**
- **Carrier User** — sees disputes/documentation requests raised against their own carrier only.
- **Retailer User** — same, for retail deductions.
- **Supplier User** — same, for Raw-Material Supplier, added 2026-09-14 (reverses an earlier "no supplier role" decision — see DELTA_DECISIONS.md). Do not build supplier-facing screens speculatively; the underlying schema repoint (addresses/contracts/etc. accepting a supplier owner, tracked in Linear AS2-100) is not yet done.

Persona/RBAC implementation itself (auth, RLS, role model) is tracked separately in Linear (AS2-101) — this skill governs how each persona's screens should look once that lands, not the backend mechanics.

## Information architecture
Overview (exec summary) · Invoice Review Queue · Exceptions · Invoice Detail / Three-Way Match · Carriers · Analytics & Savings · Documents / Data Sources · Settings.
Every page in this IA carries the same top nav (with the same tab order and the same active-tab styling) and the same right-hand AI panel — never build a page without both, UNLESS the page is being built specifically for the Executive persona, in which case only Overview and Analytics & Savings apply and the nav shows only those two tabs.

**Shell override (2026-09-20, CLAUDE.md "Delta design" note): the published canvas wins over the paragraph above on shell shape.** Build a left sidebar on Deep Navy, not a top nav — same tab set, same active-tab treatment, just oriented as a sidebar. Read the canvas (`Artifact` tool, `action: "read"`, https://claude.ai/artifact/HutMmZyqbbML4itXqZDvkE) before building or changing any screen; don't redesign this shell from the paragraph above once the artifact is reachable.

## Core workflows and metric definitions (do not blur these)
1. **Potential overbilling** — unvalidated variance flagged by the system, not yet reviewed by a person.
2. **Validated overbilling** — a person confirmed the variance is real.
3. **Recovered credits** — carrier credit actually received/confirmed.
4. **Prevented payments** — amount withheld or corrected before payment went out.
5. **Net savings realized** — recovered + prevented only. Never includes unresolved/open exposure — that is a liability figure, not a savings figure, and must never be summed into it.

## Typography — face is locked, do not substitute; apply optical discipline on top

**Face (2026-09-20 override, canvas wins — see shell override above): system font stack, no webfont link.** `-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, ui-sans-serif, system-ui, sans-serif` for body/UI; keep numeric/tabular data (dollar amounts, percentages, counts, dates) on a monospace or tabular-figure treatment — `ui-monospace, "SF Mono", "Segoe UI Mono", monospace`, always with `font-variant-numeric: tabular-nums`. This replaces the original Public Sans / IBM Plex Mono + Google Fonts link mandate below; do not re-litigate it per page or per session.

Apply a `.num` utility class (or equivalent) consistently to every numeric value across every page — do not hand-roll font-family per number.

**Optical discipline (added — adapted from Apple's WWDC 2020 "Details of UI Typography"), applies to whichever face is active:**
- Tracking (letter-spacing) is size-specific, never one value for all sizes. Large display numbers (hero KPI figures, big dollar totals) want slightly *negative* tracking; small text (table cells, labels, badges) wants near-zero or slightly positive tracking for legibility. Tighten headings and hero metrics, leave body and table text near `0`.
- Leading (line-height) tracks size inversely — tighter on large headings/hero metrics, looser on body copy and dense table rows where scanability matters more than density.
- Build hierarchy from weight + size + leading together, not size alone. Prefer a weight bump over a size bump when emphasizing a value inline (e.g. an exception count inside a sentence) — it adds presence without breaking layout rhythm.
- Scale layout with the text (`rem`/`em` spacing, not fixed `px`) so the app respects the user's OS/browser text-size setting without breaking.

## Visual system — semantic color, not decorative
Use these exact tokens everywhere. Never introduce a new hex value for something these already cover, and never repurpose a token for a meaning it doesn't own.

| Token | Hex | Use |
|---|---|---|
| Industrial Navy | #102A43 | primary text, headings |
| Deep Navy | #0B1F33 | sidebar / header background |
| Trust Blue | #2563EB | primary actions, links, baseline chart series |
| Blue Hover | #1D4ED8 | hover state of Trust Blue elements |
| Signal Orange | #D97706 | exceptions/review flags and unresolved exposure ONLY — never decorative |
| Pale Amber | #FEF3C7 | background for orange-flagged badges/pills |
| Savings Green | #15803D | confirmed savings ONLY (recovered, prevented, net savings) — never decorative |
| Pale Green | #DCFCE7 | background for green badges/pills |
| Slate Text | #334155 | secondary text, labels |
| Border | #CBD5E1 | all card/table borders and dividers |
| Work Surface | #F8FAFC | page background |
| White | #FFFFFF | card/panel background |
| Error Red | #B91C1C | overdue/error states — sparingly, never for routine warnings |

Do not introduce light "accent" tints (e.g. the lighter green/amber sometimes used for contrast on a dark banner) anywhere except directly on the Deep Navy banner background — they exist solely to stay legible on navy, not as a second palette.

**Materials — explicitly excluded, do not import.** Apple's translucent-glass system (`backdrop-filter` blur on nav/toolbars/sheets, layered frosted materials) is NOT part of this product's visual system and must not be introduced, even experimentally. It directly conflicts with the rule immediately above (flat, opaque White/Work Surface panels; no new light tints outside the Deep Navy banner). If a translucent surface is ever genuinely wanted somewhere (e.g. the AI panel), that is a scoped design decision requiring Venkatesh's explicit sign-off — never a default applied while building a screen.

## Charts — type AND color are both locked per data shape
Color rule: blue = baseline/actuals, orange = unresolved exposure ONLY, green = validated/recovered/prevented ONLY, gray = benchmark/reference line. Never color-only for status — pair color with a label or icon (WCAG AA: never convey state by hue alone).

Chart TYPE is picked by data shape, and the same shape must always render as the same chart type across every page — do not vary this by screen:
- **Ranked comparison across a small set of named entities** (top carriers by exposure, discrepancy rate by carrier, exposure by plant) → horizontal bar list, entities top-to-bottom by magnitude, value labeled at the bar's end. This is the SAME chart type wherever this shape occurs — Analytics' "top carriers" and Carriers' "discrepancy rate by carrier" are deliberately the same visual pattern, not a bug to "vary for interest."
- **Trend over time, single series** (savings trend, carrier's historical discrepancy trend) → line chart with a light dashed reference/benchmark line where relevant.
- **Trend over time, two comparable series** (savings vs. exposure by month) → grouped/paired bar chart, blue+orange or blue+green per the color rule above.
- **Composition/breakdown of one total** (reconciliation outcomes, needs-review by reason) → stacked bar or horizontal progress-bar list.
When a new page needs a chart, first check whether its data matches one of these shapes before inventing a new chart type — reuse, don't redesign.

## UX/interaction rules
- Every page: same sidebar nav, same active-tab treatment, same 320px-wide right-hand AI panel ("Ask Delta") with quick-prompt chips, a chat thread with inline example charts, and an input bar — except Executive-persona pages, which show only their two nav tabs.
- Every individual record (an invoice row, a carrier row) that has real supporting paperwork must link to it — a "View document"/"View docs" affordance next to the record, not buried only on its detail page. A page of pure aggregates (all-carrier/all-time charts with no single underlying record) does not get a fake per-item document link — don't force one just for consistency.

## Motion & interaction feel (added — adapted from Apple's WWDC "Designing Fluid Interfaces"/"Details of UI Typography," scoped to a desktop B2B app — not a touch/gesture product)

This section governs *how* things move; it never changes *what* color, font, or layout to use — those stay locked above. Scope note: Delta is a desktop-first data application, not a touch-driven consumer app, so gesture-specific material (drag-to-dismiss, swipe recognizers, flick-momentum projection, rubber-band drag physics, haptics) is deliberately left out below. If the product ever grows a genuine touch/mobile surface, pull those pieces from the `apple-design` skill directly rather than guessing them back in here.

- **Kill latency on every interactive control.** Hover/press feedback on buttons, table rows, and nav items must respond instantly (on `pointerdown`/`mouseenter`, not waiting for a debounce or a network round-trip). Anything on the input path that isn't essential — artificial timers, unnecessary debounces — is a regression.
- **Never lock out input during a transition.** A user must be able to close a drawer/modal/the AI panel and reopen it without waiting for the prior animation to finish, and re-triggering an already-open panel should not restart the animation from scratch.
- **Default to critically-damped springs, not fixed-duration easing, for anything the user directly opens/closes** (the AI panel, modals, dropdowns, the document viewer overlay). Damping `1.0` (no overshoot) is the default for opening/closing UI; reserve slight bounce (damping ~`0.8`) only for something that genuinely carried momentum into the motion (rare in this product — most opens/closes are click-triggered, not drag-released).
- **Animate from the current on-screen value, never the target value, when an animation is interrupted.** If a user re-clicks to reopen the AI panel while it's still closing, the reopen animation starts from wherever the panel actually is on screen, not from "closed."
- **Spatial consistency:** a panel/dropdown/modal enters and exits along the same path (the AI panel slides in and out from the right, not in from the right and out through the bottom). Anchor a dropdown/popover's transform-origin to the control that triggered it, not the viewport center.
- **Animate only compositor-friendly properties** (`transform`, `opacity`) for anything performance-sensitive — avoid animating `width`/`height`/`top`/`left` directly on frequently-shown elements like table row hover states or the AI panel.
- **Feedback comes in four kinds — use the right one:** status (a running/queued badge), completion (a toast or inline success state), warning (inline validation before submit, not after), error (inline, next to the field it concerns, not just a top-of-page banner). Validate inline where possible rather than waiting for submit.

## Reduced motion & reduced transparency (added, strengthens the WCAG AA rule below)
- Respect `prefers-reduced-motion: reduce`: replace slide/spring transitions (panel open/close, modal enter/exit) with a short opacity cross-fade, and drop any overshoot/bounce entirely.
- Respect `prefers-contrast: more` where applicable: fall back to solid backgrounds with a defined border rather than a soft/low-contrast divider.
- This product doesn't use translucent surfaces (see Materials exclusion above), so `prefers-reduced-transparency` has nothing to override today — keep this in mind only if that ever changes.

## Accessibility (WCAG AA)
4.5:1 contrast for normal text, 3:1 for large text/controls. Never convey status by color alone — always pair with text or icon.

## Voice
Concrete, evidence-based phrasing tied to actual numbers and dates. Never "AI magic," "unlock insights," or similarly vague marketing language.

## Craft review lens (added — adapted from Apple's "Principles of Great Design")
When reviewing a finished screen, check it against these, in this order — they're a review lens, not new rules that override anything above:
1. **Purpose** — does every element on this screen earn its place, or did something get added just because there was space?
2. **Familiarity** — do things that look the same behave the same, and live in the same place, across every page in the IA?
3. **Simplicity (not minimalism)** — is the most important number/action on the screen actually the most visually obvious one (via the locked hierarchy rules above), or is everything competing at the same weight?
4. **Craft** — is every spacing, timing, and alignment value a deliberate, defensible choice, or did something get eyeballed? Misaligned icons and inconsistent spacing read as carelessness in a finance product specifically — this audience notices.

## Deliverables checklist for a full build
- [ ] System-font stack used everywhere (no webfont link), per the 2026-09-20 shell/typography override
- [ ] Tracking/leading scaled by size per the optical-discipline rules above — not one fixed value everywhere
- [ ] Same color tokens used, no new hex values introduced, no translucent/glass surfaces introduced
- [ ] Same chart type per data shape across pages
- [ ] AI panel present on every page (except Executive-only pages, which drop the full nav too), opens/closes with a critically-damped spring, interruptible
- [ ] Reduced-motion fallback in place for any panel/modal/drawer transition
- [ ] Document links on every individual record with real paperwork behind it, none faked on pure-aggregate pages
- [ ] Metric definitions (potential/validated overbilling, recovered, prevented, net savings) used exactly as defined above, never conflated
- [ ] No use of "party"/"counterparty" anywhere in new copy or code — use the specific role name instead

## Where this fits in the Delta project
This is the UI/UX system for Project DELTA (AS2G-3WIM). DELTA_CONSTRAINTS.md's no-spend/no-provisioning rule still applies to anything this UI implies on the backend — this skill only governs front-end look, feel, structure, and consistency.
