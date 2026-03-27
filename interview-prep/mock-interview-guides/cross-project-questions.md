# Cross-Project Questions — Connecting the Dots

> Questions that require synthesizing knowledge across multiple projects.
> These test breadth, pattern recognition, and genuine understanding.

---

## 1. Kafka: RAVE Classic (Sarama) vs Cloud (kafka-go)

### Question
"You used Kafka in both RAVE Classic and RAVE Cloud with different libraries. Compare the reliability guarantees and what you'd choose for a new project."

### Expected Answer
**RAVE Classic — Shopify/Sarama:**
- Callback-based consumer group handler (Setup, Cleanup, ConsumeClaim)
- Manual offset commit via `MarkMessage` — gives explicit control over acknowledgment
- If processing fails, I skip the message but don't commit — it gets redelivered
- More boilerplate, more control
- Shopify archived the repo — maintenance concern

**RAVE Cloud — segmentio/kafka-go:**
- Imperative `ReadMessage` loop with context
- Auto-commit on read — simpler but less control
- If processing fails, the offset is already committed — message is lost unless I handle it
- Cleaner Go idioms, context-native

**For a new project:** I'd choose kafka-go with explicit commit mode (`FetchMessage` + `CommitMessages`) for the best of both — clean API with manual commit control. Sarama is archived and has a steeper learning curve.

**Cross-project insight:** The reliability gap matters for compliance-critical systems like Nimble. If a download compliance check message is lost, a user might download software they shouldn't. For RAVE wellness, losing a case creation event is bad but recoverable (retry or manual creation).

[FOLLOW_UP] "How would you add exactly-once semantics?"
[FOLLOW_UP] "What about Kafka transactions?"
[SCORING: 1-5]
- 3: Knows both libraries exist
- 5: Code-level comparison, explicit/auto commit trade-offs, reliability implications per domain

---

## 2. Auth: wellnessProducer vs Nimble DownloadGatekeeper

### Question
"Compare how authentication works in wellnessProducer vs the Nimble DownloadGatekeeper. What's similar and what's different?"

### Expected Answer
**wellnessProducer (RAVE Cloud):**
- JWT-based auth via Istio RequestAuthentication + Go middleware
- Tokens from HPE GreenLake SSO (arubathena.com / greenlake.hpe.com)
- Multi-version support (v1.0 legacy, v1.1+ modern, v3 environment-specific)
- Extracts HPE claims (workspace_id, principal_type) for multi-tenant scoping
- Both user tokens and service tokens (hpe_identity_type: service-identity)

**Nimble DownloadGatekeeper:**
- IP-based + account-based validation (not JWT)
- Checks IP geolocation against embargo lists
- Checks license/account status against GTCaaS
- No user-facing auth — the download URL itself is the "credential"
- Device-level authorization via device shadow status (ONBOARDED, BLOCKED)

**Similarities:**
- Both enforce access control before allowing an action (case creation / download)
- Both have caching to avoid redundant auth checks (JWT claims parsing / SQLite IP cache)
- Both fail-closed on error (deny access if auth service is unreachable)

**Differences:**
- wellnessProducer: identity-based (who are you?) — JWT claims
- DownloadGatekeeper: compliance-based (are you allowed?) — geographic + regulatory
- wellnessProducer: Istio + application dual-layer
- DownloadGatekeeper: pure application-level (no service mesh)

[FOLLOW_UP] "If you needed to add JWT auth to Nimble, how would you do it?"
[SCORING: 1-5]
- 3: Can describe each system's auth independently
- 5: Contrasts identity-based vs compliance-based, fail-closed pattern, caching parallels

---

## 3. Go Patterns Across RAVE, Nimble, DiceDB

### Question
"You've written Go across multiple projects. What patterns do you reuse and what do you adapt?"

### Expected Answer
**Reused patterns:**
- **Repository pattern**: All projects use interfaces for data access (MongoRepository, SQLiteCache)
- **Error wrapping**: `fmt.Errorf("context: %w", err)` everywhere
- **Context propagation**: All functions take `context.Context` as first parameter
- **Constructor injection**: `NewService(repo, kafka, auth)` — no DI frameworks

**Adapted patterns:**
- **HTTP framework**: RAVE uses Gin, Surmai uses PocketBase (built-in), DiceDB has custom TCP handler
- **Database**: MongoDB (RAVE), SQLite (Nimble, ClipStash), custom storage engine (Crono)
- **Concurrency**: RAVE uses goroutine-per-message, DiceDB uses connection-per-client with a shared command processor

