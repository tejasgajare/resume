# Answer Calibration — Weak / Acceptable / Strong

> For the 15 most likely interview questions, three quality levels are provided.
> Use these to calibrate scoring and identify gaps in your preparation.

---

## 1. "Tell me about RAVE."

### Weak Answer (Score: 2)
"RAVE is a platform at HPE that handles wellness automation. It has a bunch of microservices written in Go. I worked on the API layer and did some bug fixes. It uses MongoDB and Kubernetes."

**Why it's weak:** No specifics about architecture, personal contribution, or technical depth. "Bunch of microservices" is vague. No mention of what wellness automation means.

### Acceptable Answer (Score: 3)
"RAVE Cloud is HPE's wellness automation engine — it monitors infrastructure devices and creates support cases automatically. It's about 20 microservices running on Kubernetes. I primarily worked on the wellnessProducer service, which is the REST API layer. I built the v3 APIs with JWT authentication, implemented manual case creation, and added file upload validation. We use MongoDB for storage and Kafka for event streaming between services."

**Why it's acceptable:** Good overview, names specific service and contributions. Missing: architecture flow, security work, monitoring, trade-offs.

### Strong Answer (Score: 5)
"RAVE Cloud is HPE GreenLake's wellness automation engine — it monitors HPE infrastructure, detects health anomalies, and automatically creates support cases. The architecture is event-driven: device telemetry flows through Kafka to wellness analyzers, which trigger case creation through my primary service, wellnessProducer.

I own wellnessProducer — the Go REST API that handles case CRUD, file uploads, and JWT authentication. I evolved it through three API versions, each adding security: v1 accepted legacy tokens, v3 enforces environment-specific issuers aligned with Istio RequestAuthentication. The manual case flow routes through caseRelay (email/SendGrid) or crmRelay (CRM API) based on the tenant's domain.

I also fixed 13+ security vulnerabilities including a critical cross-workspace case creation bug, built the Prometheus/Grafana monitoring system, and added multi-layer file upload validation with magic byte verification. The platform runs 20+ services across dev/intg/prod with Helm, ArgoCD, and Istio.

What I'd improve: better distributed tracing, more granular alert suppression during deployments, and a formal SLO framework."

**Why it's strong:** Architecture flow, specific ownership, security narrative with numbers, monitoring, and honest improvement areas. Shows the candidate both built and reflects on the system.

---

## 2. "Explain the JWT auth system."

### Weak Answer (Score: 2)
"We use JWT tokens for authentication. The tokens are validated by Istio and then the Go service checks them too. It's a standard JWT implementation."

### Acceptable Answer (Score: 3)
"Authentication has two layers. Istio's RequestAuthentication validates the JWT signature using JWKS from HPE's SSO endpoint. Then my Go middleware extracts HPE-specific claims like workspace_id and principal_type for tenant scoping. v3 APIs enforce stricter token requirements than v1."

### Strong Answer (Score: 5)
"The JWT system evolved through three versions. Originally, the middleware used hardcoded regex (`arubathena.com`, `common.cloud.hpe.com`) — this meant dev tokens worked in production, which was a security gap I discovered and fixed.

The fix: environment-specific issuer validation. The middleware reads `ALLOWED_JWT_ISSUERS` from environment variables, set from Helm values matching Istio's RequestAuthentication config. Now dev tokens only work in dev.

The middleware itself uses `ParseUnverified` — Istio already verified the signature, so we just need to extract claims. It pulls HPE-specific fields: `hpe_workspace_id` (for tenant scoping), `hpe_principal_type` (user vs api-client), `hpe_identity_type` (user vs service identity). The workspace_id is injected into the context and used by all downstream handlers — this is how we prevent cross-tenant access.

I also added `auth_failure_total` metrics so we can detect misconfiguration or attack patterns in Grafana."

---

## 3. "How did you handle file uploads?"

### Weak Answer (Score: 2)
"We validate file uploads — check the extension and make sure the file isn't too big. We store them in MongoDB."

### Acceptable Answer (Score: 3)
"File uploads go through multiple validation steps: filename sanitization to prevent path traversal, an extension allowlist of 60+ types, content-type header verification, and magic byte checking to catch renamed files. We also enforce size limits."

### Strong Answer (Score: 5)
"I built a five-layer validation pipeline. Layer 1: filename sanitization — strip path components, null bytes, and special characters. Layer 2: extension allowlist — 60+ types explicitly allowed, everything else rejected. Layer 3: content-type header must match the extension. Layer 4: magic byte verification — I read the first N bytes and compare against known signatures. A PDF must start with `%PDF` (0x25504446), a PNG with `0x89504E47`. This catches renamed executables. Layer 5: per-file and total size limits.

