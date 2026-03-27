# Case Creation Performance Under Concurrent Load

## STAR Narrative

### Situation
The HPE GreenLake RAVE platform's case creation pipeline follows a multi-stage flow: WellnessProducer → Kafka → WellnessManagement → CRM. As the Platform Co-pilot team drove user adoption, they reported increasingly sluggish case creation. I analyzed 30 days of production logs from rave-stage and discovered severe P90 response time degradation under concurrent load:
- **1 user**: 8.832s
- **5 users**: 33.038s (3.7× increase for 5× load)
- **10 users**: 41.344s (4.7× increase for 10× load)

This was clearly non-linear degradation — the system was serializing work that should have been parallelized. The P90 numbers meant 10% of users were experiencing case creation times approaching a full minute under modest concurrency, threatening the platform's reliability as adoption scaled.

### Task
I was tasked with investigating the root cause of the performance degradation and proposing concrete fixes. This required:
1. Building a stage-by-stage timing analysis to isolate which pipeline component was the bottleneck.
2. Analyzing Kafka partition assignment and consumer parallelism.
3. Implementing fixes for partition distribution, observability gaps, and nil-safety issues found during investigation.
4. Ensuring pipeline timing was observable in Grafana for ongoing monitoring.

### Action
**Step 1 — Stage-by-Stage Timing Analysis:**
I instrumented and analyzed each stage of the CreateCase pipeline to build a latency budget:
- **WellnessProducer**: 24ms average — less than 1% of end-to-end time. This stage serializes the case creation event and produces it to Kafka. Not the bottleneck.
- **Kafka transit**: 657ms average — message time-in-flight from producer acknowledgment to consumer pickup. Reasonable for the cluster configuration.
- **WellnessManagement**: Consumed the bulk of end-to-end time. This service reads from Kafka, processes the case creation logic, and relays to CRM via the crmRelay pipeline.

The timing analysis immediately showed that WellnessManagement was the bottleneck, and the degradation pattern (near-linear increase with concurrency) pointed to serialization — requests were queuing rather than processing in parallel.

**Step 2 — Root Cause: Single Kafka Partition:**
I examined the Kafka topic configuration and discovered that all messages were being routed to a single partition. The producer was not setting a partition key, so Kafka's default partitioner was using a sticky partition strategy (batching to one partition before rotating). With all messages on one partition, only one crmRelay consumer in the consumer group could process messages — the other consumers were idle. This explained the near-linear degradation: 10 concurrent users meant 10 messages queued for a single consumer.

**Step 3 — Partition Key Distribution:**
I implemented partition key distribution using `updatedMetadata.UUID` as the Kafka message key for attachment-related messages. This UUID is unique per case creation event, ensuring uniform distribution across partitions via Kafka's murmur2 hash partitioner. The key selection criteria were:
- **High cardinality**: UUIDs are unique per event → excellent distribution.
- **Ordering guarantee**: Messages for the same case go to the same partition, preserving per-case ordering.
- **Deterministic**: Same UUID always routes to the same partition — important for retry semantics.

**Step 4 — Nil Guard and Pipeline Safety:**
During the investigation, I found a nil pointer risk in the WellnessProducer's `sendToWellness` pipeline. The `fp` (fingerprint) variable was used without a nil check, which could cause a panic if the upstream context was cancelled or the producer failed initialization. I added explicit nil guards:
```go
if fp == nil {
    log.Error("WellnessProducer fp is nil, skipping send")
    return
}
```

**Step 5 — Missing CRMfp.Finish() and Completion Logging:**
I discovered that `CRMfp.Finish()` was not being called in all code paths, meaning the CRM relay fingerprint wasn't being properly closed. This caused:
- Resource leaks (open spans/contexts not cleaned up).
- Missing `crmFinishedAt` timestamps in the logs, making it impossible to calculate CRM relay duration.

I added the `CRMfp.Finish()` call and ensured `crmFinishedAt` was logged on all completion paths (success and error).

