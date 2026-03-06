# System Design Questions — Interview Prep

> Based on Tejas Gajare's actual work at HPE GLCP Wellness/RAVE team.
> Each question maps to a real system I designed, built, or contributed to.

---

## 1. Design a Wellness Automation Engine (RAVE Cloud)

### Problem Statement
Design a system that monitors HPE infrastructure devices, detects health anomalies, and automatically creates support cases. The system must support multiple tenants (workspaces), integrate with CRM systems, handle email-based case routing, and provide APIs for manual case creation. Scale: 20+ microservices, 3 environments (dev/intg/prod), multi-cloud deployment on Kubernetes.

### Expected Approach

**Step 1 — Clarify Requirements**
- Ingestion: How do events arrive? (Kafka topics from device telemetry)
- Processing: What rules determine case creation? (Wellness rules engine)
- Output: Where do cases go? (CRM API for HPE domains, SendGrid email for others)
- Auth: Multi-tenant with workspace isolation
- Scale: Thousands of devices per workspace, real-time processing

**Step 2 — High-Level Architecture**
```
[Device Telemetry] → [Kafka] → [Wellness Analyzer]
                                      ↓
[wellnessProducer API] ← [User/Service] → [MongoDB]
        ↓                                     ↓
[caseRelay (email)] ←→ [crmRelay (CRM API)] → [External CRM]
        ↓
[Monitoring: Prometheus → Grafana → PagerDuty/Slack]
```

**Step 3 — Key Components**
- **wellnessProducer**: Go microservice handling REST APIs (v1/v2/v3), JWT auth, file uploads, case CRUD
- **caseRelay**: Routes cases to email domain (SendGrid) with retry logic
- **crmRelay**: Routes cases to CRM domain (DCM API) with attachment retry for race conditions
- **MongoDB**: Stores wellness objects, case metadata, attachments
- **Kafka**: Event streaming between services (segmentio/kafka-go in Cloud, Sarama in Classic)
- **Istio**: Service mesh for inter-service auth, traffic routing, RequestAuthentication

**Step 4 — Data Model**
```go
type WellnessModel struct {
    WorkspaceID   string        `bson:"workspace_id"`
    CaseNumber    string        `bson:"case_number"`
    Status        string        `bson:"status"` // open, closed, reopened
    Priority      Priority      `bson:"priority"`
    PrimaryContact Contact     `bson:"primary_contact"`
    Attachments   []Attachment  `bson:"attachments"`
    CreatedBy     string        `bson:"created_by"` // JWT sub claim
    ManualCaseData *ManualCase  `bson:"manual_case_data,omitempty"`
}
```

### Trade-offs
| Decision | Chosen | Alternative | Why |
|----------|--------|------------|-----|
| Kafka vs REST | Kafka for async | Direct REST calls | Decouples services, handles backpressure |
| MongoDB vs SQL | MongoDB | PostgreSQL | Flexible schema for varying wellness data shapes |
| Email routing | Domain-based split | Single CRM | Some tenants don't have HPE CRM access |
| Auth | JWT middleware per version | API gateway auth | Version-specific token formats needed |

### Follow-ups
- [FOLLOW_UP] "How do you handle cases where the CRM is down?" → Retry queue with exponential backoff in crmRelay, attachment retry logic for race conditions
- [FOLLOW_UP] "How does multi-tenancy work?" → Workspace ID derived server-side from JWT claims, never trusted from client
- [FOLLOW_UP] "How would you scale this 10x?" → Kafka partition scaling, MongoDB sharding by workspace_id, horizontal pod autoscaling

---

## 2. Design a File Upload System with Content Validation

### Problem Statement
Design a file upload system for a support case management platform. Users upload attachments (logs, screenshots, documents) to support cases. The system must prevent malicious uploads while supporting 60+ file types. Files are forwarded to external CRM systems.

### Expected Approach

**Step 1 — Threat Model**
- Path traversal (../../etc/passwd)
- Extension spoofing (.exe renamed to .pdf)
- Content-type mismatch (JS in an image)
- Oversized files (DoS)
- Malicious filenames (XSS, null bytes)