I know the limitations: ZIP-based Office formats (docx, xlsx) share the same PK magic bytes — can't distinguish at the byte level. Text formats have no reliable magic bytes. Polyglot files could pass magic byte checks while containing payloads.

The attachments are then forwarded to the CRM. I also fixed a race condition in crmRelay where attachments arrived before the case number was assigned — implemented retry with exponential backoff."

---

## 4. "Describe the monitoring setup."

### Weak Answer (Score: 2)
"We use Prometheus and Grafana. I set up some dashboards and alerts that go to Slack."

### Acceptable Answer (Score: 3)
"I built the monitoring system using Prometheus for metrics collection, Grafana for dashboards, and AlertManager for routing alerts to Slack and PagerDuty. Services expose metrics via /metrics endpoints. I created custom business metrics like case creation rate and CRM success/failure rates."

### Strong Answer (Score: 5)
"I designed the monitoring overhaul with a key insight: consolidate separate per-action metrics into one unified metric. Instead of separate caseCreated, caseUpdated, caseReopened counters, I created `mc_event_case_action_total` with an `action` label. Bounded cardinality (4 values), simplified dashboards, and one PromQL query catches drops across all action types.

Four alert categories: service stopped running (warning: non-running pods, critical: zero replicas), processing drops (case creation rate below threshold), CRM failures (error rate spike), and auth failures (misconfiguration or attack detection). PagerDuty for critical, Slack for warning.

What I'd add: distributed tracing with Jaeger for cross-service request correlation, SLO-based alerting instead of threshold-based, and deployment annotations in Grafana to correlate alert spikes with rollouts."

---

## 5. "Walk through a security fix."

### Weak Answer (Score: 2)
"We found a security bug where users could see other users' data. We fixed it by adding validation."

### Acceptable Answer (Score: 3)
"GLCP-333624 — the v2 Create Case API allowed cross-workspace case creation. The handler used workspace_id from the request body instead of extracting it from the JWT. I fixed it by deriving workspace from the token's hpe_workspace_id claim."

### Strong Answer (Score: 5)
"GLCP-333624 was a cross-tenant isolation failure. The v2 Create Case API accepted workspace_id in the request body and used it to scope database operations. A malicious tenant could supply another tenant's workspace_id and create cases in their namespace.

Root cause: The API was designed for convenience — frontend sends workspace_id, backend trusts it. This violated the principle that identity information must always come from the authenticated token, never the client.

Fix: I changed the handler to ignore client-supplied workspace_id entirely. It now extracts hpe_workspace_id from the JWT claims (which Istio has already verified). I added regression tests that specifically attempt cross-workspace case creation and verify they're rejected. I also added the createdBy field from the JWT sub claim for audit trail.

This led to a broader review — I found and fixed 12+ additional input validation issues in the same pass, including contact email validation, severity allowlist enforcement, and attachment filename sanitization."

---

## 6. "How does the Nimble GT system work?"

### Weak Answer (Score: 2)
"It's a system that checks if users are allowed to download software. It uses some APIs to check compliance."

### Acceptable Answer (Score: 3)
"The Nimble Download system enforces US export control on software downloads. It has four microservices: AutoOnboard screens devices against restricted party lists, DownloadGatekeeper checks IP geolocation and license status at download time, GTCacheAPI caches results in SQLite, and there are reference number tools. Devices go through a state machine from PENDING to ONBOARDED or BLOCKED."

### Strong Answer (Score: 5)
"The system has two phases. Phase 1 — onboarding: when a Nimble device is discovered, AutoOnboard screens it against the Restricted Party List (RPL) via GTCaaS. Clear devices go to ONBOARDED, matches go to BLOCKED. This happens asynchronously.

Phase 2 — download time: when a user requests a software download, DownloadGatekeeper intercepts and runs three checks: (1) IP geolocation — is the request from an embargoed country? (2) License/account status — valid and not suspended? (3) Device status — ONBOARDED, not BLOCKED?

The GTCacheAPI reduces redundant calls to GTCaaS by 60%+ using SQLite — zero external dependencies, embedded, survives pod restarts. Cache entries have TTL-based invalidation.

I'm also planning the MaxMind migration: GTCaaS is switching from Neustar to MaxMind for IP geolocation. I designed the cache to abstract over the provider — the geo_country column is kept but only populated on 403 embargo responses (MaxMind inlines country in the screening response). This minimizes code changes when the provider switches.

The system fails closed — if GTCaaS is unreachable, downloads are blocked for uncached IPs. This is correct for compliance — better to block a legitimate download temporarily than allow a prohibited one."

