# Plan: Phase 4 — Production Scoring Pipeline

## Scope

A new, decoupled Lambda that continuously scores real production calls and
writes results into Langfuse (design/04; Findings 6, 10). It is the steady-state
engine behind the north star — the "are we getting better over time" trend
(Finding 6) comes from this running on every real call. Specifically:

- A **new, separate Lambda** subscribed to `call.ended` (EventBridge rule or an
  existing internal event) — explicitly **not** an edit to
  `ultravox_webhooks.py` (design/04 §1, to avoid the Finding 4 blast radius).
- The pipeline: idempotency check → fetch inputs → score via `voiceeval` →
  write `EvalScore`s to the `call_id`-keyed Langfuse trace (design/04 §2).
- **Audio-scoring sampling** (10-20% default, 100% during calibration) —
  design/04 §3, but the audio scorer itself is Phase 5.
- **`is_eval_tenant` tagging** on every score so Phase 6 can exclude eval
  calls from aggregates (design/04 §4; design.md single derivation rule).

## Dependencies

- **Phase 1a** — the `voiceeval` library (design/04 calls into design/02).
- **Phase 1b's trace-id convention** — it writes scores against
  `trace_id = call_id` (CLAUDE.md table; design/04 §2 step 4). Note: Phase 4 can
  write scores even before the listener is fully live —
  `score_interruptions()` is skipped gracefully if the listener isn't running
  for a call (design/04 §2 step 3), so partial coverage is fine.
- Read-only access to `call-messages`, `call_order_facts`,
  `transaction_details` (design/04 §1 — the Lambda is purely additive,
  read-only against production data, writes only to Langfuse).

## Parallel work breakdown

- **Track I — infra/trigger.** The new Lambda + EventBridge rule (or internal
  event subscription), read-only IAM to the three DynamoDB tables and Langfuse
  write only (design/04 §1). Independent of the scoring-glue track.
- **Track G — scoring glue.** Fetch inputs → call `voiceeval` functions →
  assemble `EvalScore`s → write to Langfuse with `is_eval_tenant` tag (design/04
  §2). Depends on 1a (the library) but not on Track I's infra to develop/test
  in isolation.
- **Track D — idempotency / compute-avoidance.** The `call_id#event_type`
  marker that skips redundant LLM-judge + audio inference (design/04 §2 step 1)
  — shares the mechanism with Phase 1b Track C. Independent of Tracks I and G.
- **Track A — sampling logic.** The configurable audio-scoring sample-rate gate
  (design/04 §3). Independent; the audio scorer it calls is Phase 5, so this is
  built with `score_audio_quality()` injected/stubbed until Phase 5 lands.

## TDD task list

- **Track D — idempotency marker (unit + integration):**
  - RED: failing test that a second invocation for the same
    `call_id#event_type` skips the expensive LLM-judge/audio path while the
    Langfuse trace write remains a harmless idempotent upsert (design/04 §2
    step 1).
  - GREEN: marker check (DynamoDB conditional, short TTL).
  - VERIFY: skip path + first-time path covered.

- **Track G — input fetch + score assembly (unit-testable with injection):**
  - RED: failing test feeding hand-built `call-messages`/`call_order_facts`/
    `transaction_details` fixtures and asserting the assembled `EvalScore`s
    (correct `metric_name`, `value`, `tenant_id`, `scenario_id = None` for
    production calls per design.md envelope).
  - GREEN: orchestration calling injected `voiceeval` functions.
  - REFACTOR: keep fetch, score, and write as separate small functions
    (coding-style.md).
  - VERIFY: assembly covered with `voiceeval` and Langfuse client both injected.

- **Track G — `is_eval_tenant` tagging (unit-testable):**
  - RED: failing test that a score for an `eval-`-prefixed tenant is tagged
    `is_eval_tenant = true` and a normal tenant `false`, using design.md's
    single derivation rule (`tenant_id.startswith("eval-")`) — NOT re-derived
    here (design/04 §4).
  - GREEN: apply the shared derivation.
  - VERIFY: both cases covered.

- **Track A — sampling (unit-testable):**
  - RED: failing test that the sample-rate gate selects ~the configured fraction
    of calls and forces 100% when the calibration flag is set (design/04 §3).
  - GREEN: deterministic-seeded sampling gate.
  - VERIFY: default-rate and calibration-window cases covered.

- **Track I — Lambda trigger (NOT unit-testable):**
  - The EventBridge subscription + Lambda wiring is infra — **not
    unit-testable**. Alternative verification: an integration test invoking the
    Lambda with a synthetic `call.ended` event and asserting scores land on the
    Langfuse trace; plus a manual e2e smoke run on one real `call.ended`.
  - Also assert (integration) the Lambda has **no** write path to production
    order tables (design/04 §1 — purely additive, read-only).

- **`score_interruptions()` graceful skip (unit-testable):**
  - RED: failing test that when no listener state events exist for a call, the
    pipeline skips interruption scoring without erroring (design/04 §2 step 3 —
    partial coverage is fine).
  - GREEN: the skip guard.
  - VERIFY: skip path covered.

## Demo criteria

- A real `call.ended` event triggers the new Lambda, which writes
  `order_accuracy`, the configured rubric scores (incl. `escalation_handling`),
  and conversation-completion as `EvalScore`s onto that call's Langfuse trace
  (share the trace URL showing the scores).
- A replayed duplicate `call.ended` produces **no duplicate scores** (idempotent
  upsert) and **skips** the LLM-judge/audio recompute (observable via the marker
  / logs).
- An `eval-`-tenant call's scores are tagged `is_eval_tenant = true` (visible on
  the trace).

## North-star validation

Phase 4 is the continuous engine producing the headline metrics — **order
accuracy, escalation correctness, conversation completion** — on real calls at
volume (Finding 6). Before done: spot-check several real calls' written scores
against human reading of those calls, and confirm `is_eval_tenant` is correctly
set on every score (Phase 6's aggregates depend on it). **"Not aligned, needs
rework"** looks like: scores disagree with obvious human judgment of real calls;
`is_eval_tenant` is missing/wrong (eval calls would pollute the management
scorecard — design.md convention); or the Lambda has any path to mutate
production order data (design/04 §1 violation — it must be read-only/additive).

## Open spikes that must resolve first

**None block Phase 4's core.** Phase 4 depends on Phase 1b's trace-id
convention but **not** on Phase 1b's join-permission spike being resolved —
`score_interruptions()` is skipped gracefully when the listener isn't live
(design/04 §2 step 3), so Phase 4 ships with partial coverage and gains
interruption data later once 1b's spike + listener land. Audio scoring is gated
on Phase 5 (its own calibration sample), and Track A is built with the audio
scorer injected/stubbed until then.
