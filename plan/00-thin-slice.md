# Plan: Phase 0 — Thin End-to-End Slice

## Scope

Prove the entire measurement loop on **one** real call before investing depth
anywhere (Finding 2 Option C; design.md §build-sequencing recap step 1;
handoff.md "Start here"). Four pieces, kept deliberately thin:

1. One real call → **one Langfuse trace**, via the simplest viable path —
   explicitly **NOT** the Data Connection listener (that is Phase 1b, gated on
   its own spike). A manual/simplest-possible trace write is enough
   (design/01 §3).
2. **One order-accuracy score** for that call, computed directly off existing
   `transaction_details` data — a **throwaway script**, NOT the `voiceeval`
   library yet (design/02 §1; handoff.md).
3. **One Promptfoo scenario passing** against an **already-completed** real
   call using Promptfoo's **native** assertions only — no custom library, no
   call-driver harness (design/03 §1; handoff.md).
4. Confirm/create the Langfuse Cloud Hobby-tier account (Finding 3).

Nothing else (1a/1b/3/4/5/6) starts until all four work together on real data.

**Resolved scope note (was flagged in critical review as a contradiction
between two locked Findings, then resolved with the user):** idea.md's
Finding 11 establishes that Promptfoo cannot place or drive a live Ultravox
call on its own — it has no native mechanism to open a session, speak as a
caller, and capture turns; that capability is exactly what Phase 3's
call-driver harness exists to build. Phase 0 explicitly excludes that harness.
Resolution: Phase 0's Promptfoo scenario does **not** drive a live call. It
grades the transcript/tool-call payload of a real call that was already
placed (via the existing `POST /calls/web` flow, outside Promptfoo) — a
weaker but immediately buildable interpretation of "against the live agent"
that needs zero new harness code. Track C below reflects this.

## Dependencies

- **Langfuse Cloud Hobby-tier signup only** — no new infra (CLAUDE.md table,
  Phase 0 row).
- Read access to the existing `transaction_details-{branch}-{env}` DynamoDB
  table for one real call (idea.md §0.2).
- **One already-completed real call** to operate against — placed via the
  existing `POST /calls/web` flow (idea.md §0.2), or simply a recent real call
  that already exists. All three tracks below read back from this same call;
  none of them places or drives a live call as part of Phase 0.
- **No design doc gates this phase** (design.md §build-sequencing: "no design
  doc gates this — build directly").

## Parallel work breakdown

These are **three independent tracks** that only converge at the end — assign
to three engineers/agents simultaneously. They share two things that must be
agreed up front (a 30-minute alignment, not a blocking dependency): the
cross-cutting `tenant_id`/`call_id` conventions (design.md), and **the literal
`call_id` of the one already-completed real call** all three tracks will
operate against. Settling that one `call_id` first is what makes the three
tracks genuinely independent — none of them needs to wait on another to place
or generate a call.

- **Track A — Langfuse trace write.** Sign up for Hobby tier; write the
  simplest script that creates one trace with `id = call_id` and the
  `tenant_id`/`location_id`/`prompt_version`/`is_eval_tenant` tags. Depends on
  nothing but the signup.
- **Track B — order-accuracy throwaway script.** Read the agreed call's
  `transaction_details`, parse the order, compute a simple order-accuracy
  number against a hand-specified expected order. Depends only on DynamoDB
  read access. Independent of Track A.
- **Track C — Promptfoo native-assertion scenario.** Stand up one
  `promptfooconfig.yaml` with one scenario whose native assertions grade the
  agreed call's already-captured transcript/tool-call payload (expected text /
  tool / payload) — not a live-driven call. Independent of A and B.

**Convergence (serial, at the end):** stitch the three into one demonstrable
loop — the one agreed-upon real call whose trace exists in Langfuse, whose
order-accuracy number is computed, and whose scenario passes in Promptfoo.
This is the only serial step.

## TDD task list

Phase 0 is deliberately throwaway, so the bar is "lightweight tests that prove
each track does what it claims," not full 80% coverage on code that will be
replaced by `voiceeval` in 1a. Apply RED→GREEN→REFACTOR where there's real
logic; use smoke-verification where the step is inherently I/O against a live
system.

- **Track B — order-accuracy parse + compare (genuinely unit-testable):**
  - RED: write a failing test that feeds a hand-built `transaction_details`
    record + a hand-specified expected order and asserts the computed score.
  - GREEN: minimal parse-and-compare function.
  - REFACTOR: keep it small; note this is throwaway and will be superseded by
    `voiceeval.score_order_accuracy()` in 1a — do not over-build.
  - VERIFY: the comparison logic is covered; the DynamoDB fetch is integration,
    tested with a recorded/sample item.

- **Track A — Langfuse trace write (NOT unit-testable; it's an external I/O
  call):**
  - Alternative verification: a manual smoke run that writes one trace, plus a
    screenshot/URL of the trace appearing in the Langfuse UI. Optionally a thin
    integration test that writes a trace with a known `call_id` and reads it
    back via Langfuse's API to confirm `trace_id = call_id` idempotency
    (design.md convention).

- **Track C — Promptfoo native-assertion scenario (NOT unit-testable; it
  grades a real call's captured output):**
  - Grading a real call's already-captured transcript/tool-call payload is
    **not unit-testable** by design — the input is real production data, not
    a hand-built fixture, and the point is to prove the assertion runs against
    something genuinely real (Finding 9 rationale: a simulated agent/transcript
    doesn't catch real regressions). Alternative verification: the
    `promptfoo eval` command exiting 0 with the scenario passing against the
    real call's captured data, captured as a CI/console artifact.

## Demo criteria

A single runnable demonstration, on **one already-completed real call**:

- The Langfuse trace for that `call_id` is visible in the Langfuse Cloud UI
  (share the trace URL), tagged with `tenant_id` and `is_eval_tenant`.
- The throwaway script prints an order-accuracy number for that same
  `call_id`, computed from its `transaction_details` record.
- `promptfoo eval --config <config>` exits 0 with the one scenario passing via
  native assertions, grading that same real call's already-captured
  transcript/tool-call payload (console/CI output captured) — this is the
  e2e test for Phase 0, by definition: it is the only step that chains a real
  call's real data through to a pass/fail result.

Vague "works end-to-end" is not acceptable — the three concrete artifacts
above (trace URL, printed score, green Promptfoo run) are the bar.

## North-star validation

Before calling Phase 0 done: the printed order-accuracy number must be a
**real number computed from a real call's real order data**, not a stub or a
hardcoded value — because order accuracy is the headline metric (Finding 6),
and the entire point of Option C (Finding 2) is to prove that the loop can
produce that headline number on real data before any deep build. **"Not
aligned, needs rework"** looks like: the score is faked/hardcoded, the trace
is synthetic rather than from a real call, or the Promptfoo scenario grades a
mocked/synthetic transcript instead of a real call's captured output — any of
which means the integration assumptions are not actually de-risked and the
loop hasn't been proven.

## Open spikes that must resolve first

**None.** Per design.md (§build-sequencing) and handoff.md ("Open spikes still
blocking deeper phases (not Phase 0)"), no Open Spike blocks Phase 0. The
join-permission spike (design/01), the audio-frame spike (design/03), and the
Hobby-tier dashboard spike (design/05) all block *later* phases, not this one —
which is exactly why the trace write here uses the simplest manual path, not
the listener.
