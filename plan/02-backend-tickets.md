# Plan: Phase 2 — Backend-Team Tickets

## Scope

This milestone is **not code this eval team writes.** It is the filing,
tracking, and acceptance of backend-team work in the separate
`echobot-studio-backend` repo, which later phases depend on (CLAUDE.md table
Phase 2; handoff.md "Do this in parallel, starting now"). Three tickets:

- **2a — Webhook dedup fix** (idea.md Finding 4). A pre-existing production
  data-integrity bug, independent of this project's timeline.
- **2b — `eval-` test tenant + Square Sandbox credentials** (idea.md Finding 9).
  **Phase 3 cannot start without this.**
- **2c — design/01 spike-A enablement: push `joinUrl` to the listener** at call
  creation (design/01 §2 Approach A), plus confirming/adding `call_id` to each
  tool Lambda's invocation context (design/01 §2). This is the backend half of
  Phase 1b's unblocking.

Because this is ticket-tracking, the "TDD" section below is expressed as
**acceptance criteria**, not RED→GREEN→REFACTOR — the eval team does not own
the implementation or its tests.

## Dependencies

- **None** — CLAUDE.md table Phase 2: "these can and should start immediately."
  File all three now, in parallel with Phases 1a/1b, not after.
- 2c's `joinUrl`-push detail is the resolution path for design/01's
  join-permission spike — so 2c and the Phase 1b spike are the same coordinated
  conversation.

## Parallel work breakdown

All three tickets are **independent of each other** and can be filed and worked
simultaneously by the backend team:

- **2a (dedup)** — touches `src/webhooks/ultravox_webhooks.py` only.
- **2b (eval tenant + Square Sandbox)** — touches tenant registration +
  `bot_config` + a Square developer sandbox account; no overlap with 2a.
- **2c (joinUrl push + tool `call_id`)** — touches `create_web_call()` /
  `ultravox_service.py` and the tool-injection code path; no overlap with 2a
  or 2b.

The eval team's parallel responsibility is **tracking + acceptance verification**
of each, not implementation.

## TDD task list → Acceptance criteria (ticket-tracking, not RED/GREEN/REFACTOR)

**Ticket 2a — webhook dedup fix.** Copy idea.md Finding 4's problem statement
verbatim into the ticket (it's written to be pasted). Acceptance:

- A durable **DynamoDB conditional write** dedup marker
  (`pk=DEDUP#{call_id}#{event_type}`, with TTL, `ConditionExpression=
  attribute_not_exists(pk)`) is added *before* event processing; a duplicate
  `call.ended` returns `200` without reprocessing (Finding 4 decision, point 1).
- Verified by the backend team's own test that a replayed duplicate webhook does
  not produce duplicate rows in `CALLS` / `CALL_EVENTS` / `CALL_MESSAGES` /
  `CALL_RECORDINGS` / `TRANSACTION_DETAILS`.
- **Eval-side note (this team's scope, NOT this ticket):** the eval ingestion
  layer dedups defensively on read regardless of this ticket's status (Finding 4
  decision point 2) — that defensive check is built in Phase 1b/Phase 4, so the
  eval framework's correctness does not depend on 2a's timeline.

**Ticket 2b — `eval-` test tenant + Square Sandbox credentials.** Copy idea.md
Finding 9's decision + addendum verbatim. Acceptance:

- A dedicated tenant with an **`eval-` prefixed `tenant_id`** is registered
  through the normal tenant-setup process, with `bot_config` pointing to Square
  **Sandbox** credentials (not a real merchant account) — Finding 9 decision.
- A test call that completes an order against this tenant lands in Square
  Sandbox (no real order/charge/kitchen ticket) while still exercising the real
  `completeorder` / Square code path.
- **Square Sandbox GET-order (read-back) access** is available to the eval
  harness using the test tenant's Sandbox credentials only — never production
  Square read access (Finding 9 addendum).
- **This team's blocking-status check:** confirm this ticket's filing and
  completion status before starting Phase 3 — design/03 §"Hard dependency"
  states zero sandbox/test-tenant concept exists today and Phase 3 has nothing
  to run against without it.

**Ticket 2c — joinUrl push + tool `call_id` context.** Tied to design/01's
join-permission spike (Approach A). Acceptance:

- `create_web_call()` forwards the `joinUrl` to the listener service (internal
  HTTP call or SQS message) at call creation (design/01 §2 Approach A).
- The spike confirms an observer WebSocket can join that `joinUrl` without
  disrupting the customer's session (this is the design/01 spike itself — if
  Approach A is ruled out, fall back to Approach B per design/01).
- `call_id` is confirmed present (or added) in each tool Lambda's invocation
  context, enabling Phase 1b's tool-call spans (design/01 §2).

## Demo criteria

- **2a:** the backend ticket is closed; a replayed duplicate `call.ended` in a
  backend test environment produces exactly one row per affected table (backend
  team's verification artifact linked in the ticket).
- **2b:** a real call placed against the `eval-` tenant completes an order that
  appears in **Square Sandbox** (sandbox order ID retrievable via GET-order with
  the sandbox credentials) and produces **no** real merchant order.
- **2c:** the listener service receives a `joinUrl` for a freshly created call
  (logged/observed), and a tool invocation carries `call_id` in its context.

## North-star validation

Phase 2's tie is **de-risking/unblocking**, not directly moving a metric (per
plan.md's table):

- 2a protects **order-accuracy integrity** — without it, duplicate `call.ended`
  events double-count orders in `transaction_details`, silently corrupting the
  very headline metric this project produces (Finding 4 impact section).
- 2b is the **only** thing that lets Phase 3 run a real order-completion
  regression scenario without creating a real restaurant order — it directly
  unblocks the automated pre-release regression check Finding 6 names.
- 2c unblocks Phase 1b's listener (turn-taking/latency signal).

**"Not aligned, needs rework"** looks like: 2a re-enables only the old Valkey
check (Finding 4 explicitly rejects this stopgap as the final fix); 2b uses a
real merchant account instead of Square Sandbox (would pollute production order
data — Finding 9 rejected); or 2b's tenant lacks the `eval-` prefix (breaks the
`is_eval_tenant` exclusion every downstream aggregate relies on — design.md
convention).

## Open spikes that must resolve first

- **design/01's join-permission spike** is resolved *through* ticket 2c
  (Approach A) — it is the work item, so it doesn't "block" Phase 2; rather,
  Phase 2c is where it gets answered, which is what unblocks Phase 1b.
- No spike blocks 2a or 2b — file both immediately (handoff.md).
