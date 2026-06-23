# Voice Agent Evaluation Framework — Implementation Plan (Index)

This is the **build** layer, sitting on top of `idea.md` (the *why* — locked
Findings 1-11) and `design.md` + `design/01-05` (the *how*). It does not
re-argue any decision or re-derive any design; it sequences the work, splits
it into parallelizable tracks, and attaches a TDD task list and concrete demo
criteria to each milestone.

**Authoritative dependency graph:** `CLAUDE.md`'s "Implementation Order"
table (Phase 0 → 6) is binding. This plan mirrors it exactly — one milestone
file per phase, same numbering as `design/`. Where a milestone is gated on an
**Open Spike**, that spike is listed as a blocking precondition (pulled
verbatim from the relevant design doc), never resolved here — per the 95%
confidence rule in `CLAUDE.md`, an unresolved spike is a question to answer
before building, not a gap to guess across.

## Milestone map

| Plan doc | Milestone | Phase (CLAUDE.md) | Design source | Owner |
|---|---|---|---|---|
| [plan/00-thin-slice.md](plan/00-thin-slice.md) | Thin end-to-end slice | 0 | design.md §build-sequencing | Eval team |
| [plan/01a-voice-eval-layer.md](plan/01a-voice-eval-layer.md) | `voiceeval` library core (order diff + LLM judge) | 1a | design/02 §1-2 | Eval team |
| [plan/01b-observability.md](plan/01b-observability.md) | Data Connection listener + deeper Langfuse tracing | 1b | design/01 | Eval team |
| [plan/02-backend-tickets.md](plan/02-backend-tickets.md) | Backend tickets (webhook dedup, eval tenant + Square Sandbox, joinUrl push) | 2 | idea.md F4/F9, design/01 spike A | Backend team |
| [plan/03-preprod-regression.md](plan/03-preprod-regression.md) | Promptfoo packs + call-driver harness + CI gate | 3 | design/03 | Eval team |
| [plan/04-production-pipeline.md](plan/04-production-pipeline.md) | `call.ended` continuous scoring Lambda | 4 | design/04 | Eval team |
| [plan/05-audio-quality.md](plan/05-audio-quality.md) | NISQA/DNSMOS/UTMOS audio scoring + calibration | 5 | design/02 §4 | Eval team |
| [plan/06-dashboards.md](plan/06-dashboards.md) | Engineering view + management scorecard | 6 | design/05 | Eval team |

## Dependency diagram (parallel vs. serial)

```
Phase 0 (thin slice) ───── must complete first; nothing else starts until it works
        │
        ├──────────────┬──────────────────────────────────────────┐
        ▼              ▼                                            ▼
   Phase 1a        Phase 1b                                    Phase 2 (backend)
 (voiceeval core)  (listener +                          (a) webhook dedup fix
        │           deeper tracing)                     (b) eval tenant + Square Sandbox
        │           BLOCKED on design/01                    (c) joinUrl push + tool
        │           join-permission spike                       call_id context
        │              │                                        │  (resolves design/01's
        │              │                                        │   join-permission spike)
        ├──────────────┴────────────────┐                       │
        ▼                                ▼                       │
   Phase 4                          Phase 6 (eng view)           │
 (prod pipeline)                    nearly free once 1b exists   │
   needs 1a + 1b's trace                                         │
   id convention                                                 │
        │                                                        │
        │            ┌───────────────────────────────────────────┘
        ▼            ▼
   Phase 6 (mgmt scorecard)     Phase 3 (preprod regression)
   needs Phase 4 producing      needs Phase 1a (library)
   scores at real volume        AND Phase 2b (eval tenant)
                                BLOCKED on design/03 audio-frame spike

Phase 5 (audio quality) ── NO incoming arrow from 0-4 above; gated only on the
                            20-50 call human-rated calibration sample EXISTING.
                            (Note: Track M (models) is fully independent; Track
                            S (sample collection) draws calls from Phase 4's
                            pipeline at 100% sampling during the calibration
                            window — see plan/05's Dependencies section. So
                            "parallel with 2-4" is accurate for the milestone
                            as a whole, but Track S specifically waits on
                            Phase 4 being live.)
```

**What can run truly in parallel (different engineers/agents, same time):**

- **1a, 1b, and 2** all start the moment Phase 0 is proven. 1a (the library)
  does **not** depend on 1b (the listener) — per `CLAUDE.md`'s table and
  Finding 2's loosened dependency graph.
- Within **Phase 2**, the three backend tickets (2a webhook dedup, 2b eval
  tenant, 2c joinUrl-push + tool `call_id` context) are independent of each
  other and of everything the eval team builds — file all three immediately.
- **Phase 5** (audio quality) is gated only on its calibration sample
  existing, not on phases 2-4 — it can run alongside them once the sample is
  collected.

**What is strictly serial:**

