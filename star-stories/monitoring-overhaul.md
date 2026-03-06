# Prometheus/Grafana Monitoring Overhaul for RAVE Platform

## STAR Narrative

### Situation
The RAVE Cloud Platform — HPE GreenLake's case management system — had minimal observability. There were scattered metrics with inconsistent naming, no alerting for service health or processing drops, and multiple separate Prometheus counters for related actions (e.g., `MetricEventCaseCreated`, `MetricEventCaseUpdated` as independent metrics). The Grafana dashboards were ad-hoc, and a legacy `platformFamily` filter was included in queries that no longer reflected the system's architecture. When case processing rates dropped or a service stopped running, the team wouldn't know until customers reported issues. Sprint reviews required manually checking logs to understand system throughput.

### Task
I was tasked with building a comprehensive monitoring and alerting system from scratch:
1. Consolidate fragmented metrics into a unified naming scheme.
2. Create service health alerts to detect pods not running or zero replicas.
3. Add processing drop detection to alert when case creation/automation rates fell below thresholds.
4. Remove the obsolete `platformFamily` filter from all queries.
5. Design Grafana dashboards for real-time operational visibility.
6. Establish alert categories with appropriate severity levels (warning vs. critical).

### Action
**Step 1 — Metric Consolidation:**
I replaced the separate `MetricEventCaseCreated` and `MetricEventCaseUpdated` counters with a single unified counter: `mc_event_case_action_total`. This counter uses an `action` label with values `created`, `updated`, `reopened`, and `new_open`. This dramatically simplified PromQL queries — instead of summing across multiple metric names, dashboards could use:
```promql
sum(rate(mc_event_case_action_total{action="created"}[5m]))
```
The consolidation followed Prometheus naming conventions: `<namespace>_<subsystem>_<name>_<unit>` with `_total` suffix for counters.

**Step 2 — Service Health Alerts:**
I created two tiers of service health alerts:
- **Warning**: Detect non-running pods — `kube_pod_status_phase{phase!="Running"}` for RAVE service pods indicates a pod is in CrashLoopBackOff, Pending, or Error state.
- **Critical**: Zero replicas — `kube_deployment_status_replicas_available == 0` means the service is completely down.

Each alert included annotations with a runbook link, affected service name, and current state.

**Step 3 — Processing Drop Detection:**
I designed rate-based alerts that detect when case processing drops below historical baselines:
- Used `rate()` over a 15-minute window to smooth spikes.
- Set thresholds based on 2 weeks of historical data — a drop below the p10 (10th percentile) rate triggers a warning; a drop to zero triggers critical.
- Alerts distinguished between "low processing" (possible upstream issue) and "zero processing" (service definitely broken).

**Step 4 — platformFamily Filter Removal:**
The `platformFamily` label was a legacy artifact from when the platform supported multiple product families. After architecture changes, all cases flowed through a single path. I audited every PromQL query in Grafana dashboards and alert rules, removed the filter, and verified query results were identical before and after removal.

**Step 5 — Grafana Dashboard Design:**
I built operational dashboards with:
- **Overview panel**: Case creation rate, update rate, reopen rate (all from `mc_event_case_action_total`).
- **Service health panel**: Pod status, restart counts, memory/CPU utilization.
- **Processing pipeline panel**: End-to-end latency from event ingestion to case creation.
- **Alert status panel**: Current firing alerts with severity and duration.

I also conceptualized an AI-powered Grafana panel concept that would accept natural language prompts to answer questions about RAVE monitoring data — though this remained a design concept.

**Step 6 — Alert Categories and Routing:**
I organized alerts into two categories:
1. **Service stopped running** — pod health, replica counts.
2. **Processing drops** — rate-based anomaly detection on case creation/automation.

Each category had warning and critical severity levels with different notification channels (Slack for warning, PagerDuty for critical).

