# Voice Agent Evaluation Framework — Design (Index)

This is the **how** layer, built on top of `idea.md`'s **why** (the locked
business problem, scope, and Findings 1-11). `idea.md` remains the source of
truth for every decision and its rejected alternatives — this document and
its children do not re-argue those decisions, they implement them.

Each component below got its own file instead of one large document because
the components have different owners, different build sequencing (per
[[Finding 2]]/[[Finding 11]]'s roadmap), and different lifecycles — the
production pipeline and the Promptfoo harness, for example, are deepened on
separate schedules. A single file would mean unrelated sections being edited
in the same diff every time one component changes.

**Confidence discipline carried over from `idea.md`:** where this document
states something as decided, it is backed by a verified fact (cited) or a
direct decision already made in `idea.md`. Where a mechanism is genuinely
unverified, it is called out explicitly as an **Open Spike** rather than
written as if it were settled — do not treat spike sections as design,
they are the list of things that must be proven before the surrounding
design can be trusted.

## Document map

| Doc | Component | Primary owner | Hard dependency |
|---|---|---|---|
| [design/01-observability.md](design/01-observability.md) | Langfuse + Data Connection listener | Eval framework team | Backend change to surface `joinUrl` (spike) |
| [design/02-voice-evaluation-layer.md](design/02-voice-evaluation-layer.md) | Shared `voiceeval` scoring library | Eval framework team | None — can start immediately on existing data |
| [design/03-preprod-regression.md](design/03-preprod-regression.md) | Promptfoo + call-driver harness | Eval framework team | Backend ticket: eval tenant + Square Sandbox creds ([[Finding 9]]) — **not yet confirmed filed** |
| [design/04-production-pipeline.md](design/04-production-pipeline.md) | `call.ended` scoring job | Eval framework team | design/02's library; design/01's trace id convention |
| [design/05-dashboards.md](design/05-dashboards.md) | Engineering + management views | Eval framework team | design/01 (Langfuse) and design/04 (scores existing to display) |

## Cross-cutting conventions

These apply to every component doc below. Defining them once here is the
entire point of splitting the documents — none of the five docs should
redefine `tenant_id` handling, the scenario schema, or the scoring library's
interface independently.

### Tenant & call identity

- `tenant_id`, `location_id`, `call_id` are taken **verbatim** from
  `echobot-studio-backend`'s existing DynamoDB partition-key conventions
  (`idea.md` §0.2) — never re-derived, renamed, or re-encoded by any eval
  component.
- Test/eval tenants are distinguished by an **`eval-` tenant_id prefix**
  ([[Finding 9]]). Every component that computes a production-facing
  aggregate (management scorecard, dashboards) **must** filter this prefix
  out before aggregating. Components that compute per-call scores for
  engineering/debugging purposes (Langfuse traces, Promptfoo run results)
  do **not** filter it — eval-tenant calls still need visible scores, just
  not blended into production KPIs.
- **Deterministic Langfuse trace id: `trace_id = call_id`.** Verified this
  session: Langfuse upserts a trace idempotently when the same custom
  trace id is sent again within its retention window — and Langfuse Cloud
  Hobby tier's window is 30 days ([[Finding 3]]), so this lines up exactly
  with no extra expiry logic needed.
  ([How to update traces, observations, and scores?](https://langfuse.com/faq/all/tracing-data-updates),
  [Trace IDs & Distributed Tracing](https://langfuse.com/docs/observability/features/trace-ids-and-distributed-tracing))
  This means **trace-level correctness no longer depends on the [[Finding 4]]
  dedup fix** — reprocessing the same `call.ended` event twice now safely
  overwrites the same trace instead of creating a duplicate. Span-level
  idempotency still needs deterministic span ids; defined in
  [design/01](design/01-observability.md).

### Scenario / golden-call schema

No formal schema existed before this build (`idea.md` §0). Proposed shape,
stored at `scenarios/{tenant_id}/{scenario_id}.yaml` (the forward-compatible
per-tenant layout already decided in [[Finding 5]]):

```yaml
scenario_id: iscream-gelato_add_two_scoops_001
tenant_id: eval-iscream-gelato        # always the eval- prefixed test tenant
persona:
  mode: scripted                      # scripted | llm_simulated
  # llm_simulated is a stretch goal past the thin slice — see design/03
  script:
    - "Hi, can I get two scoops of pistachio in a cup?"
    - "Actually make that a waffle cone."
expected_order:
  items:
    - name: "Pistachio"
      quantity: 2
      modifiers: ["Waffle Cone"]
      unit_price: 4.50
  currency: USD
  total: 9.00
expected_tool_calls:
  - tool: AddOrderItem
    payload_contains: { item: "Pistachio", quantity: 2 }
rubric_refs: ["menu_price_accuracy", "bill_clarification"]
tags: ["regression", "happy-path"]
```

This is the ground truth for the **Promptfoo synthetic scenario** row of
[[Finding 7]]'s revised ground-truth table — the scenario author already
knows the expected order when writing it, so there is no extra labeling
cost.

### Shared scoring library interface ([[Finding 10]])

Package name: `voiceeval` — pure Python, **no Promptfoo or Langfuse imports
inside the package itself** (per [[Finding 10]]'s "one library, two
callers" decision). Full design in
[design/02](design/02-voice-evaluation-layer.md); the interface contract
that the other docs depend on:

```python
def score_order_accuracy(actual_order: Order, expected_order: Order) -> OrderAccuracyScore: ...
def judge_rubric(transcript: list[Turn], rubric_id: str, grounding: TenantMenu) -> RubricScore: ...
def score_audio_quality(audio_path: str) -> AudioQualityScore: ...
def score_interruptions(state_events: list[StateEvent]) -> InterruptionScore: ...
```

All four return a common envelope:

```python
@dataclass(frozen=True)
class EvalScore:
    call_id: str
    tenant_id: str
    scenario_id: str | None       # None for real production calls
    metric_name: str              # "order_accuracy" | "menu_price_accuracy" | ...
    value: float                  # normalized 0.0-1.0 where possible
    breakdown: dict[str, float]   # per-dimension detail, e.g. NISQA's 4 sub-scores
    rationale: str | None         # LLM-judge free-text explanation, for human review
    evaluator_version: str        # bump on any rubric/model/threshold change
    computed_at: datetime
```

`evaluator_version` exists so that a later change to the rubric or the
order-diff logic ([[Finding 7]]) doesn't silently re-interpret historical
scores — old scores keep the version they were computed under.

### Build sequencing recap

Pulled forward from [[Finding 2]] and [[Finding 11]] so each component doc
doesn't need to re-derive it:

1. **Thin slice** (no design doc gates this — build directly): one real
   call → one Langfuse trace (simplest path, not necessarily the final
   listener) → one order-accuracy score off existing data → one Promptfoo
   scenario passing with native assertions.
2. **In parallel, once the slice is proven:**
   - [design/01](design/01-observability.md)'s listener + deeper Langfuse
     tracing.
   - [design/02](design/02-voice-evaluation-layer.md)'s order-accuracy and
     LLM-judge mechanisms, off existing DynamoDB data — does not wait on
     (a).
3. [design/03](design/03-preprod-regression.md)'s full scenario pack +
   call-driver harness + release gate, once (2) has something real to grade
   against.
4. [design/04](design/04-production-pipeline.md), once (2) and (3) both
   exist — it's the orchestration glue that calls the same library
   continuously in production.
5. [design/05](design/05-dashboards.md) last — it only has something to
   show once (4) is producing scores.
6. Audio/voice-quality scoring inside design/02 stays gated on its own
   20-50 call human-rated calibration sample ([[Finding 7]]), independent
   of the numbered sequence above.

## Open cross-cutting risks carried into every component doc

- **PII ([[Finding 8]], deferred).** Every schema above — scenario YAML,
  `EvalScore.rationale`, transcripts passed into `voiceeval` — carries raw
  customer text with no redaction today. This is an accepted, tracked risk,
  not an oversight; see [[Finding 8]] for the revisit triggers.
- **The eval tenant + Square Sandbox credentials ([[Finding 9]]) are a
  backend-team ticket that this design assumes exists, but its filing/
  completion has not been independently confirmed in this session.**
  [design/03](design/03-preprod-regression.md) is blocked on it.
