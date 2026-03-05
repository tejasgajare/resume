# Monitoring & Observability — RAVE Platform

## Overview

I built and maintained the monitoring and observability stack for RAVE, including Prometheus metrics design, Grafana dashboard modifications, and alert rule configuration. My key contribution was consolidating fragmented metrics into a unified `mc_event_case_action_total` counter with action labels, designing service health alerts that distinguish between degraded (non-running pods) and critical (zero replicas) states, and implementing processing drop detection for the wellness pipeline. This work spans GLCP-325062 (monitoring alerts) and GLCP-237935 (cloud alerts).

---

## Prometheus Metrics

### Core Metrics

#### `mc_event_case_action_total` (Counter)

The primary metric I consolidated. Previously, there were separate counters for each case action (create, update, close, escalate). I consolidated them into a single counter with action labels.

```go
var mcEventCaseActionTotal = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "mc_event_case_action_total",
        Help: "Total number of case actions processed by type",
    },
    []string{"action", "domain", "status", "device_type"},
)

// Usage examples:
mcEventCaseActionTotal.WithLabelValues("create", "email", "success", "nimble").Inc()
mcEventCaseActionTotal.WithLabelValues("create", "panhpe", "failure", "alletra").Inc()
mcEventCaseActionTotal.WithLabelValues("attach_file", "email", "success", "proliant").Inc()
mcEventCaseActionTotal.WithLabelValues("close", "panhpe", "success", "nimble").Inc()
```

**Why consolidation**: Before my change, there were 6+ separate counters:
- `mc_case_created_total`
- `mc_case_updated_total`
- `mc_case_closed_total`
- `mc_case_escalated_total`
- `mc_attachment_uploaded_total`
- `mc_email_sent_total`

This made dashboards complex and alert rules verbose. With one counter and labels, a single PromQL query can slice data any way:

```promql
# Total cases created per domain
sum(rate(mc_event_case_action_total{action="create"}[5m])) by (domain)

# Failure rate by device type
sum(rate(mc_event_case_action_total{status="failure"}[5m])) by (device_type)

# All actions for a specific domain
sum(rate(mc_event_case_action_total{domain="email"}[5m])) by (action)
```

**Labels design rationale**:
- `action`: What happened (create, update, close, escalate, attach_file, send_email)
- `domain`: email or panhpe — separates the two case creation paths
- `status`: success or failure — enables error rate alerting
- `device_type`: nimble, alletra, proliant, etc. — enables per-product monitoring

#### Service Health Metrics

```go
var serviceHealthGauge = prometheus.NewGaugeVec(
    prometheus.GaugeOpts{
        Name: "rave_service_health",
        Help: "Health status of RAVE services (1=healthy, 0=unhealthy)",
    },
    []string{"service", "namespace", "cluster"},
)

var servicePodCount = prometheus.NewGaugeVec(
    prometheus.GaugeOpts{
        Name: "rave_service_pod_count",
        Help: "Number of running pods per service",
    },
    []string{"service", "namespace", "status"},  // status: running, pending, failed
)
```

#### Wellness Pipeline Metrics

```go
var wellnessObjectsProcessed = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "rave_wellness_objects_processed_total",
        Help: "Total wellness objects processed through the pipeline",
    },
    []string{"stage", "result"},  // stage: received, validated, routed, persisted
)

var wellnessProcessingLatency = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "rave_wellness_processing_duration_seconds",
        Help:    "Time taken to process wellness objects",
        Buckets: prometheus.DefBuckets,
    },
    []string{"stage", "api_version"},
)
```

---

## Grafana Dashboards

### Classic Workflow Overview Dashboard

I modified the existing "classic-workflow-overview" dashboard to improve signal and reduce noise.

#### Key Modification: Removed platformFamily Filter

**Before**: Dashboard had a `platformFamily` dropdown filter that fragmented data across product lines. Operators had to check each platform family separately to get a complete picture.

**After**: I removed the `platformFamily` filter and replaced it with aggregate views that show all platform families together, with the ability to drill down via legend clicking.

```json
{
  "panels": [
    {
      "title": "Case Actions by Domain (All Platforms)",
      "targets": [
        {
          "expr": "sum(rate(mc_event_case_action_total{action=\"create\"}[5m])) by (domain)",
          "legendFormat": "{{domain}}"
        }
      ]
    },
    {
      "title": "Case Actions by Device Type",
      "targets": [
        {
          "expr": "sum(rate(mc_event_case_action_total{action=\"create\"}[5m])) by (device_type)",
          "legendFormat": "{{device_type}}"
        }
      ]
    }
  ]
}
```

