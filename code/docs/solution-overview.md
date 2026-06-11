# Solution Overview — DevOps + AIOps Take-Home

## Decisions & Reasoning

### Task 3: Observability Metrics (done first)

**Decision: Implement Task 3 before Tasks 2 & 4.**
The alert rules and dashboard panels both reference `agent_rejections_total`, which didn't exist yet. Building the metrics first unblocked the other two tasks naturally.

**Decision: `agent_rejections_total` counter with `reason` label.**
A single counter with a `reason` label (`prompt_injection`, `secrets_request`, `dangerous_action`) lets operators answer "what kind of rejections are spiking?" with one query. Without the `reason` dimension, you'd know rejections went up but not why — useless at 3am.

**Decision: Add `agent_request_errors_total` separately from rejections.**
Rejections are intentional (the agent correctly refused a bad request). Errors are unintentional (malformed input, server bugs). Conflating them makes debugging harder — a spike in errors points to a code or infra problem, while a spike in rejections points to traffic or config changes.

**Decision: Add `agent_message_length_chars` histogram.**
Unusually long messages can indicate abuse, scraping, or prompt injection attempts. A histogram lets operators spot anomalies without reading individual messages. Cheap to collect, valuable when something weird happens.

**Decision: Add `agent_in_flight_requests` gauge.**
Shows concurrent request count at any point in time. Useful for detecting connection pool exhaustion or thread starvation — things that latency histograms reveal too late.

---

### Task 2: Alerting Strategy

**Decision: HighRejectionRate threshold at 16%.**
The traffic generator runs at `REJECTION_MIX_RATIO=0.15` (15% baseline). Setting the alert at 16% keeps it just above baseline so it can realistically fire when rejection rate drifts above normal — making it useful for demo and observation purposes. 
*(Note: I set this to 16% strictly to see how the system reacts in this mock setup. In a real production environment, this should be set to 20-25% to trigger major incidents without alert fatigue).*

**Decision: 5-minute `for:` duration on HighRejectionRate.**
Short bursts of rejections happen naturally (a few adversarial requests in a row). Requiring the condition to persist for 5 minutes filters noise while still catching sustained issues within a reasonable window.

**Decision: RejectionRateSpike uses 2× ratio against 30m baseline instead of a fixed threshold.**
A fixed threshold doesn't adapt to changes in traffic patterns. Comparing current 5m rate against the 30m average detects *relative* changes — so it works whether baseline is 10% or 20%. The 2× multiplier was chosen because normal variance rarely doubles the average.

**Decision: Add HighLatency alert (p95 > 1s).**
This app does regex matching and string responses — normal latency is sub-100ms. A p95 above 1s means something is fundamentally wrong (resource contention, GC pauses, infrastructure issues). It's a different signal than rejection rate.

**Decision: Add PromptInjectionSpike alert.**
Prompt injection is specifically worth monitoring because it indicates active attacks, not misconfigured filters. A sustained rate of >0.5/s for 3 minutes suggests automated probing rather than a human user.

**Decision: Include `{{ $value }}` in all annotations.**
An alert that says "rejection rate is high" is less useful than one that says "rejection rate is 42%". Including the current value saves the on-call engineer from having to query Prometheus just to understand the severity.

---

### Task 4: Dashboard

**Decision: Use `sum by (reason)` grouping in rejection panels.**
Breaking down by `reason` label is the most useful dimension for operators. It immediately answers "is this a prompt injection spike or are we rejecting legitimate requests?" — which determines whether to investigate security or roll back a deployment.

**Decision: Different time windows for different panels.**
- Rejection Rate panel uses `[5m]` — smooths out noise for trend analysis
- Rejections by Reason panel uses `[1m]` — more granular for real-time debugging

These serve different purposes: the rate panel is for dashboarding, the per-second panel is for active incident investigation.

---

### Task 1: CI/CD Pipeline

**Decision: Use eval-runner as the quality gate, not unit tests.**
The eval-runner tests the actual running system end-to-end (golden dataset for false positives, adversarial dataset for false negatives). This is more meaningful than unit tests for a classification system — it catches regressions that unit tests would miss (e.g., a regex change that breaks detection).

**Decision: Single `build-and-test` job using Docker Compose.**
Mirrors the local development workflow (`make up` → `make eval`). No need for separate build/test infrastructure — the system is already containerized. Keeps CI simple and reproducible.

**Decision: Deploy job updates `deployment/manifest.yml` with commit SHA.**
Makes deployments traceable to specific commits. When something breaks in production, you can immediately find the exact code that was deployed. The manifest also builds a deployment history table for audit trails.

**Decision: Deploy only runs on push to main, not on PRs.**
PRs should validate code quality but never deploy. The `if: github.ref == 'refs/heads/main'` guard ensures only merged code reaches the deployment step.

---

### Task 5: Incident Response

**Decision: Include system-specific commands, not generic procedures.**
A runbook that says "check the logs" is useless. Including exact `curl` commands, PromQL queries, and `docker compose` invocations means an engineer unfamiliar with the system can follow it without guessing.

**Decision: Severity table based on rejection rate thresholds.**
Gives the on-call engineer a quick decision framework instead of relying on judgment at 3am. The thresholds align with the alert configuration (25% = warning territory).

**Decision: Separate "mitigate vs escalate" criteria.**
Not every rejection spike needs a rollback. If adversarial traffic is being correctly rejected, the system is working as designed. The decision framework helps distinguish "the system is protecting itself" from "the system is broken."

---

## Feature Summary

### What's Implemented

| Area | Feature | Key Detail |
|---|---|---|
| **Metrics** | `agent_rejections_total` | Counter by `reason` label — powers alerts + dashboard |
| **Metrics** | `agent_request_errors_total` | Separates errors (bugs) from rejections (intended behavior) |
| **Metrics** | `agent_message_length_chars` | Histogram for input size anomaly detection |
| **Metrics** | `agent_in_flight_requests` | Gauge for real-time concurrency visibility |
| **Alerts** | `HighRejectionRate` | >17% for 5m (just above 15% baseline) |
| **Alerts** | `RejectionRateSpike` | 2× the 30m baseline — adapts to traffic changes |
| **Alerts** | `HighLatency` | p95 > 1s for 5m |
| **Alerts** | `PromptInjectionSpike` | >0.5 req/s for 3m — detects automated attacks |
| **Dashboard** | All 8 panels working | Fixed 3 broken panels, 1 auto-resolved via new metrics |
| **CI/CD** | Build → eval → deploy pipeline | Eval-runner as quality gate, SHA-based deployment traceability |
| **Runbook** | Incident response for rejection spikes | System-specific commands, severity table, decision framework |

### Worth Mentioning

- **Task dependency awareness**: Metrics (Task 3) was done first because alerts and dashboard both depend on `agent_rejections_total`. This avoided building on placeholders.
- **Eval passes 100%**: Golden dataset 20/20 accepted, adversarial dataset 15/15 rejected. No regressions introduced.
- **All alerts are healthy**: 5 rules loaded in Prometheus, all in `inactive` state (no false positives out of the gate).
- **Threshold reasoning is documented in-place**: Alert YAML comments explain why each value was chosen, so future maintainers don't have to guess.
- **Docker Compose v2 compatibility**: Fixed `docker-compose` → `docker compose` and removed obsolete `version` attribute.
