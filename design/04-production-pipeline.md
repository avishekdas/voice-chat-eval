# Design: Production Scoring Pipeline

Implements [[Finding 6]] and [[Finding 10]]'s "small production pipeline"
half of the shared-library decision. Owns: triggering continuous scoring of
real production calls and writing results into Langfuse. Calls into
[02](02-voice-evaluation-layer.md) for all scoring; relies on
[01](01-observability.md)'s trace-id convention for idempotency.

## 1. Trigger

**Decision: a new, separate Lambda — not an edit to the existing webhook
handler.** `src/webhooks/ultravox_webhooks.py` already carries real
business risk ([[Finding 4]]'s disabled-dedup finding lives in this exact
file) — adding scoring logic into it increases the blast radius of touching
an already-fragile, business-critical path. Instead:

- Subscribe a new Lambda to the same `call.ended` signal via an
  EventBridge rule (or, if the webhook handler already publishes an
  internal event/queue message on `call.ended` for other consumers, hook
  into that instead of the raw webhook) — decoupled from, and unable to
  regress, the existing handler's order/transaction logic.
- This Lambda is purely additive: read-only against `call-messages`,
  `call_order_facts`, `transaction_details`; its only write is to Langfuse
  (scores) — it has no path to mutate production order data.

## 2. Pipeline steps

1. **Idempotency check.** Per `design.md`'s convention, the
   trace write itself (`trace_id = call_id`) is now safely idempotent by
   construction — a duplicate `call.ended` delivery re-running this whole
   pipeline produces an identical upsert, not a duplicate trace. Still keep
   the lightweight `call_id#event_type` marker (same mechanism as
   [01](01-observability.md)'s compute-avoidance check) purely to skip
   redundant LLM-judge calls and audio-model inference, which cost real
   money/time to redo — not because correctness depends on it anymore.
2. **Fetch inputs:** transcript (`call-messages`), tool-call payloads
   (`call_order_facts`), order record (`transaction_details`), recording
   URL (on-demand from Ultravox, per [01](01-observability.md)).
3. **Score:** call `voiceeval.score_order_accuracy()`,
   `judge_rubric()` for each configured rubric, and (sampled — see below)
   `score_audio_quality()`. `score_interruptions()` runs against
   [01](01-observability.md)'s buffered state events if the listener is
   live for this call; skip gracefully if not (e.g. listener spike from
   [01](01-observability.md) hasn't landed yet — partial coverage is fine
   at this stage, per the thin-slice principle).
4. **Write:** attach each `EvalScore` to the `call_id`-keyed Langfuse
   trace as a score. Tag every score with `is_eval_tenant`, computed via
   `design.md`'s single derivation rule (`tenant_id.startswith("eval-")`,
   [[Finding 9]]) rather than re-implemented here — this is the only thing
   that makes the [05](05-dashboards.md) exclusion filter possible
   downstream.

## 3. Audio-scoring sampling

Running NISQA/DNSMOS/UTMOS inference on every single production call is a
cost/latency decision, not an architectural one — **proposed default: 10-
20% sample rate**, configurable, with **100% sampling during the
[[Finding 7]] calibration window** (the 20-50 call human-rated collection
phase) so the calibration sample is drawn from the same pipeline that will
run in steady state, not a separate one-off script. This number is a
starting guess, not a researched figure — revisit once real inference cost/
latency is measured.

## 4. `eval-` tenant exclusion

Per `design.md`'s convention: **individual** eval-tenant calls still get
full scoring and a visible Langfuse trace (useful for validating the
[03](03-preprod-regression.md) harness itself, and for engineering
debugging) — the exclusion only applies when **aggregating** for the
management scorecard ([05](05-dashboards.md)). This pipeline's
responsibility is just to make sure the `is_eval_tenant` tag is always
correctly set on every score it writes; the filtering itself happens at
the dashboard/aggregation layer, not here.

## Out of scope for this doc

- The scoring algorithms → [02](02-voice-evaluation-layer.md).
- Promptfoo's separate, CI-triggered use of the same library →
  [03](03-preprod-regression.md).
- How scores get displayed → [05](05-dashboards.md).
