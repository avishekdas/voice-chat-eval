# Handoff

**Status:** Planning phase complete. `idea.md` (decisions/why),
`design.md` + `design/01-05` (component design/how), and `CLAUDE.md`
(project guide + implementation order + standing 95%-confidence rule) are
written, committed, and pushed to `main`. **No code exists yet.**

## Next step: Phase 0 — the thin end-to-end slice

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

## Standing rule

Don't skip ahead on confidence — if anything below is ambiguous (which
trace-write path counts as "simplest," how to interpret an Open Spike,
whether a Finding still holds), ask rather than guess. Full rule in
`CLAUDE.md` → "Standing rule: the 95% confidence rule."