**Step 2 — Multi-Layer Validation Pipeline**
```
[Upload Request]
    → Layer 1: Filename sanitization (strip path, null bytes, special chars)
    → Layer 2: Extension allowlist (60+ types: pdf, png, log, csv, etc.)
    → Layer 3: Content-type header check
    → Layer 4: Magic byte verification (first N bytes match expected file type)
    → Layer 5: File size limits (per-file and total)
    → [MongoDB GridFS / blob storage] → [CRM API forward]
```

**Step 3 — Magic Byte Implementation**
```go
var magicBytes = map[string][]byte{
    "pdf":  {0x25, 0x50, 0x44, 0x46}, // %PDF
    "png":  {0x89, 0x50, 0x4E, 0x47}, // .PNG
    "zip":  {0x50, 0x4B, 0x03, 0x04}, // PK..
    "jpg":  {0xFF, 0xD8, 0xFF},        // ÿØÿ
}

func validateFileContent(file []byte, ext string) error {
    expected, ok := magicBytes[ext]
    if !ok {
        return nil // Extension not in magic byte registry — allow (validated by allowlist)
    }
    if !bytes.HasPrefix(file, expected) {
        return ErrContentTypeMismatch
    }
    return nil
}
```

**Step 4 — Filename Sanitization**
```go
func sanitizeFilename(name string) string {
    name = filepath.Base(name)          // Strip path components
    name = strings.ReplaceAll(name, "\x00", "") // Remove null bytes
    name = regexp.MustCompile(`[<>:"/\\|?*]`).ReplaceAllString(name, "_")
    if len(name) > 255 { name = name[:255] }
    return name
}
```

### Trade-offs
| Decision | Chosen | Alternative | Why |
|----------|--------|------------|-----|
| Magic bytes | Inline check | ClamAV scan | Speed vs thoroughness — ClamAV adds latency |
| Storage | MongoDB + forward to CRM | S3 | CRM needs the file directly, no need for S3 |
| Allowlist | Explicit allowlist | Blocklist | Safer — unknown types rejected by default |
| Size limits | Per-file + total | Just total | Prevents single giant file |

### Follow-ups
- [FOLLOW_UP] "What about zip bombs?" → Check compressed vs uncompressed ratio, limit nesting depth
- [FOLLOW_UP] "How do you handle concurrent uploads?" → Mutex per case ID, or optimistic concurrency with MongoDB upsert
- [FOLLOW_UP] "What about polyglot files?" → Magic bytes mitigate but don't eliminate — defense in depth

---

## 3. Design a Multi-Tenant Case Management System

### Problem Statement
Design a system where multiple organizational tenants (workspaces) can create, view, and manage support cases. Cases contain structured data, contacts, attachments, and must be routed to different backends based on the tenant's email domain. Critical requirement: absolute tenant isolation — no cross-workspace data leakage.

### Expected Approach

**Step 1 — Tenant Isolation Model**
```
[JWT Token] → Extract workspace_id from claims (server-side only)
           → Never trust client-supplied workspace identifiers
           → All database queries scoped: db.Find({"workspace_id": ws_id})
```

**Step 2 — Case Creation Flow**
```
Client → wellnessProducer API (JWT auth) → Validate & enrich
    → Determine routing:
        Email domain (e.g., @gmail.com) → caseRelay → SendGrid
        HPE domain (e.g., @hpe.com)     → crmRelay → DCM CRM API
    → Persist in MongoDB (wellness_objects collection)
    → Return case reference to client