---

## 7. "Tell me about a production incident."

### Weak Answer (Score: 2)
"We had some issues with the service and I helped fix them."

### Acceptable Answer (Score: 3)
"There was a race condition in crmRelay where file attachments arrived before the case number was assigned by the CRM. The attachment upload failed because it couldn't find the case. I added retry logic with backoff."

### Strong Answer (Score: 5)
"The crmRelay attachment race condition was interesting. When a manual case is created with attachments, two async operations happen: (1) case creation → CRM API → returns case number, (2) attachments forwarded to CRM referencing the case number. Sometimes the CRM was slow to return the case number, but attachment processing was already triggered.

I diagnosed this by adding metrics (`crm_relay_attachment_retry_total`) and correlating spikes with CRM response latency in Grafana. The fix: retry with exponential backoff and context-aware cancellation. The handler checks if the case exists before uploading, retries up to N times with increasing delay, and respects context cancellation so long retries don't outlive the request.

This wasn't a catastrophic incident — it affected a small percentage of cases with attachments. But it was insidious because the failure was silent until we had monitoring. The attachment was simply missing from the CRM case."

---

## 8. "What's your experience with Kubernetes?"

### Weak Answer (Score: 2)
"We deploy to Kubernetes. I've worked with pods and deployments."

### Acceptable Answer (Score: 3)
"I manage K8s deployments for RAVE across 3 environments. I work with Deployments, Services, ConfigMaps, Secrets, ServiceAccounts, and Istio CRDs. I provisioned custom ServiceAccounts for the ESO migration and configured rolling update strategies."

### Strong Answer (Score: 5)
"I manage 20+ microservices on K8s across dev/intg/prod. The deployment pipeline is GitOps: code push → GitHub Actions CI → Docker build → ArgoCD detects Helm chart changes → syncs to K8s.

I work at multiple K8s abstraction levels. Infrastructure: custom ServiceAccount provisioning with AWS IRSA annotations for ESO — each service gets least-privilege access to its own Parameter Store paths. Service mesh: Istio RequestAuthentication for JWT validation, VirtualService for version routing, DestinationRules for circuit breaking. Application: Deployments with rolling update (maxUnavailable: 0, maxSurge: 1), liveness/readiness probes, resource limits.

For the ESO migration, I had to: create ServiceAccounts with IAM role annotations, create SecretStore and ExternalSecret CRDs, verify backward-compatible K8s Secret names, and ensure Helm hook ordering (SecretStore before ExternalSecret). I completed 3 services and documented the pattern for the remaining 17."

---

## 9. "Why should we hire you?"

### Weak Answer (Score: 2)
"I'm a good developer who works hard and learns fast."

### Acceptable Answer (Score: 3)
"I bring strong Go backend experience from HPE, where I've built and maintained production microservices, fixed security vulnerabilities, and worked across the full stack. I also have breadth across Swift, TypeScript, and infrastructure."

### Strong Answer (Score: 5)
"Three things differentiate me. First, I build production systems end-to-end — not just the happy path. At HPE, I owned wellnessProducer from API design through security hardening to monitoring. I don't ship features without thinking about security, observability, and failure modes.

Second, I go beyond what's asked. I built automation tools (sprint review generator, sprint assignment) that save the team hours per sprint. I proactively identified and fixed 13+ security vulnerabilities that weren't on anyone's roadmap.

Third, I have genuine breadth. I'm not just a Go developer — I've built native macOS apps in Swift (ClipStash, Kairo), contributed to open-source (DiceDB), and I'm building a time series database (Crono). Each project teaches patterns I bring back to my main work."

---

## 10. "What's your biggest weakness?"

### Weak Answer (Score: 2)
"I'm a perfectionist" or "I work too hard."

### Acceptable Answer (Score: 3)
"I sometimes spend too long on technical elegance before shipping. I've learned to scope iterations — ship the MVP, then refine."

### Strong Answer (Score: 5)
"I've historically under-invested in automated integration tests. At HPE, I caught the JWT issuer bug during a manual review, not automated testing. That experience taught me that security-critical paths need integration tests that verify configuration alignment (e.g., Istio config matches middleware config). I've since added more integration tests and regression tests for security fixes, but I still think there's a gap — I'd love to implement contract testing or chaos engineering for auth paths."

---

## 11. "How do you handle technical debt?"

### Weak Answer (Score: 2)
"We try to keep the code clean."

### Acceptable Answer (Score: 3)
"I identify tech debt during code reviews and sprint planning. At HPE, we had a dedicated security sprint where I addressed 13+ validation issues. I also refactored the metrics system from separate counters to a unified metric."

