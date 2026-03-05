# RAVE Cloud Platform — System Architecture

## Overview

I work on RAVE (Rapid Automated Validation Engine), the wellness automation engine within HPE GreenLake Cloud Platform (GLCP). RAVE is a distributed system of 20+ Go microservices that monitors HPE infrastructure health, analyzes telemetry, and automates support case creation and management. The platform processes wellness events from thousands of devices across HPE's customer base, routing them through analysis pipelines that ultimately create support cases via email (SendGrid) or CRM (PAN HPE / DCM) integrations.

I've contributed across the full stack — JWT authentication, API versioning, manual case workflows, file upload security, monitoring, and cross-service security hardening.

---

## Service Topology

### Core Services (Go Microservices)

| Service | Role | Key Interactions |
|---------|------|-----------------|
| **wellnessProducer** | Primary API gateway. Receives wellness objects, validates, routes, persists. | MongoDB, Kafka, S3 |
| **caseRelay** | Email-based case creation via SendGrid. Handles email domain manual cases. | SendGrid API, Kafka, MongoDB |
| **crmRelay** | CRM-based case creation via PAN HPE / DCM API. Handles panhpe domain cases. | DCM CRM API, Kafka, MongoDB |
| **dataParser** | Parses and normalizes incoming telemetry data from various device types. | Kafka, MongoDB |
| **spamGuard** | Deduplication and rate-limiting for wellness events. Prevents case flooding. | MongoDB, Kafka |
| **healthChecker** | Monitors service health across the platform. Feeds Prometheus metrics. | Kubernetes API, Prometheus |
| **templateEditor** | Manages email and case templates for different device families and alert types. | MongoDB |
| **wellnessProxy** | Python/FastAPI proxy for cross-launch token support and Nimble route handling. | wellnessProducer, Nimble APIs |
| **deviceShadow** | Maintains current state representation of monitored devices. | MongoDB, Kafka |
| **alertProcessor** | Evaluates wellness rules against device telemetry to generate alerts. | Kafka, MongoDB |
| **notificationService** | Manages notification preferences and delivery channels. | SendGrid, Kafka |
| **configService** | Centralized configuration management for platform-wide settings. | MongoDB |
| **schedulerService** | Cron-based job scheduling for periodic health checks and report generation. | Kafka |
| **auditLogger** | Tracks all case actions, API calls, and state transitions for compliance. | MongoDB, Kafka |

### Supporting Infrastructure

- **MongoDB**: Primary data store for wellness objects, device shadows, case metadata, templates, configuration
- **Apache Kafka**: Event bus for inter-service communication (segmentio/kafka-go library)
- **Amazon S3**: Attachment storage for manual case file uploads
- **SendGrid**: Email delivery for case creation notifications and email-domain cases
- **PAN HPE / DCM CRM API**: Enterprise CRM integration for panhpe-domain case creation

---

## Data Flow: Automated Alerts

```
Device Telemetry
    │
    ▼
[dataParser] ──parse & normalize──▶ Kafka: wellness-events
    │
    ▼
[alertProcessor] ──evaluate rules──▶ Kafka: alerts
    │
    ▼
[spamGuard] ──deduplicate──▶ Kafka: validated-alerts
    │
    ▼
[wellnessProducer] ──persist──▶ MongoDB: wellnessObjects
    │
    ├──▶ [caseRelay] ──SendGrid──▶ Support Case (email)
    │
    └──▶ [crmRelay] ──DCM API──▶ Support Case (CRM)
```

