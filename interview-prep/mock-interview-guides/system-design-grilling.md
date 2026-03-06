# System Design Grilling — Deep Question Chains

> 5-7 question chains per system design topic, drilling progressively deeper.
> Each question has expected answer, red flags, and green flags.

---

## Chain 1: Multi-Tenant Case Management System

### Q1: "Design a system where multiple tenants can create and manage support cases."
**Why asked:** High-level design thinking — can they scope the problem?
**Expected:** Clarify requirements (tenants, case types, routing), propose API layer → storage → routing → external integration architecture.
**Red flags:** 🚩 Jumps straight to database schema without clarifying requirements
**Green flags:** ✅ Asks clarifying questions, draws component boundaries first

### Q2: "How do you ensure tenant isolation?"
**Why asked:** Security-critical design — multi-tenancy is the hardest part.
**Expected:** Workspace ID from JWT (never client-supplied), all queries scoped, database-level scoping (shared with filters vs database-per-tenant), audit logging.
**Red flags:** 🚩 "We'd have separate databases per tenant" without trade-off discussion, or ignoring the question
**Green flags:** ✅ Explains RAVE's approach: shared DB with query scoping, server-side workspace derivation, connects to GLCP-333624 fix

### Q3: "A tenant reports they can see another tenant's cases. How do you investigate?"
**Why asked:** Operational skills — incident response for security.
**Expected:** Check audit logs for cross-workspace queries, review recent deployments, check JWT middleware configuration, look for queries missing workspace filter.
**Red flags:** 🚩 "That shouldn't happen" without investigating
**Green flags:** ✅ Systematic approach, mentions specific checks, connects to real experience

### Q4: "How do you handle case routing (some tenants want email, some want CRM)?"
**Why asked:** Design flexibility — not all tenants are the same.
**Expected:** Domain-based routing (email domain → SendGrid via caseRelay, HPE domain → CRM API via crmRelay), configurable per tenant.
**Red flags:** 🚩 Hardcoded routing with no extensibility
**Green flags:** ✅ Strategy pattern, configuration-driven routing, extensibility for new routes

### Q5: "The CRM API is slow and sometimes down. How does your design handle this?"
**Why asked:** Resilience and failure handling.
**Expected:** Retry with exponential backoff, circuit breaker, async processing via queue, alerting on failure rates.
**Red flags:** 🚩 "The CRM should just be reliable" — no resilience thinking
**Green flags:** ✅ Specific patterns (retry, circuit breaker), connects to crmRelay attachment retry experience

### Q6: "This system needs to handle 10x current traffic. What breaks first?"
**Why asked:** Scale thinking — understand bottlenecks.
**Expected:** MongoDB write throughput → sharding by workspace_id. API layer → horizontal scaling with HPA. CRM API → rate limiting concern, need queue buffer. Kafka → add partitions.
**Red flags:** 🚩 "Just add more pods" without thinking about data layer
**Green flags:** ✅ Identifies bottlenecks by component, proposes specific scaling strategies per component

### Q7: "How do you monitor this system?"
**Why asked:** Observability as part of design, not afterthought.
**Expected:** Prometheus metrics (request latency, case creation rate, routing failures), Grafana dashboards, PagerDuty/Slack alerts, log aggregation.
**Red flags:** 🚩 Doesn't mention monitoring unprompted
**Green flags:** ✅ Proposes monitoring as integral to design, specific metrics and alert categories

[SCORING: 1-5]
- 1-2: Can't design the system coherently
- 3: Reasonable design but missing security or failure handling
- 4: Solid design with tenant isolation, failure handling, and monitoring
- 5: Above + scale analysis, connects to real experience, discusses trade-offs throughout

---

## Chain 2: File Upload Security System

### Q1: "Design a file upload system for a case management platform."
**Why asked:** Can they think about security from the start?
**Expected:** Multi-layer validation pipeline, not just "upload to S3."
**Red flags:** 🚩 "Accept the file and store it" — no security consideration
**Green flags:** ✅ Immediately mentions validation, threat model, or attack vectors

### Q2: "What's your threat model?"
**Why asked:** Security mindset — think like an attacker.
**Expected:** Path traversal, extension spoofing, content-type mismatch, zip bombs, XSS via filename, DoS via large files, malware upload.
**Red flags:** 🚩 "Just check the file extension"
**Green flags:** ✅ Multiple attack vectors, prioritizes by severity, mentions defense in depth

### Q3: "How does magic byte validation work? Write the core function."
**Why asked:** Implementation depth — can they code it?
**Expected:** Map of extension → byte signature, read first N bytes, compare with bytes.HasPrefix. Should know specific signatures (PDF=0x25504446, PNG=0x89504E47).
**Red flags:** 🚩 Knows the concept but can't write the code
**Green flags:** ✅ Writes clean function, knows specific byte values, handles edge cases (text files without magic bytes)

