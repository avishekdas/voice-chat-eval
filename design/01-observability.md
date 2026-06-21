# Design: Observability Plane

Implements [[Finding 1]] and [[Finding 3]]. Owns: Langfuse Cloud setup, the
Data Connection listener, and the trace/span schema everything else writes
into. Does **not** own scoring logic (→ [02](02-voice-evaluation-layer.md))
or how/when scores get triggered in production (→
[04](04-production-pipeline.md)).

## 1. Langfuse Cloud setup

- Hobby tier, single shared project, org membership restricted to the core
  1-2 eval engineers — no tenant/customer access ever ([[Finding 3]],
  [[Finding 5]]). No per-tenant projects; per-tenant RBAC isn't available on
  this tier, so splitting projects would be theater ([[Finding 5]]).
- Every trace and span carries `tenant_id`, `location_id`, `prompt_version`,
  and a boolean `is_eval_tenant` tag from day one, regardless of project
  structure — this is what makes a future migration to per-tenant projects
  (if ever needed) a query/export operation instead of re-instrumentation.
  `is_eval_tenant` is computed via `design.md`'s single derivation rule
  (`tenant_id.startswith("eval-")`), not re-derived independently here.

## 2. Data Connection listener

### What it is

A long-running Python (asyncio) service. For each in-progress call, it holds
a WebSocket connection to Ultravox's Data Connection channel and buffers:
`state` transitions (`idle`/`listening`/`thinking`/`speaking`, each
timestamped), incremental transcript deltas, and ping/pong latency samples.
On the existing `call.ended` webhook, it flushes its buffer for that
`call_id` into Langfuse spans and closes the trace.

### Open spike — must be resolved before writing listener code

[[Finding 1]] already flagged "how a third-party, non-client service joins
another session's Data Connection mid-call" as unresolved. This session's
research narrowed but did not close it:

- **Verified:** Ultravox's `/api/calls` create response includes a
  `joinUrl` — a `wss://` WebSocket stream URL — and joining that URL is how
  *any* client (browser SDK, telephony bridge, or a server-side WebSocket
  client) starts participating in the call's real-time session.
  ([Quickstart](https://fixie-ai.github.io/ultradox/guides/quickstart/),
  [The Start Call 2-Step](https://docs.ultravox.ai/gettingstarted/rule5))
- **Verified (this codebase):** today, only `echobot-studio-backend`'s
  `create_call()` (`ultravox_service.py:361-376`) ever sees this response —
  it is passed straight through to the route handler
  (`ultravox_routes.py:558-566`) and from there to the browser client. The
  listener service is not the one creating the call, so it has no way to
  discover a `joinUrl` for a call it didn't initiate.
- **Not verified (genuine unknown, not yet researched to a confident
  answer):** whether Ultravox's protocol allows a *second*, independent
  WebSocket connection to join the same `joinUrl` purely to observe (without
  acting as the primary audio client and without disrupting the real
  customer's session). The "monitoring call events or routing data to
  external systems" framing in Ultravox's Data Connection docs
  ([Protocol & Data Messages](https://docs.ultravox.ai/apps/datamessages))
  suggests this kind of secondary/observer connection is an intended use
  case, but this has not been confirmed against the actual protocol spec.

**Two candidate approaches to resolve this, in spike-priority order:**

| Approach | Mechanism | Why it's preferred / risk |
|---|---|---|
| **A. Push (recommended first spike)** | Small addition to `create_web_call()` in `echobot-studio-backend`: forward the `joinUrl` to the listener service (e.g. an internal HTTP call or SQS message) the moment a call is created. | Small, scoped backend change — same shape/size as the [[Finding 4]] and [[Finding 9]] backend tickets. Fully within our control; doesn't depend on an unconfirmed Ultravox multi-consumer capability. |
| **B. Pull via webhook + lookup** | On the existing `call.started` webhook, call an Ultravox REST endpoint to retrieve the `joinUrl` for that `call_id`, if such a lookup exists. | [[Finding 1]] already confirmed the Get Call REST endpoint exposes no latency/transcript data; whether it exposes the `joinUrl` post-creation has not been checked. Depends on an Ultravox capability we haven't verified exists. |

**Decision for this design: spike Approach A first.** It requires
coordinating one small change with the backend team rather than betting the
whole listener on an unconfirmed third-party API capability. Do not write
any listener code until this spike either confirms A works or rules it out
in favor of B.

### Trace / span schema

- **Trace:** `id = call_id` (see `design.md`'s cross-cutting convention).
  Tags: `tenant_id`, `location_id`, `prompt_version` (read from
  `tenant_prompts` per `idea.md` §0.2's note on `switch-prompt`),
  `is_eval_tenant`.
- **Turn span:** one per `listening → thinking → speaking` cycle. Span id is
  **deterministic**: `hash(call_id + turn_index)`, where `turn_index` is the
  listener's own ordinal count of state cycles within the call — re-running
  the same buffered event stream through the flush step regenerates
  identical ids, making span writes idempotent upserts rather than
  duplicate-prone appends.
- **Tool-call span:** emitted **directly by each Lambda tool** at the moment
  it's invoked ([[Finding 1]], point 3) — not by the listener, which cannot
  see server-side tool calls at all. Each tool Lambda must receive `call_id`
  in its invocation context (today it receives `x-user-id`/`interaction_id`
  per `idea.md` §0.2; whether `call_id` specifically is already present in
  that context was not confirmed this session — verify before assuming it's
  free; if missing, it's a small addition to the same tool-injection code
  path the backend already uses to inject `x-api-key`/`x-user-id`).
- **Latency:** TTFB-equivalent per turn = `speaking_started_at -
  thinking_started_at`, taken directly from the `state` timestamps — the
  genuine signal [[Finding 1]] set out to recover.

### Idempotency layering

Deterministic trace/span ids (above) make duplicate writes safe by
construction — a duplicate `call.ended` delivery now produces an identical
upsert, not a duplicate. This **supersedes** needing the
[[Finding 4]] dedup check for trace correctness specifically. It does
**not** remove the need for *some* "already processed" check elsewhere:
re-running the flush step twice would still mean re-doing wasted compute
(e.g. re-deriving turn boundaries) even though the final Langfuse write is
harmless. Keep a lightweight `call_id#event_type` marker (DynamoDB item,
short TTL) purely as a **compute-avoidance optimization** in the listener's
flush step — not as the correctness mechanism, which the deterministic ids
now own.

### Audio handling

Recordings are fetched on demand from Ultravox per `call_id`, mirroring the
backend's existing `get_call_recording()` pattern (`idea.md` §0.2) — **not**
persisted into our own S3 by default, to avoid creating an uncontrolled
second copy of customer audio ([[Finding 8]]). The one exception is the
[[Finding 7]] calibration sample (20-50 human-rated calls), which is
intentionally persisted; its storage location and access control are gated
on the [[Finding 8]] revisit triggers, not decided here.

## 3. Sequencing

Per `design.md`'s build-sequencing recap: the **thin slice** should prove
the Langfuse write path using the simplest viable trace source (even a
manually-triggered single-tenant join), not the production-grade listener
described above. Don't build listener robustness (reconnect logic, buffer
overflow handling, multi-call concurrency) until the thin slice has
validated that Approach A's push mechanism actually works end-to-end.

## Out of scope for this doc

- Scoring logic that consumes these traces/spans → [02](02-voice-evaluation-layer.md).
- What triggers scoring to run, and how scores get attached → [04](04-production-pipeline.md).
- Dashboard rendering of this data → [05](05-dashboards.md).
