# CLAUDE.md

Guidance for any Claude Code session working in this repository.

## What this project is

An **evaluation framework** for an already-live voice ordering chatbot
(Ultravox runtime + AWS Lambda/API Gateway, in the separate
`echobot-studio-backend` repo). This repo does not contain the voice agent
itself — it contains the planning/design for, and eventually the code for,
the system that observes, scores, and regression-tests that agent.

**Current repo state (as of this writing): planning is complete, no code
exists yet.** This is not a git repository. The two source-of-truth
documents below are the entire deliverable so far.

## The business problem (don't re-derive this — it's already answered)

The voice agent is live and gets patched reactively, with **no way to tell
whether fixes actually improve it**, no reviewable quality dashboard for
non-engineers, and no automated pre-release regression check. **Order
accuracy** is the headline metric, alongside escalation correctness and
conversation completion. Full reasoning: `idea.md` → Finding 6.

## Document map — read in this order

1. **`idea.md`** — the *why*. Confirmed build scope, the existing
   frontend/backend architecture as actually investigated (not assumed),
   a 4-agent critical review (§0.3), and 11 fully-resolved findings (§0.4)
   covering every major architectural decision, **including rejected
   alternatives and the reasoning behind each choice**. Findings 1-11 are
   binding — treat them as already-decided, not open questions, unless new
   evidence contradicts them.
2. **`design.md`** — the *how*, index + cross-cutting conventions
   (tenant/call ID rules, the scenario schema, the shared scoring library's
   interface contract, the build-sequencing recap). Read this before any
   of the component docs below — they all depend on these conventions.
3. **`design/01-observability.md`** — Langfuse + the Data Connection
   listener.
4. **`design/02-voice-evaluation-layer.md`** — the shared `voiceeval`
   scoring library (order accuracy, LLM-judge rubrics, interruption
   detection, audio quality).
5. **`design/03-preprod-regression.md`** — Promptfoo scenario packs, the
   call-driver harness, CI release gating.
6. **`design/04-production-pipeline.md`** — continuous production scoring.
7. **`design/05-dashboards.md`** — engineering + management views.

Each design doc explicitly marks **Open Spike** sections for anything not
yet verified — those are not settled design, they're the list of things
that must be proven before building that part.

## Architecture at a glance

Three layers, built and deepened on different schedules (not "all in
parallel" — see Implementation Order below):

- **Observability** — Langfuse traces every call; a custom listener
  recovers real-time turn-taking/latency signal that Ultravox's webhooks
  alone don't provide.
- **Voice Evaluation Layer** — one shared scoring library, computing order
  accuracy, conversational-rubric judgments, interruption metrics, and
  audio quality. Called by both the pre-prod and production paths below —
  never duplicated between them.
- **Pre-Prod Regression** — Promptfoo scenario packs run against a
  dedicated test tenant (real Ultravox calls, Square *Sandbox* orders),
  gating releases in CI.

A fourth and fifth piece consume the above: a **production pipeline** that
scores real calls continuously, and a **dashboard layer** (engineering
trace view + a management scorecard) that displays the results.

## Implementation order

This is the recommended build sequence — phases are ordered by actual
dependency, not by document number. Don't start a phase whose prerequisite
column isn't satisfied.

| Phase | Build | Depends on |
|---|---|---|
| **0. Thin slice** | One real call → one Langfuse trace (simplest available path) → one order-accuracy score off existing `transaction_details` data → one Promptfoo scenario passing with Promptfoo's *native* assertions (no custom library yet). Nothing else starts until this loop works end-to-end on real data. | Langfuse Cloud signup only — no new infra. |
| **1a. Voice Evaluation Layer core** | `voiceeval` library: order-accuracy diff + LLM-judge rubric (`design/02`). | Phase 0. Does **not** depend on 1b. |
| **1b. Observability deepening** | Data Connection listener + deeper Langfuse tracing (`design/01`). | Phase 0, **and** resolving `design/01`'s join-permission Open Spike first — do not write listener code before that's settled. |
| **2. Backend-team tickets** | File/track, in parallel with 1a/1b, not after: (a) the webhook-dedup fix (`idea.md` Finding 4 — a pre-existing production bug, independent of this project's timeline), (b) the `eval-` test tenant + Square Sandbox credentials (`idea.md` Finding 9), (c) push the `joinUrl` to the listener at call creation + confirm/add `call_id` to each tool Lambda's invocation context — the backend half of resolving `design/01`'s join-permission Open Spike (Approach A). | None — these can and should start immediately; everything in Phase 3 is blocked waiting on (b); Phase 1b's listener is blocked waiting on (c). |
| **3. Pre-Prod Regression, full build** | Full scenario packs, the call-driver harness, Square Sandbox read-back, CI release-gate wiring (`design/03`). | Phase 1a (needs the library) **and** Phase 2b (needs the test tenant). |
| **4. Production pipeline** | New, decoupled Lambda scoring real calls continuously (`design/04`). | Phase 1a (the library) and Phase 1b (the trace-id convention it writes against). |
| **5. Audio/voice quality** | NISQA/DNSMOS/UTMOS scoring, gated on collecting and calibrating against a 20-50 call human-rated sample (`idea.md` Finding 7). | Can run in parallel with Phases 2-4 once the calibration sample is collected — not gated on a calendar date, gated on that sample existing. |
| **6. Dashboards** | Engineering view is nearly free once Phase 1b exists (native Langfuse UI). Management scorecard needs Phase 4 producing scores at real volume before there's anything meaningful to show (`design/05`). | Phase 1b (engineering view); Phase 4 (scorecard). |

## Standing rule: the 95% confidence rule

**Never proceed with an architectural, scope, or implementation-approach
decision without at least 95% confidence that it is correct.** If
confidence is below that bar, **ask the user a clarifying question instead
of guessing, assuming a default, or silently picking an option** — this
applies to picking between researched alternatives, interpreting an
ambiguous requirement, or deciding how to resolve one of the Open Spikes
in the design docs.

This is not a one-off instruction from a single session — it has governed
every decision recorded in `idea.md` (each Finding either reached 95%
confidence on its own from verified facts, or was put directly to the user
as a question) and must continue to govern this project going forward.
Concretely:

- Don't convert an **Open Spike** (in any `design/*.md`) into shipped code
  by guessing the unresolved part — resolve the spike first (by research,
  by a small experiment, or by asking the user), then build.
- If new information contradicts a locked Finding in `idea.md`, don't
  silently override it — surface the conflict and confirm before changing
  course.
- When a genuinely new decision gets made (not just an implementation
  detail), record it back into `idea.md` (if it's a "why" decision, in the
  same Findings format) or the relevant `design/*.md` (if it's a "how"
  decision) — don't let decisions live only in chat history.