### Q4: "DOCX and XLSX have the same magic bytes (PK). How do you distinguish them?"
**Why asked:** Edge case awareness — real-world nuances.
**Expected:** Both are ZIP-based — can unzip and check internal file structure (word/document.xml vs xl/workbook.xml). Or accept that magic bytes alone can't distinguish and rely on extension + content-type header as additional signals.
**Red flags:** 🚩 "That's not possible" or doesn't know about ZIP-based Office formats
**Green flags:** ✅ Explains the ZIP container format, proposes internal structure check, discusses trade-off (performance vs thoroughness)

### Q5: "A security researcher found a polyglot file that passes your validation. Now what?"
**Why asked:** Handling failure of your security model.
**Expected:** Acknowledge the limitation, propose additional layers: ClamAV scanning, sandboxed execution, content-disposition headers forcing download, per-tenant upload rate limiting.
**Red flags:** 🚩 "That can't happen with our validation" — overconfidence
**Green flags:** ✅ Acknowledges limitation, proposes defense in depth, discusses ClamAV or similar

### Q6: "If you could redesign this from scratch, what would you change?"
**Why asked:** Design maturity — learning from experience.
**Expected:** Pre-signed S3 URLs (avoid file transit through API), async processing pipeline (upload → scan → notify CRM), ClamAV integration, per-tenant quotas.
**Red flags:** 🚩 "It's fine as-is"
**Green flags:** ✅ Specific improvements with rationale, S3 pre-signed URLs for performance and security

---

## Chain 3: Global Trade Compliance Gateway

### Q1: "Design a system that enforces export control on software downloads."
**Why asked:** Domain-specific design with regulatory requirements.
**Expected:** Separate onboarding (screening) from download-time (enforcement), caching for performance, fail-closed for compliance.
**Red flags:** 🚩 Doesn't mention fail-closed or compliance implications
**Green flags:** ✅ Regulatory awareness, two-phase architecture, fail-closed default

### Q2: "Why separate onboarding from download-time checks?"
**Why asked:** Architecture justification.
**Expected:** RPL screening is expensive and slow — do it once at onboarding. ELC/IP checks are per-download but can be cached. Separating allows different scaling and SLA.
**Red flags:** 🚩 "Just check everything at download time"
**Green flags:** ✅ Performance rationale, SLA differences, caching strategy

### Q3: "How do you cache compliance checks without violating regulations?"
**Why asked:** Trade-off between performance and correctness.
**Expected:** TTL-based caching, cache invalidation on status changes, stale-while-revalidate for performance, but never serve a cached "allow" past TTL without recheck.
**Red flags:** 🚩 "Cache everything forever"
**Green flags:** ✅ TTL awareness, distinction between cacheable (IP geo) and non-cacheable (device status changes), fail-closed on cache miss

### Q4: "GTCaaS is down. What happens to download requests?"
**Why asked:** Failure mode design for compliance systems.
**Expected:** Cached results serve stale-but-known-good for recently checked IPs. New/uncached IPs are blocked (fail-closed). Alert on GTCaaS unavailability.
**Red flags:** 🚩 "Allow downloads" — fail-open is a compliance violation
**Green flags:** ✅ Fail-closed, stale cache with caveats, alerting, degraded mode awareness

### Q5: "A device shadow is stuck in PENDING state. How do you diagnose?"
**Why asked:** Operational skills with state machines.
**Expected:** Check AutoOnboard logs, verify GTCaaS RPL API is responding, check MongoDB for the document, check if there's a Kafka message stuck, look at retry metrics.
**Red flags:** 🚩 "Restart the service"
**Green flags:** ✅ Systematic debugging, state machine understanding, specific checks per component

### Q6: "You're migrating from one geolocation provider to another. How?"
**Why asked:** Migration planning — real-world scenario.
**Expected:** Abstract the provider behind an interface, implement new provider, dual-run period, compare results, cut over. Minimize cache schema changes.
**Red flags:** 🚩 "Just swap the API call"
**Green flags:** ✅ Interface abstraction, dual-run validation, cache compatibility, rollback plan

---

## Chain 4: Monitoring & Alerting System

### Q1: "Design a monitoring system for 20+ microservices."
**Why asked:** Observability architecture at scale.
**Expected:** Prometheus for metrics, Grafana for visualization, AlertManager for routing, log aggregation (ELK/Humio), distributed tracing.
**Red flags:** 🚩 Only mentions logging
**Green flags:** ✅ Three pillars: metrics, logs, traces. Plus alerting.

