# Plan: Phase 3 — Pre-Prod Regression (Promptfoo + Call-Driver Harness + CI Gate)

## Scope

The full pre-prod regression layer (design/03; Findings 2, 9, 10, 11):

- **Promptfoo scenario packs**, organized `scenarios/{tenant_id}/*.yaml` per
  the design.md scenario schema, with native `python` assertions that call into
  `voiceeval` (design/03 §1).
- The **call-driver harness** — a Python service that places a real Ultravox
  call against the `eval-` tenant, drives a scripted persona, and captures the
  transcript/audio (design/03 §2).
- **Square Sandbox read-back** — GET the created order back from Sandbox to use
  as the `actual_order` ground truth for test-tenant calls (design/03 §3;
  Finding 7 correction).
- **CI release-gate wiring** — a `just eval-gate` recipe in the backend's
  `deploy-reusable.yml` (design/03 §4) — backend-coordinated.

## Dependencies

- **Phase 1a** — needs the `voiceeval` library (CLAUDE.md table; design/03
  calls into design/02 for all scoring).
- **Phase 2b** — the `eval-` test tenant + Square Sandbox credentials.
  **Hard blocker:** design/03 §"Hard dependency" — "Verify ticket status before
  starting any work below, or the call-driver harness and Square Sandbox
  read-back have nothing to run against." Everything in Phase 3 is blocked on
  2b (CLAUDE.md table Phase 2 note).
- The CI-gate wiring is **backend-repo CI configuration** — backend-team-
  coordinated, not landable unilaterally (design/03 §4).

## Parallel work breakdown

Several tracks are independent once 1a and 2b are in place; the harness's
audio-send step is the one piece gated on a spike.

- **Track P — scenario packs + assertion shims.** Author scenario YAMLs
  (`scenarios/eval-iscream-gelato/*.yaml`) and the thin `assertions/*.py` shims
  that load a scenario and call `voiceeval` (design/03 §1). Independent of the
  harness internals — the shims just need `voiceeval` (Phase 1a). Can start as
  soon as 1a lands.
- **Track H1 — harness call placement + capture (non-audio-send parts).** Call
  `POST /calls/web` for the `eval-` tenant, receive `joinUrl`, connect as
  WebSocket client, capture agent audio + transcript deltas + `state` messages
  (design/03 §2 steps 1, 2, 4). Independent of Track P. **Note:** the harness
  *is* the call creator, so it has no join-permission problem (design/03 §2 key
  insight) — distinct from Phase 1b's listener.
- **Track H2 — harness audio-send (BLOCKED on the audio-frame spike).** Step 3:
  synthesize each scripted line via TTS and send audio frames into the session
  (design/03 §2 step 3). **Do not build until the audio-frame-format spike
  closes** (design/03 §2 Open Spike).
- **Track S — Square Sandbox read-back.** GET-order from Sandbox after a test
  call, shape it into an `Order` for `score_order_accuracy()` (design/03 §3).
  Depends on 2b's sandbox credentials; independent of the harness's audio path.
- **Track C — CI gate.** The `just eval-gate` recipe + `deploy-reusable.yml`
  insertion (design/03 §4). Backend-coordinated; can be specified in parallel
  but lands via a backend ticket.

Convergence (serial): one full scenario run = harness places call (H1+H2) →
captures → read-back (S) → assertions (P) score via `voiceeval` → pass/fail.

## TDD task list

- **Track P — assertion shims (unit-testable):**
  - RED: failing test that a shim loads a scenario YAML and returns Promptfoo's
    expected pass/fail/score shape, with `voiceeval` injected/stubbed.
  - GREEN: minimal shim (load YAML → call `voiceeval` → return shape). No
    scoring logic in the shim (design/03 §1).
  - REFACTOR: keep each shim tiny.
  - VERIFY: shim covered; `voiceeval` itself is already covered in 1a.