**DiceDB-specific:**
- More low-level: direct TCP handling, custom protocol parsing
- Memory management matters — in-memory DB can't leak goroutines
- Testing is more unit-focused (command-level) vs RAVE's integration tests

**What I've learned across projects:**
Go's stdlib is enough for most things. I avoid frameworks when possible (RAVE uses Gin for routing convenience, but most logic is stdlib). The strongest pattern is interfaces for testing — it works across every project.

[FOLLOW_UP] "How do you decide when a framework (Gin) is justified vs stdlib?"
[SCORING: 1-5]
- 3: Can name patterns used in each project
- 5: Explains WHY patterns are reused, what's adapted and why, cross-project learning

---

## 4. MongoDB: RAVE Wellness Objects vs Nimble Device Shadows

### Question
"You use MongoDB in both RAVE and Nimble. Compare the data models and why MongoDB is the right choice for each."

### Expected Answer
**RAVE — Wellness Objects:**
```go
type WellnessModel struct {
    WorkspaceID  string       `bson:"workspace_id"`
    CaseNumber   string       `bson:"case_number"`
    Status       string       `bson:"status"`
    Priority     Priority     `bson:"priority"`      // Nested document
    Contacts     []Contact    `bson:"contacts"`       // Array of embedded docs
    Attachments  []Attachment `bson:"attachments"`    // Variable-length array
    ManualCaseData *ManualCaseData `bson:"manual_case_data,omitempty"` // Optional
}
```
- Flexible schema: different case types have different fields
- Deeply nested documents (priority, contacts, attachments)
- Query pattern: always scoped by workspace_id + case_number
- Index: compound `{workspace_id: 1, case_number: 1}`

**Nimble — Device Shadows:**
```go
type DeviceShadow struct {
    SerialNumber string    `bson:"serial_number"`
    State        string    `bson:"state"` // PENDING, ONBOARDED, BLOCKED
    RPLResult    string    `bson:"rpl_result"`
    LastScreened time.Time `bson:"last_screened"`
    Metadata     bson.M    `bson:"metadata"` // Variable device attributes
}
```
- Simpler document structure, fewer nesting levels
- State machine pattern: state transitions are the primary operation
- Query pattern: lookup by serial_number
- Index: `{serial_number: 1}` unique

**Why MongoDB for both:**
- RAVE: Flexible schema handles varying case types, embedded documents avoid JOINs for contacts/attachments, workspace-scoped access pattern is natural for document DB
- Nimble: Device metadata varies by device type — flexible schema avoids rigid migrations. State machine fits document updates.

**Key difference:** RAVE documents are read-heavy (dashboards, case views) with complex queries. Nimble documents are write-heavy (state transitions) with simple lookups.

[FOLLOW_UP] "When would you NOT use MongoDB?"
[FOLLOW_UP] "How would you handle a cross-workspace report in RAVE?"
[SCORING: 1-5]
- 3: Can describe each data model
- 5: Contrasts access patterns, explains why MongoDB fits each, knows indexing for both

---

## 5. State Machines: Nimble Device Shadow vs RAVE Case Status

### Question
"Both Nimble and RAVE have state management. Compare the state machines."

### Expected Answer
**Nimble Device Shadow States:**
```
PENDING → (RPL clear) → ONBOARDED
PENDING → (RPL match) → BLOCKED
ONBOARDED → (periodic re-screen) → ONBOARDED | BLOCKED
BLOCKED → (manual review) → ONBOARDED
```
- 4 states: PENDING, ONBOARDED, FAILED, BLOCKED
- Transitions driven by external compliance screening results
- State determines download permission (ONBOARDED = allowed, BLOCKED = denied)
- Irreversible in most cases (BLOCKED requires manual intervention)

**RAVE Case Status:**
```
open → (agent update) → in_progress
open → (auto-close) → closed
in_progress → (resolution) → closed
closed → (new event) → reopened
reopened → (resolution) → closed
```
- States: open, in_progress, closed, reopened
- Transitions driven by both automated events and manual actions
- Any state can eventually transition to any other (more fluid)
- Status affects monitoring metrics and dashboard display

