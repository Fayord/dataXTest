# Incident Response: Rejection Rate Spike

## 1. Initial Triage (first 5 minutes)

**Confirm the alert is real:**

```bash
# Check current rejection rate (should be ~15% baseline)
curl -s 'http://localhost:9090/api/v1/query?query=sum(rate(agent_rejections_total[5m]))/sum(rate(agent_requests_total{route="/ask"}[5m]))' | jq '.data.result[0].value[1]'

# Check API is responding
curl -s http://localhost:8080/healthz | jq .
```

**Quick severity assessment:**

| Rejection Rate | Severity | Action |
|---|---|---|
| 15-25% | Low | Monitor, likely normal variance |
| 25-50% | Medium | Investigate, may affect legitimate users |
| >50% | High | Immediate investigation, likely config change or attack |

## 2. Investigation

**Identify which rejection reason is spiking:**

```bash
# Breakdown by reason
curl -s 'http://localhost:9090/api/v1/query?query=sum by (reason) (rate(agent_rejections_total[5m]))' | jq '.data.result[] | {reason: .metric.reason, rate: .value[1]}'
```

**Check if a deployment caused it:**

```bash
# Check current prompt version
curl -s http://localhost:8080/healthz | jq .prompt_version

# Check recent deployment manifest
cat code/deployment/manifest.yml | grep image_tag

# Compare rejection rate before/after by time range in Grafana
# Dashboard: Agent API Monitoring → Rejection Rate by Reason panel
```

**Check application logs:**

```bash
docker compose logs --tail=100 agent-api | grep -i "reject\|error"
```

**Check traffic patterns:**

```bash
# Is request volume unusual?
curl -s 'http://localhost:9090/api/v1/query?query=sum(rate(agent_requests_total{route="/ask"}[5m]))' | jq '.data.result[0].value[1]'

# Check traffic generator config
docker compose exec traffic-generator env | grep -E "REJECTION_MIX|REQUEST_INTERVAL"
```

## 3. Decision Framework

**Mitigate if:**
- Legitimate user requests are being incorrectly rejected (false positives)
- A recent deployment correlates with the spike → **rollback** the deployment
- Rejection patterns were changed → revert the code change

**Escalate if:**
- Spike is caused by an active attack (prompt injection volume from external sources)
- Root cause is unclear after 15 minutes of investigation
- Multiple services are affected

**Mitigation actions:**

```bash
# Rollback to previous deployment
git revert HEAD
git push

# Or restart the service
docker compose restart agent-api

# Temporarily adjust traffic generator (if it's the source)
docker compose exec traffic-generator env REJECTION_MIX_RATIO=0.05
docker compose restart traffic-generator
```

## 4. Post-Incident

- [ ] Write timeline: when alert fired, when investigated, when resolved
- [ ] Identify root cause (deployment, config change, attack, bug)
- [ ] If false positive: adjust alert thresholds in `prometheus/alert-rules.yml`
- [ ] If real issue: add regression test to eval-runner datasets
- [ ] Update this runbook with lessons learned