```

**Step 3 — Security-Critical Design Decisions**
- Workspace ID MUST come from JWT `hpe_workspace_id` claim, not request body
- v2 had a bug (GLCP-333624) where client-supplied workspace was trusted — I fixed this
- Server-side input validation: all fields validated (required contacts, valid email format, allowed priority values)
- CreatedBy field extracted from JWT `sub` claim for audit trail

**Step 4 — Data Flow for Manual Case**
```go
type ManualCaseData struct {
    Subject     string   `json:"subject" validate:"required,max=200"`
    Description string   `json:"description" validate:"required,max=5000"`
    Severity    string   `json:"severity" validate:"oneof=critical high medium low"`
    Category    string   `json:"category"`
}
```

### Trade-offs
- **Database-per-tenant vs shared-with-scoping**: Chose shared MongoDB with query scoping. Trade-off: simpler ops, but must be extremely careful with every query.
- **Routing by email domain**: Simple but inflexible — what if a tenant uses both domains? Added configurable domain mapping.

### Follow-ups
- [FOLLOW_UP] "How do you prevent a developer from accidentally writing an unscoped query?" → Code review, middleware that injects workspace filter, integration tests
- [FOLLOW_UP] "What about cross-workspace reporting?" → Separate admin API with super-admin JWT claims, never exposed to regular tenants
- [FOLLOW_UP] "How do you handle tenant offboarding?" → Soft delete with TTL, audit log of all access

---

## 4. Design a Global Trade Compliance Gateway (Nimble Download)

### Problem Statement
Design a system that enforces US export control regulations on software downloads. Before any user downloads Nimble Storage software, the system must verify: (1) the device is properly onboarded and not on restricted party lists, (2) the download request doesn't originate from an embargoed country, and (3) the user's license/account is valid. Must handle device lifecycle from discovery through onboarding with external compliance screening.

### Expected Approach

**Step 1 — Four-Microservice Architecture**
```
[Device Discovery via RPL Feed]
    → AutoOnboard Service
        → GTCaaS RPL Screening API
        → MongoDB (Device Shadow: PENDING → ONBOARDED | BLOCKED)
    
[Download Request]
    → DownloadGatekeeper
        → GTCacheAPI (IP geolocation + license check)
            → SQLite local cache (reduces redundant API calls by 60%+)
        → GTCaaS ELC Screening API
        → Allow / Deny download
```

**Step 2 — Device Shadow State Machine**
```
PENDING → (RPL clear) → ONBOARDED
PENDING → (RPL match) → BLOCKED
ONBOARDED → (periodic re-screen) → ONBOARDED | BLOCKED
BLOCKED → (manual review) → ONBOARDED
```

**Step 3 — IP Geolocation & Caching**
```go
type IPCacheEntry struct {
    IP         string    `db:"ip"`
    Country    string    `db:"geo_country"`
    LastCheck  time.Time `db:"last_check"`
    Result     string    `db:"result"` // allow, deny, embargo
    TTL        int       `db:"ttl_seconds"`
}
```
- SQLite for local cache (low-latency, no external dependency)
- 60%+ reduction in redundant API calls to GTCaaS
- Cache invalidation on TTL expiry or manual flush

**Step 4 — GTCaaS Migration (Neustar → MaxMind)**
- Moving from HPE Geolocation Service (Neustar) to MaxMind within GTCaaS
- Eliminates separate geo API call — IP passed directly to License Screening API
- geo_country column kept in cache but only populated on 403 embargo responses

### Trade-offs
| Decision | Chosen | Alternative | Why |
|----------|--------|------------|-----|
| Cache store | SQLite | Redis | Zero external deps, embedded, sufficient throughput |
| State machine | MongoDB document | PostgreSQL with constraints | Flexible schema for varying device attributes |
| Screening | External GTCaaS | In-house lists | Compliance liability — use certified provider |
| Architecture | 4 separate services | Monolith | Each has different scaling/SLA needs |

### Follow-ups
- [FOLLOW_UP] "What if GTCaaS is down?" → Cache serves stale results with warning, downloads blocked for uncached IPs (fail-closed)
- [FOLLOW_UP] "How do you handle false positives on RPL?" → Manual review workflow with BLOCKED → ONBOARDED transition
- [FOLLOW_UP] "How would you scale to 100x downloads?" → Horizontal scaling of DownloadGatekeeper, larger SQLite cache or migrate to Redis, pre-warm cache for known IP ranges

---

## 5. Design a Monitoring and Alerting System

### Problem Statement
Design a monitoring system for a microservices platform running 20+ services across 3 environments. Must track service health, business metrics (case creation rates, CRM success/failure), and provide automated alerting via Slack and PagerDuty. Must detect both infrastructure issues (pod crashes) and business logic issues (case creation pipeline failures).

### Expected Approach

**Step 1 — Metrics Taxonomy**
```
Infrastructure Metrics:
  - Pod status (running, crash-looping, OOMKilled)
  - Request latency (p50, p95, p99)
  - Error rates (4xx, 5xx)
  