**Comparison:**
| Aspect | Nimble | RAVE |
|--------|--------|------|
| Transition source | External (GTCaaS) | Mixed (automation + human) |
| Reversibility | Low (BLOCKED needs review) | High (cases reopen) |
| Security implication | High (determines compliance) | Medium (affects support SLA) |
| Implementation | MongoDB field update | MongoDB field + Kafka event |

[FOLLOW_UP] "How do you prevent invalid state transitions?"
[SCORING: 1-5]
- 3: Can describe each state machine
- 5: Contrasts transition drivers, reversibility, security implications

---

## 6. Caching Strategies: SQLite (Nimble) vs In-Memory (wellnessProducer)

### Question
"You've used different caching strategies across projects. Compare them."

### Expected Answer
**Nimble GTCacheAPI — SQLite:**
- Persistent local cache for IP geolocation and license status
- TTL-based invalidation
- Why SQLite: zero external dependencies, embedded, sufficient for single-instance throughput
- 60%+ reduction in GTCaaS API calls
- Survives pod restart (data persists on disk)

**wellnessProducer — In-Memory:**
- JWT claims parsing cached in Go structs (context per request)
- Prometheus metrics registry (global in-memory state)
- No persistent cache — each request re-validates from source (MongoDB)
- Acceptable because MongoDB queries are fast with proper indexes

**When to use each:**
- SQLite: External API calls that are expensive/rate-limited, data that should survive restarts
- In-memory: Per-request data, global configuration, metrics
- Redis (not used, but would consider): Multi-instance shared cache, high-throughput reads

[FOLLOW_UP] "When would you migrate from SQLite to Redis?"
[SCORING: 1-5]
- 3: Can describe each approach
- 5: Trade-off analysis, persistence vs performance, when to upgrade

---

## 7. Security Patterns Across Projects

### Question
"You've fixed multiple security issues. What patterns do you see across your projects?"

### Expected Answer
**Common security patterns:**
1. **Never trust client-supplied identity data**: RAVE workspace_id must come from JWT, not request body. Nimble device authorization comes from server-side state, not the download request.
2. **Defense in depth**: File uploads have 5 validation layers. Auth has 2 layers (Istio + app). Compliance has 3 checks (RPL + ELC + IP geo).
3. **Fail-closed**: Nimble denies downloads if GTCaaS is unreachable. RAVE denies access if JWT validation fails. Better to block a legitimate user temporarily than allow an unauthorized one.
4. **Input validation at every boundary**: Filename sanitization, email format validation, severity allowlist, magic byte verification.
5. **Least privilege**: ESO uses per-service IAM roles (IRSA), not shared credentials. Each ServiceAccount only accesses its own secrets.

**Cross-project application:** The "never trust client data" principle applies everywhere — RAVE, Nimble, and even side projects. In ClipStash, clipboard data from external apps is sanitized before display to prevent UI injection.

[FOLLOW_UP] "How do you bake security into the development process vs. fixing it after?"
[SCORING: 1-5]
- 3: Lists security fixes
- 5: Identifies patterns across projects, articulates principles, discusses process

---

## 8. Monitoring Across RAVE Variants

### Question
"You have RAVE Cloud, Classic, and LVS. How do you ensure consistent monitoring across all three?"

### Expected Answer
**RAVE Cloud:** Prometheus client_golang, Grafana dashboards, AlertManager → PagerDuty/Slack. I built the monitoring overhaul here.

**RAVE Classic:** Also Prometheus, but different metric names (legacy). I upgraded monitoring to align metrics across variants. Uses Shopify/Sarama Kafka metrics.

**RAVE LVS:** Similar to Cloud, deployed via Helm with matching Prometheus annotations. ESO for secrets.

**Consistency approach:**
- Unified metric naming conventions (adopted mc_event_case_action_total pattern across variants)
- Shared Grafana dashboards with data source switching per environment
- Humio/LogScale for centralized logging across all variants
- PagerDuty routing rules per variant + shared escalation policy

**Challenge:** Classic uses different Kafka libraries and DB connections (PostgreSQL, Vertica) — metric labels differ. I had to create mapping functions in Grafana to normalize across variants.

[FOLLOW_UP] "How do you handle alert fatigue across 3 variants × 3 environments?"
[SCORING: 1-5]
- 3: Knows monitoring exists in each variant
- 5: Explains normalization challenges, unified approach, cross-variant Grafana techniques

---

## 9. Language Selection Across Projects