**Why this architecture**: Each service has a single responsibility. spamGuard prevents case flooding (one device reporting the same error repeatedly shouldn't create 100 cases). dataParser normalizes across device families (Nimble, Alletra, ProLiant all have different telemetry formats). This separation lets us scale each service independently — caseRelay might need more replicas during an outage wave while dataParser stays steady.

---

## Data Flow: Manual Case Creation

```
Client (GLCP UI)
    │
    ▼
[wellnessProducer] ──POST /v3/wellness──▶ Validate request
    │                                        │
    │                                        ├── JWT auth (v1.1+ for v3)
    │                                        ├── ManualCaseData validation
    │                                        ├── Contact validation (primary email required)
    │                                        ├── Category validation (allowlisted values)
    │                                        └── Attachment validation (if present)
    │
    ├── domain: "email" ──▶ Kafka ──▶ [caseRelay] ──▶ SendGrid
    │                                                    │
    │                                                    ├── Build email with template
    │                                                    ├── CC: primary + alternate + createdBy
    │                                                    └── Attach files (deferred if uploading)
    │
    └── domain: "panhpe" ──▶ Kafka ──▶ [crmRelay] ──▶ DCM CRM API
                                                        │
                                                        ├── Create CRM case
                                                        ├── Add contacts
                                                        └── Attach files
```

See: [manual-case-workflow.md](./manual-case-workflow.md) for detailed validation pipeline.

---

## Communication Patterns

### Synchronous (REST APIs)

- **External**: Client → wellnessProducer (v1/v2/v3 endpoints)
- **Internal**: wellnessProducer → S3 (file uploads), caseRelay → SendGrid, crmRelay → DCM API
- **Cross-launch**: wellnessProxy → wellnessProducer (Nimble routes)

### Asynchronous (Kafka)

- **Event-driven**: All inter-service communication uses Kafka topics via `segmentio/kafka-go`
- **Topics**: wellness-events, alerts, validated-alerts, case-actions, case-status-updates
- **Consumer groups**: Each service has its own consumer group for independent processing
- **Why Kafka over direct REST**: Decouples services, enables replay, handles backpressure. If crmRelay is down during a DCM outage, messages queue in Kafka and process when it recovers. No data loss.

### Service Mesh (Istio)

- **mTLS**: All inter-service traffic encrypted via Istio sidecar proxies
- **RequestAuthentication**: JWT validation at the mesh level, configured per environment via Helm values
- **EnvoyFilter**: API version routing (v3 endpoints enforce v1.1+ tokens)
- **Traffic management**: Canary deployments, circuit breaking, retry policies

---

## Data Stores

### MongoDB

- **wellnessObjects**: Core collection — each document represents a wellness event with device info, alert details, case status
- **deviceShadows**: Current state of each monitored device (last seen, health score, active alerts)
- **caseMetadata**: Case lifecycle tracking (created, updated, resolved, attachments)
- **templates**: Email and case templates per device family
- **BSON handling**: I fixed a critical bug where `primitive.D` (ordered BSON documents) weren't being handled in `convertBsonDToMap`. MongoDB's Go driver v1.11.1 uses `primitive.A` (type definition over `[]interface{}`) for arrays and `primitive.D` for nested documents. The type switch wasn't catching `primitive.D`, causing silent data loss in nested document queries.

### Amazon S3

- **Attachment storage**: Files uploaded via manual case workflow
- **Status tracking**: Each attachment tracked through lifecycle: `fileUploading` → `fileUploaded` → `fileAttachedToCase`
- **Retention**: Configured per environment

---

## Deployment Architecture

### Kubernetes + Helm

- **Namespaces**: Each service has its own namespace for isolation
- **Helm charts**: Parameterized per environment (dev, intg, prod)
- **Resource management**: CPU/memory requests and limits tuned per service
- **HPA**: Horizontal Pod Autoscaler configured for high-throughput services (wellnessProducer, dataParser)

### GitOps with ArgoCD

- **Deployment model**: All changes flow through Git → ArgoCD syncs to cluster
- **Environment promotion**: dev → intg → prod with manual gates for prod
- **Rollback**: ArgoCD enables instant rollback to any previous Git state

### Multi-Environment

| Environment | Purpose | Auth Issuer |
|-------------|---------|-------------|
| dev | Development/testing | dev issuer URL |
| intg | Integration testing | intg issuer URL |
| prod | Production | Multiple prod issuers (arubathena.com, greenlake.hpe.com) |

**Why per-environment config**: I replaced hardcoded regex-based issuer validation with environment-specific configuration that matches Istio RequestAuthentication Helm values. This eliminated the brittleness of regex patterns and ensured the Go middleware and Istio mesh-level validation agree on which issuers are valid.

See: [jwt-auth-system.md](./jwt-auth-system.md) for auth architecture details.

### CI/CD Pipeline (GitHub Actions)

- **Build**: Go build + Docker image creation
- **Test**: Unit tests, integration tests against test MongoDB/Kafka
- **Security**: Image scanning, dependency vulnerability checks
- **Deploy**: Push to container registry → ArgoCD detects and syncs

---

## Security Architecture

### Authentication

- **JWT tokens**: Multi-version support (v1.0 arubathena.com, v1.1+ greenlake.hpe.com)
- **Service tokens**: Identified by `hpe_principal_type: api-client`
- **Middleware chain**: Signature verification → issuer validation → claims extraction → context propagation

### Authorization

- **Tenant isolation**: Workspace ID extracted from token (not client request) — I fixed GLCP-333624 where cross-workspace case creation was possible
- **Cross-tenant prevention**: GLCP-325645 — additional tenant boundary enforcement
- **Input validation**: GLCP-325722 — comprehensive request validation

### File Security

- **Filename sanitization**: Path traversal prevention (`../` rejection, null byte rejection)
- **Extension allowlist**: 60+ allowed types
- **Content verification**: Magic byte validation
- **Size limit**: 10 MiB per file

See: [file-upload-system.md](./file-upload-system.md) for complete security pipeline.

---

## Scale & Performance

- **20+ microservices** across multiple Kubernetes clusters
- **500+ Go source files** in the main repository
- **Multi-cluster**: Services distributed across AWS regions for availability
- **Kafka throughput**: Handles bursts during infrastructure incidents
- **MongoDB**: Indexed for wellness object queries by device, time, status

---

## Cross-References

- [JWT Auth System](./jwt-auth-system.md) — Authentication architecture
- [Manual Case Workflow](./manual-case-workflow.md) — Case creation pipeline
- [File Upload System](./file-upload-system.md) — Attachment security
- [API Versioning](./api-versioning.md) — v1/v2/v3 evolution
- [Monitoring & Observability](./monitoring-observability.md) — Prometheus/Grafana
- [Cross-Launch Tokens](./cross-launch-tokens.md) — Wellness Proxy

---

## [KEY_POINTS]

- RAVE is a distributed system of 20+ Go microservices processing wellness telemetry for HPE infrastructure
- Event-driven architecture with Kafka decouples services and enables resilience
- Two case creation paths: email (SendGrid) and CRM (PAN HPE / DCM)
- Multi-layer security: JWT auth, tenant isolation, file validation, input sanitization
- GitOps deployment with ArgoCD, Istio service mesh, multi-environment promotion

## [COMMON_MISTAKES]

- Confusing RAVE with a simple monitoring tool — it's a full case automation platform with CRM integration
- Assuming services communicate synchronously — most inter-service communication is Kafka-based
- Thinking JWT validation happens only in Go middleware — Istio RequestAuthentication also validates at mesh level
- Overlooking the BSON primitive.D bug — it's a subtle Go/MongoDB driver interaction

## [FOLLOW_UP]

- "Walk me through what happens when a device reports a critical alert" → Full automated flow
- "How do you handle service failures?" → Kafka replay, circuit breaking, ArgoCD rollback
- "What's the difference between email and panhpe domains?" → Different case creation backends
- "How do you ensure tenant isolation?" → JWT workspace extraction, not client-supplied IDs
