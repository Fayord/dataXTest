# System Architecture Overview

## Runtime Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          Docker Network: aiops-network                         │
│                                                                                │
│  ┌──────────────────────┐         ┌──────────────────────────────────────────┐  │
│  │  Traffic Generator   │  POST   │             Agent API (:8080)            │  │
│  │                      │ /ask    │                                          │  │
│  │  ~2 req/s            ├────────►│  Flask App (Python 3.11)                 │  │
│  │  85% normal          │         │                                          │  │
│  │  15% adversarial     │         │  ┌────────────────────────────────────┐  │  │
│  │                      │         │  │         Request Flow               │  │  │
│  └──────────────────────┘         │  │                                    │  │  │
│                                   │  │  POST /ask                         │  │  │
│                                   │  │    │                               │  │  │
│                                   │  │    ▼                               │  │  │
│                                   │  │  Parse JSON body                   │  │  │
│                                   │  │    │                               │  │  │
│                                   │  │    ▼                               │  │  │
│                                   │  │  classify_rejection(message)       │  │  │
│                                   │  │    │                               │  │  │
│                                   │  │    ├──► REJECTED ──► reason:       │  │  │
│                                   │  │    │    • prompt_injection          │  │  │
│                                   │  │    │    • secrets_request           │  │  │
│                                   │  │    │    • dangerous_action          │  │  │
│                                   │  │    │                               │  │  │
│                                   │  │    └──► ACCEPTED ──► response      │  │  │
│                                   │  │                                    │  │  │
│                                   │  └────────────────────────────────────┘  │  │
│                                   │                                          │  │
│                                   │  Endpoints:                              │  │
│                                   │    GET /healthz   ── health check        │  │
│                                   │    GET /metrics   ── prometheus scrape   │  │
│                                   └──────────┬───────────────────────────────┘  │
│                                              │                                  │
│                                              │ GET /metrics (every 5s)          │
│                                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                        Prometheus (:9090)                                 │  │
│  │                                                                           │  │
│  │  Scrapes ──► Stores ──► Evaluates Alert Rules                             │  │
│  │                                                                           │  │
│  │  Alert Rules:                                                             │  │
│  │    ⚠  AgentAPIDown          up{job="agent-api"} == 0        (1m, crit)   │  │
│  │    ⚠  HighRejectionRate     rejection_rate > 17%            (5m, warn)   │  │
│  │    ⚠  RejectionRateSpike    5m_rate > 2× 30m_baseline      (3m, warn)   │  │
│  │    ⚠  HighLatency           p95 > 1s                       (5m, warn)   │  │
│  │    ⚠  PromptInjectionSpike  injection_rate > 0.5/s          (3m, warn)   │  │
│  └──────────┬────────────────────────────────────────────────────────────────┘  │
│             │                                                                   │
│             │ Datasource (http://prometheus:9090)                               │
│             ▼                                                                   │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                        Grafana (:3000)                                    │  │
│  │                        admin/admin                                        │  │
│  │                                                                           │  │
│  │  Dashboard: "Agent API Monitoring" (8 panels)                             │  │
│  │  ┌─────────────────────────┐  ┌─────────────────────────────┐             │  │
│  │  │ Request Rate      (rps) │  │ Rejection Rate by Reason (%)│             │  │
│  │  └─────────────────────────┘  └─────────────────────────────┘             │  │
│  │  ┌─────────────────────────┐  ┌─────────────────────────────┐             │  │
│  │  │ Rejections by Reason    │  │ Request Latency (p50, p95)  │             │  │
│  │  └─────────────────────────┘  └─────────────────────────────┘             │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────┐ ┌────────────┐               │  │
│  │  │ Overall  │ │ Total    │ │ Total        │ │ API Status │               │  │
│  │  │ Rej Rate │ │ Requests │ │ Rejections   │ │  UP/DOWN   │               │  │
│  │  └──────────┘ └──────────┘ └──────────────┘ └────────────┘               │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │  Eval Runner (on-demand, profile: eval)                                   │  │
│  │                                                                           │  │
│  │  make eval ──► docker compose run --rm eval-runner                        │  │
│  │                                                                           │  │
│  │  Golden Dataset (20 msgs)      ──► expect ACCEPTED  ──► accuracy ≥ 90%   │  │
│  │  Adversarial Dataset (15 msgs) ──► expect REJECTED  ──► rej rate ≥ 60%   │  │
│  │                                                                           │  │
│  │  Exit 0 = PASS │ Exit 1 = FAIL  ──► used as CI quality gate              │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Metrics Flow

```
                          Agent API (app.py)
                                │
                  Emits 6 metric families on GET /metrics
                                │
          ┌─────────────┬───────┼───────┬──────────────┬──────────────┐
          ▼             ▼       ▼       ▼              ▼              ▼
   ┌────────────┐ ┌──────────┐ ┌──────────────┐ ┌──────────┐ ┌────────────┐
   │ requests   │ │rejections│ │ latency      │ │ errors   │ │ msg_length │
   │ _total     │ │ _total   │ │ _seconds     │ │ _total   │ │ _chars     │
   │            │ │          │ │              │ │          │ │            │
   │ Counter    │ │ Counter  │ │ Histogram    │ │ Counter  │ │ Histogram  │
   │            │ │          │ │              │ │          │ │            │
   │ Labels:    │ │ Labels:  │ │ Labels:      │ │ Labels:  │ │ Labels:    │
   │ •version   │ │ •version │ │ •version     │ │ •version │ │ •version   │
   │ •route     │ │ •reason  │ │ •route       │ │ •err_type│ │            │
   └─────┬──────┘ └────┬─────┘ └──────┬───────┘ └────┬─────┘ └─────┬──────┘
         │             │              │              │              │
         │    ┌────────┘              │              │              │
         │    │                       │              │              │
         ▼    ▼                       ▼              │              │
   ┌──────────────┐            ┌───────────┐         │              │
   │ Alert Rules  │            │ Dashboard │         │              │
   │              │            │ Panels    │         │              │
   │ HighRej...   │◄───────────│ 1-8       │         │              │
   │ Spike...     │            └───────────┘         │              │
   │ Latency...   │◄─────────────────┘               │              │
   │ Injection... │                                   │              │
   └──────────────┘                   ┌───── Future ──┘──────────────┘
                                      │  (available for dashboards
                                      │   & alerts if needed)
                                      └──────────────────────────────

   ┌──────────────────┐
   │ in_flight_requests│   Gauge — not used in alerts/dashboard yet,
   │                   │   but visible at /metrics for ad-hoc debugging
   └───────────────────┘
```

## CI/CD Pipeline Flow

```
   ┌──────────────────┐     ┌──────────────────┐
   │   Pull Request   │     │   Push to main    │
   │   to main        │     │                   │
   └────────┬─────────┘     └────────┬──────────┘
            │                        │
            ▼                        ▼
   ┌─────────────────────────────────────────────┐
   │           Job: build-and-test                │
   │                                              │
   │  1. Checkout code                            │
   │  2. docker compose build                     │
   │  3. docker compose up -d                     │
   │  4. Wait for /healthz (30 retries × 2s)      │
   │  5. docker compose run --rm eval-runner      │
   │     ├── Golden:      20/20 accepted? ────┐   │
   │     └── Adversarial: 15/15 rejected? ────┤   │
   │                                          │   │
   │     Exit 0 ──► ✅ PASS                    │   │
   │     Exit 1 ──► ❌ FAIL (blocks merge)     │   │
   │                                          │   │
   │  6. Upload eval-results/ as artifact     │   │
   │  7. docker compose down (always)         │   │
   └─────────────────────┬────────────────────┘   │
                         │                         │
              ┌──────────┘                         │
              │  (only on push to main)            │
              ▼                                    │
   ┌──────────────────────────────┐                │
   │       Job: deploy            │                │
   │                              │                │
   │  1. Checkout code            │                │
   │  2. Update manifest.yml:     │                │
   │     image_tag ← commit SHA   │                │
   │     history  ← timestamp     │                │
   │  3. git commit + push        │                │
   └──────────────────────────────┘

              deployment/manifest.yml
   ┌──────────────────────────────────────┐
   │  image_tag: "abc123def..."           │
   │                                      │
   │  # Deployment History:               │
   │  # | 2026-06-11 | abc123d | GH Act | │
   │  # | 2026-06-10 | 789def0 | GH Act | │
   └──────────────────────────────────────┘
```

## Incident Response Flow

```
   ⚠ Alert: HighRejectionRate / RejectionRateSpike
   │
   ▼
   ┌──────────────────────────────────────────┐
   │  1. TRIAGE (first 5 min)                 │
   │                                          │
   │  curl prometheus/api/v1/query            │
   │    → confirm rejection rate value        │
   │  curl /healthz                           │
   │    → confirm API is responding           │
   │                                          │
   │  15-25% → Low    (monitor)               │
   │  25-50% → Medium (investigate)           │
   │  >50%   → High   (immediate action)      │
   └────────────────┬─────────────────────────┘
                    │
                    ▼
   ┌──────────────────────────────────────────┐
   │  2. INVESTIGATE                          │
   │                                          │
   │  Query by reason:                        │
   │    prompt_injection ──► attack?          │
   │    secrets_request  ──► probing?         │
   │    dangerous_action ──► abuse?           │
   │                                          │
   │  Check:                                  │
   │    • Recent deployment (manifest.yml)    │
   │    • Prompt version change               │
   │    • Traffic generator config            │
   │    • Application logs                    │
   └────────────────┬─────────────────────────┘
                    │
          ┌─────────┴──────────┐
          ▼                    ▼
   ┌──────────────┐    ┌──────────────┐
   │  MITIGATE    │    │  ESCALATE    │
   │              │    │              │
   │  • Rollback  │    │  • Unknown   │
   │  • Restart   │    │    root cause│
   │  • Revert    │    │  • Multiple  │
   │    config    │    │    services  │
   │              │    │  • >15 min   │
   │  (false +ve  │    │    no answer │
   │   or known   │    │              │
   │   cause)     │    │              │
   └──────┬───────┘    └──────┬───────┘
          │                   │
          └─────────┬─────────┘
                    ▼
   ┌──────────────────────────────────────────┐
   │  4. POST-INCIDENT                       │
   │                                          │
   │  • Write timeline                        │
   │  • Root cause analysis                   │
   │  • Adjust thresholds if false positive   │
   │  • Add regression tests to eval-runner   │
   │  • Update runbook                        │
   └──────────────────────────────────────────┘
```

## File Map

```
code/
├── agent-api/
│   ├── app.py                 ◄── Task 3: added 4 metrics
│   ├── Dockerfile
│   └── requirements.txt
│
├── prometheus/
│   ├── prometheus.yml         ◄── scrapes agent-api every 5s
│   └── alert-rules.yml        ◄── Task 2: 5 alert rules
│
├── grafana/
│   ├── dashboards/
│   │   └── agent-monitoring.json  ◄── Task 4: fixed 3 broken panels
│   └── provisioning/
│       ├── dashboards/dashboards.yml
│       └── datasources/datasources.yml
│
├── .github/workflows/
│   └── ci.yml                 ◄── Task 1: CI/CD pipeline
│
├── deployment/
│   └── manifest.yml           ◄── updated by CD with commit SHA
│
├── docs/
│   ├── incident-response.md   ◄── Task 5: runbook
│   └── solution-overview.md   ◄── design decisions document
│
├── eval-runner/
│   ├── runner.py              ◄── quality gate (golden + adversarial)
│   ├── Dockerfile
│   └── requirements.txt
│
├── traffic-generator/
│   ├── generator.py           ◄── synthetic load (~2 req/s, 15% malicious)
│   ├── Dockerfile
│   └── requirements.txt
│
├── docker-compose.yml
└── Makefile                   ◄── make up / eval / down
```
