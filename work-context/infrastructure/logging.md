# Logging & Observability

> **Tags**: `#interview-ready` `#logging` `#observability` `#humio` `#logscale` `#new-relic` `#prometheus` `#monitoring`
> **Role**: Software Engineer — RAVE Platform, HPE GreenLake
> **Cross-references**: [kubernetes-helm.md](./kubernetes-helm.md), [cicd-pipelines.md](./cicd-pipelines.md), [aws-terraform.md](./aws-terraform.md)

---

## Overview

I manage the logging and observability stack for the RAVE platform, which encompasses Humio/LogScale for log aggregation, Prometheus for metrics collection, PagerDuty for alerting, and New Relic for APM. My responsibilities include configuring Humio access groups and roles, defining Prometheus alerting rules, setting up Istio sidecar metrics scraping, and ensuring all 20+ RAVE microservices have consistent observability coverage. I also contribute to the cross-team `ccp-humio-resources-config` repo that manages Humio resources as code.

---

## Humio / LogScale

### What is Humio/LogScale?

CrowdStrike's Humio (now branded as LogScale) is our centralized log aggregation platform. It ingests logs from all EKS clusters across HPE GreenLake and provides real-time search, dashboards, and alerting. For RAVE, Humio is the primary tool for debugging production issues, tracing request flows, and monitoring service health.

### Access Management via ccp-humio-resources-config

I contribute to the `ccp-humio-resources-config` repository, which manages Humio resources as GitOps-managed code. The repo structure:

```
ccp-humio-resources-config/
├── clusters/                        # Humio cluster configurations
├── resources/
│   ├── groups/
│   │   ├── rave.yaml               # RAVE team access group
│   │   ├── ccs.yaml                # CCS team access
│   │   ├── sre.yaml                # SRE team access
│   │   ├── compute.yaml            # Compute team
│   │   ├── storage.yaml            # Storage team
│   │   ├── security.yaml           # Security team
│   │   └── ...                     # Other teams
│   ├── roles/
│   │   └── global.yaml             # Global role definitions
│   └── scheduled_searches/
│       ├── storage.yaml            # Storage team scheduled searches
│       └── compute.yaml            # Compute team scheduled searches
├── utils/                           # Utility scripts
├── SELF-SERVICE.md                  # Self-service documentation
└── README.md
```

### RAVE Humio Group

The `rave.yaml` group definition manages who has access to RAVE log repositories:

```yaml
# (c) Copyright 2025 Hewlett Packard Enterprise Development LP

groups:
  - name: rave-users
    users:
      - jianrobert.qiu@hpe.com
      - ramesh.sunkara@hpe.com
      - sreeja.ravikumar@hpe.com
      - tejas.gajare@hpe.com
      - vijaykumarg@hpe.com
      - sagar.kale@hpe.com
      - matthew.hiles@hpe.com
      - sai-sumanth.dudyala@hpe.com
      - peng.chen@hpe.com
      - ravi.elumalai@hpe.com
      - corey.newport@hpe.com
      - mariyappan.santhanavelu@hpe.com
      - joshua.freeman@hpe.com
      - christopher.hubbard@hpe.com
      - randy.gilbert@hpe.com
      - daniel.knowles@hpe.com
      - christopher.milham@hpe.com
      - harold.shaver@hpe.com
      - brandon.yates@hpe.com
```

When a new team member joins RAVE, I add their email to this group file, open a PR, and once merged, a pipeline applies the change to Humio. This is our GitOps approach to access management — no manual Humio admin clicks, full audit trail in Git.

### Humio Log Repositories

RAVE logs are organized into Humio repositories by environment:
- `rave-dev` — Development environment logs
- `rave-stage` — Staging environment logs
- `rave-prod` — Production environment logs

Each repository receives logs from the corresponding EKS cluster. Logs are shipped via Fluent Bit (managed by the CCP team) running as DaemonSets on cluster nodes.