### Strong Answer (Score: 5)
"I categorize tech debt by risk: security debt (hardcoded JWT issuers) gets immediate attention; design debt (separate vs unified metrics) gets planned in sprints; cosmetic debt (code style) gets fixed opportunistically in related PRs.

The ESO migration is a good example — 3 services migrated, 17 remaining. I documented the migration pattern so any team member can complete it. The 17 remaining aren't urgent (Vault still works) but represent operational debt. I track and advocate for allocating sprint capacity to it."

---

## 12. "Explain a system design trade-off you made."

### Weak Answer (Score: 2)
"We chose MongoDB because it's good for our use case."

### Acceptable Answer (Score: 3)
"For the Nimble IP cache, I chose SQLite over Redis. SQLite has zero external dependencies and survives pod restarts. Redis would add an external component to manage. The throughput was sufficient for our use case."

### Strong Answer (Score: 5)
"The Nimble GTCacheAPI trade-off is instructive. Options were: (1) Redis — fast, shared across instances, but adds an external dependency and operational burden, (2) SQLite — embedded, zero-ops, persistent, but single-instance only, (3) In-memory — fastest but lost on restart.

I chose SQLite because: single DownloadGatekeeper instance (no sharing needed), GTCaaS is rate-limited (caching prevents hitting limits, not performance-critical), data should survive restarts (avoid cold-cache stampede), and zero operational overhead.

When would I switch? If we scaled to multiple DownloadGatekeeper replicas (need shared cache → Redis), if cache miss latency became critical (in-memory tier in front of SQLite), or if data exceeded single-disk capacity (Redis cluster or DynamoDB)."

---

## 13. "How do you approach learning a new technology?"

### Weak Answer (Score: 2)
"I read the documentation and try it out."

### Acceptable Answer (Score: 3)
"I build something with it. I learned Swift by building ClipStash (clipboard manager) and Kairo (calendar). I learn best by applying to real projects, not tutorials."

### Strong Answer (Score: 5)
"Three-phase approach: First, build something small and real — not a tutorial project. I learned Swift 6 by building ClipStash (a full macOS app with GRDB, SwiftUI, and snapshot testing). Real projects surface real problems.

Second, go deep on one specific area. For Rust, I'm porting Kairo to Tauri — this forces me to learn Rust's ownership model in the context of IPC with a UI.

Third, bring patterns back to main work. ClipStash taught me about floating panel architecture that informed how I think about UI feedback in RAVE's Grafana dashboards. DiceDB taught me about high-performance Go patterns that I applied to wellnessProducer."

---

## 14. "Walk me through debugging a performance issue."

### Weak Answer (Score: 2)
"I'd look at the logs and try to find what's slow."

### Acceptable Answer (Score: 3)
"I'd check Prometheus metrics for latency spikes, identify the slow endpoint, check MongoDB query performance with explain(), look for missing indexes, and check if Kafka consumption is falling behind."

### Strong Answer (Score: 5)
"Systematic approach: (1) Define the symptom — which metric (latency, error rate, throughput) is degraded? (2) Isolate the component — Grafana dashboards per service, check if it's wellnessProducer or crmRelay or MongoDB. (3) Check the obvious — pod resource limits (OOM?), connection pool exhaustion, recent deployment. (4) Dive deep — MongoDB explain() for slow queries, Go pprof for CPU/memory profiling, Kafka consumer lag for backpressure. (5) Fix and verify — deploy fix, watch metrics recover, add alerts to catch recurrence.

A real example: I noticed CRM relay failures spiking. The Grafana dashboard showed `crm_relay_failure_total` increasing. Root cause: the CRM API was returning slower responses, causing our attachment uploads to timeout. I added the retry logic with exponential backoff, and the failure rate dropped back to baseline."

---

## 15. "Where do you see yourself in 5 years?"

### Weak Answer (Score: 2)
"I want to be a senior engineer or maybe a manager."

### Acceptable Answer (Score: 3)
"I want to be a Staff-level engineer owning a significant distributed system, ideally in an area that combines backend engineering with systems thinking — databases, infrastructure, or developer tools."

### Strong Answer (Score: 5)
"I'm drawn to the intersection of systems engineering and developer experience. Building Crono (time series DB) is exploring the systems side — storage engines, indexing, and performance at the data layer. The automation tools I build (sprint skills, Lucene search) explore the developer experience side.

In 5 years, I want to be designing systems that other engineers build on — whether that's infrastructure (databases, messaging systems), platforms (developer productivity tools), or core services at scale. I value depth over breadth in my career growth, even though I explore broadly in side projects."