**Why remove the filter**: Platform-agnostic metrics give a better overall health picture. If Nimble cases are failing while Alletra cases are fine, you want to see both on the same panel to compare. The old filter hid cross-platform issues.

### Dashboard Panels I Created/Modified

| Panel | Type | Purpose |
|-------|------|---------|
| Case Creation Rate | Time series | Rate of case creation across domains |
| Case Failure Rate | Time series | Error rate with threshold line |
| Service Pod Status | Stat | Current pod counts per service |
| Attachment Pipeline | Time series | File upload stages and success rates |
| API Version Traffic | Pie chart | Request distribution across v1/v2/v3 |
| Processing Latency | Heatmap | p50/p95/p99 processing time |
| Kafka Consumer Lag | Time series | Consumer group lag per topic |

---

## Alert Categories

### Service Health Alerts (GLCP-325062)

#### Warning: Non-Running Pods

```yaml
groups:
  - name: rave-service-health
    rules:
      - alert: RaveServicePodsNotRunning
        expr: |
          rave_service_pod_count{status!="running"} > 0
        for: 5m
        labels:
          severity: warning
          team: rave
        annotations:
          summary: "{{ $labels.service }} has {{ $value }} non-running pods"
          description: |
            Service {{ $labels.service }} in {{ $labels.namespace }}/{{ $labels.cluster }}
            has {{ $value }} pods in {{ $labels.status }} state for more than 5 minutes.
            This may indicate deployment issues or resource constraints.
          runbook: "https://confluence.hpe.com/rave/runbook/pod-issues"
```

**Why 5-minute threshold**: Pods frequently enter non-running states briefly during deployments (rolling updates). A 5-minute `for` clause prevents alert fatigue during normal deployments while still catching stuck pods.

#### Critical: Zero Replicas

```yaml
      - alert: RaveServiceZeroReplicas
        expr: |
          sum(rave_service_pod_count{status="running"}) by (service, namespace) == 0
        for: 2m
        labels:
          severity: critical
          team: rave
        annotations:
          summary: "{{ $labels.service }} has ZERO running pods"
          description: |
            Service {{ $labels.service }} has no running pods in {{ $labels.namespace }}.
            This is a service outage. Immediate investigation required.
          runbook: "https://confluence.hpe.com/rave/runbook/service-outage"
```

**Why 2 minutes for critical**: Zero replicas means complete service outage. We tolerate 2 minutes for Kubernetes to recover (restart crashed pods, complete deployments) but alert quickly after that.

#### Severity Design Rationale

| Condition | Severity | Why |
|-----------|----------|-----|
| Some pods not running | Warning | Service degraded but still operational |
| Zero running pods | Critical | Complete service outage |
| High error rate (>5%) | Warning | Cases failing but not all |
| Error rate 100% | Critical | No cases processing |

### Processing Drop Detection

```yaml
      - alert: RaveProcessingDropDetected
        expr: |
          (
            sum(rate(rave_wellness_objects_processed_total{stage="received"}[10m]))
            -
            sum(rate(rave_wellness_objects_processed_total{stage="persisted"}[10m]))
          ) / sum(rate(rave_wellness_objects_processed_total{stage="received"}[10m]))
          > 0.1
        for: 10m
        labels:
          severity: warning
          team: rave
        annotations:
          summary: "More than 10% of wellness objects dropping between receive and persist"
          description: |
            {{ $value | humanizePercentage }} of wellness objects are being received 
            but not persisted. This indicates validation failures, MongoDB issues, 
            or service errors in the processing pipeline.
```

**Why received vs. persisted**: This captures drops at any stage in the pipeline. Objects are counted when received (before validation) and when persisted (after all processing). The difference catches validation rejections, serialization errors, and database failures.

### Kafka Consumer Lag Alerts

```yaml
      - alert: RaveKafkaConsumerLag
        expr: |
          kafka_consumer_group_lag{group=~"rave-.*"} > 1000
        for: 15m
        labels:
          severity: warning
          team: rave
        annotations:
          summary: "Kafka consumer lag > 1000 for {{ $labels.group }}"
          description: |
            Consumer group {{ $labels.group }} on topic {{ $labels.topic }}
            has a lag of {{ $value }} messages. This indicates the consumer
            is falling behind, possibly due to slow processing or increased input rate.
```

---

## Humio/LogScale Integration

### Structured Logging