Business Metrics:
  - mc_event_case_action_total{action="created|updated|reopened|new_open"}
  - case_relay_success_total / case_relay_failure_total
  - crm_relay_attachment_retry_total
  - wellness_producer_auth_failure_total
```

**Step 2 — Architecture**
```
[Go Services] → Prometheus client_golang → /metrics endpoint
    → [Prometheus Server] (scrape interval: 30s)
        → [Grafana Dashboards] (service health, business metrics)
        → [AlertManager]
            → [Slack webhooks] (warning)
            → [PagerDuty API] (critical)
```

**Step 3 — Unified Metrics Design**
I consolidated separate per-action metrics into a single counter with labels:
```go
// Before (bad): separate metrics per action
caseCreatedTotal = prometheus.NewCounter(...)
caseUpdatedTotal = prometheus.NewCounter(...)

// After (good): single metric with action label
mcEventCaseAction = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "mc_event_case_action_total",
        Help: "Manual case events by action type",
    },
    []string{"action"},
)
```

**Step 4 — Alert Categories**
1. **Service Stopped Running** — Warning: non-running pods detected. Critical: zero replicas.
2. **Processing Drops** — Case creation/automation rate drops below threshold for sustained period.
3. **CRM Failures** — crmRelay returning errors above threshold.
4. **Auth Failures** — Spike in JWT validation failures (potential attack or misconfiguration).

### Trade-offs
| Decision | Chosen | Alternative | Why |
|----------|--------|------------|-----|
| Metrics format | Single metric + labels | Separate metrics | Reduces cardinality, simplifies dashboards |
| Alerting | PagerDuty + Slack | Email | Real-time response needed |
| Scrape interval | 30s | 15s/60s | Balance between latency and load |
| Dashboard tool | Grafana | Datadog | Already in platform, cost-effective |

### Follow-ups
- [FOLLOW_UP] "How do you handle metric cardinality explosion?" → Limit label values (no user IDs in labels), use recording rules for expensive queries
- [FOLLOW_UP] "What's the on-call process?" → PagerDuty rotation, runbooks linked from alerts, escalation policy
- [FOLLOW_UP] "How do you distinguish between a real outage and a deployment?" → Deployment annotations in Grafana, alert suppression during rollouts

---

## General System Design Tips

### Framework for Any Question
1. **Clarify** — Ask about scale, users, SLAs, constraints
2. **High-level design** — Draw boxes and arrows, identify core components
3. **Deep dive** — Pick the hardest/most interesting component, design in detail
4. **Trade-offs** — Every decision has alternatives; articulate them
5. **Scale & failure** — How does it handle 10x load? What breaks first?
6. **Monitoring** — How do you know it's working? How do you detect failures?

### Common Patterns from My Work
- **Event-driven architecture** (Kafka) for decoupling and backpressure
- **State machines** (device shadow states) for lifecycle management
- **Multi-layer validation** (file uploads) for defense in depth
- **Server-side authorization** (workspace from JWT) over client-supplied data
- **Unified metrics with labels** over separate metric names
- **Fail-closed** for security-critical paths (trade compliance)
- **SQLite for local caching** when external cache isn't justified

[KEY_POINTS] Always connect system design answers to real implementations. Interviewers value "I built this" over "I would build this."
[COMMON_MISTAKES] Don't forget monitoring/alerting in your design. Don't hand-wave security ("we'd add auth later").
[SCORING: 1-5] 5 = connects to real implementation, explains trade-offs with alternatives, anticipates failures