### Question
"You've used Go, Python, Swift, TypeScript, Java, and Rust. How do you decide which language to use?"

### Expected Answer
| Project | Language | Why |
|---------|----------|-----|
| RAVE Cloud | Go | Team standard, microservice fit, fast compilation, static binaries |
| Wellness Proxy | Python (FastAPI) | Routing logic, rapid development, Python ecosystem for data parsing |
| ClipStash | Swift | macOS native app, SwiftUI requirement, Apple framework integration |
| Kairo | Swift + Rust (Tauri) | Swift for native macOS, Rust for cross-platform port |
| Surmai | Go + TypeScript | Go backend (PocketBase), React/TS frontend |
| DiceDB | Go | Existing project language, performance for in-memory DB |
| Lucene Search | Java | Spring Boot + Vaadin for search analysis UI |

**Decision framework:**
1. **Platform constraint**: macOS → Swift, cross-platform desktop → Rust/Tauri
2. **Team/project standard**: Existing codebase → match the language
3. **Performance needs**: High-throughput → Go or Rust; scripting → Python
4. **Ecosystem**: Search/analytics → Java (Lucene); web frontend → TypeScript
5. **Personal growth**: Rust (Kairo Tauri) was partly to learn Rust in a real project

[FOLLOW_UP] "If you could rewrite RAVE in any language, which and why?"
[SCORING: 1-5]
- 3: Can explain language choice per project
- 5: Articulates a decision framework, discusses trade-offs, personal growth angle

---

## 10. Infrastructure Patterns: RAVE vs Grouped vs Yardi

### Question
"You've worked with K8s/Istio at HPE, AWS ECS at Grouped, and on-prem at Yardi. Compare the infrastructure approaches."

### Expected Answer
| Aspect | HPE RAVE | Grouped | Yardi |
|--------|---------|---------|-------|
| Orchestration | K8s + Istio + ArgoCD | AWS ECS + Fargate | On-prem IIS |
| Deployment | GitOps (ArgoCD) | CI/CD + ECS deploy | Manual deploy |
| Service mesh | Istio (RequestAuth, VirtualService) | ALB routing | N/A |
| Secrets | ESO (was Vault) | AWS Secrets Manager | Config files |
| Scaling | HPA + K8s native | Fargate auto-scaling (5 instances) | Manual |
| Monitoring | Prometheus/Grafana/PagerDuty | CloudWatch | Basic logging |
| Databases | MongoDB, PostgreSQL | RDS (PostgreSQL) | MSSQL |

**Evolution of my infrastructure skills:**
- Yardi: Traditional on-prem, learned database fundamentals (T-SQL, MSSQL)
- Grouped: Cloud-native (AWS), learned ECS/Fargate, multi-AZ design
- HPE: Full Kubernetes ecosystem — Istio, Helm, ArgoCD, ESO, Terraform

Each role deepened my understanding. Yardi taught me database reliability matters. Grouped taught me cloud scaling. HPE taught me that orchestration complexity grows non-linearly with service count.

[FOLLOW_UP] "What's the biggest operational lesson from each role?"
[SCORING: 1-5]
- 3: Can describe each infrastructure
- 5: Shows progression, contrasts operational lessons, connects experiences

---

## 11. AI Agent Architecture: Sahaya vs Genie Wellness Agent

### Question
"You've built two LLM-powered agent systems — Sahaya (personal) and the Genie Wellness Agent (HPE). Compare the architecture, design decisions, and trade-offs."

### Expected Answer
| Aspect | Sahaya | Genie Wellness Agent |
|--------|--------|---------------------|
| Framework | LangGraph (multi-agent with domain subgraphs) | Custom FastAPI + LLM prompts |
| Agent model | Multi-agent with intent routing | Single-purpose agent with specialized prompts |
| LLM | Gemini | LLM (via HPE infra) |
| Memory | pgvector semantic search + keyword fallback | Stateless (per-request) |
| Deployment | Docker Compose on Raspberry Pi ARM64 | Multi-cluster K8s (5 clusters) |
| Scale | Single user, self-hosted | Enterprise, multi-cluster |
| Streaming | SSE for real-time responses | Synchronous responses |

**Key insight**: Sahaya needed multi-agent coordination (chat, tasks, meals, calendar, wellness — each with different tools). Genie was single-purpose (wellness event analysis) so a simpler architecture with well-tuned prompts was more appropriate. Over-engineering with LangGraph would have added latency without benefit for Genie's focused use case.