**Step 6 — Pipeline Timing Observability:**
The existing pipeline timing logs were at Debug level, meaning they were invisible in production (which runs at Info level). I promoted the critical timing logs to Info level so they would appear in Grafana/Humio without requiring a log level change:
- `wellnessProducerDuration`
- `kafkaTransitDuration`
- `wellnessManagementDuration`
- `crmRelayDuration`
- `endToEndDuration`

This enabled the stage-by-stage latency dashboards that caught this issue in the first place.

### Result
- **Horizontal scaling enabled**: Distributed Kafka messages across all partitions, allowing all crmRelay consumers to process concurrently. Under 10-user load, the work now distributes across N consumers instead of serializing on one.
- **Pipeline observability**: Stage-by-stage timing now visible in Grafana dashboards at Info log level — no more changing log levels to diagnose latency.
- **Nil safety**: Eliminated a potential nil-pointer panic in the WellnessProducer pipeline.
- **Resource cleanup**: CRMfp.Finish() properly called on all paths, preventing resource leaks and enabling accurate CRM relay duration metrics.
- **Foundation for scaling**: The partition key strategy means adding more crmRelay consumer instances immediately improves throughput — linear horizontal scaling.

## Interview Delivery Tips

### How to Open This Story
"I diagnosed and fixed a performance bottleneck in our case creation pipeline where P90 response times ballooned from 9 seconds to 41 seconds under just 10 concurrent users. The root cause was all Kafka messages going to a single partition, serializing what should have been parallel work. I implemented partition key distribution, fixed observability gaps, and enabled horizontal scaling of our CRM relay consumers."

### Time Budget (5-minute answer)
- Situation: 45 seconds (P90 numbers, non-linear degradation pattern)
- Task: 30 seconds (stage-by-stage analysis, root cause, fix)
- Action: 2.5 minutes (focus on single-partition root cause + key distribution)
- Result: 1 minute (horizontal scaling, observability, nil safety)

### Pivot Points
- **If interviewer asks about Kafka**: Deep dive into partitioning, consumer groups, sticky partitioner, murmur2 hash
- **If interviewer asks about performance**: Discuss profiling methodology, latency budgets, P90 vs. P50
- **If interviewer asks about Go**: Nil guards, defer patterns, context cancellation, resource cleanup
- **If interviewer asks about observability**: Log levels, Grafana dashboards, stage-by-stage timing

---

## Rubric Annotations

### Key Phrases/Concepts That MUST Appear
- Kafka partition key distribution (not random/sticky partitioning)
- Single-partition bottleneck as root cause of serialization
- `updatedMetadata.UUID` as deterministic, high-cardinality key
- Stage-by-stage timing analysis (WellnessProducer → Kafka → WellnessManagement)
- Consumer group parallelism (idle consumers when single partition)
- Nil guard for WellnessProducer fp
- CRMfp.Finish() on all code paths
- Log level promotion (Debug → Info) for production observability

