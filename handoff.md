# Handoff

**Status:** Planning phase complete *and* architecture-reviewed.
`idea.md` (decisions/why), `design.md` + `design/01-05` (component
design/how), and `CLAUDE.md` (project guide + implementation order +
standing 95%-confidence rule) are written, committed, and pushed to
`main` (GitHub: `avishekdas/voice-chat-eval`). A critical architecture
review pass (commit `3d38850`) found and fixed 5 consistency issues
across the design docs — see "Already resolved by the review" below so
a new session doesn't need to re-derive them. **No code exists yet.**

## Start here: Phase 0 — the thin end-to-end slice

This is the next action for a new Claude Code session picking this up —
not a re-read of the planning docs. Read `CLAUDE.md` first for project
orientation, then begin the checklist below.

Per `CLAUDE.md`'s Implementation Order table and `design.md`'s build
sequencing recap, this is the only thing to build right now — nothing else
starts until this loop works end-to-end on one real call:

- [ ] Confirm/create the Langfuse Cloud account (Hobby tier — `idea.md`
      Finding 3). No infra to stand up.
- [ ] Get **one** real call into **one** Langfuse trace, via whichever path
      is fastest to stand up first — explicitly **not** the Data Connection
      listener (`design/01-observability.md`), which is gated on its own
      unresolved Open Spike. A manual/simplest-possible trace write is
      enough to prove the integration.
- [ ] Compute **one** order-accuracy score for that call directly off the
      existing `transaction_details` data — a throwaway script is fine
      here, not the `voiceeval` library yet (`design/02`, §1).
- [ ] Get **one** Promptfoo scenario passing against the live agent using
      Promptfoo's **native** assertions only (no custom library, no
      call-driver harness yet).
- [ ] Only once all four are working together on real data: move to Phase
      1a/1b per `CLAUDE.md`'s table.

## Do this in parallel, starting now (long lead time, not on our team)

These block later phases and are owned by the backend team — file/track
them immediately rather than waiting for Phase 0 to finish:

- [ ] **Webhook dedup fix** — `idea.md` Finding 4. Pre-existing production
      data-integrity bug, independent of this project's timeline.
- [ ] **`eval-` test tenant + Square Sandbox credentials** — `idea.md`
      Finding 9. **Phase 3 (`design/03-preprod-regression.md`) cannot
      start without this** — confirm ticket status before assuming it's in
      progress.

## Already resolved by the review (don't re-litigate these)

The 2026-06-21 architecture review pass found and fixed these — a new
session can treat them as settled, not open questions:

- `is_eval_tenant` now has one canonical derivation rule
  (`tenant_id.startswith("eval-")`) stated once in `design.md`'s
  cross-cutting conventions; `design/01` and `design/04` reference it
  rather than re-deriving it.
- `design/05`'s "Interruption recovery %" KPI is explicitly flagged as
  having no data until `design/01`'s listener spike (below) is resolved.
- `design.md`'s dependency table now lists the scorecard's Grafana
  export-Lambda dependency.
- `design/03`'s call-driver flow no longer reads as if the Ultravox
  audio-send format were confirmed — it's explicitly tied to the spike
  below.
- The "WER / speech understanding score" KPI was **dropped** from
  `design/05`'s scorecard (was previously listed with no designed
  producer — Finding 7 explicitly rejected building WER). No replacement
  metric was substituted.

## Open spikes still blocking deeper phases (not Phase 0)

None of these block Phase 0 itself, but don't start the phase that
depends on them until resolved:

- **`design/01`'s join-permission spike** — how a third-party service
  joins another session's Ultravox Data Connection mid-call to observe.
  Blocks the real listener build (Phase 1b), not the thin slice's manual
  trace write.
- **`design/03`'s audio-frame-format spike** — exact codec/framing
  Ultravox expects when the call-driver harness sends synthesized TTS
  audio into a session. Blocks the harness build (Phase 3), not Promptfoo's
  native-assertion-only scenario in Phase 0.
- **`design/05`'s Langfuse Hobby-tier dashboard spike** — whether Hobby
  tier supports custom score-aggregation views (Approach A) or whether the
  Grafana export-Lambda fallback (Approach B) is required. Blocks the
  management scorecard (Phase 6), far downstream of Phase 0.

## Standing rule

Don't skip ahead on confidence — if anything below is ambiguous (which
trace-write path counts as "simplest," how to interpret an Open Spike,
whether a Finding still holds), ask rather than guess. Full rule in
`CLAUDE.md` → "Standing rule: the 95% confidence rule."