### Log Format & Structured Logging

RAVE services emit structured JSON logs:
```json
{
  "timestamp": "2026-01-15T10:30:00Z",
  "level": "INFO",
  "service": "crmrelay",
  "message": "Case relay processed successfully",
  "caseId": "CASE-12345",
  "correlationId": "abc-def-123",
  "duration_ms": 145,
  "environment": "prod"
}
```

I enforce structured logging standards across the team:
- **JSON format**: Machine-parseable, Humio-friendly
- **Correlation IDs**: Every request carries a correlation ID for distributed tracing
- **Service name**: Identifies which microservice emitted the log
- **Duration**: For performance tracking and P95/P99 calculations
- **Log level**: Configurable per environment (`debug` in dev, `info` in prod — though currently we use `debug` everywhere, which is something I'd change)

### Humio Query Patterns I Use

**Service error investigation:**
```
#repo=rave-prod | service=crmrelay | level=ERROR | groupby(message, function=[count()])
```

**Request latency analysis:**
```
#repo=rave-prod | service=wellnessproducer | duration_ms > 1000 | timechart(avg(duration_ms))
```

**Distributed trace:**
```
#repo=rave-prod | correlationId="abc-def-123" | sort(timestamp)
```

**Pod restart investigation:**
```
#repo=rave-prod | kubernetes.container_name=caserelay | "OOMKilled" OR "CrashLoopBackOff"
```

---

## Prometheus & Metrics

### Prometheus Architecture

RAVE services expose metrics on port 9070 (configurable via `global.monitoring.port`):

```yaml
# Helm values
global:
  monitoring:
    port: 9070
    enabled: "true"
```

Prometheus scrapes these endpoints via ServiceMonitor CRDs managed in the `kustomize-prometheus` repo.

### Istio Sidecar Metrics

I configured a PodMonitor to scrape Envoy sidecar metrics from all RAVE pods:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: istio-sidecar-metrics
  namespace: rave
  labels:
    release: tenant-monitoring
spec:
  selector:
    matchLabels:
      security.istio.io/tlsMode: istio
  podMetricsEndpoints:
    - port: 15090              # Envoy stats port
      path: /stats/prometheus
      interval: 15s
      relabelings:
        - action: keep
          sourceLabels: [__meta_kubernetes_pod_container_name]
          regex: "istio-proxy"
```

This gives us Envoy-level metrics without any application code changes:
- **istio_requests_total**: Request count by service, response code, and source/destination
- **istio_request_duration_milliseconds**: Request latency histograms
- **istio_tcp_connections_opened_total**: TCP connection tracking (useful for MongoDB monitoring)
- **envoy_cluster_upstream_cx_active**: Active connections per upstream cluster

### PrometheusRule Alerts

I define alerting rules for RAVE services:

**wellnessproducer alerts** (`wellnessproducer-prometheusrule.yaml`):
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: wellnessproducer-alerts
  namespace: rave
spec:
  groups:
    - name: wellnessproducer.rules
      rules:
        - alert: WellnessProducerDown
          expr: up{app="wellnessproducer"} == 0
          for: 5m
          labels:
            severity: critical
            team: rave
          annotations:
            summary: "wellnessproducer is down"

        - alert: WellnessProducerHighErrorRate
          expr: |
            rate(http_requests_total{app="wellnessproducer",status=~"5.."}[5m])
            / rate(http_requests_total{app="wellnessproducer"}[5m]) > 0.05
          for: 10m
          labels:
            severity: warning
            team: rave
          annotations:
            summary: "wellnessproducer error rate > 5%"
```

**comenricher alerts** (`comenricher-prometheusrule.yaml`):
- High processing latency alerts
- Queue backlog alerts
- OOM risk alerts (memory approaching limits)

### AlertManagerConfig

Production alerts route to PagerDuty via the `pagerduty-secret`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pagerduty-secret
  namespace: rave
type: Opaque
stringData:
  routingKey: <pagerduty-routing-key>
```

The AlertManager configuration routes `severity: critical` alerts to PagerDuty for on-call notification, while `severity: warning` alerts go to a Slack channel for awareness.

---

## New Relic APM

### Integration

RAVE services are instrumented with New Relic APM agents (Go agent for Go services). New Relic provides:
- **Transaction traces**: End-to-end request traces showing time spent in each function
- **Error tracking**: Automatic error capture with stack traces
- **Database monitoring**: MongoDB query performance and slow query detection
- **External service calls**: Latency to upstream services (Harmony, Consul)

### How I Use New Relic

I use New Relic primarily for:
1. **Performance profiling**: Identifying slow database queries or inefficient code paths
2. **Error triage**: Correlating error spikes with deployments or infrastructure events
3. **Capacity planning**: Understanding baseline resource usage to set appropriate Kubernetes resource limits
4. **Cross-service tracing**: Following a request from the API gateway through multiple RAVE services to the database

### New Relic vs. Humio

| Aspect | Humio/LogScale | New Relic |
|--------|---------------|-----------|
| Primary use | Log aggregation & search | APM & performance monitoring |
| Data type | Structured log lines | Metrics, traces, events |
| Strength | Free-text search, real-time | Automatic instrumentation, dashboards |
| Cost model | Volume-based (ingest GB/day) | Per-host licensing |
| RAVE usage | Debugging, incident investigation | Performance optimization, alerting |

I use both tools together: New Relic for identifying *what* is slow, Humio for investigating *why* by searching the detailed logs.

---

## Observability Stack Integration

### End-to-End Monitoring Flow

```
RAVE Service
  │
  ├─→ Structured JSON logs → Fluent Bit → Humio/LogScale
  │     (debug, info, warn, error)
  │
  ├─→ Application metrics (port 9070) → Prometheus → Grafana dashboards
  │     (request count, latency, custom business metrics)
  │
  ├─→ Istio sidecar metrics (port 15090) → Prometheus → Grafana
  │     (mesh traffic, connection pools, TLS stats)
  │
  ├─→ New Relic APM agent → New Relic cloud
  │     (transactions, errors, database, external calls)
  │
  └─→ PrometheusRule alerts → AlertManager → PagerDuty / Slack
        (critical: page on-call; warning: Slack notification)
```

### Debugging a Production Incident — My Workflow

1. **Alert fires** (PagerDuty page or Slack notification)
2. **Check New Relic** — Is it a specific transaction failing? High error rate? Elevated latency?
3. **Check Prometheus/Grafana** — Pod restarts? Memory pressure? CPU throttling?
4. **Check Humio** — Search for error-level logs in the affected service, filter by time window
5. **Trace with correlation ID** — If available, search Humio for the correlation ID to see the full request path
6. **Check Istio metrics** — Connection pool exhaustion? TLS handshake failures? Upstream 503s?
7. **Check Kubernetes events** — `kubectl get events -n rave --sort-by=.lastTimestamp` for OOM kills, evictions, scheduling failures
8. **Mitigate** — Scale up replicas, restart pods, or rollback deployment depending on root cause

---

## Scheduled Searches & Dashboards

### Humio Scheduled Searches

The `ccp-humio-resources-config` repo supports scheduled searches that run periodically and trigger alerts. While I haven't created RAVE-specific scheduled searches yet, the pattern is:

```yaml
# resources/scheduled_searches/rave.yaml (future)
scheduled_searches:
  - name: rave-high-error-rate
    query: '#repo=rave-prod | level=ERROR | count() > 100'
    schedule: '*/15 * * * *'    # Every 15 minutes
    action:
      type: email
      recipients:
        - infosight-wellness-automation@hpe.com
```

### Grafana Dashboards

I contribute to Grafana dashboards that visualize RAVE metrics:
- **Service Overview**: Request rate, error rate, and P95 latency for all RAVE services
- **MongoDB Performance**: Connection pool usage, query latency, operation counts
- **Resource Utilization**: CPU and memory for each deployment
- **Istio Mesh**: Service-to-service traffic flow, error rates by source/destination

---

## Interview Deep-Dives

### Q: How do you approach observability for a microservices platform?

I implement three pillars: logs (Humio/LogScale), metrics (Prometheus), and traces (New Relic APM). Structured JSON logging with correlation IDs enables cross-service request tracing. Prometheus scrapes both application metrics (business logic) and Istio sidecar metrics (network layer). New Relic provides automatic transaction tracing without code changes. AlertManager routes critical alerts to PagerDuty for the on-call engineer. The key principle is: metrics tell you *what* is broken, traces tell you *where* it's breaking, and logs tell you *why*.

### Q: How do you manage logging access at scale?

We use GitOps for Humio access management. The `ccp-humio-resources-config` repo defines team groups in YAML files. When I need to add or remove a team member's Humio access, I update the RAVE group file and open a PR. Once merged, a pipeline syncs the change to Humio. This provides a full audit trail, prevents unauthorized access, and scales across all HPE GreenLake teams without manual admin intervention.

### Q: Walk me through debugging a MongoDB connectivity issue.

First, I check the Istio sidecar metrics in Prometheus: `envoy_cluster_upstream_cx_connect_fail` for the MongoDB PrivateLink hosts. If connections are failing, I check the DestinationRule TCP keepalive settings—we've had issues where idle connections were being dropped by AWS NAT Gateways. Then I search Humio for the service logs: `#repo=rave-prod | service=crmrelay | "mongo" | level=ERROR`. I look for specific error types: connection timeouts (network issue), auth failures (credential rotation), or topology changes (MongoDB Atlas maintenance). If it's a PrivateLink DNS issue, I check the ServiceEntry configuration in the Istio mesh.

### Q: What log level do you run in production?

Currently, we run `debug` in all environments (`global.logLevel: "debug"` in values files). Honestly, this is something I'd change — debug-level logging in prod generates excessive volume and increases Humio costs. I'd switch prod to `info` level and use dynamic log level switching (via Consul config or environment variable) for targeted debugging when investigating issues. The challenge is convincing the team that we can effectively debug with `info` logs — it requires ensuring our `info` logs are comprehensive enough for common scenarios.

### Q: How would you improve the observability stack?

Three improvements: (1) Implement distributed tracing with OpenTelemetry instead of relying on New Relic's proprietary agent — this gives us vendor flexibility and richer trace context. (2) Add Humio scheduled searches for automated anomaly detection (error rate spikes, latency outliers). (3) Create SLO (Service Level Objective) dashboards that track error budgets — currently, we monitor metrics but don't have formal SLO definitions for RAVE services. An error budget approach would help us make data-driven decisions about when to prioritize reliability vs. new features.

---

## Key Repos

| Repo | Purpose | My Role |
|------|---------|---------|
| `ccp-humio-resources-config` | Humio access/roles as code | RAVE group management, user access |
| `rave-cloud` | Helm chart monitoring templates | PrometheusRules, PodMonitors, alerting |
| `kustomize-prometheus` | Prometheus configs | RAVE ServiceMonitors, alert rules |
| `mcd-deploy-proj-rave` | Monitoring-related Helm values | Metrics ports, log levels |

---

## Metrics & Impact

- **20+ services** with consistent structured logging to Humio
- **3 Humio repositories** (dev/stage/prod) with team-scoped access
- **Istio sidecar metrics** scraped via PodMonitor for network-layer observability
- **PagerDuty integration** for production critical alerts
- **19 team members** managed via GitOps access control in Humio
- **New Relic APM** for transaction tracing and performance profiling
- **Sub-5-minute** mean time to detect production issues via alerting stack