- **Track P — scenario YAML validation (unit-testable):**
  - RED: failing test that a malformed scenario YAML (missing `expected_order`,
    or a non-`eval-` `tenant_id`) is rejected at load (input validation at the
    boundary, coding-style.md).
  - GREEN: schema validation against design.md's scenario shape.
  - VERIFY: valid + each invalid case covered.

- **Track S — Square Sandbox read-back parse (unit-testable):**
  - RED: failing test feeding a recorded Sandbox GET-order JSON and asserting it
    maps to the correct `Order`.
  - GREEN: parse/shape function (the network GET is a thin wrapper, injected).
  - VERIFY: parse covered; the live GET is integration (recorded-response test).

- **Track H1 — call placement + capture (mostly NOT unit-testable):**
  - "Place a real Ultravox call and capture turns" is **not unit-testable** by
    design (Finding 9: the real agent must be exercised). Alternative
    verification: integration test replaying a recorded session into the
    capture buffer, plus a manual e2e smoke run placing one real `eval-`-tenant
    call and confirming captured transcript/state events. The pure
    turn-parsing logic (shared with Phase 1b Track S) is unit-tested there.

- **Track H2 — audio-send (BLOCKED; NOT buildable until spike closes):**
  - design/03 §2 step 3: "This step is not yet buildable as described." Do not
    write RED/GREEN for the send path until the audio-frame spike resolves the
    codec/sample-rate/framing. Once resolved: integration-test the frame
    encoding against the confirmed format, then an e2e run confirming the agent
    responds to the synthesized line.

- **Track C — CI gate (verification, backend-coordinated):**
  - The `just eval-gate` recipe is verified by: `promptfoo eval` returning a
    threshold-based exit code (0 pass / non-zero fail) in CI; and the backend
    team confirming `deploy-reusable.yml` runs it before `sam deploy`
    (design/03 §4). Note `deploy.yml`'s direct path bypasses it unless also
    routed through the reusable workflow (design/03 §4) — flag this to backend.

## Demo criteria

- `promptfoo eval --config scenarios/eval-iscream-gelato/promptfooconfig.yaml`
  runs end-to-end: the harness places a **real call** against the `eval-`
  tenant, drives a scripted persona, the order lands in **Square Sandbox**, the
  read-back order is scored by `voiceeval`, and the scenario passes/fails with a
  threshold-based exit code (capture the console/CI output).
- A deliberately-broken scenario (wrong expected order) **fails** the gate —
  proving the gate actually gates, not just passes everything.
- The CI gate runs in `deploy-reusable.yml` before `sam deploy` (backend
  confirmation / workflow-run link).

## North-star validation

Phase 3 delivers the **automated pre-release regression check** Finding 6 names
directly, gating on **order accuracy** (diff + Sandbox read-back), **escalation
correctness** (rubric), and **conversation completion** per scenario. Before
done: a known-good scenario must pass and a known-bad scenario must fail, on
**real** calls against the **real** Ultravox agent (Finding 9 rejected mocking
the agent precisely because a simulated agent doesn't catch real regressions).
**"Not aligned, needs rework"** looks like: the harness tests a mock instead of
the real agent; the gate passes a scenario with a genuinely wrong order
(order-accuracy not actually wired through); or test orders land in production
instead of Sandbox (Finding 9 violation — pollutes the headline metric this
project produces).

## Open spikes that must resolve first

- **Blocking (design/03 §2 Open Spike), pulled verbatim:** *the exact audio
  frame format/codec Ultravox expects when a server-side WebSocket client sends
  synthesized speech into a session has not been confirmed.* Resolve by reading
  `docs.ultravox.ai/apps/websockets` directly before writing Track H2's
  audio-sending code (design/03 §2). This blocks **only Track H2**, not the rest
  of Phase 3 — Tracks P, H1 (capture), and S can proceed.
- **Hard precondition (not a spike):** Phase 2b ticket confirmed filed AND
  complete — verify status first (design/03 §"Hard dependency").
