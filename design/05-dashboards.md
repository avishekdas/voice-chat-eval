# Design: Dashboard & Reporting Layer

Implements [[Finding 5]] and [[Finding 6]]'s dashboard requirement. Owns:
the engineering trace-debugging view and the management scorecard. Reads
scores written by [04](04-production-pipeline.md) and
[03](03-preprod-regression.md) — computes nothing itself.

## 1. Engineering / QA dashboard

Native Langfuse Cloud trace explorer, filtered by the `tenant_id` tag.
Nothing custom to build — this is Langfuse's default UI, used as-is for
trace-by-trace debugging and failure inspection. No design decision needed
here beyond what [01](01-observability.md) already specifies for tagging.

## 2. Management scorecard

### Open spike — not yet resolved

Whether Langfuse Cloud's **Hobby tier** supports custom score-aggregation
dashboards (beyond its default trace/score views) was **not verified this
session** — this needs a direct check against current Langfuse Cloud
features before committing to either path below.

### Two candidate approaches

| Approach | Mechanism | Verdict |
|---|---|---|
| **A. Native Langfuse dashboards** | If Hobby tier supports custom dashboard/scorecard views, build the [[Finding 6]] 10-KPI scorecard directly there. | Simplest if available — zero new infra. Blocked on the spike above. |
| **B. Export scores into the existing Prometheus/Grafana stack (recommended fallback)** | A small Lambda periodically pulls scores via Langfuse's public API and exposes them as Prometheus metrics (or pushes to a pushgateway), feeding the **already-existing** EC2-based Prometheus+Grafana stack confirmed in `monitoring/` (`prometheus-grafana-working.yaml`, `prometheus.yml` with a CloudWatch exporter job). | Reuses verified, already-operated infra rather than building a new BI surface — consistent with the project's "adopt, don't build" pattern ([[Finding 7]]). |

**Recommendation: spike A first (it's free to check); fall back to B if
Hobby tier doesn't support it.** B is the safer bet to actually plan around
given Langfuse hasn't confirmed this capability on the free tier in any
source reviewed.

### Why Approach B is more than a fallback — it also resolves a tension between Findings 5 and 6

[[Finding 5]] restricts Langfuse org membership to the core 1-2 eval
engineers, specifically because the Hobby tier has no per-project RBAC —
anyone with a login sees every tenant's raw transcripts. [[Finding 6]]
separately wants the scorecard visible to "anyone who needs to review the
agent's quality," a broader audience than that. These two requirements are
in direct tension **only if the scorecard lives inside Langfuse itself**
(broadening access there means broadening raw-transcript access too).

**If Approach B is used, this tension dissolves cleanly:** only aggregated
*scores* (not transcripts) leave Langfuse and land in Grafana, which has
its own, separate access-control model. Grafana viewer access can be
granted broadly — satisfying [[Finding 6]] — without ever widening who can
browse raw Langfuse traces — preserving [[Finding 5]]. This is a genuine
side benefit of choosing B, not just a cost-avoidance argument, and is
worth weighing into the spike decision above even if Hobby tier turns out
to support native dashboards.

### KPI list

Reuses `idea.md` §5's scorecard unchanged — restated here as the literal
contract this dashboard must render, all filtered to exclude
`is_eval_tenant = true` records per [04](04-production-pipeline.md)'s
tagging:

1. Order completion success %
2. Add/Edit/Delete accuracy %
3. Correct tool call %
4. Correct tool payload %
5. Interruption recovery %
6. WER / speech understanding score
7. Average + P95 response latency
8. Voice quality / naturalness score (NISQA/DNSMOS/UTMOS, post-calibration per [[Finding 7]])
9. Escalation correctness %
10. Prompt version delta vs previous baseline

### Release-readiness view

Separate from the management scorecard — this is Promptfoo's own run
output ([03](03-preprod-regression.md)), consumed directly from CI
results/logs rather than through Langfuse or Grafana. No new design needed
here; it's already covered by [03](03-preprod-regression.md)'s release-gate
section.

## Out of scope for this doc

- Score computation → [02](02-voice-evaluation-layer.md).
- Score writing/tagging → [04](04-production-pipeline.md).