### Result
- **Unified metrics** reduced the number of distinct metric names by 60% while providing richer data through labels.
- **Service health alerts** caught 3 pod crashloop incidents within the first week that would have gone unnoticed.
- **Processing drop alerts** detected an upstream Kafka partition rebalance issue that reduced throughput by 40% — team was notified within 15 minutes instead of hours.
- **Grafana dashboards** became the go-to for sprint reviews, replacing manual log analysis.
- **platformFamily removal** simplified all queries and eliminated a confusion point for new team members.
- Alerting infrastructure now covers all RAVE microservices with consistent naming and severity conventions.

## Interview Delivery Tips

### How to Open This Story
"I built the observability stack for our case management platform from the ground up — consolidated fragmented metrics into a unified Prometheus naming scheme, designed Grafana dashboards, and created two categories of alerts: service health and processing drop detection. Before this, the team wouldn't know about outages until customers reported them."

### Time Budget (5-minute answer)
- Situation: 45 seconds (no alerting, inconsistent metrics, manual sprint reviews)
- Task: 30 seconds (6 responsibilities)
- Action: 2.5 minutes (focus on metric consolidation + alert design)
- Result: 1 minute (caught 3 incidents first week, 40% throughput drop detected in 15 min)

### Pivot Points
- **If interviewer asks about data**: Focus on Prometheus data model, PromQL, cardinality
- **If interviewer asks about operations**: Focus on alert fatigue, severity tiers, runbooks
- **If interviewer asks about dashboards**: Discuss panel design, user stories, the AI concept
- **If interviewer asks about scale**: Discuss recording rules, Mimir, long-term storage

---

## Rubric Annotations

### Key Phrases/Concepts That MUST Appear
- `mc_event_case_action_total` with `action` label (unified counter)
- Prometheus counter naming convention (`_total` suffix)
- `rate()` function for counter-based alerting
- Two alert categories: service health + processing drops
- Warning vs. critical severity tiers
- platformFamily filter removal rationale
- Defense-in-depth monitoring (pod health + business metrics)

### Common Mistakes
- Describing dashboards without explaining the underlying data model
- Using `increase()` instead of `rate()` for alerting (less smooth, harder to threshold)
- Not explaining why metric consolidation matters (query complexity, cardinality)
- Ignoring label cardinality concerns when adding dimensions
- Describing alerts without severity levels or escalation paths

### Scoring Rubric [1-5]
- 1 — Poor: Cannot explain Prometheus data model or why metrics were consolidated
- 2 — Below Average: Describes dashboards but lacks alerting strategy; no mention of rate-based detection
- 3 — Acceptable: Covers metric consolidation and basic alerts but light on severity tiers and processing drop detection
- 4 — Strong: Full narrative with alert categories, rate-based detection, and Grafana design rationale
- 5 — Exceptional: All of the above plus discusses cardinality trade-offs, alert fatigue mitigation, and operational impact

---

## Follow-Up Questions (Progressively Harder)

### Follow-Up 1: Why use labels on a single counter instead of separate metric names? What are the trade-offs?
**Expected Answer**: Labels provide dimensional data — you can `sum by (action)` to get per-action rates or drop the label to get totals. Separate metric names require listing all names in queries and are harder to extend. The trade-off is label cardinality: each unique label combination creates a new time series. With low-cardinality labels like `action` (4 values), this is fine. High-cardinality labels (e.g., user IDs) would explode storage and query time.
**Red Flags**: Doesn't mention cardinality; says "labels are always better than separate metrics."
**Green Flags**: Articulates the cardinality trade-off, mentions time series explosion risk.

### Follow-Up 2: How would you detect a gradual degradation vs. a sudden outage in your alerting system?
**Expected Answer**: Sudden outage: `rate() == 0` over a short window (5m). Gradual degradation: compare current `rate()` against a historical baseline using `predict_linear()` or a recording rule that captures the rolling 7-day average. Alert when current rate drops below a percentage of the baseline (e.g., 50% of 7-day average). You can also use Prometheus's `deriv()` function to detect sustained negative trends.
**Red Flags**: Only describes threshold-based alerting; doesn't know `predict_linear()` or trend detection.
**Green Flags**: Mentions recording rules for baselines, `predict_linear()`, and distinguishes between absolute thresholds and relative thresholds.