### Q2: "How do you design metrics that scale?"
**Why asked:** Cardinality awareness — common pitfall.
**Expected:** Bounded labels only, no user IDs or request IDs. Unified metrics with labels vs separate metrics. Recording rules for expensive queries.
**Red flags:** 🚩 Puts unbounded values in labels
**Green flags:** ✅ Cardinality awareness, label hygiene, recording rules concept

### Q3: "How do you avoid alert fatigue?"
**Why asked:** Operational maturity.
**Expected:** Alert grouping, severity tiers (PagerDuty for critical, Slack for warning), alert suppression during deployments, runbooks linked from alerts, SLO-based alerts instead of threshold.
**Red flags:** 🚩 "Every metric gets an alert"
**Green flags:** ✅ Tiered alerting, deployment awareness, runbooks, mentions SLOs

### Q4: "A dashboard shows a spike in error rates. Walk me through your investigation."
**Why asked:** Debugging methodology.
**Expected:** Identify which service, which endpoint, what error codes. Check recent deployments. Check upstream dependencies (MongoDB, Kafka, CRM). Check pod health. Check logs for the specific error.
**Red flags:** 🚩 "Check the logs" without a structured approach
**Green flags:** ✅ Systematic top-down approach, correlates multiple signals (metrics + logs + deployments)

### Q5: "How do you monitor business metrics, not just infrastructure?"
**Why asked:** Business awareness in engineering.
**Expected:** Case creation rate, CRM success/failure ratio, automation vs manual case split, response time SLA compliance. These matter more to stakeholders than CPU utilization.
**Red flags:** 🚩 Only monitors CPU/memory/request count
**Green flags:** ✅ Business KPIs as first-class metrics, understands stakeholder perspective

### Q6: "What would you add to the monitoring system if you had unlimited resources?"
**Why asked:** Vision and understanding of the art of the possible.
**Expected:** Distributed tracing (Jaeger), SLO framework with error budgets, AI-powered anomaly detection (connects to the Grafana AI concept), chaos engineering, synthetic monitoring.
**Red flags:** 🚩 "It's good enough"
**Green flags:** ✅ Multiple improvement areas with clear rationale for each

---

## Chain 5: Kubernetes Deployment Architecture

### Q1: "Design the deployment architecture for a microservices platform."
**Why asked:** Infrastructure design skills.
**Expected:** Container orchestration (K8s), CI/CD pipeline, GitOps (ArgoCD), service mesh (Istio), secrets management, monitoring.
**Red flags:** 🚩 "Deploy Docker containers to VMs"
**Green flags:** ✅ Modern K8s stack with GitOps, service mesh, secrets management

### Q2: "How do you handle secrets in a Kubernetes environment?"
**Why asked:** Security in infrastructure.
**Expected:** ESO (External Secrets Operator) syncing from cloud secret stores (AWS Parameter Store, Vault), IRSA for IAM, never in plaintext, never in Git (even encrypted is controversial).
**Red flags:** 🚩 "Environment variables in the deployment YAML"
**Green flags:** ✅ ESO or Vault, IRSA for auth, secret rotation, least privilege

### Q3: "How do you do zero-downtime deployments?"
**Why asked:** Deployment strategy knowledge.
**Expected:** Rolling update (maxUnavailable: 0, maxSurge: 1), readiness probes, PDB (Pod Disruption Budget), ArgoCD sync waves, database backward compatibility.
**Red flags:** 🚩 "We just do `kubectl apply`"
**Green flags:** ✅ Rolling update config, probes, PDB, database migration awareness

### Q4: "A deployment caused a regression. How do you rollback?"
**Why asked:** Operational recovery.
**Expected:** ArgoCD rollback to previous sync, or `kubectl rollout undo`. Depends on whether the issue is code (rollback image) or config (rollback Helm values). Database migrations may not be reversible.
**Red flags:** 🚩 Doesn't mention database compatibility or Helm value rollback
**Green flags:** ✅ Multiple rollback strategies based on issue type, database awareness, ArgoCD history

### Q5: "How does Istio fit into this architecture?"
**Why asked:** Service mesh understanding.
**Expected:** Sidecar proxy (Envoy), mTLS between services, RequestAuthentication for JWT, VirtualService for traffic routing, DestinationRule for circuit breaking.
**Red flags:** 🚩 "It's just for service discovery"
**Green flags:** ✅ Multiple Istio features with specific CRD examples, connects to real auth implementation

### Q6: "What are the downsides of your Istio + K8s architecture?"
**Why asked:** Critical thinking about your own infrastructure.
**Expected:** Complexity (YAML hell), debugging difficulty (Envoy proxy issues are opaque), resource overhead (sidecar per pod), learning curve for new team members, upgrade risk.
**Red flags:** 🚩 "There are no downsides" or can't name any
**Green flags:** ✅ Honest about complexity, specific operational challenges, suggests mitigations
