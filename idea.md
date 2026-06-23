# Voice Agent Evaluation Architecture (Ultravox + Langfuse + Promptfoo)

## 0. Confirmed Build Scope (North Star)

- **Current state:** A voice ordering chatbot is already live — Ultravox runtime +
  AWS (API Gateway/Lambda tools) are deployed and working. Nothing else
  (observability, evaluation, regression gating) exists yet.
- **Goal:** Build an **eval framework** around the existing live bot — not a
  POC/demo and not a rebuild of the bot itself.
- **Scope of this build:** All three layers in parallel, not phased:
  1. **Observability** — stand up self-hosted Langfuse and a trace adapter so
     every conversation produces a trace with turn/STT/LLM/TTS/tool spans.
  2. **Voice Evaluation Layer** — transcript/outcome evaluator + audio/voice
     evaluator, writing scores back into Langfuse.
  3. **Pre-Prod Regression** — Promptfoo scenario packs + release gate.
- **Langfuse hosting:** Self-hosted, to be stood up from scratch (likely on AWS
  alongside the existing Lambda/API Gateway stack) — not Langfuse Cloud.
- **Implementation language:** Python for the eval framework (trace adapter,
  evaluators, WER/audio scoring). Promptfoo is orchestrated via its own
  CLI/config regardless of host language.
- **Scenario / golden-call data:** No formal scenario dataset exists yet — only
  real production call transcripts. Part of this build is defining the
  scenario/expected-outcome schema and deriving an initial golden-call set
  from those real transcripts (rather than importing a pre-built dataset).

## 0.1 Existing Frontend Landscape (echobot-studio-ui)

Investigated the live hosted UI (`hostui-dev.dev.pannalabs.ai/?tenant_id=iscream-gelato`)
by reading its source at `/Users/abhishridas/workspace/Pannalabs/echobot-studio-ui`
(the page itself is a JS-rendered Next.js SPA, not analyzable by fetching raw HTML).

- **What it is:** A multi-tenant admin/staff portal (Next.js + React) for
  managing voice-ordering chatbots — separate from any eval tooling. Three
  tenant categories: Restaurant (orders/menu), Assistant (appointments),
  Telecaller (outbound sales). `iscream-gelato` is a Restaurant-category tenant.
- **Tenant model:** True multi-tenant via AWS Cognito auth + tenant routing —
  not just a query-param flag. Tenant selection happens through
  `/dashboard/tenants` or `/dashboard/staff-app/[tenant_id]`.
- **Backend it talks to:** `https://voicebotapi-dev.pannalabs.ai/v2`, JWT
  bearer auth from Cognito. Endpoints cover tenant CRUD, prompt
  listing/switching (`/tenants/{id}/prompts`, `.../switch-default`), and
  restaurant catalog/menu (`/catalogs?tenant_id=...`).
- **Critical finding — no call/transcript/order history UI exists today.**
  The "Recent Activity" analytics section
  (`src/components/analytics-section.tsx`) is hardcoded mock data. Voice
  agent transcripts are presumably stored backend-side (DynamoDB via
  Lambda, per the architecture doc above) but are **not surfaced anywhere**
  in this frontend.
- **Implication for the eval framework:** this is a blank slate, not a
  conflict to design around. Real production transcripts will need to be
  pulled from the backend store directly (not from this UI). If a UI is
  ever wanted for the eval framework, this app's `DashboardLayout`, auth/
  tenant context, and `@tanstack/react-query` data-fetching pattern could be
  reused — but the eval framework itself does not need to live inside or
  modify this repo.
- **Relevant file paths (for future reference, not modified):**
  - API client: `src/lib/api.ts`
  - Auth context: `src/lib/auth-context.tsx`
  - Tenant service: `src/lib/tenant-service.ts`
  - Staff app router: `src/app/(dashboard)/dashboard/staff-app/[id]/page.tsx`
  - Shared layout: `src/components/staff-ui/shared/DashboardLayout.tsx`

## 0.2 Existing Backend Architecture (echobot-studio-backend)

Scanned the AWS SAM (Python) backend at
`/Users/abhishridas/workspace/Pannalabs/echobot-studio-backend` — this is the
real source of truth for how calls/transcripts/audio/tools actually work,
and the system the eval framework will read from / instrument.

**Ultravox integration**

- Call creation: `POST /calls/web` → `create_web_call()` in
  `src/routes/ultravox_routes.py:477`. Pulls per-tenant config (Secrets
  Manager, cached in Valkey), builds a `CallConfig` with system prompt +
  menu + tools, and posts to `https://api.ultravox.ai` via
  `src/services/ultravox_service.py:create_call()`. Auth is a per-tenant
  `X-API-Key` from `tenant.bot_config.ultravox_key`.
- Webhooks (events flow *back* from Ultravox): `POST /webhook/ultravox` and
  `/v2/webhook/ultravox`, handled in
  `src/webhooks/ultravox_webhooks.py:UltravoxWebhookHandler.handle()`.
  HMAC-signed. Handles `call.started` / `call.updated` / `call.ended`.
  Tenant is resolved from `call.metadata.tenant_id` in the webhook body.
- **Tools are not backend Lambdas in this stack.** They're HTTP endpoints
  registered as `temporaryTool` objects inside the tenant config; Ultravox
  itself calls them directly over HTTP during the conversation (the
  backend injects `x-api-key`/`x-user-id` auth headers into each tool
  definition before sending it to Ultravox). The actual order-tool handler
  code lives outside this repo's SAM templates (likely `customer-template.yaml`
  or an external service — not yet confirmed).

**Data model (DynamoDB + S3)**

| Table | Key | Stores |
|---|---|---|
| `calls-{branch}-{env}` | PK `TENANT#{tenant_id}#LOC#{location_id}`, SK `CALL#{created_at}#{call_id}` | call metadata, status, duration, recording URL |
| `call-messages-{branch}-{env}` | PK `CALL#{call_id}`, SK `MSG#{timestamp}` | transcript turns (speaker, text) |
| `call_recordings-{branch}-{env}` | — | recording metadata (s3_url, duration, format) |
| `call_events-{branch}-{env}` | — | stage transitions, tool-call events |
| `call_order_facts-{branch}-{env}` | — | order state extracted from the call |
| `transaction_details-{branch}-{env}` | — | payment/transaction records |
| `tenant_prompts-{branch}-{env}` | PK `TENANT#{tenant_id}`, SK `PROMPT#{prompt_name}` | versioned prompts per tenant |

S3 (`{RECORDINGS_BUCKET_NAME}`) holds per-call recording/metadata/events/messages
JSON under `calls/{call_id}/...`, written by the Ultravox API side, not proactively
mirrored by this backend.

**Critical gaps this confirms for the eval framework's scope**

- **Transcripts are not real-time.** `call-messages` is only populated after
  the `call.ended` webhook fires — there is no live transcript stream to hook
  into; the eval framework will ingest post-call.
- **Audio isn't persisted locally.** It lives in Ultravox's own storage; the
  backend calls `get_call_recording()` on demand rather than archiving it.
  The eval framework will need to fetch audio from Ultravox (or have the
  backend start archiving to S3) rather than assuming it's already in our S3.
- **No tracing today.** Logging is structured JSON (`src/common/logger.py`)
  plus Lambda Powertools metrics (namespace `EchobotStudio`) feeding a
  standalone Prometheus/Grafana stack (`monitoring/`). There is no X-Ray, no
  Langfuse, no per-call/per-turn trace today — confirms the Observability
  Plane (§2.B) is genuinely greenfield, not a wrapper around something that
  exists.
  Other relevant gap: webhook event deduplication logic exists but is
  currently **disabled** (commented out) — duplicate `call.ended` events are
  possible and the eval ingestion needs to be idempotent against that.
- **Tool calls happen outside this backend's visibility.** Since Ultravox
  calls tools directly over HTTP, this repo doesn't see tool invocations as
  they happen — `call_order_facts` and `call_events` are reconstructed
  after the fact from the webhook payload, not captured live. Tool-call
  correctness scoring (§3.B) will depend on what's actually present in
  those post-hoc payloads, not live interception.
- **Multi-tenancy is logical, not infrastructure-isolated** — same Lambdas
  serve every tenant; isolation is via tenant_id partition keys and
  per-tenant Secrets Manager config/API keys. The eval framework should
  follow the same `tenant_id` (+ `location_id`) partitioning when storing
  scenario/eval data, for consistency.

**Prompt management** — `tenant_prompts` table, switched via
`POST /tenants/{tenant_id}/switch-prompt` (`src/routes/tenant_routes.py:355`),
which updates the tenant's active prompt and busts the Valkey cache. This is
the hook the Pre-Prod Regression Plane (§2.D) should read "current prompt
version" from when tagging a Promptfoo run.

**File paths for integration**

- Ultravox client: `src/services/ultravox_service.py`
- Webhook handler: `src/webhooks/ultravox_webhooks.py`
- Call creation: `src/routes/ultravox_routes.py:477`
- Transcript/messages: `src/services/call_messages_dynamodb_service.py`
- Calls table service: `src/services/calls_dynamodb_service.py`
- Tenant/prompt service: `src/services/tenant_service.py`, `src/services/tenant_prompt_service.py`
- Logging/metrics: `src/common/logger.py`, `src/common/metrics_publisher.py`

## 0.3 Critical Analysis (Council Review)

Ran a 4-agent critical review panel against this document (architecture
feasibility, security/privacy, problem-statement devil's advocate,
build-vs-buy/cost) — each agent read the doc independently, with no
visibility into the others' findings. Listed below in order of how many
reviewers independently converged on the same point, which is the
strongest signal of what's actually load-bearing.

1. **The tracing model in §1–§7 doesn't match reality.** (architect +
   devil's advocate) The reference architecture assumes live per-turn
   STT/LLM/TTS spans with TTFB. §0.2 already established transcripts only
   land after `call.ended` fires, audio isn't persisted locally, and tool
   calls are invisible to this backend. What's actually buildable is a
   **post-call reconstructor**, not a runtime tracer — latency/TTFB columns
   in the scorecard will be empty or fabricated unless this is corrected
   up front. Sell/scope this as "observability of outcomes," not "runtime
   tracing."