### Follow-Up 3: How do you avoid alert fatigue? What strategies did you use?
**Expected Answer**: (1) Severity tiers — warning for "investigate soon," critical for "act now." (2) Grouping — Alertmanager groups related alerts into single notifications. (3) Inhibition — if the entire cluster is down, suppress individual service alerts. (4) Silence windows for known maintenance. (5) Tuning thresholds iteratively based on false positive rates. (6) `for` duration in alert rules — require the condition to persist for N minutes before firing, filtering transient spikes.
**Red Flags**: "We just send all alerts to Slack."
**Green Flags**: Mentions `for` duration, inhibition rules, and iterative threshold tuning.

### Follow-Up 4: How did removing the platformFamily filter affect existing dashboards and alerts?
**Expected Answer**: I audited every PromQL query in Grafana (dashboards and alert rules) that referenced the `platformFamily` label. For each: (1) Ran the query with and without the filter, comparing results. (2) Since all traffic now flows through a single platform path, the filter was a no-op (it matched all data). (3) Removing it simplified queries but didn't change results — verified with `diff` on query outputs. (4) Updated alert thresholds after removal because the aggregated data might show different rates. (5) Committed changes to the Grafana-as-code repository (dashboards stored as JSON in Git). This cleanup reduced cognitive load for anyone reading the queries.
**Red Flags**: "We just removed it everywhere without checking."
**Green Flags**: Describes the audit process, verification via query comparison, and threshold re-evaluation.

---

## Grilling Chain (7 questions drilling deeper)

### Q1: Explain the Prometheus data model. What is a time series?
**Why asked**: Tests foundational understanding of the monitoring system.
**Expected answer**: A time series is a stream of timestamped values identified by a metric name and a set of key-value label pairs. For example, `mc_event_case_action_total{action="created", service="wellnessProducer"}` is one time series. Each unique combination of metric name + labels = one time series. Prometheus stores these as append-only sequences of (timestamp, float64) samples, scraped at regular intervals.
**Red flags**: Confuses metrics with logs; doesn't understand that labels create distinct time series.
**Green flags**: Explains label-based identification, mentions scrape intervals and append-only storage.

### Q2: What's the difference between `rate()` and `increase()` in PromQL? When would you use each?
**Why asked**: Tests PromQL fluency — a common source of subtle bugs.
**Expected answer**: `rate()` computes the per-second average rate of increase over the window. `increase()` computes the total increase over the window. `rate()` is better for alerting and graphing because it's normalized (per-second), making it comparable across different window sizes. `increase()` is useful for "how many events happened in the last hour" type queries. Both handle counter resets correctly. A gotcha: `rate()` requires at least two samples in the window to compute a rate.
**Red flags**: Says they're interchangeable; doesn't mention counter reset handling.
**Green flags**: Explains per-second normalization, mentions minimum two-sample requirement.

