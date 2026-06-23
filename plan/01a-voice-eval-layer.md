# Plan: Phase 1a — `voiceeval` Library Core (Order Diff + LLM Judge)

## Scope

Build the core of the shared `voiceeval` scoring library — the **one place
scoring logic lives** (Finding 10; design/02). Phase 1a covers the two
mechanisms that depend on nothing external:

- **Order-accuracy diffing** — `score_order_accuracy()` (design/02 §1).
- **LLM-judge rubric scoring** — `judge_rubric()` for `menu_price_accuracy`,
  `bill_clarification`, `escalation_handling` (design/02 §2).

Plus the shared `EvalScore` envelope and the `Order`/`OrderItem`/`Turn`
dataclasses (design.md interface contract; design/02 package layout).

**Explicitly out of Phase 1a:** `score_audio_quality()` (Phase 5) and
`score_interruptions()` (needs Phase 1b's listener) — those are separate
milestones per CLAUDE.md's table.

## Dependencies

- **Phase 0 proven** (CLAUDE.md table: "Depends on Phase 0").
- **Does NOT depend on Phase 1b** (CLAUDE.md table: "Does **not** depend on
  1b"; Finding 2's loosened graph).
- Read access to the tenant catalog API (`/catalogs?tenant_id=...`) for the
  LLM judge's grounding (design/02 §2) — used by a thin wrapper *outside* the
  pure package.

## Parallel work breakdown

Two largely independent tracks plus a shared-foundation task that both need
first.

- **Shared foundation (serial, do first, small):** the `EvalScore` envelope
  (design.md), `Order`/`OrderItem`, and `Turn` dataclasses (design/02 package
  layout). Frozen dataclasses, immutable (coding-style.md). Both tracks below
  import these, so land them first — but it's a small, fast task.

Then, in parallel:

- **Track A — order diff (`voiceeval/order/`).** `diff.py`
  (`score_order_accuracy()`) + `models.py`. Pure, deterministic, no model.
  Independent of Track B.
- **Track B — LLM judge (`voiceeval/judge/`).** `rubric.py`
  (`judge_rubric()`), the three rubric YAMLs, and `grounding.py` (fetches/
  shapes tenant menu data). The pure rubric-application logic is independent;
  the live catalog fetch is a thin wrapper outside the package. Independent of
  Track A.

Both tracks converge only on the shared `EvalScore` return type.

## TDD task list

- **Shared dataclasses (unit-testable):**
  - RED: test that `EvalScore`, `Order`, `OrderItem`, `Turn` are frozen and
    reject mutation (immutability per coding-style.md).
  - GREEN: define the frozen dataclasses per design.md/design/02.
  - REFACTOR: one small file per concern (design/02 package layout).
  - VERIFY: 100% trivial coverage on dataclasses.

- **`score_order_accuracy()` — exact match (unit-testable):**
  - RED: failing test — hand-built `actual`/`expected` `Order` with identical
    items asserts `item_precision == 1.0`, `item_recall == 1.0`,
    `total_matches == True`, empty `missing_items`/`extra_items`.
  - GREEN: minimal exact-name, case-insensitive match + total compare
    (design/02 §1).
  - REFACTOR: extract item-matching into a small helper.
  - VERIFY: coverage on the match/diff paths.

- **`score_order_accuracy()` — missing / extra / quantity / total mismatch
  (unit-testable):**
  - RED: separate failing tests for each: a missing item, an extra item, a
    quantity mismatch, a total mismatch — asserting the correct
    `missing_items`/`extra_items`/precision/recall/`total_matches`.
  - GREEN: extend the diff.
  - VERIFY: each branch covered.

- **`score_order_accuracy()` — fuzzy name match (unit-testable, with a flagged
  open question):**
  - RED: failing test where `actual` item name differs from `expected` only by
    casing/phrasing and should still match.
  - GREEN: token-overlap / edit-distance fallback used only when no exact match
    exists (design/02 §1).
  - **Note:** design/02 §1 flags the exact fuzzy *threshold* as an unresolved
    tuning question needing real mismatched-name examples — do NOT guess the
    final threshold here; implement the mechanism with a clearly-marked
    provisional threshold and a TODO to calibrate once real Promptfoo runs
    exist. (Per the 95% rule, this is a flagged follow-up, not a silent
    default.)

- **`judge_rubric()` — pure rubric application (unit-testable with an injected
  judge):**
  - RED: failing test that injects a stub judge LLM (deterministic canned
    response) and asserts `RubricScore.score` and `rationale` are parsed
    correctly for a binary rubric.
  - GREEN: minimal rubric loader + judge-call + response parse, with the LLM
    client **injected as a parameter** (keeps the package pure per design/02
    §5).
  - REFACTOR: one small loader, separate from the call/parse.
  - VERIFY: rubric load + parse covered without any real LLM call.

- **`judge_rubric()` — graded vs. binary scoring (unit-testable):**
  - RED: failing tests for a `graded` rubric returning a 0.0-1.0 with a
    documented threshold, and a `binary` rubric returning pass/fail.
  - GREEN: implement both `scoring` modes (design/02 §2 rubric format).
  - VERIFY: both modes covered.

- **`grounding.py` — catalog fetch + shape (integration, NOT pure-unit):**
  - The live `/catalogs?tenant_id=...` fetch is **not unit-testable in
    isolation** — it's a network call. Alternative verification: an integration
    test against a recorded/sample catalog JSON response, asserting item names/
    prices map into the `TenantMenu` grounding shape.
  - **Flagged unknown (design/02 §2):** the exact field name Square's
    `price_money` surfaces under is "not yet confirmed" — nail it down against
    a real catalog response during implementation; do not hardcode a guessed
    field name.

- **Real judge accuracy check (NOT unit-testable):**
  - "Does the chosen judge model actually grade transcripts correctly" is not
    unit-testable. Alternative verification (design/02 §2): run the judge over a
    handful of hand-checked real transcripts and confirm agreement; pick the
    cheapest model that passes. Capture the hand-check results as an artifact.

## Demo criteria

- `pytest` green on the `voiceeval/order/` and `voiceeval/judge/` test suites
  with **80%+ coverage** reported on those packages.
- A runnable snippet: import `voiceeval`, call `score_order_accuracy()` on a
  real (or realistic) order pair and print the `OrderAccuracyScore`; call
  `judge_rubric()` with `escalation_handling` on a real transcript and print
  the `RubricScore` (score + rationale).
- Confirm **no `import promptfoo` and no `import langfuse`** anywhere inside the
  `voiceeval` package (a grep/test enforcing design/02 §5 / Finding 10's purity
  rule).

## North-star validation

Order accuracy and escalation correctness are two of the three headline metrics
(Finding 6). Before done: `score_order_accuracy()` must produce a correct,
explainable diff on a **real** order pair (matching what a human would
conclude), and `judge_rubric("escalation_handling")` must agree with human
judgment on the hand-checked transcript sample. **"Not aligned, needs rework"**
looks like: the diff disagrees with obvious human reading of the order (e.g.
calls a correct order wrong because of a too-strict name match), or the
escalation rubric's rationale doesn't actually reflect the rubric's
`pass_criteria` — meaning the headline metric it produces can't be trusted.

## Open spikes that must resolve first

**None block Phase 1a.** Design/02 contains flagged *open questions* (the fuzzy
match threshold; the `price_money` field name; the judge model choice) but
design/02 explicitly classes these as implementation-tuning details to resolve
during build, not structural Open Spikes that gate the milestone. Implement the
mechanisms with provisional values and clearly-marked TODOs; do not silently
finalize the fuzzy threshold without real examples (95% rule).
