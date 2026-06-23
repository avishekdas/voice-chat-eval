# Plan: Phase 5 — Audio / Voice Quality Scoring

## Scope

The `voiceeval.score_audio_quality()` mechanism — pretrained, open-source,
no-reference perceptual-quality models, no custom training (design/02 §4;
Findings 7, 11):

- **NISQA** (overall + noisiness/coloration/discontinuity/loudness), **DNSMOS**
  (sig/bak/ovr), and **UTMOS/UTMOSv2** (TTS-naturalness) combined in
  `voiceeval/audio/score.py` (design/02 §4 package layout).
- The **one-time calibration procedure**: collect the 20-50 human-rated call
  sample, run all three models, and document a threshold mapping or simple
  linear correction (design/02 §4 calibration; Finding 7).

## Dependencies

- **The 20-50 call human-rated calibration sample must EXIST** — this is the
  hard gate, not a calendar date (CLAUDE.md table Phase 5; design/02 §4: "do not
  treat raw NISQA/DNSMOS/UTMOS output as a trustworthy score in any dashboard
  until this sample has been collected and the mapping documented").
- Otherwise **independent of Phases 2-4** — can run in parallel with them once
  the sample exists (CLAUDE.md table Phase 5).
- Recordings fetched on-demand from Ultravox per `call_id` (design/01 §audio
  handling); the calibration sample is the one intentionally-persisted exception
  (design/01 §audio handling), with its storage/access gated on Finding 8's
  revisit triggers — **not decided here**.

## Parallel work breakdown

- **Track M — model integration.** Wire NISQA, DNSMOS, UTMOS as three separate
  scorer modules (`nisqa.py`, `dnsmos.py`, `utmos.py`) plus `score.py` combining
  them into `AudioQualityScore` (design/02 §4). The model-integration code is
  independent of the calibration data — build and golden-fixture-test it while
  the sample is being collected.
- **Track S — sample collection + human rating.** Collect the 20-50 calls
  (drawn from Phase 4's pipeline at 100% sampling during the calibration window,
  design/04 §3) and obtain human ratings. This is data-gathering, not code —
  runs in parallel with Track M. **Gated on Finding 8 PII revisit triggers**
  before the sample is handed to human raters (design.md cross-cutting risks;
  Finding 8 revisit trigger 2).
- **Track C — calibration analysis.** Compare the three models' output against
  the human ratings; produce the documented threshold mapping / linear
  correction (design/02 §4). **Serial after both M and S** — needs both the
  model outputs and the human labels.

## TDD task list

- **Track M — `score_audio_quality()` combiner (unit-testable):**
  - RED: failing test that injects stub per-model scorers and asserts the
    combined `AudioQualityScore` shape (nisqa dict, dnsmos dict, utmos float)
    per design/02 §4.
  - GREEN: minimal combiner.
  - REFACTOR: one small module per model + a thin combiner (design/02 package
    layout).
  - VERIFY: combiner covered with models injected.

- **Track M — per-model wrappers (NOT pure-unit — model output IS the thing
  under test):**
  - design/02 §5: audio functions are "the one exception that need real model
    checkpoints to test meaningfully — use a handful of fixed sample WAVs as
    golden fixtures rather than mocking the models themselves." Alternative
    verification: run each model on a small set of fixed golden-fixture WAVs and
    assert outputs fall within expected ranges / match recorded golden values.
    Do **not** mock the models (mocking would defeat the validation).

- **Track C — calibration mapping (verification, not RED/GREEN):**
  - The calibration is a one-time analysis producing a documented mapping, not
    shippable scoring logic with conventional tests. Verification: the
    documented threshold mapping is checked against held-out human-rated calls
    (e.g. "NISQA overall < X correlates with human-rated 'poor'" holds on the
    held-out subset). Record the mapping + `evaluator_version` (design.md) so
    historical audio scores aren't silently re-interpreted when the mapping
    changes.

- **e2e (manual, this milestone's required e2e test per plan.md's
  cross-cutting conventions):** run the full `score_audio_quality()` pipeline
  against one real recording fetched on-demand from Ultravox for a real
  `call_id` (not just a golden-fixture WAV), and confirm the result falls
  within the calibrated mapping's expected band before treating Phase 5 as
  done — golden fixtures alone don't prove the on-demand fetch path works.

## Demo criteria

- `score_audio_quality(<golden_fixture.wav>)` returns a populated
  `AudioQualityScore` (NISQA 4 sub-scores, DNSMOS 3 sub-scores, UTMOS float)
  with `pytest` green and 80%+ coverage on the combiner.
- A documented calibration mapping artifact exists (e.g. a short doc/table
  mapping model output to human-rated quality bands), validated on held-out
  calls.
- Only **after** the mapping exists: audio scores appear in the Phase 6
  scorecard's "Voice quality / naturalness" KPI (design/05 KPI 7,
  "post-calibration per Finding 7").

## North-star validation

**This milestone does NOT move a headline metric** (order accuracy, escalation,
completion) — it produces a secondary customer-experience metric. Its
justification for staying in scope (per plan.md's table): Finding 7 already
scoped it as adopt-not-build (low cost), it rounds out the management
scorecard's customer-experience picture, and it is correctly sequenced **last**
and gated on its own calibration sample so it never delays a headline-metric
milestone. Before done: the calibration mapping must be validated against real
human ratings (Finding 7's hard gate). **"Not aligned, needs rework"** looks
like: raw NISQA/DNSMOS/UTMOS output is surfaced in the dashboard *without* the
calibration step (design/02 §4 explicitly forbids this — these models were
validated on general speech, not phone/drive-through audio), or building this
ahead of any headline-metric milestone (it must stay last).

## Open spikes that must resolve first

- **Not a design Open Spike, but a hard gate (design/02 §4):** the 20-50 call
  human-rated calibration sample must be collected and the mapping documented
  before audio scores are trusted anywhere.
- **Finding 8 PII revisit trigger** must be addressed before the sample is
  handed to human raters (design.md cross-cutting risks; Finding 8 revisit
  trigger 2) — do not hand raw customer audio/transcripts to raters without
  resolving the deferred PII decision first (95% rule: this is a question to
  surface, not to guess past).
- The calibration sample's **storage location and access control** are gated on
  Finding 8's revisit triggers and are **not** decided in design/01 — resolve
  before persisting the sample (design/01 §audio handling).