- Phase 0 before everything.
- **Phase 3 is blocked on Phase 2b** (the eval tenant + Square Sandbox creds)
  — it has nothing to run against without it. Also blocked on Phase 1a (needs
  the library) and on design/03's audio-frame-format spike.
- **Phase 4 needs Phase 1a** (the library) and **Phase 1b's trace-id
  convention** (it writes scores against `trace_id = call_id`).
- **Phase 6's management scorecard needs Phase 4** producing scores at real
  volume; the engineering view only needs Phase 1b.
- **Phase 1b's listener build is blocked on design/01's join-permission Open
  Spike** — do not write listener code before it resolves.

## Cross-cutting TDD conventions

Every milestone below (except Phase 2, which is backend ticket-tracking, not
code this team writes) follows the org-mandated workflow from
`~/.claude/rules/common/testing.md`:

- **RED → GREEN → REFACTOR → VERIFY** per unit of work:
  1. **RED** — write the failing test first; run it; confirm it fails for the
     right reason.
  2. **GREEN** — write the minimal implementation to pass.
  3. **REFACTOR** — improve structure with tests staying green (immutability,
     small functions <50 lines, files <800 lines, no deep nesting >4 levels —
     per `coding-style.md`).
  4. **VERIFY** — confirm 80%+ coverage on the unit of work.
- **All three test types are required across the project:** unit (pure
  functions, dataclasses), integration (DynamoDB reads, Langfuse writes,
  Square Sandbox calls, Promptfoo assertions), e2e (a real call through
  Ultravox → score → dashboard).
- **Minimum 80% coverage**, measured per package/milestone.
- **`voiceeval` is pure** (design/02 §5): no Promptfoo/Langfuse/network code
  inside the package, so its core is directly unit-testable against
  hand-built `Order`/`StateEvent`/transcript fixtures. Network calls
  (grounding fetch, audio fetch) live in thin wrappers outside the package or
  are injected as parameters.
- **Genuinely-not-unit-testable steps are called out explicitly** in each
  milestone (e.g. "place one real Ultravox call", "run NISQA on a real WAV"),
  with the alternative verification stated (golden-fixture WAVs, a manual
  smoke run, a recorded integration artifact) rather than pretending a mock
  proves them.
- **`evaluator_version`** (design.md envelope) is bumped on any
  rubric/model/threshold change, and tests assert old scores keep their
  original version — historical scores are never silently re-interpreted.

## North star alignment

The standing business problem (`idea.md` Finding 6): *"We have a live voice
ordering agent that we keep patching reactively, with no way to measure
whether changes actually improve it, no reviewable quality dashboard, and no
automated pre-release regression check. Order accuracy is the headline
metric, alongside escalation correctness and conversation completion."*

Every milestone must move one of those three headline metrics — **order
accuracy, escalation correctness, conversation completion** — or de-risk
something that blocks one. The table states the tie for each; any milestone
whose tie is "de-risking" rather than "directly moving a metric" carries an
explicit justification for why it's still in scope.

| Milestone | Headline metric it moves / de-risks | Justification if only de-risking |
|---|---|---|
| 0 Thin slice | Proves the whole measurement loop produces a real **order-accuracy** number on a real call | De-risks: validates the integration assumptions (Langfuse write, tenant partitioning, Promptfoo fit) before any deep investment — Finding 2 Option C |
| 1a voiceeval core | Directly computes **order accuracy** (diff) and **escalation correctness** (rubric: `escalation_handling.yaml`) | — |
| 1b Observability | Enables turn-taking/latency capture; feeds interruption metrics and gives the **trend line** that proves fixes work | De-risking: no headline metric is computed here, but it produces the trace substrate every score attaches to and the "are we getting better over time" signal Finding 6 names as the core gap |
| 2 Backend tickets | Protects **order-accuracy** integrity (dedup prevents double-counted orders) and unblocks the **regression check** (eval tenant) | De-risking/unblocking: Finding 4 dedup is a correctness precondition for trustworthy order-accuracy aggregates; Finding 9 tenant is the only thing that lets Phase 3 run a real order scenario without polluting production |
| 3 Pre-prod regression | Delivers the **automated pre-release regression check** Finding 6 names directly; gates on **order accuracy** + **escalation** + **completion** per scenario | — |
| 4 Production pipeline | Continuously scores real calls on **order accuracy, escalation, completion** — the steady-state engine behind the whole north star | — |
| 5 Audio quality | Voice-quality/naturalness — a secondary metric, not a headline one | De-risking justification required: this milestone does **not** move a headline metric. It stays in scope only because Finding 7 already scoped it as adopt-not-build (low cost) and it rounds out the management scorecard's customer-experience picture. It is correctly sequenced **last** and gated on its own calibration sample, so it never delays a headline-metric milestone. |
| 6 Dashboards | Makes **order accuracy, escalation, completion** trends reviewable — the "reviewable quality dashboard" Finding 6 names directly | — |