### Q3: How does Prometheus scraping work? What happens if a target is temporarily unreachable?
**Why asked**: Tests understanding of data collection reliability.
**Expected answer**: Prometheus pulls metrics from targets at a configured `scrape_interval` (e.g., 15s). If a target is unreachable, the scrape fails — no data point is recorded for that interval (it's a gap, not a zero). Prometheus exposes `up` metric (0 or 1) per target to detect scrape failures. After the target recovers, scraping resumes normally. Counter-based metrics handle gaps correctly because `rate()` only considers present samples. The gap itself is a signal — you can alert on `up == 0`.
**Red flags**: Says Prometheus pushes metrics to targets; says a gap means the metric is zero.
**Green flags**: Explains pull model, `up` metric, gap handling in `rate()`.

### Q4: Your alert uses a 15-minute window for rate calculation. How did you choose this window size?
**Why asked**: Tests practical tuning experience.
**Expected answer**: The window must be at least 4x the scrape interval (if scraping every 15s, minimum window is 60s) to guarantee at least two samples for `rate()`. I chose 15 minutes to smooth out natural traffic fluctuations — case creation is bursty (business hours vs. nights). A shorter window (e.g., 5m) would cause false positives during low-traffic periods. A longer window (e.g., 1h) would delay detection. 15 minutes balanced sensitivity and stability. I also used the `for` clause (e.g., `for: 10m`) so the alert only fires if the condition persists.
**Red flags**: Arbitrary choice with no reasoning; doesn't know about the scrape interval relationship.
**Green flags**: Explains 4x scrape interval rule, balances sensitivity vs. false positives, mentions `for` clause.

### Q5: What are recording rules and when would you use them?
**Why asked**: Tests knowledge of Prometheus optimization.
**Expected answer**: Recording rules pre-compute frequently used or expensive PromQL expressions and store the results as new time series. Use them for: (1) Dashboard queries that run on every page load — pre-compute to reduce query latency. (2) Alert rules that reference complex aggregations — avoid recomputing on every evaluation. (3) Cross-metric computations that are expensive (e.g., histogram quantiles). Naming convention: `level:metric:operations` (e.g., `job:mc_event_case_action_total:rate5m`).
**Red flags**: Doesn't know what recording rules are; confuses them with alerting rules.
**Green flags**: Mentions the naming convention, gives practical examples of when to use them.

### Q6: How would you handle metric cardinality explosion? Give a real example.
**Why asked**: Tests ability to identify and mitigate a common production issue.
**Expected answer**: Cardinality explosion happens when a label has too many unique values (e.g., user IDs, request IDs, IP addresses). Each unique combination creates a new time series, consuming memory and storage. Example: if I had added `case_id` as a label on `mc_event_case_action_total`, with 100K cases, that's 100K × 4 actions = 400K time series for one metric. Solution: don't use high-cardinality values as labels. Instead, log them and use log-based analysis (e.g., Humio/LogScale) for per-case queries. For necessary high-cardinality cases, use histograms or summary metrics instead of individual time series.
**Red flags**: Doesn't understand cardinality; says "just add more labels for more detail."
**Green flags**: Gives concrete numbers, recommends log-based analysis for high-cardinality data, mentions histograms.

### Q7: If you could redesign your monitoring stack today, what would you change?
**Why asked**: Tests forward-thinking and awareness of monitoring ecosystem evolution.
**Expected answer**: (1) Add distributed tracing (Jaeger/Tempo) to correlate requests across RAVE microservices — metrics tell you WHAT is slow, traces tell you WHERE. (2) Use Grafana Mimir for long-term metric storage instead of local Prometheus. (3) Implement SLOs (Service Level Objectives) with error budgets using Sloth or Pyrra — shift from "is it broken?" to "are we meeting our reliability targets?" (4) Add structured logging with correlation IDs that link to traces and metrics. (5) Consider OpenTelemetry as the unified telemetry standard instead of Prometheus-specific instrumentation.
**Red flags**: "Nothing, our setup is complete."
**Green flags**: Mentions observability pillars (metrics + traces + logs), SLOs/error budgets, OpenTelemetry.

---

## Tags & Cross-References

### Related STAR Stories
- [Multi-Environment Infra](multi-environment-infra.md) — monitoring covers all environments managed by this infra
- [Global Trade Compliance](global-trade-compliance.md) — GTC metrics feed into these dashboards
- [Manual Case Workflow](manual-case-workflow.md) — case creation metrics are the `mc_event_case_action_total` counter

### Interview Question Categories This Covers
- System Design: Observability architecture, metrics data model
- DevOps/SRE: Alerting strategy, incident detection, SLOs
- Data Engineering: PromQL queries, time series databases, cardinality
- Operations: Alert fatigue, runbooks, dashboard design

### Behavioral Dimensions Demonstrated
- **Initiative**: Built from scratch, not assigned — identified the gap
- **Impact measurement**: Caught real incidents within the first week
- **Technical depth**: Prometheus internals, PromQL, Grafana
- **Team enablement**: Dashboards replaced manual sprint review analysis