All RAVE services use structured JSON logging that feeds into Humio/LogScale:

```go
type LogEntry struct {
    Timestamp    string `json:"timestamp"`
    Level        string `json:"level"`
    Service      string `json:"service"`
    RequestID    string `json:"requestId"`
    WorkspaceID  string `json:"workspaceId"`
    Action       string `json:"action"`
    Message      string `json:"message"`
    Error        string `json:"error,omitempty"`
    DurationMS   int64  `json:"durationMs,omitempty"`
    APIVersion   string `json:"apiVersion,omitempty"`
}
```

### Key Humio Queries

```
// Find all failed case creations in the last hour
service=wellnessProducer action=createCase level=error
| timechart(count(), span=5m)

// Trace a specific request across services
requestId="req-uuid-123"
| sort(timestamp)

// Cross-workspace access attempts (security)
message="workspace ID mismatch"
| groupBy(workspaceId)
```

### Log-to-Metric Bridge

Some metrics are derived from logs rather than direct instrumentation:

- **Security events**: Cross-workspace attempts, invalid token patterns
- **Business metrics**: Case categories, device types (for reporting dashboards)
- **Error classification**: Exception types, downstream API failures

---

## Metric Consolidation Rationale

### Before Consolidation

```
mc_case_created_total{domain="email"} 1234
mc_case_created_total{domain="panhpe"} 567
mc_case_updated_total{domain="email"} 890
mc_case_updated_total{domain="panhpe"} 345
mc_case_closed_total{domain="email"} 678
mc_case_closed_total{domain="panhpe"} 234
mc_attachment_uploaded_total{} 456
mc_email_sent_total{} 789
```

**Problems**:
1. 8+ separate metrics = 8+ separate Grafana panels
2. Adding a new action means a new metric, new panel, new alert rule
3. Cross-action analysis requires joining multiple metrics in PromQL
4. Inconsistent label sets across metrics

### After Consolidation

```
mc_event_case_action_total{action="create", domain="email", status="success", device_type="nimble"} 1234
mc_event_case_action_total{action="create", domain="panhpe", status="success", device_type="alletra"} 567
mc_event_case_action_total{action="update", domain="email", status="success", device_type="nimble"} 890
mc_event_case_action_total{action="close", domain="panhpe", status="failure", device_type="proliant"} 234
```

**Benefits**:
1. Single metric, slice any way with labels
2. Adding a new action = new label value, no code change in metrics
3. Unified dashboards with flexible filtering
4. Consistent label set enables cross-action correlation

---

## Grafana Alerting vs. Prometheus Alerting

We use **Prometheus alerting** (not Grafana alerting) because:

1. **Reliability**: Prometheus alerting evaluates rules even if Grafana is down
2. **GitOps**: Alert rules stored in YAML files, deployed via Helm/ArgoCD
3. **Alertmanager integration**: Routing, silencing, grouping handled by Alertmanager
4. **Consistency**: Same rules evaluated in all environments (dev/intg/prod values may differ)

Grafana dashboards are for **visualization only** — no alerting from Grafana panels.

---

## Related Jira Issues

| Issue | Description | My Contribution |
|-------|-------------|-----------------|
| GLCP-325062 | Monitoring alerts setup | Service health alerts, processing drops |
| GLCP-237935 | Cloud alerts integration | Prometheus + Grafana pipeline |

---

## [KEY_POINTS]

- Consolidated 8+ fragmented metrics into single `mc_event_case_action_total` with labels
- Two-tier service health alerts: warning (degraded pods) vs. critical (zero replicas)
- Processing drop detection compares received vs. persisted rates
- Removed platformFamily filter from Grafana for aggregate visibility
- Prometheus alerting (not Grafana) for reliability and GitOps compatibility

## [COMMON_MISTAKES]

- Using Grafana for alerting instead of Prometheus — Grafana is visualization only
- Alerting on instant values instead of rates — spikes cause false alarms, use `rate()` over windows
- Setting alert thresholds too tight — 5-min `for` on warnings prevents deployment noise
- Creating separate metrics per action — use labels for cardinality, not metric names

## [FOLLOW_UP]

- "Why consolidate metrics?" → Fewer metrics, flexible querying, easier maintenance
- "How do you prevent alert fatigue?" → Severity tiers, `for` clauses, Alertmanager routing/silencing
- "What monitoring tells you a deployment went wrong?" → Zero replicas alert + error rate spike
- "How do you trace a request across services?" → requestId in structured logs, queried in Humio
