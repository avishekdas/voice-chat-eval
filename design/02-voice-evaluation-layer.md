# Design: Voice Evaluation Layer (`voiceeval` library)

Implements [[Finding 7]] and [[Finding 10]]. This is the one place actual
scoring logic lives. Per [[Finding 10]]'s decision, it is a **plain
importable Python library with no Promptfoo- or Langfuse-specific code
inside it** — [03](03-preprod-regression.md) and
[04](04-production-pipeline.md) are both thin callers into this package,
never the other way around.

## Package layout

Per the file-organization preference (many small, cohesive files over one
large module):

```
voiceeval/
  __init__.py
  order/
    diff.py            # score_order_accuracy()
    models.py           # Order, OrderItem dataclasses
  judge/
    rubric.py           # judge_rubric()
    rubrics/             # one YAML per rubric_id, see below
      menu_price_accuracy.yaml
      bill_clarification.yaml
      escalation_handling.yaml
    grounding.py         # fetches/shapes tenant menu data for the judge
  audio/
    nisqa.py
    dnsmos.py
    utmos.py
    score.py             # score_audio_quality(), combines the three
  timing/
    interruptions.py     # score_interruptions()
  score.py               # EvalScore dataclass (shared envelope, see design.md)
```

## 1. Order-accuracy diffing

**Mechanism:** deterministic structured-data diff, no model involved
(`idea.md` §0.4 Finding 7 table). Canonical `Order` representation:

```python
@dataclass(frozen=True)
class OrderItem:
    name: str
    quantity: int
    modifiers: tuple[str, ...]
    unit_price: float

@dataclass(frozen=True)
class Order:
    items: tuple[OrderItem, ...]
    currency: str
    total: float
```

**Diff algorithm:** match items by name first (exact, case-insensitive),
falling back to a fuzzy match (e.g. token-overlap or edit-distance
threshold) only when no exact match exists — menu item names spoken by a
customer and parsed by the agent will not always match the scenario
author's casing/phrasing exactly. **Open question, not yet resolved:** how
aggressive the fuzzy threshold should be is a tuning decision that needs a
handful of real mismatched-name examples to calibrate against, not
something to guess at design time — flag as a small follow-up once a few
real Promptfoo runs exist to look at.

Output: per-item precision/recall plus an overall boolean/score:

```python
@dataclass(frozen=True)
class OrderAccuracyScore:
    item_precision: float
    item_recall: float
    total_matches: bool      # exact total match, currency-aware
    missing_items: tuple[str, ...]
    extra_items: tuple[str, ...]
```

**Ground truth source varies by call type** — this is already locked in
`idea.md`'s Finding 7 correction and restated here only because it's the
input contract `score_order_accuracy()` must support, not a new decision:

| Call type | `expected_order` comes from | `actual_order` comes from |
|---|---|---|
| Promptfoo synthetic scenario | The scenario YAML's `expected_order` (design.md) | Agent's tool-call payload, captured during the call |
| Promptfoo test-tenant call | Same scenario YAML | **Square Sandbox GET-order read-back** (see [03](03-preprod-regression.md)) |
| Real production call | `transaction_details` table — accepted as an imperfect proxy ([[Finding 7]]'s correction: it reflects the agent's own self-reported order, not independently-verified customer intent) | Same `transaction_details` record |

## 2. LLM-judge rubric scoring

**Mechanism:** off-the-shelf judge LLM (no training), scored against a
rubric authored once per dimension, grounded against the tenant's actual
menu/pricing data.

```python
def judge_rubric(transcript: list[Turn], rubric_id: str, grounding: TenantMenu) -> RubricScore: ...

@dataclass(frozen=True)
class RubricScore:
    score: float            # 0.0-1.0
    rationale: str          # judge's free-text explanation — feeds EvalScore.rationale, for human spot-checking
```

**Rubric format** — one YAML per dimension, authored by the team (no tool
can define product-specific correctness, per [[Finding 7]]):

```yaml
rubric_id: menu_price_accuracy
description: >
  Did the agent state menu item names and prices that match the tenant's
  actual catalog, and did it correctly total the order?
pass_criteria:
  - "Every menu item name mentioned matches a real catalog item (fuzzy match acceptable for minor phrasing)."
  - "Every price mentioned matches the catalog's price_money for that item."
  - "The stated total matches sum(item prices) within $0.01."
fail_examples:
  - "Agent quotes a price that doesn't exist in the catalog."
scoring: binary   # binary | graded — graded rubrics return a 0.0-1.0 score with a documented threshold for pass/fail
```

**Grounding context — verified this session:** the tenant catalog API
(`/catalogs?tenant_id=...`, `catalog_routes.py:125-185`, built by
`squareup_service.format_items_by_category()`) returns:

