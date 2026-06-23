# Plan: Phase 6 — Dashboards (Engineering View + Management Scorecard)

## Scope

The dashboard/reporting layer (design/05; Findings 5, 6). Reads scores written
by Phase 4 and Phase 3 — **computes nothing itself** (design/05 intro):

- **Engineering / QA view** — native Langfuse Cloud trace explorer, filtered by
  `tenant_id`. Nothing custom to build (design/05 §1).
- **Management scorecard** — the 9-KPI view (design/05 §2 KPI list), filtered to
  exclude `is_eval_tenant = true`. Two candidate approaches (A native Langfuse,
  B Grafana export), **gated on an Open Spike** (design/05 §2).
- **Release-readiness view** — Promptfoo's own run output, consumed from CI
  (design/05 §release-readiness; already covered by Phase 3).

**Note (fixed by the plan review):** `design/05-dashboards.md`'s Approach-A
row previously said "10-KPI scorecard," a stale reference predating the
WER-KPI removal documented later in the same file. It has been corrected to
"9-KPI" to match the actual KPI list and this plan's count.

## Dependencies

- **Engineering view: Phase 1b** — "nearly free once Phase 1b exists (native
  Langfuse UI)" (CLAUDE.md table Phase 6; design/05 §1).
- **Management scorecard: Phase 4** — needs Phase 4 producing scores at real
  volume before there's anything meaningful to show (CLAUDE.md table Phase 6;
  design/05 intro).
- **Gated on design/05's Hobby-tier dashboard Open Spike** (which approach is
  viable) before committing to A or B.
- KPI 5 (Interruption recovery) has **no/partial data until Phase 1b's
  join-permission spike + listener are live** (design/05 KPI 5; handoff.md
  "Already resolved by the review").

## Parallel work breakdown

- **Track E — engineering view.** Configure the native Langfuse trace explorer
  with `tenant_id` filtering (design/05 §1). Essentially configuration, not a
  build — available as soon as Phase 1b traces exist. Independent of the
  scorecard track.
- **Track K — scorecard, Approach decision + build.** First run the Open Spike
  (does Hobby tier support custom score-aggregation views?). Then:
  - **If A (native Langfuse):** build the 9-KPI scorecard in Langfuse.
  - **If B (Grafana export):** build the small export Lambda that pulls scores
    via Langfuse's API and exposes them as Prometheus metrics into the existing
    EC2 Prometheus/Grafana stack (design/05 §2 Approach B). B also resolves the
    Finding 5↔6 access tension (only aggregated scores, not transcripts, leave
    Langfuse — design/05 §"Why Approach B is more than a fallback").
  - Depends on Phase 4 scores existing. Independent of Track E.
- **Track R — release-readiness view.** Already covered by Phase 3's CI output
  (design/05 §release-readiness) — no new build; just confirm CI results are
  consumable. Independent.

## TDD task list

- **Track E — engineering view (NOT code — configuration):**
  - Native Langfuse UI; **not unit-testable**. Verification: a manual check that
    the trace explorer filters correctly by `tenant_id` and shows per-call
    spans/scores (design/05 §1) — captured as a screenshot/URL.

- **Track K — scorecard aggregation filter (unit-testable, if Approach B):**
  - If B's export Lambda computes any aggregation: RED — failing test that the
    aggregation **excludes** `is_eval_tenant = true` records before computing
    each KPI (design/05 §2; design.md convention — eval calls must never blend
    into production KPIs). GREEN — apply the shared derivation filter. VERIFY —
    both included and excluded cases covered.
  - RED: failing test that each of the 9 KPIs maps to the correct underlying
    `EvalScore.metric_name`(s) and renders an empty/partial state gracefully
    when no data exists (e.g. KPI 5 interruption recovery before the listener is
    live — design/05 KPI 5). GREEN: KPI mapping. VERIFY: covered.

- **Track K — export Lambda I/O (NOT unit-testable; Approach B):**
  - The Langfuse-API pull + Prometheus-metric exposure is integration — **not
    unit-testable**. Alternative verification: integration test pulling scores
    from a Langfuse test project and asserting the exposed Prometheus metrics;
    plus a manual check the metrics appear in Grafana.

- **Track R — release-readiness (verification only):**
  - No new code; confirm Phase 3's `promptfoo eval` output is visible as a CI
    artifact (design/05 §release-readiness).

- **e2e (manual, this milestone's required e2e test per plan.md's
  cross-cutting conventions):** place one real call, let Phase 4 score it,
  and confirm the score appears correctly in both the engineering trace view
  and the management scorecard within a reasonable time window — the chain
  "real call → score → dashboard" is the literal e2e definition from
  plan.md, named explicitly here rather than left implicit in the Demo
  criteria below.

## Demo criteria

- **Engineering view:** a shareable Langfuse trace-explorer URL filtered to one
  `tenant_id`, showing real calls with spans + scores (design/05 §1).
- **Management scorecard:** a live dashboard URL (Langfuse if A, Grafana if B)
  rendering the 9 KPIs, with `eval-` tenant calls **excluded** from the
  aggregates — demonstrable by placing an `eval-` call and showing it does NOT
  move the production KPIs. KPI 5 (interruption) and KPI 7 (voice quality)
  render as empty/partial with a clear "no data yet" state if their upstream
  (Phase 1b listener / Phase 5 calibration) isn't live yet (design/05 KPI 5/7).
- **Release-readiness:** a CI run link showing the Promptfoo pass/fail summary
  (design/05 §release-readiness).

## North-star validation

Phase 6 delivers the **"reviewable quality dashboard"** Finding 6 names
directly — making **order accuracy, escalation correctness, conversation
completion** trends visible to "anyone who needs to review the agent's quality."
Before done: confirm the scorecard's order-accuracy / escalation / completion
KPIs match the underlying real Phase 4 scores (no transformation drift), and
that `eval-` calls are excluded from production aggregates (design.md
convention; design/05 §2). **"Not aligned, needs rework"** looks like: a KPI
shows a number that doesn't trace back to real `EvalScore` records; eval-tenant
calls leak into production aggregates (Finding 9 / design.md violation — the
"fake test orders pollute the very metrics this project exists to produce"
failure Finding 9 warns about); or (if access is widened) raw transcripts become
broadly viewable, violating Finding 5 — which Approach B specifically prevents
by keeping only aggregated scores in Grafana (design/05 §"Why Approach B...").

## Open spikes that must resolve first

- **Blocking the scorecard (design/05 §2 Open Spike), pulled verbatim:**
  *whether Langfuse Cloud's Hobby tier supports custom score-aggregation
  dashboards (beyond its default trace/score views) was not verified — needs a
  direct check against current Langfuse Cloud features before committing to
  either path.* Spike A first (free to check); fall back to B (Grafana export)
  if Hobby tier doesn't support it (design/05 §2 recommendation). Do not build
  the scorecard until this resolves which approach to take (95% rule).
- **Upstream dependency (design/05 KPI 5):** KPI 5 (Interruption recovery) has
  no/partial data until design/01's join-permission spike resolves and the
  Phase 1b listener is live — the dashboard renders whatever Phase 4 has
  written, it does not backfill (design/05 KPI 5). Not a blocker for building
  Phase 6, but the KPI must render a graceful "no data yet" state until then.
- **Upstream dependency (design/05 KPI 7):** Voice quality KPI is empty until
  Phase 5's calibration completes (design/05 KPI 7).