2. **"Build all three layers in parallel" is the wrong call, not just
   aggressive.** (architect + devil's advocate + build-vs-buy ranking) The
   dependency chain is real: scores need traces to attach to; Promptfoo's
   regression gate is meaningless without an evaluator to grade scenario
   runs against. Forced order: fix capture/idempotency → outcome evaluator
   on existing data → thin Promptfoo gate → Langfuse/voice-quality
   investment once value is proven.
3. **Self-hosting Langfuse from scratch is disproportionate infra sprawl.**
   (architect + security + build-vs-buy, three independent reviewers
   unprompted) It's a new Postgres+ClickHouse+Redis stack on top of an
   already-separate Prometheus/Grafana setup, with no stated on-call,
   backup, or storage-growth plan, and no auth/RBAC model for who can see
   which tenant's transcripts once they're in it.
4. **Disabled webhook dedup (§0.2) breaks score idempotency if untouched.**
   (architect + devil's advocate) Duplicate `call.ended` events become
   duplicate traces and duplicate scores, silently double-counting every
   aggregate in the management scorecard. Needs a deterministic upsert key
   derived from `call_id` before any scoring layer goes live.
5. **New cross-tenant exposure risk.** (architect + security) Today a
   partition-key bug is contained inside one Lambda's DynamoDB query. Once
   Langfuse/Promptfoo sit on top with shared dashboards, a missing tenant
   filter (or no per-tenant Langfuse project) means one engineer's login
   could browse every tenant's customer calls — strictly worse than today's
   per-tenant Secrets Manager isolation. Neither the original plan nor §1–§7
   addresses Langfuse RBAC or per-tenant project scoping.
6. **No stated business problem driving this.** (devil's advocate) Nothing
   in §0–§0.2 names a complaint, a bad-order rate, or an incident. "Build an
   eval framework" is currently a description of the artifact, not a sized
   justification, which makes it hard to judge whether the self-hosted-
   Langfuse-plus-custom-audio-evaluator level of investment is proportionate.
7. **The Voice Evaluation Layer is the long pole and the riskiest custom
   build.** (build-vs-buy + devil's advocate) WER needs ground-truth
   transcripts that don't exist yet (§0 already admits no scenario dataset).
   Naturalness/prosody are flagged in the doc *itself* as needing human
   calibration — building automated scorers before that rubric exists is
   backwards. Recommendation: ship objective metrics first (task success,
   tool correctness via Promptfoo assertions, latency from spans); defer
   WER/naturalness until a real golden-call set and rating rubric exist.
8. **PII handling gap.** (security) Pulling real production transcripts/
   audio into a "golden call" S3 dataset creates a second, less-controlled
   copy of customer data (names, phone numbers, payment-adjacent context)
   with no stated redaction, retention, or access-control policy.
9. **Promptfoo could pollute production.** (security) Running scenario
   packs against the live Ultravox-enabled agent implies real API keys and
   real Lambda tool calls unless a separate pre-prod tenant/key namespace is
   explicitly stood up — otherwise test orders land in real
   `transaction_details`/`calls` tables.
10. **Double-built risk with Promptfoo.** (build-vs-buy) Promptfoo's native
    assertion engine already covers pass/fail-by-scenario and tool/payload
    correctness — exactly what the custom "Transcript/Outcome Evaluator"
    (§3.B) is being built to do. Worth checking whether that evaluator can
    just be Promptfoo assertions instead of a second bespoke scoring system.

**Net effect on §0's confirmed scope:** the "build all three layers in
parallel, self-hosted Langfuse, fully custom Voice Evaluation Layer"
decisions in §0 are contested by this review. Resolutions are being worked
through one finding at a time in §0.4 below, rather than revising §0 in
one pass — §0.4 is the live source of truth where it overrides §0/§1-§7.

## 0.4 Resolutions

Working through §0.3's findings one at a time. Each entry below is a locked
decision, not a suggestion — it supersedes whatever §1–§7 originally assumed
for that point.

### Resolved — Finding 1: tracing model mismatch

Before picking an option, verified directly against Ultravox's own docs
(not just this repo's code) what's actually possible, since the original
§2.B design borrowed Langfuse's Pipecat voice-trace pattern (conversation →
turns → STT/LLM/TTS spans), and that pattern assumes a component pipeline.

**Verified facts:**

- **Ultravox has no discrete STT/LLM/TTS pipeline to trace.** It's a
  multimodal model that processes audio directly into the LLM — there is
  no separate ASR stage to instrument, even in principle. The Pipecat
  pattern Langfuse's docs showcase doesn't apply to Ultravox's architecture.
  ([Understanding Latency in Voice AI Systems](https://www.ultravox.ai/voice-ai/understanding-latency-in-voice-ai-systems))
- **Webhooks are coarse**: only `call.started`, `call.connected`,
  `call.ended`, `call.billed` exist — no per-turn webhook events.
  ([Available Webhooks](https://docs.ultravox.ai/webhooks/available-webhooks))
- **The REST Get Call endpoint has no latency/TTFB/transcript data** — only
  metadata, billing, and summaries.
  ([Get Call](https://docs.ultravox.ai/api-reference/calls/calls-get))
- **An unused live channel exists**: Ultravox's "Data Connection"
  (WebSocket/WebRTC) streams real-time `state` messages
  (`idle`/`listening`/`thinking`/`speaking`, timestamped — the actual
  TTFB-equivalent signal), incremental transcript deltas, and ping/pong
  latency, and is explicitly documented for "monitoring call events or
  routing data to external systems." The current backend never joins this
  — it only uses webhooks + post-call REST.
  ([Protocol & Data Messages](https://docs.ultravox.ai/apps/datamessages))
- **Tool-call messages on that channel (`ClientToolInvocation`/
  `ClientToolResult`) only cover client-executed tools.** This backend
  registers tools as server-side HTTP endpoints (`temporaryTool` +
  `baseUrl`), so joining the Data Connection would *not* surface tool-call
  events — that gap needs a separate fix.
- **Audio never appears in this protocol** — recording stays on the
  separate WebRTC/telephony channel and is fetched on-demand after the
  call regardless of which option below is chosen.

**Decision: Option C + D (hybrid).**

1. Build a lightweight listener service that joins each call's Data
   Connection and captures `state` transitions + transcript deltas +
   ping/pong in real time, emitting Langfuse spans as the call happens.
   This gets genuine TTFB-equivalent latency (listening→thinking→speaking
   timestamps) and live transcript ingestion — the actual capability the
   original design wanted, via the correct integration point.
2. Audio-quality metrics (dead-air, talk-over, interruption recovery,
   naturalness) are explicitly scoped as **offline/batch analysis on the
   fetched recording**, not a tracing problem — they were always going to
   be post-hoc regardless of (1), since audio never flows through the Data
   Connection.
3. Tool-call visibility is fixed independently of Ultravox's protocol: each
   tool Lambda (`AddOrderItem` etc.) emits its own event the moment it's
   invoked, using the `interaction_id`/`x-user-id` context it already
   receives. Fully within our control; closes the gap regardless of (1)/(2).

**Rejected:** Option A (post-call-only reconstruction) — viable as a
fallback if the Data Connection listener proves infeasible to build, but
rejected as the primary plan since it would mean permanently relabeling
"TTFB"/"latency" scorecard items to "turn-to-turn delta," a materially
weaker metric than what's actually available via (1).

**Still open / not yet de-risked:** exactly how a third-party, non-client
service authenticates and joins another session's Data Connection
mid-call is not fully spelled out in the docs reviewed so far — this needs
a spike/proof-of-concept before committing engineering time to building
the listener service.

### Resolved — Finding 2: "build all three layers in parallel" is the wrong call

**Problem statement (re-examined):** the original §0.3 critique framed this
as a strict dependency chain — "scores need traces to attach to; Promptfoo's
gate is meaningless without an evaluator" — implying a forced waterfall.
That framing turned out to be only partially true. Re-checking what each
layer actually requires:

- **Promptfoo** can run scenario packs against the live agent today using
  its own native assertions (expected tool call, expected payload, expected
  text) — no dependency on Langfuse or a custom evaluator. This is also the
  cheapest layer per the build-vs-buy review (Finding 7), precisely because
  it needs nothing else built first.
- **Outcome Evaluator** (task success %, tool correctness) can compute
  scores directly from existing DynamoDB data (`call-messages`,
  `call_order_facts`) without Langfuse existing — Langfuse is only where
  scores get displayed/trended, not a prerequisite for computing them.
- **Observability** (Langfuse + the Data Connection listener from Finding 1)
  is independent infrastructure work — it doesn't block, or get blocked by,
  the evaluator.
- **Audio/voice-quality evaluator** is the one piece with a real hard
  blocker: no golden-call dataset or rating rubric exists yet (Finding 7),
  so it can't meaningfully start regardless of how the other layers are
  sequenced.

So the real dependency graph is looser than "one strict waterfall" — two of
the three layers can move independently; the third is blocked by missing
data, not by the other layers.

**Options considered:**

| Option | Approach | Pros | Cons |
|---|---|---|---|
| A. Strict waterfall | Capture/idempotency fixes → Outcome Evaluator (off existing data) → Promptfoo (native assertions) → Langfuse/listener → audio eval last | Simplest to reason about, no coordination overhead, matches effort-ranking from the build-vs-buy review | Delays observability value (trace-level debugging) even though it's independently buildable sooner |
| B. Two parallel tracks | Track 1: Observability (Langfuse + listener). Track 2: Outcome Evaluator + thin Promptfoo gate (native assertions) — wired up immediately since neither depends on Track 1. Audio eval deferred regardless. | Matches the real dependency graph; gets a working release gate *and* trace visibility sooner than A | Needs two efforts in flight (or context-switching) and an upfront shared schema agreement (tenant_id/call_id conventions) so tracks don't diverge |
| C. Thin end-to-end slice first, then deepen | Prove the whole loop on one real call before investing depth anywhere: one Langfuse trace (simplest available path), one outcome score, one passing Promptfoo scenario — then deepen each layer (in A or B order) | De-risks integration assumptions (Langfuse write API, tenant partitioning, Promptfoo's actual fit) for a few hours of work, before deep investment in any one layer; produces a concrete artifact to show, which also helps surface the still-missing "why are we doing this" answer from Finding 6 | Easy to let "just one more thing" creep back into full-depth parallel building if the slice isn't kept genuinely thin |

**Decision: Option C.**

Build the thin end-to-end slice first: one real call → one trace (via
whichever path is fastest to stand up first, not necessarily the final
Data-Connection listener from Finding 1) → one outcome score (e.g. task
success %, computed from existing `call_order_facts`) → one Promptfoo
scenario passing against the live agent using native assertions. Nothing
else gets built until this loop is proven working end-to-end on real data.

**Future enhancement path (after the thin slice proves out):** deepen in
roughly this order, leaning toward Option B's parallelization once the
slice has de-risked the integration points:

1. Stand up the Data Connection listener (Finding 1) and deepen Langfuse
   tracing — independent track, can start as soon as the slice's "one
   trace" path is validated.
2. In parallel, deepen the Outcome Evaluator (add/edit/delete correctness,
   escalation correctness, conversation completion %) directly off existing
   DynamoDB data — does not wait on (1).
3. Deepen Promptfoo into a real scenario pack + release gate, re-evaluating
   Finding 10 (whether the Outcome Evaluator should be replaced by, or fold
   into, Promptfoo's native assertions rather than staying a separate
   bespoke system) once there's a real scenario pack to test that against.
4. Audio/voice-quality evaluator stays last, gated on Finding 7's
   prerequisite (a real golden-call dataset and a human-rating rubric for
   the subjective metrics) — not on a calendar date.

### Resolved — Finding 3: self-hosted Langfuse is disproportionate infra

**Problem statement (re-examined):** §0 originally locked in "self-hosted,
stood up from scratch" specifically rejecting Langfuse Cloud. The council
review (architect + security + build-vs-buy, three independent reviewers)
flagged this as infra sprawl — a new Postgres+ClickHouse+Redis stack on top
of an already-separate Prometheus/Grafana setup, with no stated ops/backup
plan and no auth/RBAC model.

**Verified facts that changed the picture:**

- The "self-host" critique assumed the heavy production deployment
  (Kubernetes/Helm, HA, multi-instance). Langfuse also offers an explicitly
  lightweight **Docker Compose** deployment — officially "recommended for
  testing and low-scale deployments," no HA/backup/horizontal scaling, but
  also no Kubernetes-scale ops burden.
  ([Docker Compose Deployment](https://langfuse.com/self-hosting/deployment/docker-compose))
- **Langfuse Cloud's compliance posture is stronger than assumed**: SOC 2
  Type II + ISO 27001 audited, with separated US/EU/HIPAA/Japan data
  regions. ([Security & Compliance](https://langfuse.com/security),
  [Data Regions](https://langfuse.com/security/data-regions)) This removes
  "data residency" as an automatic reason to self-host.
- **Langfuse Cloud has a free Hobby tier**: 50,000 units/month, 30-day
  retention, 2 users, no credit card required — comfortably covers a
  thin-slice POC (per Finding 2) with zero infrastructure to stand up at
  all. ([Pricing](https://langfuse.com/pricing))
- Asked directly: there is no hard data-residency/compliance reason behind
  the original "self-host" choice — it was the default assumption, not a
  constraint. The actual preference is "whichever option is free/cheap and
  not complex for the POC."

**Decision: use Langfuse Cloud's free Hobby tier for the POC/MVP phase.**
Zero infra to stand up — lighter than even the Docker Compose self-host
path — and the 50k-units/30-day-retention cap is well above what a
thin-slice POC and early MVP usage will need.

**Documented fallback (not yet needed):** if the Hobby tier's cap is ever
hit, or a genuine compliance/data-residency requirement emerges later,
migrate to self-hosted Langfuse via the **Docker Compose** deployment
(matched to actual scale at that time) rather than jumping straight to the
full Kubernetes/HA production deployment — re-evaluate the heavier
self-host path only if real traffic actually demands HA.

### Resolved — Finding 4: disabled webhook dedup breaks idempotency

**Problem statement (for the backend developer — copy/paste this section
verbatim when filing the ticket):**

In `echobot-studio-backend`, file
`src/webhooks/ultravox_webhooks.py`, the `UltravoxWebhookHandler.handle()`
method has its deduplication check **fully implemented but commented out**,
at lines 95-104:

```python
# call_data = body.get("call", {})
# call_id = call_data.get("callId")

# Deduplication check
# if call_id:
#     if self._is_duplicate_event(call_id, event):
#         logger.info(f"⏭️ Duplicate event ignored: {event} for call {call_id}")
#         return {"statusCode": 200, "message": "Duplicate event ignored"}
# else:
#     logger.warning("⚠️ Ultravox event without callId (cannot dedupe)")
```

The supporting methods `_is_duplicate_event()` (lines 1064-1076) and
`_mark_event_processed()` (lines 1078-1094) still exist and are still
*called* after processing (lines 236, 320, 391) — i.e. the system still
marks events as processed in Valkey/Redis, it just never checks that mark
before reprocessing. **No comment or TODO in the code explains why the
check was disabled.** It is unknown whether this was an intentional
rollback (e.g. the check was causing false-positive drops) or an
accidental leftover from debugging.

**Why this matters (impact, not just style):** webhook providers — Ultravox
included — use at-least-once delivery, meaning duplicate deliveries of the
same event are an expected, normal occurrence, not an edge case. Today,
every write triggered by a webhook is **unconditional** (`put_item()` with
no `ConditionExpression`) across:
- `CALLS` table (`pk=TENANT#{tenant_id}#LOC#{location_id}`,
  `sk=CALL#{call_created_at}#{call_id}`) — duplicate `call.ended` →
  duplicate overwrite of status/duration/performance fields.
- `CALL_EVENTS` table (`pk=CALL#{call_id}`, `sk=EVENT#{timestamp}`) —
  duplicate or near-duplicate event rows.
- `CALL_MESSAGES` table — duplicate transcript writes.
- `CALL_RECORDINGS` table — duplicate recording metadata entries.
- `TRANSACTION_DETAILS` table — **duplicate order/transaction records**,
  which is a real business-data correctness bug, not just an eval concern
  (e.g. could double-count an order in reporting).

For this project specifically: any Outcome Evaluator score (order
correctness, escalation correctness) or Langfuse trace built from this data
would inherit the duplication, producing false "regressions" or false
confidence that have nothing to do with the agent's actual behavior.

**No unique webhook event ID exists in the payload.** The best available
idempotency key is `call_id + event_type` (optionally `+ timestamp` if an
event can legitimately recur, e.g. `call.updated`).

**Decision: fix the backend root cause + add a defensive check in the eval
ingestion layer (defense-in-depth).**

1. **Backend fix (echobot-studio-backend, owned by the backend team):**
   replace the Valkey-based check with a **durable DynamoDB conditional
   write** instead of just re-enabling the old Valkey check as-is. Write a
   dedup marker item (e.g. `pk=DEDUP#{call_id}#{event_type}`, with a TTL
   attribute for auto-expiry) using
   `ConditionExpression="attribute_not_exists(pk)"` *before* processing the
   event; if the condition fails, treat it as a duplicate and return `200`
   without reprocessing. This is more durable than the cache-based check
   (survives Valkey restarts/evictions, and DynamoDB is already the
   system's source of truth) and is a small, scoped change — same shape as
   the disabled code, different (more durable) backing store.
2. **Eval-layer defense (this project, our scope):** when the eval
   framework's Data Connection listener / webhook-derived ingestion feeds
   Langfuse spans or the Outcome Evaluator, dedup on read using
   `call_id + event_type` (+ timestamp where relevant) before writing any
   span or score — regardless of whether the backend fix has shipped yet.
   This follows the standing rule of never trusting external data at a
   system boundary, and means the eval framework's correctness doesn't
   depend on the backend team's fix timeline.

**Rejected: quick stopgap (just re-enable the existing Valkey check
as-is).** Faster, but inherits the cache's TTL/eviction weakness and does
nothing to explain or prevent whatever caused it to be disabled in the
first place — the same failure could resurface silently. Not chosen, but
noted as an emergency fallback if the DynamoDB-conditional-write fix can't
land before this becomes urgent.

**Action item:** file this section (problem statement + decision) as a
ticket against `echobot-studio-backend`, owned by the backend team — it is
a pre-existing production data-integrity bug independent of this eval
project, surfaced *by* this review.

### Resolved — Finding 5: cross-tenant exposure via shared Langfuse/Promptfoo dashboards

**Problem statement (re-examined):** Today, tenant data in DynamoDB is
"logically" multi-tenant only — same Lambdas serve all tenants, isolation
is by partition key (`TENANT#{tenant_id}#LOC#{location_id}`), not by
infrastructure. Introducing a single shared Langfuse project (and,
eventually, Promptfoo scenario packs) as the eval layer's dashboard makes
it materially *easier* for any engineer with access to casually browse
**every tenant's call transcripts side by side** — even if that data was
technically already reachable via DynamoDB/AWS console, a friendly
dashboard UI lowers the friction for cross-tenant snooping far below
today's baseline. The risk is amplified, not newly created from zero, but
still a real regression in practice.

**Verified facts that shaped the decision:** Langfuse's access-control
model is project/org based: organizations are the top-level entity, and
projects group data for RBAC. Critically, **fine-grained per-project RBAC
(restricting which org members can see which project) is only available
on Pro with the paid Teams add-on (+$300/month) or Enterprise** — on the
free Hobby tier locked in for [[Finding 3]], access control is org-wide
only, and the org itself is capped at 2 user seats. This means even
splitting tenants into separate Langfuse projects today would **not**
actually restrict who can view them — every org member can see every
project regardless, until/unless Teams or Enterprise is purchased. So at
this stage, the only access-control lever that actually does anything is
**how many people get an org login at all** — not how the data is
partitioned into projects.

**Options considered:**

| Option | Isolation mechanism | Verdict |
|---|---|---|
| A. Per-tenant Langfuse projects now | Splits trace *data* by project | Gives clean data segmentation, but on Hobby tier provides **zero actual access-control benefit** (no per-project RBAC without Teams/Enterprise) — adds ops overhead (per-tenant keys/config) for a protection that doesn't exist yet, working against the [[Finding 2]] thin-slice principle |
| B. Single shared project, restrict org membership | Controls *who has a login* | Matches the Hobby tier's real constraint (2-user cap anyway), simplest, zero added cost or ops overhead |
| C. Defer until [[Finding 8]] (PII) resolved | N/A — punts the decision | Reasonable in theory (severity depends on what's actually in the traces), but leaves the access question unanswered indefinitely with no forcing function |

**Decision: Option B — single shared Langfuse project, tag every trace
with `tenant_id`, and restrict Langfuse org membership to only the core
eval engineers (the 1-2 people actually building this) — never grant
tenant/customer-facing access, ever, under any circumstance.** This is
the only lever that does real work at the current (free Hobby tier,
POC/MVP) stage; per-tenant projects would be theater without Teams/
Enterprise. Every trace and span must carry `tenant_id` as a tag/metadata
field from day one regardless of project structure, so that filtering and
a future migration to per-tenant projects (if ever needed) is a query/
export operation, not a re-instrumentation effort.

**Accepted risk + revisit trigger:** commingled tenant data inside a
single Langfuse project, visible to all org members, is an accepted risk
for the POC/MVP phase specifically because the org is small (2 people,
both already trusted with full production data access today). **Revisit
this decision before any of the following:** (1) onboarding a second real
paying tenant into the *live* (non-test) eval pipeline with its own data
agreement, (2) granting Langfuse access to anyone beyond the core 1-2 eval
engineers (support staff, other teams, the tenants/customers themselves),
or (3) a tenant contract that explicitly requires data isolation. At that
point, re-evaluate upgrading to Pro+Teams (per-project RBAC) or splitting
into per-tenant Langfuse projects/orgs.

**Forward-compatible step taken now regardless of the decision above:**
Promptfoo scenario packs and any exported golden-call datasets (per
[[Finding 7]] / [[Finding 8]]) should be organized on disk by tenant from
day one (e.g. `scenarios/{tenant_id}/...`), so that tenant separation in
eval *artifacts* doesn't depend on whatever the Langfuse project structure
ends up being — this costs nothing extra today and avoids a re-derivation
effort if stricter isolation is needed later.

### Resolved — Finding 6: no stated business problem driving this work

**Problem statement (re-examined):** the original docx jumped straight to
"build observability + voice eval + regression testing" without saying
what pain that fixes. Unlike the other findings, this isn't a technical
trade-off to research — only the team building the live product knows the
real driver. Asked directly.

**The actual business problem, in the team's own words:** the voice agent
is already live and bugs keep getting found and fixed reactively, but
**there is currently no way to tell whether those fixes are actually
making the agent better over time, across the metrics that matter** —
fixing is happening blind, without a before/after signal. On top of that,
there's no artifact today that lets **anyone who needs to review the
agent's quality** (not just the engineer who happens to be debugging) see
a dashboard of how it's performing. And there's no mechanism to catch a
regression **before it ships** — ideally as part of the deploy pipeline,
or at minimum as an on-demand regression report a human can run pre-release.
**Production order accuracy** was explicitly named as one of the metrics
that matters most.

**What this confirms/changes about scope:** this validates (rather than
contradicts) the three-layer scope already locked in at §0 — it's not a
new layer, it's the *reason* the three layers exist:
- **Observability (Langfuse):** exists to give the "what happened, and is
  it trending better or worse" signal that's missing today — not just
  per-call debugging, but a trend line over time so "we keep fixing bugs"
  turns into "and here's proof it's working."
- **Voice Evaluation Layer (Outcome + audio evaluators):** order accuracy
  is confirmed as a top-tier metric the Outcome Evaluator must compute
  reliably and report on, not a nice-to-have alongside escalation
  correctness.
- **Pre-prod regression (Promptfoo):** confirmed as needing real pipeline
  integration (CI/CD gate), not just a standalone script someone remembers
  to run — "part of the pipeline, or at least able to produce regression
  test results" is the literal bar to hit.
- **Dashboard for "anyone who wants to review"** is a new explicit
  requirement that wasn't called out by name in §0: the eval framework's
  output isn't just a debugging tool for the builder, it's a reviewable
  artifact for other stakeholders. This has a direct interaction with
  [[Finding 5]]'s access-restriction decision — "anyone who wants to
  review" the agent's *quality metrics* (aggregated scores, pass/fail
  trends) is a much lower-risk audience than "anyone who wants to review"
  raw call transcripts with tenant PII. The dashboard should be designed
  so metric/trend views can eventually be shared more broadly than raw
  trace data ever should be.

**Decision:** adopt this as the standing business-problem statement for
the project (to be referenced instead of re-deriving it): *"We have a live
voice ordering agent that we keep patching reactively, with no way to
measure whether changes actually improve it, no reviewable quality
dashboard, and no automated pre-release regression check. Order accuracy
is the headline metric, alongside escalation correctness and conversation
completion."* No alternative framings were rejected — the team confirmed
this directly rather than choosing between researched options.

### Resolved — Finding 7: Voice Evaluation Layer is the riskiest custom build

**Problem statement (re-examined):** the council flagged the Voice
Evaluation Layer (Outcome Evaluator + audio/voice-quality evaluator) as
the long pole because classic approaches need things that don't exist
yet: WER needs a ground-truth transcript dataset, and naturalness/prosody
scoring needs a human-calibrated rubric. Building both from scratch (a
custom WER pipeline plus a custom-trained perceptual audio-quality model)
is a multi-month, ML-heavy undertaking for a small team — exactly the kind
of "double-built risk" this review exists to catch.

**Verified facts that changed the picture:**

- "Semantic WER" (LLM-as-judge scoring meaning-preservation instead of
  word-for-word match) is an emerging, more representative metric than
  classic WER — but it still requires a **reference transcript** per call;
  it changes the *comparison method*, not the need for ground truth.
  ([AssemblyAI: Word error rate is broken](https://www.assemblyai.com/blog/word-error-rate-is-broken))
- For **order accuracy specifically** — the headline metric per
  [[Finding 6]] — a reference already exists without building anything:
  the `transaction_details` DynamoDB table already records what was
  actually ordered. Comparing the agent's recorded order against that is a
  deterministic structured-data diff, not a transcript/WER problem at all.
- For **audio/voice quality**, open-source, pretrained, no-reference
  perceptual-quality models already exist — **NISQA** (predicts overall
  quality plus noisiness/coloration/discontinuity/loudness dimensions) and
  **DNSMOS** (predicts signal/background/overall quality) — eliminating
  the need to train a custom MOS-prediction model from scratch.
  ([NISQA paper](https://ar5iv.labs.arxiv.org/html/2104.09494))
  These still need validation against a small human-rated sample before
  being trusted, since they were validated on general speech/noise-
  suppression domains, not specifically on phone/drive-through audio.

**The follow-up question that sharpened this decision:** "adopt, don't
build" must still cover judgment calls like *order item accuracy*, *did
the agent interrupt the customer*, *did it state menu items/prices
correctly*, *did it clarify the bill properly*. These turned out to need
**three distinct mechanisms**, not one tool — clarified and locked in
here so the Outcome Evaluator's design doesn't collapse them into a
single (wrong) approach:

| Judgment | Mechanism | Why |
|---|---|---|
| Order item accuracy | Deterministic diff against `transaction_details` (see correction below — this is the agent's own derived record, not an independently-verified ground truth) | No model/LLM needed for the diff itself, but see correction below for what it actually validates |
| Menu/price mention correctness, bill clarification, escalation handling | **LLM-as-judge over the transcript, scored against an authored rubric**, grounded against the tenant's actual menu/pricing data as reference | "Adopt" in the sense of using an off-the-shelf judge LLM rather than training a model — but the rubric itself ("what counts as properly clarifying a bill") must be authored once by the team; no tool can define correctness for the product |
| Interruptions / talked-over-customer | Turn-taking timestamps from the Ultravox **Data Connection listener** ([[Finding 1]]) | This is a timing fact about overlapping speech, not something visible in transcript text — transcript-based LLM judging can't detect it reliably |
| Audio/voice quality (naturalness, prosody, glitches) | **NISQA/DNSMOS** (pretrained, no-reference), validated against ~20-50 human-rated calls | Avoids training a custom perceptual-quality model; the human-rated sample is a small, bounded one-time calibration, not an ongoing ML program |

**Decision: adopt rather than build, across all four mechanisms above.**
Concretely: no custom WER engine, no custom-trained audio-quality model,
no large pre-built golden-call dataset required to start. What *is*
still required, and is real one-time work, not a tool to buy: (1)
authoring the LLM-judge rubric(s) for conversational behavior, (2)
wiring tenant menu/pricing data in as grounding reference for that judge,
(3) collecting the small (~20-50 call) human-rated sample to calibrate
NISQA/DNSMOS for this specific phone/drive-through audio domain.

**Rejected: Option C (defer the whole Voice Evaluation Layer until a
formal labeled dataset exists).** Too conservative given [[Finding 2]]'s
thin-slice-first principle — order-accuracy scoring can start immediately
using data that already exists (`transaction_details`), and there is no
reason to block that on a dataset effort that's only actually needed for
the audio-quality slice.

**Sequencing note (ties to [[Finding 2]]'s roadmap):** order-accuracy
diffing and the LLM-judge rubric work can both start now, in parallel,
since neither depends on the Data Connection listener or a labeled audio
dataset. Interruption detection waits on the Data Connection listener
build. Audio/voice quality stays last, gated specifically on collecting
the ~20-50 call human-rated calibration sample — a small, scoped
prerequisite now, not an open-ended "wait for a real dataset."

**Correction (discovered while implementing Finding 9's test-tenant
design): `transaction_details` is not an independent ground truth.**
Tracing the actual code path
(`menuboard_service.py:1819-1926` → `squareup_service.create_order()` →
`ultravox_webhooks.py:203-209,818-913` →
`transaction_details_dynamodb_service.py:38-57`) showed that:
- Square's API call is **write-only** — its response (order ID, Square's
  computed total, line items) is never read back or persisted anywhere.
  `transaction_details.order_details` is in fact always an empty object
  (`ultravox_webhooks.py:906`).
- The DynamoDB write happens **separately**, in the post-call webhook,
  and is **decoupled** from whether the Square call succeeded — it fires
  either way.
- What actually lands in `transaction_details` (items, total, currency)
  is **the agent's own parsed understanding** of the order, taken from
  the tool-call payload or, failing that, re-derived from the transcript.

In other words, diffing against `transaction_details` mainly checks "did
the tool call agree with the transcript" (internal self-consistency) —
it does **not** independently verify that the agent correctly understood
what the customer asked for, or that Square actually received a valid,
matching order.

**Revised ground-truth design, by call type:**

| Call type | Ground truth source | What it catches |
|---|---|---|
| Promptfoo synthetic test scenarios | **The scenario's own authored expected order** (the test writer already knows what was supposed to be ordered when writing the scenario — zero extra labeling cost) | Agent misunderstanding the (scripted) customer intent |
| Promptfoo test-tenant calls specifically | **Square Sandbox order read-back** (GET the created order back from Square Sandbox after the test call, per [[Finding 9]]'s test tenant) | Integration-layer bugs — agent's intent was correct, but a malformed/incomplete payload reached Square |
| Real production calls (no authored expected order exists) | `transaction_details`, accepted as a known-imperfect proxy for now | Tool-call/transcript inconsistency only — does *not* catch the agent genuinely mishearing the customer; revisit with a human/LLM-judge transcript-derived ground truth if this gap proves costly in practice |

This refines, rather than replaces, the Finding 7 decision: order-accuracy
scoring still starts immediately (no blocking on a new dataset), but the
Promptfoo regression path now needs a small added capability — reading an
order back from Square Sandbox — layered on top of [[Finding 9]]'s test
tenant, rather than treating `transaction_details` as sufficient on its
own.

### Resolved — Finding 8: PII handling gap for golden-call datasets

**Problem statement (re-examined):** voice orders likely contain customer
names, phone numbers, and possibly addresses. Per [[Finding 5]] and
[[Finding 7]], raw call transcripts are about to start flowing into
Langfuse traces, Promptfoo scenario packs (likely git-committed, since
that's the natural home for test fixtures), and the human-rated audio
calibration sample — all surfaces with no PII handling today.

**Verified options:** **Microsoft Presidio** — an open-source, Python-
native PII detection/redaction library (regex + NLP, handles names,
phones, addresses; supports replace/redact/hash) — was identified as a
direct adopt-don't-build fit, consistent with the approach already taken
in [[Finding 7]]. ([microsoft/presidio](https://github.com/microsoft/presidio))

**Decision: defer.** No redaction pipeline is being built right now;
PII risk for POC-stage scenario packs and the Finding 7 calibration sample
is accepted as-is for the time being.

**Risk explicitly accepted by deferring (recorded so it isn't
rediscovered as a surprise later):** unlike Langfuse traces — which at
least age out under the Hobby tier's 30-day retention ([[Finding 3]]) —
Promptfoo scenario packs and any exported calibration-sample transcripts
have **no automatic expiry**. If these get git-committed (the natural
place for test fixtures to live) before redaction exists, PII can persist
in git history indefinitely and is very hard to fully scrub retroactively
even if redaction is added later.

**Revisit trigger:** before any of the following: (1) Promptfoo scenario
packs or calibration-sample transcripts are committed to a shared/git
repo, (2) the Finding 7 audio-quality calibration sample is collected and
handed to human raters, (3) any production rollout beyond the current
POC, or (4) Langfuse org membership is widened beyond the core 1-2
engineers ([[Finding 5]]). At that point, adopt Presidio (or equivalent)
for text-based redaction before artifacts are written/shared — this
finding's research does not need to be redone, the option is already
identified above.

### Resolved — Finding 9: Promptfoo could pollute production

**Problem statement (re-examined and escalated):** the original council
concern was "no pre-prod tenant/key separation" between Promptfoo test
runs and production. Investigating the actual call-completion path showed
this is more severe than a data-pollution concern: when a call completes
an order, the backend's **`completeorder` tool creates a real order via
the Square POS API** (`src/services/squareup_service.py:1037-1069`,
invoked from `src/routes/order_routes.py:258,316`). A Promptfoo regression
test that exercises a full order-completion scenario — exactly the kind
of scenario the Outcome Evaluator/regression suite needs to run, per
[[Finding 6]]/[[Finding 7]] — would create a **real order at a real
restaurant**, potentially triggering a real kitchen ticket.

**Verified facts:**

- No test/sandbox concept exists anywhere in the codebase today (no
  `is_test`, `dry_run`, `sandbox`, or similar flag).
- `tenant_id` **is** validated against an allowlist (Secrets Manager +
  DynamoDB tenant registry, `src/services/tenant_service.py:57,247-253`)
  — so a caller cannot invent an arbitrary tenant_id, but a legitimate
  dedicated test tenant *can* be registered through the normal tenant
  setup process, same as any real tenant.
- Each tenant's Ultravox key comes from its own `bot_config`
  (`src/services/ultravox_service.py:81,375-376`) — no separate dev/prod
  Ultravox key split exists today, so Ultravox calls are always billed for
  real regardless of tenant.
- **Square offers a real Sandbox API** — separate credentials from
  production, fake payment values that are never charged, and explicitly
  no real fulfillment — built for exactly this kind of testing.
  ([Square Sandbox docs](https://developer.squareup.com/docs/devtools/sandbox/overview))

**Decision: register a dedicated test/eval tenant whose `bot_config`
points to Square Sandbox credentials instead of a real merchant account.**
Promptfoo test runs use this tenant for every scenario, including ones
that complete an order:
- **Ultravox calls remain real** (the actual voice agent is exercised, not
  a mock) — this is what makes it a genuine regression test rather than a
  simulation. The small, bounded Ultravox billing cost per test run is an
  accepted cost of regression testing, not a blocker.
- **Square calls go through the real `completeorder` code path** (so a
  malformed order payload or broken integration would still be caught)
  but land in Square's Sandbox, which never produces a real order, charge,
  or kitchen ticket.
- The test tenant_id is tagged distinctly (e.g. an `eval-` prefix) so it
  can be filtered out of production dashboards and order-accuracy metrics
  ([[Finding 6]]/[[Finding 7]]) — without this, fake test orders would
  pollute the very metrics this project exists to produce.

**Rejected: mock Square + Ultravox entirely.** Zero cost and fully
deterministic, but tests a simulated agent instead of the real Ultravox
voice agent — undermines the actual purpose of pre-prod regression
testing, which is to catch real behavior regressions before they ship.

**Rejected: real tenant, but no-op the Square tool instead of using
Sandbox.** Simpler to wire up than real Sandbox credentials, but skips
testing the actual Square integration code path entirely — a broken order
payload (e.g. a schema change in `squareup_service.py`) would go
undetected until it hit production.

**Action item:** registering the test tenant and obtaining Square Sandbox
credentials is backend-team work (same ticket-filing pattern as
[[Finding 4]]), since it requires creating a real tenant record and a
Square developer sandbox account — not something the eval framework
project can do unilaterally from outside `echobot-studio-backend`.

**Addendum (from the Finding 7 ground-truth correction): the eval harness
also needs Square Sandbox *read* access, not just the tenant's create
path.** Since Square's order response is never persisted by the backend
today (`order_details` is always empty — see Finding 7's correction), the
eval harness itself must call Square Sandbox's GET-order API directly
after a test call completes, to verify Square actually received a valid,
matching order. This is scoped strictly to the test tenant's Sandbox
credentials — it does not require, and must never use, production Square
read access.

### Resolved — Finding 10: double-built risk between Outcome Evaluator and Promptfoo

**Problem statement (re-examined):** Promptfoo ships a native assertion
engine — `python`/`javascript` custom assertions that call out to
arbitrary code and return pass/fail or a score, and a native `llm-rubric`
assertion (LLM-as-judge against a rubric, with a configurable pass
threshold). Findings 6/7 had already designed an Outcome Evaluator doing
exactly this kind of scoring (order-accuracy diffing, rubric-graded LLM
judging) as if it were a separate system. The council's concern: building
a custom evaluator that duplicates what Promptfoo already does natively
is wasted effort, and risks the two scoring paths silently drifting apart
over time.

**Verified facts:** Promptfoo's `python` assertion type can call any
external Python function, which can in turn import and call arbitrary
project code — there is no technical barrier to Promptfoo invoking the
exact same scoring logic the production pipeline uses.
([Promptfoo Python assertions](https://www.promptfoo.dev/docs/configuration/expected-outputs/python/),
[LLM Rubric](https://www.promptfoo.dev/docs/configuration/expected-outputs/model-graded/llm-rubric/))
However, Promptfoo is fundamentally a **batch/CI test-runner**: it runs a
defined set of test cases on demand or in a pipeline step, with no native
mechanism to trigger off a live `call.ended` webhook or write per-call
scores into Langfuse as production traffic happens.

**Decision: one shared Python scoring library, two callers — not two
scoring engines.** The order-accuracy diff function and the LLM-judge
rubric functions from [[Finding 7]] are built exactly once, as a plain
importable library with no Promptfoo- or Langfuse-specific code inside
it. Two thin orchestration layers call into it:
- **Promptfoo**, via `python`/`llm-rubric` assertions, for synthetic
  pre-prod regression scenarios run in CI ([[Finding 2]], [[Finding 9]]'s
  test tenant).
- **A small production pipeline** (e.g. a Lambda triggered on
  `call.ended`, or a periodic batch job), for real production calls,
  writing scores into Langfuse and the quality dashboard ([[Finding 6]]).

Each orchestrator is matched to what it's actually good at — Promptfoo
for batch CI runs with pass/fail gating, a small Lambda/batch job for
continuous production scoring — while the actual scoring *logic* never
exists in two places. This also means a future change to the rubric or
the order-diff logic ([[Finding 7]]) automatically applies to both
pre-prod tests and production scoring, with no risk of the two drifting
out of sync.

**Rejected: fully separate custom evaluator + independently-configured
Promptfoo assertions.** This is the exact double-built risk the council
flagged — duplicate logic, double the maintenance, and a real risk that
"pass" in a Promptfoo regression test stops meaning the same thing as
"pass" in production over time.

**Rejected: use Promptfoo for all scoring, including production.**
Forcing real-time production call scoring through a batch/CI tool would
require building webhook-triggering and per-call Langfuse-writing glue
code anyway — about as much work as the small dedicated production
pipeline, but a more awkward fit for a tool that wasn't designed for
continuous/event-driven operation.

**Naming note:** earlier findings (6, 7, 9) referred to "the Outcome
Evaluator" as if it were a single standalone service — read that
terminology going forward as **the shared scoring library**, called from
both Promptfoo (pre-prod) and the production pipeline (post-prod), per
this finding's resolution.

### Resolved — Finding 11: market validation against June 2026 alternatives

**Problem statement:** after locking in Findings 1-10, the team asked for
an explicit check — given how fast the AI-tooling market moves — of
whether Langfuse, Promptfoo, and a custom NISQA/DNSMOS-based Voice
Evaluation Layer are still the best available choices as of **June 2026**,
against the full current market, with a stated preference for open-source
unless a licensed alternative makes a "huge difference." Ran three
parallel research passes (purpose-built voice-agent eval platforms;
general LLM observability platforms; pre-prod eval frameworks + audio
quality models) rather than relying on training-data knowledge, since
this specific market segment (voice-agent testing SaaS) barely existed
before 2025.

**Verified facts, by component:**

- **Observability (Langfuse, [[Finding 3]]):** still MIT-licensed and
  free to self-host with no event/user caps; acquired by ClickHouse, Inc.
  (Jan 2026) with no pricing/license change since. Checked against
  LangSmith, Arize Phoenix, Helicone, Braintrust, and others — Phoenix
  (the other real OSS option) is explicitly *weaker* on audio/voice
  tracing than Langfuse; Helicone is disqualified (acquired by Mintlify,
  Mar 2026, now in maintenance mode, no new features planned); LangSmith
  and Braintrust are proprietary with no voice-specific advantage found
  that clears the "huge difference" bar. No general-purpose observability
  platform — open-source or licensed — has first-class voice-native trace
  primitives (turn-taking, barge-in, TTFW); Langfuse just renders audio as
  an attached blob, same as the others.
- **Pre-prod CI gating (Promptfoo, [[Finding 2]]/[[Finding 10]]):** still
  MIT-licensed, 350k+ developers, 130k+ MAU. **New fact: OpenAI acquired
  Promptfoo in March 2026**, folding it into "OpenAI Frontier." OpenAI has
  publicly committed to keep the OSS core MIT-licensed and model-agnostic.
  DeepEval (OSS, pytest-native) is a viable comparable alternative if this
  ever needs hedging.
- **Audio quality scoring (NISQA/DNSMOS, [[Finding 7]]):** neither has
  been superseded; both remain actively maintained reference baselines in
  2026, including being used as the benchmark in newer 2025/2026 papers
  rather than replaced by them. **UTMOS/UTMOSv2** has emerged as a strong
  *complementary* signal specifically for TTS-naturalness scoring (not a
  replacement for DNSMOS's noise-suppression focus or NISQA's
  multidimensional conversational-distortion coverage). Confirmed
  separately: talk-over/dead-air are conversation-*timing* phenomena
  (VAD/diarization-based), not something any MOS-style model — NISQA,
  DNSMOS, or UTMOS — measures; the plan already treats these as a
  separate mechanism (per [[Finding 1]]'s Data Connection listener), which
  this research confirms is the correct split.
- **Purpose-built voice-agent eval platforms (Hamming, Coval, Cekura,
  Bespoken — a category that has emerged since 2025, not previously
  evaluated):** all closed-source SaaS, none offer self-hosting. None
  advertise the two most product-specific, hardest pieces of this
  project's Voice Evaluation Layer: deterministic order-accuracy diffing
  against a structured backend (the Square Sandbox read-back design from
  [[Finding 7]]/[[Finding 9]]), or per-tenant menu-grounded rubric
  judging — both would still need to be custom-built and maintained on
  top of any of these tools. They are genuinely ahead on voice-native
  metrics (turn-taking, barge-in, time-to-first-word) that general
  observability platforms lack.

**Decision: no change to the locked architecture.** Langfuse, Promptfoo,
and NISQA/DNSMOS remain the right choices — confirmed against the current
market rather than assumed. Two additions:

1. **Add UTMOS/UTMOSv2 as a third audio-quality signal** alongside
   NISQA/DNSMOS (per [[Finding 7]]), specifically for TTS-naturalness
   scoring — low-cost, additive, no conflict with the existing calibration
   plan.
2. **Promptfoo's OpenAI acquisition is logged as a watch-item, not acted
   on.** No functional or licensing change has actually occurred; the
   decision is to proceed as planned and revisit only if Promptfoo's
   roadmap, pricing, or licensing materially shifts away from OSS/
   model-agnostic — not to pre-emptively hedge with a second framework now.

**A gap this research surfaced that Findings 1-10 had not addressed:**
confirmed directly that **Promptfoo cannot place or drive an actual
real-time voice call against Ultravox** — it is fundamentally text-in/
text-out (or audio-file grading for some models), with no native
mechanism to open a live session, speak as a simulated caller, and
capture the resulting turns. [[Finding 2]] and [[Finding 9]] already
assumed "Promptfoo scenarios exercise the real Ultravox agent," but the
component that actually *makes the call* was never identified on its own
— and this gap exists regardless of which CI eval framework sits
downstream of it, so it isn't a reason to reconsider Promptfoo itself.

**Decision: custom-build the call-driver harness.** A Python service that
opens an Ultravox Data Connection session (the same integration point as
[[Finding 1]]'s listener), drives a scripted or LLM-simulated caller
persona through the conversation, and captures the resulting audio +
transcript — which then feeds into Promptfoo's scenario assertions same
as before. This keeps the architecture fully open-source/in-house, at the
cost of real engineering effort to build it.

**Rejected: adopt Cekura (or Hamming/Coval) for call-placement only.**
Cekura has transparent self-serve pricing from $30/mo and is
runtime-agnostic (dials in as a black box, doesn't assume a pipeline
architecture), which would have removed this engineering lift entirely.
Not chosen because it's closed-source SaaS, conflicting with the
open-source-first preference, and because none of these vendors'
Ultravox-compatibility was independently verified — revisit this option
specifically (not the others surveyed) if the custom call-driver proves
substantially harder to build than expected.

**Net effect on the build plan:** this finding adds one new concrete
deliverable — the Ultravox call-driver harness — to the roadmap from
[[Finding 2]], sequenced alongside the Data Connection listener (Finding
1) since they share the same integration point, and feeding the Promptfoo
scenario layer that already depended on it implicitly.

### Addendum to Finding 11 (2026-06-23): expanded vendor/benchmark survey

A follow-up request asked for two more vendors (Maxim AI, ServiceNow EVA)
plus five additional names surfaced from memory of early research
(VoiceBench, VoiceAgentBench, ESPnet-SDS, Braintrust.dev, DeepEval). Each
was independently verified via web search rather than recalled — one
materially changed category mid-research (EVA), so it's worth reading
the table below rather than assuming the name implies "vendor."

**Commercial SaaS comparison (same category as Hamming/Coval/Cekura/
Bespoken above):**

| | Hamming AI | Coval | Cekura | Maxim AI |
|---|---|---|---|---|
| Category | Closed SaaS, voice-QA + prod monitoring | Closed SaaS (YC), agent sim/eval | Closed SaaS (YC), voice/chat QA | Closed SaaS, general GenAI eval+observability |
| Self-hosting | No | No | No | No (on-prem only on Enterprise tier) |
| Voice-native simulation | Yes — 50K+ concurrent sim calls, accents/noise/interruptions, auto-converts prod failures into scenarios | Yes — persona-based sim, load/permutation testing | Yes — persona sim, prod-conversation monitoring | Yes — accent/noise/interruption/emotion simulation, no-code UI |
| Order-accuracy / structured-backend diffing | Not advertised | General tool-call/param validation (POS/EHR/API), not order-specific diffing | Not advertised | Not advertised |
| Menu-grounded rubric judging | No | No | No | No (generic eval rubrics) |
| Ultravox compatibility verified | No (LiveKit/Pipecat/ElevenLabs/Retell/Vapi confirmed; Ultravox not mentioned) | No | No (Retell/VAPI/ElevenLabs/LiveKit/Pipecat/Bland confirmed; Ultravox not mentioned) | No |
| Pricing | Enterprise/custom (SOC 2) | Custom | Free tier (300 credits) → from $30/mo | Free (10K logs) → $29-49/seat/mo → Enterprise custom |

**Decision: no change.** Same reasoning as the original Finding 11 verdict
— closed-source, no self-hosting, and Ultravox compatibility unverified
on all four; none compute deterministic order-accuracy diffing or
menu-grounded rubrics, which remain this project's hardest, most
product-specific pieces regardless of which vendor sits underneath.

**ServiceNow EVA — reclassified, not a vendor.** Initially assumed to be
a commercial ServiceNow product; verified to actually be an **open-source
research benchmark framework** (GitHub `ServiceNow/eva` + HuggingFace
datasets, co-developed with NVIDIA for NOWAI-Bench). It scores bot-to-bot
WebSocket audio conversations (two AIs talking to each other, not a real
caller against a live agent) across generic enterprise domains (Airline
CSM, IT Service Management, HR Service Delivery) — not retail ordering —
producing two aggregate scores (EVA-A accuracy, EVA-X experience). It has
no integration point for a specific production agent like ours; adopting
it would still require building our own bot-to-bot driver (the same
call-driver harness work Finding 11 already committed to) just to feed it
data, and its domains/metrics don't cover order-accuracy or menu
grounding. **Rejected as inapplicable** — it's a model-benchmarking
artifact, not a tool that plugs into a live Ultravox deployment.

**Academic benchmarks (VoiceBench, VoiceAgentBench, ESPnet-SDS) — same
"not a tool" classification as EVA:**

- **VoiceBench** (arXiv 2410.17196, TACL'26, OSS on GitHub) — a static
  dataset of 6,783 spoken instructions scoring general knowledge,
  instruction-following, and safety for LLM-based voice assistants. No
  agentic/tool-call evaluation, no order-domain coverage, no live-agent
  integration — it's a leaderboard dataset, not a testing harness.
- **VoiceAgentBench** (arXiv 2510.07978) — closer in spirit to our needs
  (it benchmarks tool-call correctness: single/parallel/dependent calls,
  dialogue-based invocation, safety), but still a static, English +
  6-Indic-language synthetic dataset for comparing base voice-LLMs against
  each other, not a harness for testing one specific deployed agent. No
  order-accuracy or menu-grounding concept. Worth noting as **methodology
  validation** — its tool-call-correctness categories corroborate that
  "right tool, right payload" is an industry-recognized axis, which is
  exactly what `design/02`'s Outcome Evaluator already scores — but it is
  not adoptable as-is.
- **ESPnet-SDS** (NAACL'25 demo, OSS toolkit) — research infrastructure
  for assembling and comparing your *own* cascaded/end-to-end spoken
  dialogue pipelines (swappable ASR/LLM/TTS components) against a
  human-human conversation baseline, with a Gradio demo UI. It assumes you
  bring the SDS components being evaluated; it has no concept of
  evaluating an externally-hosted, already-built production agent like
  ours. Not adoptable for this project's shape.

**Decision: none of the three academic artifacts change the architecture.**
They're useful as external validation that this project's metric
categories (tool-call correctness, instruction-following, audio quality)
are the industry-recognized axes — not as tools to adopt — because all
three evaluate models/pipelines in the abstract, not a specific deployed
production agent reachable only through its real integration surface
(Ultravox + Square + our Lambdas).

**Braintrust.dev and DeepEval — both already named in the original Finding
11 pass, now confirmed with current facts, no change to either verdict:**

- **Braintrust** — closed SaaS, free tier ($0, 1GB/10K scores), Pro
  $249/mo, self-hosting restricted to Enterprise (hybrid control-plane/
  data-plane split, customer runs the data plane in their own cloud via
  Terraform). Still a general text-first LLM eval/observability platform
  with **no voice-native simulation or audio-quality scoring** found —
  confirms the original Finding 11 verdict ("no voice-specific advantage
  found that clears the 'huge difference' bar").
- **DeepEval** (`deepeval.com` / `confident-ai/deepeval` — corrected from
  the `deepval.com` typo) — confirmed 100% free, MIT-licensed, pytest-
  native, and **explicitly has no voice evaluation support** ("Python-
  only, text-based"). This sharpens the original Finding 11 mention (which
  only called it "a viable comparable alternative" to Promptfoo, not
  voice-aware) — DeepEval is not a candidate for the Voice Evaluation
  Layer or audio scoring under any circumstance, only ever a possible
  pre-prod-gating hedge if Promptfoo's roadmap shifts.

**Net effect on the build plan: none.** This addendum adds no new
deliverable (unlike the first Finding 11 pass, which added the call-driver
harness) — every newly-surveyed option is either the same category as
already-rejected SaaS competitors, or a research artifact with no
production-agent integration surface at all.

---

## 1. Overview

This document outlines a reference architecture for evaluating and
monitoring a voice-based AI agent (e.g., drive-through ordering) using:

- Ultravox (voice runtime)
- AWS (API Gateway, Lambda, S3, DynamoDB)
- Langfuse (observability + evaluation tracking)
- Promptfoo (pre-prod regression)
- Voice Evaluation Layer (custom metrics for STT/TTS quality)

---

## 2. Architecture Layers

### A. Runtime / Interaction Layer

- Customer / Test Caller
- Ultravox Voice Runtime
- Prompt + Agent Logic
- AWS API Gateway + Lambda (Tools)
- DynamoDB (order/session state)
- S3 (audio + artifacts)

---

### B. Observability Layer (Langfuse)

- Captures full conversation traces
- Tracks:
  - Turns
  - STT / LLM / TTS spans
  - Tool calls
  - Latency
  - Errors
- Acts as system of record for all runs

---

### C. Voice Evaluation Layer

Separates "quality" from "trace data"

Includes:

- Transcript Evaluator
- Audio Evaluator
- Optional Human Review

Handles:

- Task success
- Tool correctness
- WER (speech accuracy)
- Voice quality
- Interruption handling

---

### D. Pre-Prod Regression Layer (Promptfoo)

- Runs scenario packs
- Compares prompt versions
- Enforces pass/fail conditions
- Integrates with CI/CD

---

### E. Dashboard Layer

- Langfuse dashboards (engineering view)
- Management scorecard (KPIs)
- Drill-down trace inspection

---

## 3. Metric Computation

### Runtime Metrics (Langfuse)

- Turn count
- Latency (P50/P95)
- Tool success/failure
- TTFB

### Outcome Metrics (Transcript Evaluator)

- Order success %
- Add/Edit/Delete accuracy
- Tool correctness
- Payload correctness
- Escalation accuracy

### Voice Metrics (Audio Evaluator)

- WER (speech accuracy)
- Interruption recovery
- Dead air / silence
- Talk-over rate
- Voice naturalness score

### Pre-Prod Metrics (Promptfoo)

- Scenario pass rate
- Regression detection
- Prompt comparison results

---

## 4. Score Flow

1. Runtime generates traces → Langfuse
2. Evaluators compute scores
3. Scores pushed to Langfuse
4. Dashboards aggregate scores

---

## 5. Management Scorecard

- Order success %
- Tool accuracy %
- Payload correctness %
- Interruption recovery %
- WER
- Latency (P95)
- Voice quality score
- Prompt version delta

---

## 6. Key Design Principles

- Langfuse = "what happened + why"
- Voice Layer = "how good was it"
- Promptfoo = "did it improve"

---

## 7. Summary

This architecture allows:

- Deep debugging (trace-level)
- Objective scoring (voice + task)
- Regression tracking across prompt iterations
- Clear management visibility

## Reference Architecture

Below is a **reference architecture** for your current environment
(**Ultravox + AWS**) that adds **Langfuse for observability/evals**,
**Promptfoo for pre-prod regression orchestration**, and a separate
**Voice Evaluation layer** to cover the voice-specific gaps discussed
(STT/TTS quality, interruption handling, conversational quality).

Two things are kept clearly separated:

- **Source-backed facts** about what Langfuse / Promptfoo support.
  【1-58e8e1】【2-607477】【3-28ab0a】【4-db216b】【5-5456b1】
- **Architectural recommendation** for how to combine them in the
  AWS/Ultravox landscape.

Internal architecture documents also reinforce that the environment
already centers around **AWS Lambda / API Gateway / S3 / DynamoDB** and
that logging/observability should be handled through logs rather than
using DynamoDB as the main analytics layer.
【6-0affb2】【7-de4f71】【8-37d4fd】

### 1) Reference architecture — logical component view

#### A. Runtime / Customer Interaction Plane

These are the components that run the actual drive-through voice agent.

**Components**

1. **Customer / Test Caller Channel**
   - Phone, browser, or synthetic caller
2. **Ultravox Voice Runtime**
   - Handles live voice session and agent conversation flow *(from
     current setup / user context)*
3. **Prompt + Agent Logic**
   - Current prompt(s), conversation policies, tool selection logic
4. **AWS Tooling Layer**
   - **Amazon API Gateway**
   - **AWS Lambda** tool endpoints for:
     - FetchMenu
     - AddOrderItem
     - EditOrderItem
     - DeleteOrderItem
     - GetOrderSummary
5. **Application Data Stores**
   - **DynamoDB** for transcripts / order state / scenario artifacts
     *(already in current flow per description)*
   - **S3** for raw artifacts such as audio, exports, and batch
     evaluation outputs *(aligned with internal AWS pattern)*
     【6-0affb2】【7-de4f71】

#### B. Observability & Trace Plane

This layer explains **what happened** in each conversation and why.

**Components**

1. **Langfuse (self-hosted)**
   - Used for:
     - traces
     - observations / spans
     - evaluation scores
     - score analytics
     - monitoring trends
2. **Trace / Event Adapter**
   - A lightweight instrumentation layer between Ultravox / tool
     executions / AWS events and Langfuse
   - For every conversation, this adapter should create:
     - **Conversation trace**
     - **Turn spans**
     - **STT span**
     - **LLM span**
     - **TTS span**
     - **Tool spans**
     - **Error / escalation events**

Langfuse's documentation explicitly supports tracing with **traces,
sessions, observations, tools, latency, cost, and evaluation scores**,
and its Pipecat voice integration shows the exact voice trace pattern
to reproduce conceptually: **conversation → turns → STT / LLM /
TTS** with TTFB and usage metrics.
【9-349b91】【2-607477】【3-28ab0a】【1-58e8e1】

#### C. Voice Evaluation Plane

This layer explains **how good the conversation was**, which is where
Langfuse alone is weaker.

**Components**

1. **Transcript / Outcome Evaluation Service**
   - Computes transcript- and tool-driven metrics
2. **Audio / Voice Quality Evaluation Service**
   - Computes voice-specific metrics that are not natively produced by
     Langfuse
3. **Human Review Workbench (optional but strongly recommended)**
   - For sampled calls, edge cases, and calibration of automated scores

**Why this layer exists**

Langfuse's sources reviewed support tracing, monitoring, scores, tool
outcomes, latency, and evaluations, but **they do not explicitly say
Langfuse natively computes voice-specific metrics such as WER or TTS
naturalness/MOS**. The sources support that Langfuse can store and trend
scores; they do **not** document native built-in WER/TTS-quality
computation. 【3-28ab0a】【2-607477】【9-349b91】

So in this architecture, the **Voice Evaluation Plane** computes those
scores externally and then writes them back into Langfuse as evaluation
scores.

#### D. Pre-Prod Regression & Release Gate Plane

This is where prompt changes are evaluated before they go live.

**Components**

1. **Promptfoo**
   - Used as the **pre-prod orchestration harness**
   - Runs scenario suites, assertions, and regression packs in CI/CD
2. **Scenario Dataset Store**
   - Test conversations, expected order states, expected tool calls,
     expected outcomes
3. **Golden Call Library**
   - Known good and known bad voice scenarios
4. **Release Gate**
   - Blocks or flags prompt changes if critical metrics degrade

Promptfoo's documentation explicitly states it is an open-source
CLI/library for **evaluating prompts, agents, and RAGs**, supports
**automated scoring**, and works in **CI/CD**. Its guides also
explicitly include evaluation workflows for agents and voice-related
integrations such as **ElevenLabs Voice AI**.
【4-db216b】【5-5456b1】【10-4eb6d5】

#### E. Dashboard & Reporting Plane

This layer is for management and operational visibility.

**Components**

1. **Langfuse Operational Dashboard**
   - Engineering / QA / AI delivery team dashboard
2. **Management Scorecard Dashboard**
   - Simplified KPI view
3. **Investigation Drill-Down View**
   - Trace-by-trace analysis for root cause

Langfuse's evaluation and monitoring docs explicitly support **score
analytics**, **dashboards**, **tracking trends over time**, and
surfacing specific traces for investigation. 【2-607477】【3-28ab0a】

### 2) Recommended end-to-end flow

**Pre-Prod flow**

1. Prompt change is proposed.
2. Promptfoo runs the **scenario pack**.
3. Each scenario invokes the Ultravox-enabled agent and AWS tools.
4. Runtime events are traced into Langfuse.
5. Voice Evaluation Plane computes:
   - outcome quality
   - tool correctness
   - STT/TTS/turn quality metrics
6. Scores are attached to the scenario run in Langfuse.
7. Promptfoo compares current run vs baseline.
8. Release gate passes/fails based on threshold rules.
9. Management sees **trend comparison**: "Prompt v17 vs v16".

**Occasional production monitoring flow**

1. Real calls are sampled or routed for monitoring.
2. Runtime traces go to Langfuse.
3. Voice Evaluation Plane runs asynchronously on selected calls.
4. Scores are written back to Langfuse.
5. Dashboards show:
   - current quality
   - degradation
   - top failure modes
   - problematic traces for investigation

### 3) Where each metric is computed

Below is the recommended metric ownership model.

#### A. Metrics computed primarily from runtime traces / tool events

These are metrics that should come from the **Ultravox + AWS tool
execution + Langfuse trace model**.

| **Metric** | **Where computed** | **Source inputs** | **Purpose** |
|----|----|----|----|
| **Turn count** | Trace Adapter / Langfuse | Conversation + turn spans | Detect looping / friction |
| **Latency per turn** | Langfuse | STT / LLM / TTS / tool span timings | Find delay and regression |
| **TTFB / first response delay** | Langfuse | Voice service span timings | Detect awkward pauses |
| **Tool call count** | Langfuse | Tool observations | See over-calling / retries |
| **Tool call success/failure** | Langfuse | Tool outputs + errors | Runtime reliability |
| **Prompt version / model version / scenario ID** | Trace Adapter metadata into Langfuse | Deployment metadata | Compare runs correctly |

Langfuse's docs explicitly support observations for **tools**,
**latency**, **cost**, **errors**, and score analytics over time.
【11-918ab2】【3-28ab0a】【2-607477】

#### B. Metrics computed in the Transcript / Outcome Evaluation Service

These are metrics best computed from transcript + tool output + expected
outcome.

| **Metric** | **Where computed** | **Source inputs** | **Purpose** |
|----|----|----|----|
| **Task success %** | Outcome evaluator | Final order state vs expected order state | Core KPI |
| **Add item correctness %** | Outcome evaluator | Tool calls + final order state | Ordering accuracy |
| **Edit item correctness %** | Outcome evaluator | Tool calls + final order state | Ordering accuracy |
| **Delete item correctness %** | Outcome evaluator | Tool calls + final order state | Ordering accuracy |
| **Right tool called?** | Outcome evaluator | Expected tool vs actual tool | Tool routing quality |
| **Right payload passed?** | Outcome evaluator | Tool input payload vs expected payload | Parameter correctness |
| **Escalation correctness %** | Outcome evaluator | Escalation event + scenario rule | Detect missed handoffs |
| **Conversation completion %** | Outcome evaluator | Call end state | Business completion |

Langfuse's evaluation docs explicitly support scoring traces and running
offline/online evaluations, and tool-call correctness can be evaluated
through datasets and score logging. 【12-58e0cd】【2-607477】

#### C. Metrics computed in the Audio / Voice Quality Evaluation Service

These are the metrics needed because Langfuse is not a voice-native
scorer by itself.

| **Metric** | **Where computed** | **Source inputs** | **Purpose** |
|----|----|----|----|
| **WER / STT accuracy** | Audio evaluator | Audio + ground-truth script / labeled dataset | Speech recognition quality |
| **Interruption recovery %** | Audio evaluator | Audio timeline + turn events | Can it handle cut-ins? |
| **Barge-in handling score** | Audio evaluator | Playback interruption + response timing | Natural conversation feel |
| **Silence / dead-air time** | Audio evaluator | Timing signals + audio | Detect awkward long pauses |
| **Talk-over / overlap rate** | Audio evaluator | Audio overlap analysis | Detect poor turn-taking |
| **TTS naturalness score** | Audio evaluator / human review | Audio sample + rubric | Voice quality |
| **Pacing / prosody score** | Audio evaluator / human review | Audio sample + rubric | Customer experience |
| **Re-prompt / repeat rate** | Audio + transcript evaluator | Transcript + turn events | Detect misunderstanding |

This table is a **recommended architecture design**. The sources
reviewed do **not** explicitly document Langfuse as natively computing
these audio-native metrics, which is why they belong in a separate
evaluation layer. 【3-28ab0a】【2-607477】

#### D. Metrics computed by Promptfoo in pre-prod

Promptfoo should remain the **regression orchestrator**, not the system
doing all voice analytics itself.

| **Metric / Result** | **Where computed** | **Source inputs** | **Purpose** |
|----|----|----|----|
| **Pass / fail by scenario** | Promptfoo | Assertions over run outputs | Release gating |
| **Prompt A vs Prompt B winner** | Promptfoo | Scenario comparison | Controlled A/B comparison |
| **Regression detected?** | Promptfoo | Current run vs baseline | Safe prompt iteration |
| **Critical assertion failures** | Promptfoo | Scenario rules | Prevent bad release |

Promptfoo's documentation explicitly supports **testing
prompts/agents**, **automated scoring**, and **CI/CD-based evaluation**,
which makes it a strong regression orchestrator for the pre-prod flow.
【4-db216b】【5-5456b1】

### 4) How scores flow into dashboards

Here is the simplest and cleanest score flow.

**Step 1 — Runtime events create traces**

- Ultravox / tool events / AWS tool calls are instrumented into
  Langfuse.
- Every conversation becomes a trace with turn-level structure.
- Tool spans contain:
  - tool name
  - inputs
  - outputs
  - status
  - latency

**Step 2 — Evaluators compute scores**

Three evaluator types write back to the same call / scenario:

**Evaluator type 1: Runtime trace-derived**

Examples:

- latency
- tool success
- turn count

**Evaluator type 2: Outcome rules**

Examples:

- order completed correctly
- correct item edited
- correct delete action
- correct tool payload

**Evaluator type 3: Voice-quality rules**

Examples:

- WER
- interruption recovery
- dead-air
- naturalness

**Step 3 — Scores are attached to the call / run in Langfuse**

Langfuse supports evaluations/scores and score analytics over time.
【2-607477】【3-28ab0a】

**Step 4 — Dashboards read the same score set in three views**

**A. Engineering / QA dashboard**

Purpose:

- debug failures
- inspect traces
- inspect bad turns
- inspect tool payload problems

Recommended KPIs:

- failed scenarios
- top broken tool calls
- calls with high dead-air
- calls with poor interruption recovery
- calls with low task success

**B. Product / management dashboard**

Purpose:

- answer "are we getting better?"

Recommended KPIs:

- overall task success %
- tool correctness %
- order edit/delete correctness %
- interruption recovery %
- average / P95 response latency
- WER trend
- voice-quality trend
- baseline vs current prompt version comparison

**C. Release-readiness dashboard**

Purpose:

- go / no-go for prompt change

Recommended KPIs:

- pass rate by scenario pack
- critical regressions
- blocked release reasons
- top changed metrics from previous baseline

### 5) Suggested management scorecard structure

Management should see only **8–10 KPIs**, not raw traces.

**Management scorecard**

1. **Order completion success %**
2. **Add/Edit/Delete accuracy %**
3. **Correct tool call %**
4. **Correct tool payload %**
5. **Interruption recovery %**
6. **WER / speech understanding score**
7. **Average + P95 response latency**
8. **Voice quality / naturalness score**
9. **Escalation correctness %**
10. **Prompt version delta vs previous baseline**

These are the KPIs that translate best from technical signals into
business confidence.

### 6) Recommended artifact placement in the AWS landscape

Because internal docs already emphasize that operational logging
should flow into logs and observability systems rather than using
DynamoDB as the primary analytics layer, the recommended split is:
【6-0affb2】【8-37d4fd】

**Keep in DynamoDB**

- transcript records (if already there)
- order session state
- scenario metadata if needed by application flow

**Keep in S3**

- raw audio
- transcript exports
- scenario datasets
- golden call artifacts
- offline evaluation outputs

**Keep in Langfuse**

- traces
- turn spans
- tool observations
- evaluation scores
- trend analytics

**Keep in Promptfoo**

- test definitions
- assertion logic
- regression-run results (and optionally publish summary into Langfuse /
  BI)

### 7) Final recommended architecture summary

**Core stack**

- **Ultravox** — live voice runtime *(current solution)*
- **AWS API Gateway + Lambda** — menu/order tools and business actions
- **DynamoDB + S3** — transcripts, state, artifacts
- **Langfuse** — observability, traces, evaluation storage, trend
  analytics
- **Voice Evaluation Layer** — transcript + audio quality scoring
- **Promptfoo** — pre-prod regression orchestration and release gating

**Design principle**

- **Langfuse = what happened + why**
- **Voice Evaluation Layer = how good was it**
- **Promptfoo = did the new prompt improve or regress**

**Bottom-line recommendation**

For a management-credible POV, build the demo around **two
dashboards**:

1. **Operational trace dashboard in Langfuse**
   - For engineering and QA
2. **Executive scorecard dashboard**
   - For management, showing prompt-version-over-prompt-version
     improvement

That gives both:

- **root-cause visibility**
- **decision-ready improvement evidence**

## One-page architecture diagram description (ready for PPT / Visio)

Below is a **clean, leadership-ready diagram layout** that can be
directly converted into a slide.

### Layout: 5 horizontal layers (top → bottom)

**Layer 1: User / Test Entry (top of diagram)**

Customer / Synthetic Test Caller
↓
- Real users (drive-through calls)
- Pre-prod simulated calls (Promptfoo)

**Layer 2: Voice Runtime & Execution Plane**

```
Ultravox Voice Runtime
        ↓
Prompt + Agent Logic
        ↓
AWS API Gateway
        ↓
Lambda Tool Services
(Add / Edit / Delete / Fetch Menu)
        ↓
DynamoDB + S3
```

**Purpose:**

- Runs the actual conversation
- Executes ordering flows
- Maintains session and order state

**Layer 3: Observability & Trace Plane (Langfuse)**

Position this *parallel* to runtime (right-hand side connection):

```
┌─────────────────────────────┐
Runtime ──> │ Langfuse Observability      │
            │ - Conversation traces       │
            │ - Turn-level spans          │
            │ - STT / LLM / TTS tracking  │
            │ - Tool calls + payloads     │
            │ - Latency + errors          │
            └─────────────────────────────┘
```

**Purpose:**

- Captures "WHAT happened"
- Enables debugging and trace analysis

**Layer 4: Evaluation Layer (split into 2 blocks)**

Position this *below Langfuse with arrows both ways*

```
┌──────────────────────────┐
│ Transcript Evaluator     │
│ - Task success           │
│ - Tool correctness       │
│ - Payload validation     │
│ - Escalation accuracy    │
└──────────────────────────┘

┌──────────────────────────┐
│ Audio / Voice Evaluator  │
│ - WER (STT accuracy)     │
│ - Interruption handling  │
│ - Dead air / latency     │
│ - Voice quality          │
└──────────────────────────┘
```

**Flow:**

- Both evaluators:
  - Read traces + audio + transcripts
  - Compute scores
  - Push scores → **Langfuse**

**Layer 5: Pre-Prod & Governance (Promptfoo)**

Position this on the **left side feeding runtime**

```
Promptfoo (CI/CD Regression Engine)
        ↓
Scenario Dataset / Golden Calls
        ↓
Triggers test runs in Ultravox
```

**Purpose:**

- Compares prompt versions
- Detects regressions
- Enforces release gate

**Layer 6: Dashboard & Consumption Layer (bottom)**

```
┌────────────────────────────┐
│ Engineering Dashboard      │
│ (Langfuse)                 │
│ - Trace debugging          │
│ - Failure inspection       │
└────────────────────────────┘

┌────────────────────────────┐
│ Management Scorecard       │
│ - Task success %           │
│ - Tool accuracy %          │
│ - WER trend                │
│ - Latency (P95)            │
│ - Voice quality score      │
│ - Prompt comparison        │
└────────────────────────────┘
```

### End-to-end flow arrows (critical for diagram clarity)

Draw these main flows:

**Runtime flow**

User → Ultravox → Tools → AWS → Response

**Observability flow**

Runtime → Langfuse (continuous tracing)

**Evaluation flow**

Langfuse + Audio → Evaluators → Scores → Langfuse

**Pre-prod flow**

Promptfoo → Ultravox → Langfuse → Evaluators → Scores → Pass/Fail

### One-line summary (diagram caption)

**Ultravox executes the voice agent, Langfuse records and explains every
interaction, the Evaluation Layer measures quality, and Promptfoo
ensures each prompt change improves performance before production.**