### Common Mistakes
- Blaming Kafka transit latency without analyzing partition distribution
- Not explaining WHY the single partition caused serialization (consumer group semantics)
- Proposing "add more consumers" without fixing partition distribution (doesn't help with 1 partition)
- Ignoring the ordering guarantee implications of the partition key choice
- Describing the fix without quantifying the before/after degradation pattern
- Overlooking the nil safety and resource cleanup improvements found during investigation

### Scoring Rubric [1-5]
- 1 — Poor: Cannot explain Kafka partitioning or consumer groups; vague about what was slow
- 2 — Below Average: Identifies "Kafka was slow" but doesn't pinpoint single-partition root cause; no mention of consumer group semantics
- 3 — Acceptable: Explains single-partition bottleneck and partition key fix, but light on key selection rationale and ancillary fixes (nil guards, Finish(), log levels)
- 4 — Strong: Full narrative with stage-by-stage analysis, partition key rationale (cardinality, ordering, determinism), and observability improvements
- 5 — Exceptional: All of the above plus discusses murmur2 hashing, sticky partitioner behavior, horizontal scaling implications, and how this connects to broader pipeline reliability

---

## Follow-Up Questions (Progressively Harder)

### Follow-Up 1: Why did you choose `updatedMetadata.UUID` as the partition key instead of, say, the user ID or case type?
**Expected Answer**: The partition key needs three properties: (1) **High cardinality** — UUIDs are unique per event, ensuring uniform distribution across partitions. User IDs would create hot partitions for power users. Case type has very low cardinality (maybe 5-10 types), which would concentrate load on a few partitions. (2) **Per-case ordering** — messages about the same case go to the same partition, so a consumer processes them in order. With user ID, different cases from the same user would be serialized unnecessarily. (3) **Deterministic routing** — retries for the same event go to the same partition, preventing duplicate processing across consumers.
**Red Flags**: "Any unique ID would work the same"; doesn't consider cardinality or ordering.
**Green Flags**: Explains all three properties; contrasts with alternatives and their failure modes.

### Follow-Up 2: Adding more partitions helps throughput, but what are the trade-offs?
**Expected Answer**: (1) **Rebalancing cost** — adding partitions triggers a consumer group rebalance, causing a brief processing pause. (2) **End-to-end latency** — more partitions means more consumer assignments and potentially more network connections. (3) **Ordering scope** — ordering is only guaranteed within a partition. If you need global ordering, you're limited to one partition (which is what we accidentally had). (4) **Metadata overhead** — each partition has leader/follower state in ZooKeeper/KRaft. Thousands of partitions increase cluster coordination overhead. (5) **Consumer count ceiling** — you can't have more active consumers than partitions in a consumer group. The ideal is consumers ≈ partitions.
**Red Flags**: "More partitions are always better"; doesn't mention rebalancing or ordering.
**Green Flags**: Discusses rebalancing cost, ordering trade-off, and consumer-to-partition ratio.

### Follow-Up 3: How would you design a load test to verify the fix actually improved concurrent performance?
**Expected Answer**: (1) **Baseline**: Run CreateCase with 1, 5, 10, 20, 50 concurrent users on the old code. Record P50, P90, P99, and throughput. (2) **After fix**: Same test matrix against the new code. (3) **Metrics to compare**: P90 response time scaling factor (ideally sub-linear with concurrency), per-partition message distribution (should be uniform), consumer lag per partition (should decrease uniformly). (4) **Test setup**: Use a dedicated staging environment with production-like Kafka cluster (same partition count, replication factor). Simulate realistic payload sizes and case types. (5) **Verification**: Check Kafka consumer group lag in Grafana — all partitions should show active consumption, not just one. (6) **Soak test**: Run at moderate concurrency for 30+ minutes to detect resource leaks from the CRMfp.Finish() fix.
**Red Flags**: "Just test with 10 users and see if it's faster"; no metrics framework.
**Green Flags**: Defines scaling factor as success metric, checks partition distribution, includes soak test for resource leaks.

---

## Grilling Chain (7 questions drilling deeper)

### Q1: Walk me through how Kafka consumer groups work and why a single partition caused serialization.
**Why asked**: Tests foundational Kafka knowledge.
**Expected answer**: A consumer group is a set of consumers that cooperate to consume a topic. Each partition is assigned to exactly one consumer in the group. If a topic has 6 partitions and 6 consumers, each consumer gets 1 partition. If a topic has 1 partition (or all messages go to 1 partition), only 1 consumer is active — the rest are standby. Messages within a partition are processed sequentially by the assigned consumer. So with all messages on partition 0, consumer 0 processes them one at a time while consumers 1-5 sit idle. 10 concurrent CreateCase requests = 10 messages queued on partition 0 = serial processing.
**Red flags**: Confuses partitions with consumer groups; says consumers can share a partition.
**Green flags**: Explains the 1-partition-to-1-consumer assignment, idle consumer concept.

### Q2: What is Kafka's sticky partitioner, and why did it cause all messages to go to one partition?
**Why asked**: Tests deep Kafka producer knowledge.
**Expected answer**: When no key is set, Kafka's producer uses the sticky partitioner (since KIP-480, Kafka 2.4+). Instead of round-robin per message, it batches messages to a single partition until the batch is full or the linger time expires, then switches to another partition. This improves batching efficiency (fewer, larger batches). In low-throughput scenarios — like our case creation rate — the batch never fills up, so all messages stick to the same partition indefinitely. The old round-robin behavior was more uniform but created many small batches. The fix is to set an explicit key, which overrides the sticky partitioner and uses murmur2 hash to deterministically assign partitions.
**Red flags**: Says Kafka always round-robins without a key; doesn't know about sticky partitioner.
**Green flags**: Explains KIP-480, why low throughput exacerbates the problem, and how key-based routing overrides it.

### Q3: Your fix distributes load across partitions. What happens if one consumer is slower than others?
**Why asked**: Tests understanding of consumer lag and backpressure.
**Expected answer**: If one consumer is slower (e.g., CRM is slow for a subset of cases), that partition's consumer lag grows while others stay healthy. This creates a hot partition — not from message distribution but from processing time. Mitigations: (1) Monitor per-partition consumer lag in Grafana. (2) Set `max.poll.records` to limit batch size per poll, preventing a single slow batch from blocking the consumer. (3) Use `max.poll.interval.ms` to detect stuck consumers — if a consumer doesn't poll within this interval, it's removed from the group and its partitions are reassigned. (4) Consider async processing within the consumer — dequeue from Kafka quickly, process in a worker pool with backpressure.
**Red flags**: "The load is balanced so all consumers are equally fast."
**Green flags**: Discusses consumer lag, max.poll settings, and async processing patterns.

### Q4: How did you verify that the partition key distribution was actually uniform?
**Why asked**: Tests validation methodology.
**Expected answer**: (1) **Kafka tooling**: Used `kafka-consumer-groups.sh --describe` to check per-partition offsets and lag — all partitions should show growing offsets, not just one. (2) **Grafana dashboard**: Created a panel showing message rate per partition using `kafka_server_brokertopicmetrics_messagesin_total` broken down by partition. (3) **Log analysis**: Checked the stage-by-stage timing logs to verify different `crmRelay` consumer instances were processing messages (each consumer logs its instance ID). (4) **Hash distribution test**: Verified that UUIDs produce uniform hash distribution across the target partition count — murmur2 hash of UUIDs is well-distributed by design, but I validated with a sample of 10K UUIDs against 6 partitions.
**Red flags**: "I just checked that it was faster."
**Green flags**: Multiple verification methods, mentions partition-level metrics, hash distribution validation.

### Q5: Why was CRMfp.Finish() not being called, and what were the consequences beyond missing timestamps?
**Why asked**: Tests understanding of resource management in Go.
**Expected answer**: The `Finish()` call was missing on an early-return error path — the happy path called it, but an error branch returned before reaching the `Finish()` call. Consequences: (1) **Open spans**: If `fp` wraps a tracing span, the span is never closed — this creates orphan spans in the trace, making trace-based debugging unreliable. (2) **Context leak**: If the fingerprint holds a `context.CancelFunc`, not calling Finish() means the context and its resources (goroutines, connections) are never cleaned up until GC. (3) **Metric inaccuracy**: Duration metrics derived from start/finish timestamps are missing data points for error cases — survivorship bias in latency data. The fix was to use a `defer CRMfp.Finish()` immediately after creation, guaranteeing cleanup on all paths.
**Red flags**: "It was just a logging issue."
**Green flags**: Mentions span leaks, context leaks, survivorship bias in metrics, and the defer pattern as the idiomatic fix.

### Q6: The P90 went from 8.8s to 41.3s at 10 users. What would the theoretical improvement be with your fix?
**Why asked**: Tests ability to reason about performance characteristics.
**Expected answer**: With proper partition distribution across N partitions and N consumers, the throughput scales linearly up to N. If we have 6 partitions and 6 consumers, 10 concurrent requests distribute as ~2 per consumer. Each consumer processes serially, so the P90 should approach 2 × single-request-time ≈ 2 × 8.8s ≈ 17.6s — a ~60% improvement. The theoretical floor is the single-request latency (8.8s) if we have ≥ 10 partitions and ≥ 10 consumers. However, CRM is an external dependency — if it rate-limits or has its own bottleneck, that becomes the new ceiling. The fix removes the Kafka-layer serialization bottleneck, but end-to-end performance depends on the slowest stage.
**Red flags**: "It should be instant now"; doesn't consider partition count or CRM bottleneck.
**Green flags**: Calculates based on partition count, acknowledges external dependency limits.

### Q7: If the CRM downstream system became the bottleneck after your fix, how would you address it?
**Why asked**: Tests system thinking beyond the immediate fix.
**Expected answer**: (1) **Circuit breaker**: Implement a circuit breaker pattern in the crmRelay consumer — if CRM error rate exceeds a threshold, stop sending and retry with exponential backoff. (2) **Bulkhead**: Limit concurrent CRM requests per consumer to prevent overwhelming the downstream system. (3) **Queue buffering**: Use Kafka's natural backpressure — if consumers slow down, lag grows but nothing is lost. Monitor lag and scale consumers accordingly. (4) **Caching**: Cache CRM responses for idempotent operations (e.g., case status checks) to reduce call volume. (5) **Async relay**: Decouple the CRM relay from the Kafka consumer — consumer writes to a local queue/database, a separate worker drains to CRM at a controlled rate. (6) **SLA negotiation**: Work with the CRM team to understand their rate limits and optimize request batching.
**Red flags**: "Just add more consumers" (doesn't help if CRM is the bottleneck).
**Green flags**: Mentions circuit breaker, bulkhead, backpressure, and rate-based control.

---

## Metrics & Evidence

### Performance Data (Before Fix)
| Concurrent Users | P90 Response Time | Scaling Factor |
|-----------------|-------------------|----------------|
| 1               | 8.832s            | 1.0×           |
| 5               | 33.038s           | 3.7×           |
| 10              | 41.344s           | 4.7×           |

### Pipeline Timing Breakdown (30-day average)
| Stage               | Avg Latency | % of E2E |
|---------------------|-------------|----------|
| WellnessProducer    | 24ms        | <1%      |
| Kafka Transit       | 657ms       | ~7%      |
| WellnessManagement  | ~8,100ms    | ~92%     |

### JIRA References
- Performance investigation and Kafka partitioning fix
- Related to WellnessProducer nil guard
- Related to CRMfp.Finish() completion logging

---

## Weak / Acceptable / Strong Answer Calibration

### Weak Answer
"Our case creation was slow so I looked at the logs and found it was a Kafka issue. I changed the partition key and it got faster."
- **Why weak**: No quantification, no explanation of root cause mechanics, no mention of investigation methodology, no ancillary improvements.

### Acceptable Answer
"P90 response times for CreateCase degraded from 9 seconds to 41 seconds under 10 concurrent users. I analyzed the pipeline stage by stage and found that all Kafka messages were going to a single partition, so only one crmRelay consumer was doing all the work. I set the partition key to the case UUID so messages distribute across partitions, enabling parallel processing. I also fixed some logging to make the pipeline timing visible in Grafana."
- **Why acceptable**: Has numbers, identifies root cause, explains the fix. Missing: key selection rationale, nil guards, CRMfp.Finish(), deeper Kafka mechanics.

### Strong Answer
"The Platform Co-pilot team reported degrading case creation performance as adoption grew. I analyzed 30 days of rave-stage logs and built a stage-by-stage timing breakdown: WellnessProducer at 24ms was negligible, Kafka transit at 657ms was reasonable, but WellnessManagement consumed the bulk of the time. P90 went from 8.8 seconds at 1 user to 41.3 seconds at 10 — a non-linear scaling pattern that pointed to serialization. The root cause was Kafka's sticky partitioner sending all keyless messages to a single partition, meaning only one of our crmRelay consumers was active while the rest sat idle. I implemented partition key distribution using updatedMetadata.UUID — chosen for high cardinality, per-case ordering preservation, and deterministic routing for retries. During the investigation, I also found and fixed a nil pointer risk in WellnessProducer's fp variable, a missing CRMfp.Finish() call that was leaking resources and hiding error-path duration metrics, and promoted pipeline timing logs from Debug to Info for production observability. The fix enables linear horizontal scaling — adding more crmRelay consumers now actually improves throughput."
- **Why strong**: Quantified throughout, explains investigation methodology, root cause mechanics (sticky partitioner), key selection rationale (3 properties), ancillary fixes with their impact, and scaling implications.

---

## Tags & Cross-References

### Related STAR Stories
- [Monitoring Overhaul](monitoring-overhaul.md) — Grafana dashboards that surfaced this latency issue
- [Manual Case Workflow](manual-case-workflow.md) — case creation pipeline that this performance work optimizes
- [File Upload Security](file-upload-security.md) — attachment pipeline connected via the same Kafka topics

### Interview Question Categories This Covers
- System Design: Message queue partitioning, horizontal scaling, pipeline analysis
- Performance: P90 analysis, latency budgets, bottleneck identification
- Kafka: Partitioning, consumer groups, sticky partitioner, key selection
- Go Programming: Nil guards, defer patterns, resource cleanup
- Observability: Log levels, stage-by-stage timing, Grafana dashboards

### Behavioral Dimensions Demonstrated
- **Analytical rigor**: Stage-by-stage timing breakdown to isolate the bottleneck
- **Root cause depth**: Went beyond "Kafka is slow" to sticky partitioner mechanics
- **Thoroughness**: Fixed ancillary issues (nil guards, Finish(), log levels) found during investigation
- **Scalability thinking**: Solution enables horizontal scaling, not just a point fix

---

## Gold Standard Answer (Rubric-Annotated)

> *[SITUATION — 45s]* "The Platform Co-pilot team reported degrading case creation as adoption grew. **[QUANTIFY]** I analyzed 30 days of rave-stage logs and found P90 response times went from 8.8s at 1 user to 33s at 5 users to 41.3s at 10 — a non-linear scaling pattern indicating serialization, not just load."
>
> *[TASK — 30s]* "I was tasked with root-causing this and proposing fixes. **[SCOPE]** That meant building a stage-by-stage timing analysis across the full pipeline: WellnessProducer → Kafka → WellnessManagement → CRM."
>
> *[ACTION — 2.5min]* "First, I built a latency budget. **[KEY INSIGHT]** WellnessProducer was 24ms — negligible. Kafka transit was 657ms — reasonable. WellnessManagement consumed the bulk. **[ROOT CAUSE]** Digging into Kafka, I found all messages were landing on a single partition — the producer had no key set, so Kafka's sticky partitioner batched everything to one partition. With only one active crmRelay consumer, the rest were idle. **[FIX]** I implemented partition key distribution using updatedMetadata.UUID — chosen for high cardinality, per-case ordering, and deterministic retry routing. **[ANCILLARY]** During investigation I also found and fixed a nil pointer risk in WellnessProducer fp, a missing CRMfp.Finish() call leaking resources, and promoted pipeline timing from Debug to Info for production visibility."
>
> *[RESULT — 1min]* "**[IMPACT]** The fix enables linear horizontal scaling — adding crmRelay consumers now actually improves throughput. Pipeline timing is now observable in Grafana without log level changes. The nil guard and Finish() fixes eliminated resource leaks and metric blind spots on error paths."

### Why This Scores 5/5
- **Quantified throughout**: P90 numbers, stage-by-stage breakdown, scaling factor
- **Root cause mechanics**: Sticky partitioner, consumer group semantics, not just "wrong partition"
- **Key selection rationale**: Three properties (cardinality, ordering, determinism)
- **Ancillary improvements**: Shows thoroughness — didn't stop at the immediate fix
- **Scaling implications**: Connects the fix to horizontal scaling capability