```json
{
  "catalogs": [
    {
      "category_name": "...",
      "category_description": "...",
      "items": [
        { "id": "...", "name": "...", "description": "...", "...modifiers/pricing fields..." }
      ]
    }
  ],
  "count": 12
}
```

Item names, descriptions, categories, and modifiers are confirmed present.
**Not yet confirmed:** the exact field name Square's `price_money` surfaces
under in this response — a minor data-mapping detail to nail down during
implementation, not an architectural unknown; `grounding.py` should fetch
this endpoint live (or from a short-lived cache) rather than hardcoding
catalog data anywhere in the rubric YAML.

**Judge model choice is deliberately left open here** — this is an
implementation parameter (which LLM, what temperature, how grounding is
formatted into the prompt), not a design decision with the same weight as
the mechanism choice itself. Pick the cheapest model that passes a handful
of hand-checked transcripts during the thin slice, revisit if judge
accuracy proves to be the bottleneck.

## 3. Interruption / talk-over detection

**Mechanism:** turn-taking timestamps from
[01](01-observability.md)'s Data Connection listener — a timing fact, not
something detectable from transcript text ([[Finding 7]]).

```python
def score_interruptions(state_events: list[StateEvent]) -> InterruptionScore: ...

@dataclass(frozen=True)
class StateEvent:
    state: Literal["idle", "listening", "thinking", "speaking"]
    timestamp: float

@dataclass(frozen=True)
class InterruptionScore:
    talk_over_count: int      # customer speech detected while agent state == speaking
    dead_air_seconds: float   # cumulative gaps between turns exceeding a threshold
    interruption_recovery: float  # did the agent yield/resume sensibly after an overlap — heuristic, see below
```

**Design decision:** `talk_over` is defined as a `listening` state
transition occurring before the prior `speaking` state has ended, sourced
directly from the listener's buffered event stream
([01](01-observability.md)) rather than re-deriving it from raw audio —
the Data Connection already gives state transitions as the ground-truth
signal, so no separate VAD/diarization step is needed for this metric.
`interruption_recovery` is the one sub-metric here that's a heuristic
rather than a hard fact (did the agent's next turn meaningfully address
what was said during the overlap) — flagged as the weakest part of this
mechanism and a candidate for LLM-judge augmentation later if the heuristic
proves unreliable in practice.

## 4. Audio/voice quality scoring

**Mechanism:** pretrained, open-source, no-reference models — NISQA,
DNSMOS, and UTMOS/UTMOSv2 as a complementary third signal
([[Finding 7]], [[Finding 11]]). No custom training.

```python
def score_audio_quality(audio_path: str) -> AudioQualityScore: ...

@dataclass(frozen=True)
class AudioQualityScore:
    nisqa: dict[str, float]   # overall, noisiness, coloration, discontinuity, loudness
    dnsmos: dict[str, float]  # sig, bak, ovr
    utmos: float              # TTS-naturalness signal
```

**Calibration procedure (one-time, per [[Finding 7]]):** collect the 20-50
human-rated call sample, run all three models against the same audio, and
compare. This produces either a simple linear correction or, more likely,
just a documented threshold mapping ("NISQA overall < X correlates with
human-rated 'poor'") — not a retraining exercise. This calibration is a
**hard gate**: do not treat raw NISQA/DNSMOS/UTMOS output as a trustworthy
score in any dashboard until this sample has been collected and the mapping
documented, since these models were validated on general speech/noise-
suppression domains, not specifically on phone/drive-through audio
([[Finding 7]]).

## 5. Testing approach

All four mechanisms above are pure functions over plain data structures —
no Promptfoo, Langfuse, or network calls inside `voiceeval` itself (network
calls for grounding/audio-fetch happen in thin wrapper code outside the
package, or are injected as parameters). This makes them directly
unit-testable per the project's TDD standard: write the diff/rubric/timing
test cases first, against hand-constructed `Order`/`StateEvent` fixtures,
before wiring in real LLM-judge or model-inference calls. The audio-scoring
functions are the one exception that need real model checkpoints to test
meaningfully — use a handful of fixed sample WAVs as golden fixtures rather
than mocking the models themselves, since the model output *is* the thing
being validated.

## Out of scope for this doc

- What calls these functions and when (→ [03](03-preprod-regression.md),
  [04](04-production-pipeline.md)).
- Where `EvalScore` results get displayed (→ [05](05-dashboards.md)).
- PII redaction of transcripts passed into `judge_rubric()` — deferred per
  [[Finding 8]], tracked in `design.md`'s cross-cutting risks.
