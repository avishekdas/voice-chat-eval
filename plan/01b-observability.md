# Plan: Phase 1b — Data Connection Listener + Deeper Langfuse Tracing

## Scope

Recover the real-time turn-taking/latency signal that webhooks alone can't
provide, and deepen Langfuse tracing beyond Phase 0's manual write (design/01;
Finding 1). Specifically:

- The **Data Connection listener** — an asyncio service that joins each call's
  WebSocket, buffers `state` transitions + transcript deltas + ping/pong, and
  flushes into Langfuse spans on `call.ended` (design/01 §2).
- The **trace/span schema** — deterministic trace id (`call_id`) and
  deterministic span ids (`hash(call_id + turn_index)`) for idempotent upserts
  (design/01 §2 trace/span schema, §idempotency layering).
- **Tool-call span emission** from each Lambda tool (design/01 §2; Finding 1
  point 3) — coordinated with the backend, since the listener cannot see
  server-side tool calls.
- **Latency** = `speaking_started_at - thinking_started_at` per turn
  (design/01 §2).

## Dependencies

- **Phase 0 proven** (CLAUDE.md table).
- **AND design/01's join-permission Open Spike resolved first** — CLAUDE.md
  table: "do not write listener code before that's settled." handoff.md lists
  this as the blocking spike for 1b.
- The tool-call-span piece depends on **Phase 2** confirming whether `call_id`
  is already in each tool Lambda's invocation context (design/01 §2 notes this
  is "not confirmed this session — verify before assuming it's free").

## Parallel work breakdown

The spike must close before the listener code starts, but several pieces can
run in parallel once it does — and one piece (the schema) can be written
*independently of the spike entirely*.

- **Pre-spike, can start now (independent of join-permission spike):**
  - **Track S — trace/span schema + deterministic id logic.** The
    `trace_id = call_id` and `span_id = hash(call_id + turn_index)` derivation,
    and the turn-boundary derivation from a buffered state-event stream, are
    pure functions over event data (design/01 §2). These need **no live
    connection** to develop and test — build and unit-test them against
    hand-built event streams while the spike is being run.

- **Blocked on the join-permission spike (design/01 §2 Open Spike → spike
  Approach A first):**
  - **Track L — listener connection + buffering.** Only after the spike
    confirms how the listener obtains a `joinUrl` (Approach A: backend push, the
    recommended first spike) and that an observer WebSocket can join without
    disrupting the customer's session.

- **Parallel, backend-coordinated (Phase 2 overlap):**
  - **Track T — tool-call span emission.** Each Lambda tool emits its own span
    on invocation (design/01 §2). Independent of the listener; depends on
    confirming/adding `call_id` to the tool invocation context (a small backend
    change, same shape as the Phase 2 tickets).

- **Parallel (independent):**
  - **Track C — compute-avoidance marker.** The lightweight
    `call_id#event_type` DynamoDB marker (short TTL) used only to skip redundant
    flush compute — NOT for correctness (design/01 §idempotency layering).
    Independent of the listener internals.

## TDD task list

- **Track S — deterministic ids (unit-testable):**
  - RED: failing test asserting `trace_id == call_id` and that
    `span_id(call_id, turn_index)` is stable across repeated calls (idempotency
    by construction, design/01).
  - GREEN: implement the deterministic id derivation.
  - REFACTOR: small pure helper module.
  - VERIFY: covered.

- **Track S — turn-boundary derivation from state events (unit-testable):**
  - RED: failing test feeding a hand-built `listening→thinking→speaking` event
    stream and asserting the correct turn spans + per-turn latency
    (`speaking_started_at - thinking_started_at`).
  - GREEN: implement the boundary/latency derivation.
  - REFACTOR: keep parse separate from id assignment.
  - VERIFY: covered, including a duplicate-event-stream test proving re-running
    the flush regenerates identical span ids (idempotent upsert).

- **Track L — listener connection + buffering (mostly NOT unit-testable):**
  - The WebSocket join + live buffering is integration/e2e against Ultravox —
    **not unit-testable**. Alternative verification: an integration test
    against a recorded Data Connection event stream replayed into the buffer,
    plus a manual e2e smoke run joining one real call and confirming buffered
    `state`/transcript/ping-pong events. The flush-to-Langfuse step is verified
    by the Track S unit tests on the pure derivation, plus an integration test
    confirming spans land on the `call_id` trace.
  - Robustness (reconnect, buffer overflow, multi-call concurrency) is
    explicitly deferred until the thin slice / first real join proves the basic
    path (design/01 §3) — do not build it speculatively.

- **Track T — tool-call span (integration):**
  - Not pure-unit (it emits from a Lambda). Verification: an integration test
    asserting an invoked tool produces a span on the correct `call_id` trace,
    after confirming `call_id` is present in the invocation context.

- **Track C — compute-avoidance marker (integration):**
  - RED-ish: a test asserting a second flush for the same `call_id#event_type`
    skips the expensive recompute while the Langfuse write remains a harmless
    upsert. Backed by DynamoDB integration; mark the conditional-write path.

## Demo criteria

- One **real call** produces a Langfuse trace (`id = call_id`) with per-turn
  spans showing genuine `listening→thinking→speaking` timestamps and a computed
  TTFB-equivalent latency per turn (share the trace URL — design/01's
  "genuine signal Finding 1 set out to recover").
- Re-running the flush for the same call produces **no duplicate spans** (the
  same trace upserts) — demonstrable by flushing twice and showing the trace is
  unchanged.
- At least one tool-call span appears on a real call's trace (Track T).

## North-star validation

1b doesn't itself compute a headline metric — its north-star tie is
**de-risking** (per plan.md's table): it produces the trace substrate every
score attaches to and the over-time trend signal Finding 6 names as the core
gap ("no way to tell whether fixes are making the agent better over time").
Before done: confirm the per-turn latency and turn-taking timestamps are
**real captured values**, because Phase 4's interruption metric and the
scorecard's latency KPI depend on them being genuine, not synthesized.
**"Not aligned, needs rework"** looks like: latency/turn columns are empty or
fabricated (exactly the failure Finding 1 set out to avoid by rejecting
post-call-only reconstruction), meaning the trend line can't be trusted.

## Open spikes that must resolve first

**Blocking (design/01 §2 Open Spike), pulled verbatim:** *how a third-party,
non-client service joins another session's Data Connection mid-call to observe.*
Resolve via **spike Approach A first** (small backend change to push the
`joinUrl` to the listener at call creation — design/01's decision), confirming
both that the push works and that an observer WebSocket can join without
disrupting the customer's session. **Do not write any listener code (Track L)
until this spike confirms A works or rules it out in favor of B.** Track S (the
pure schema/id logic) is the only part that may proceed before the spike
closes.

Secondary precondition (not a spike, a verification): confirm whether `call_id`
is already present in each tool Lambda's invocation context before assuming
Track T is free (design/01 §2).