[FOLLOW_UP] "When would you choose multi-agent vs single-agent architecture?"
[SCORING: 1-5]
- 3: Can describe both systems
- 5: Articulates why different architectures suited different problems, discusses trade-offs

---

## 12. Deployment Spectrum: K8s at Scale vs Raspberry Pi

### Question
"You deploy RAVE to enterprise Kubernetes clusters with Istio and ArgoCD, but you also deploy Sahaya to a Raspberry Pi. Compare these deployment experiences."

### Expected Answer
| Aspect | RAVE (Enterprise K8s) | Sahaya (Raspberry Pi) |
|--------|----------------------|----------------------|
| Orchestration | K8s + Istio + ArgoCD + HPA | Docker Compose (simplified from K3s) |
| Networking | Istio VirtualService, RequestAuthentication | Cloudflare Tunnel |
| Secrets | ESO → AWS Parameter Store | .env file + Docker secrets |
| CI/CD | GitHub Actions + SonarQube + ArgoCD GitOps | `make ship` (build + push + deploy) |
| Scale | HPA auto-scaling, multi-cluster | Single instance, ARM64 |
| Monitoring | Prometheus + Grafana + PagerDuty | Docker logs |

**Key lesson**: I initially tried K3s on the Pi (cargo-culting enterprise patterns). K3s worked but added unnecessary complexity — restarts took 2+ minutes, resource usage was high. Docker Compose + Cloudflare tunnel achieved the same result with 10x simpler operations. The lesson: match infrastructure complexity to actual requirements.

[FOLLOW_UP] "What made you switch from K3s to Docker Compose? At what scale would you switch back?"
[SCORING: 1-5]
- 3: Can describe both environments
- 5: Shows judgment about when to simplify vs when to use full orchestration

---

## 13. FastAPI Across Projects: Wellness Proxy vs Sahaya vs Genie

### Question
"You've used FastAPI in three different contexts — the Wellness Proxy, Sahaya backend, and Genie Wellness Agent. How did your usage evolve?"

### Expected Answer
1. **Wellness Proxy**: Simple routing proxy — async endpoints forwarding requests to Nimble/HPE APIs. Cross-launch token handling. Minimal — ~20 files.
2. **Genie Wellness Agent**: LLM-powered API — FastAPI + Gunicorn serving AI analysis endpoints. Custom prompts, MCP tools integration. Medium complexity.
3. **Sahaya**: Full application — app factory pattern, SQLAlchemy ORM integration, Alembic migrations, SSE streaming endpoints, WebSocket support, dependency injection for repos, 87+ endpoints.

**Evolution**: Proxy → AI agent API → full application framework. Each use pushed me to learn deeper FastAPI patterns: from basic routing to async SQLAlchemy sessions, SSE streaming generators, and middleware chains.

[FOLLOW_UP] "What's the most advanced FastAPI pattern you've used?"
[SCORING: 1-5]
- 3: Can describe each project's FastAPI usage
- 5: Shows clear progression and deeper patterns learned at each stage

---

## 14. Performance Debugging: RAVE Case Creation vs System Design

### Question
"Walk me through the case creation performance investigation. If you were designing this system from scratch, how would you prevent the P90 degradation?"

### Expected Answer
**Investigation**: P90 went from 8.8s (1 user) to 41.3s (10 concurrent). I broke down the pipeline:
- WellnessProducer: 24ms (<1%) — not the bottleneck
- Kafka transit: 657ms — reasonable
- WellnessManagement: dominant — sequential processing, single Kafka partition

**Root cause**: All Kafka messages went to the same partition (default key hashing). crmRelay consumers couldn't parallelize.

**Fix**: Used updatedMetadata.UUID as partition key → even distribution → horizontal scaling.

**If designing from scratch**: 
1. Explicit partition strategy from day one (hash on case ID or tenant)
2. Rate limiting per tenant at the API gateway
3. Async acknowledgment pattern (accept request, return ID, process asynchronously)
4. Circuit breaker on CRM calls to prevent cascade failures
5. Pre-allocated consumer groups sized for expected concurrency

[FOLLOW_UP] "What if the CRM downstream is the bottleneck, not Kafka?"
[SCORING: 1-5]
- 3: Can describe the investigation
- 5: Connects to system design principles, proposes preventive architecture
