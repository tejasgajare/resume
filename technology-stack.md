# Technology Stack — Tejas Gajare

A comprehensive catalog of technologies I use, organized by category with proficiency levels
and project context.

**Proficiency Key:**
- 🟢 **Expert** — Deep knowledge, can architect systems, debug complex issues, mentor others
- 🔵 **Advanced** — Strong working knowledge, regular production use, comfortable independently
- 🟡 **Intermediate** — Functional proficiency, have shipped projects, can ramp up quickly
- ⚪ **Familiar** — Have used meaningfully, understand concepts, can contribute with ramp-up

---

## Programming Languages

### Go 🟢 Expert
**Primary backend language.** Used daily for 2+ years across all HPE RAVE services.

| Project                | Usage                                                  |
|-----------------------|--------------------------------------------------------|
| RAVE Cloud            | 500+ Go files, 20+ microservices, core platform        |
| RAVE Classic          | 219 Go files, legacy platform variant                  |
| RAVE LVS              | 500+ Go files, on-prem/hybrid variant                  |
| Nimble Download       | 52 Go files, 4 microservices for global trade          |
| DiceDB (OSS)          | Open-source contribution, 109+ Go files                |
| Crono / ChronoStore   | Personal time-series database projects                 |
| Surmai                | Go + PocketBase backend for travel PWA                 |

**Key patterns:**
- HTTP middleware chains (auth, logging, metrics)
- MongoDB driver (mgo/mongo-driver v1.11.1) with BSON type handling
- Kafka consumers/producers (segmentio/kafka-go, Shopify/Sarama)
- Prometheus metrics instrumentation
- Goroutine-based concurrency (workers, batch processing)
- Interface-based dependency injection for testability
- Custom storage engines (ChronoStore, Crono)

### Python 🟢 Expert
**Used for AI-agent backends, proxy services, automation, and infrastructure scripting.**

| Project                | Usage                                                  |
|-----------------------|--------------------------------------------------------|
| Sahaya Backend        | FastAPI + LangGraph multi-agent orchestrator, 18 SQLAlchemy models, Alembic migrations, pgvector memory |
| Genie Wellness Agent  | LLM-powered wellness event analysis agent (HPE), FastAPI + Gunicorn, multi-cluster K8s |
| Wellness Proxy        | FastAPI-based proxy for Nimble/HPE API routing          |
| LocalDev              | Local MongoDB and PortalDB provisioning scripts         |
| Infrastructure        | Automation scripts, data processing                     |

**Key patterns:**
- FastAPI with async endpoints and SSE streaming
- LangGraph/LangChain multi-agent orchestration with domain subgraphs
- SQLAlchemy ORM + Alembic migrations
- pgvector for semantic vector search
- Cross-launch token handling
- HPA (Horizontal Pod Autoscaler) configuration

### Swift 🔵 Advanced
**Native macOS application development with SwiftUI.**

| Project                | Usage                                                  |
|-----------------------|--------------------------------------------------------|
| ClipStash             | macOS clipboard manager, Swift 6, SwiftUI + AppKit     |
| Kairo                 | macOS menu bar calendar widget, SwiftUI                 |

**Key patterns:**
- SwiftUI + AppKit integration (NSPanel, NSPasteboard)
- MVVM architecture with service/model layers
- GRDB (SQLite) for persistence
- EventKit and Weather API integration
- SPM (Swift Package Manager) dependency management
- Snapshot testing with renderInWindow() helpers

### TypeScript 🔵 Advanced
**Mobile and frontend development with React Native/Expo and micro-frontends.**

| Project                | Usage                                                  |
|-----------------------|--------------------------------------------------------|
| Sahaya App            | React Native/Expo mobile app — chat, wellness, calendar, tasks |
| GreenLake Infra MFE   | Micro-frontend infrastructure (27 TS files)            |
| Surmai                | React frontend for travel PWA                          |

### Java 🟡 Intermediate
**Enterprise applications and search analysis tooling.**

| Project                | Usage                                                  |
|-----------------------|--------------------------------------------------------|
| Grouped               | Spring Boot microservices for social media platform    |
| Lucene Search Analysis | Spring Boot + Vaadin for Lucene/Atlas Search analysis  |

### Rust 🟡 Intermediate
**Cross-platform application development.**

| Project                | Usage                                                  |
|-----------------------|--------------------------------------------------------|
| Kairo (Tauri port)    | Cross-platform desktop app via Tauri framework         |

### C# ⚪ Familiar
**Enterprise web development (prior role).**

| Project                | Usage                                                  |
|-----------------------|--------------------------------------------------------|
| Yardi Systems         | ASP.NET applications, Harbor Management System          |

### T-SQL ⚪ Familiar
**Database programming (prior role).**

| Project                | Usage                                                  |
|-----------------------|--------------------------------------------------------|
| Yardi Systems         | CCPA/GDPR compliance data scrubbing tool               |

### C++ ⚪ Familiar
**Academic and competitive programming.**

---

## Databases

### MongoDB 🟢 Expert
**Primary data store across RAVE platform and Nimble Download.**

- Wellness objects, device shadows, case metadata
- BSON type handling (primitive.D, primitive.A conversions)
- Aggregation pipelines, compound indexes
- Document schema design for nested entities
- Used in: RAVE Cloud, RAVE Classic, RAVE LVS, Nimble Download

### SQLite 🔵 Advanced
**Embedded database for caching and desktop applications.**

- IP/license caching in Nimble Download (DownloadGatekeeper)
- GRDB wrapper in ClipStash (clipboard history storage)
- Knowledge graph database (this portfolio)
- Used in: Nimble Download, ClipStash, resume knowledge graph

### PostgreSQL 🟡 Intermediate
- Relational data in RAVE Classic
- Used in: RAVE Classic, academic projects

### Microsoft SQL Server ⚪ Familiar
- Enterprise database at Yardi Systems
- Complex stored procedures, hierarchical data
- Used in: Yardi compliance tool, Harbor Management System

---

## Infrastructure & Cloud

### Kubernetes 🟢 Expert
**Container orchestration across all RAVE services.**

- Custom ServiceAccount provisioning (pod-level AWS IAM)
- HPA (Horizontal Pod Autoscaler) configuration
- Multi-environment management (dev, intg, prod)
- Pod health monitoring and alerting
- Used in: All RAVE variants, Wellness Proxy, Nimble Download

### Docker 🔵 Advanced
- Container image building for all microservices
- Multi-stage builds for Go services
- Colima for local container runtime (macOS)
- ColimaUI desktop application built for managing containers
- Used in: All HPE services, side projects

### AWS 🔵 Advanced

| Service     | Usage                                            |
|-------------|--------------------------------------------------|
| S3          | Attachment storage for case creation              |
| IAM         | Pod-level roles via ServiceAccount                |
| EKS         | Kubernetes cluster management                     |
| ECS/Fargate | Used at Grouped (5 scaled instances)             |
| RDS         | Database hosting (Grouped)                        |

### Terraform 🟡 Intermediate
- Infrastructure-as-code for AWS resources
- Used in: HPE cloud infrastructure management

### Helm 🔵 Advanced
- Chart management for all RAVE Kubernetes deployments
- Values-based per-environment configuration
- Used in: RAVE Cloud, RAVE LVS, Wellness Proxy

### Istio 🔵 Advanced
- Service mesh configuration for RAVE platform
- RequestAuthentication for JWT validation
- EnvoyFilter configuration for API versioning
- mTLS between services
- Used in: RAVE Cloud, RAVE LVS

### ArgoCD 🟡 Intermediate
- GitOps-based deployment pipeline
- Used in: RAVE Cloud, RAVE LVS, infrastructure repos

---

## Messaging & Event Streaming

### Kafka 🔵 Advanced
**Event streaming between RAVE microservices.**

| Library          | Used In        | Notes                          |
|-----------------|----------------|--------------------------------|
| segmentio/kafka-go | RAVE Cloud  | Modern Go Kafka client          |
| Shopify/Sarama  | RAVE Classic   | Legacy Go Kafka client          |

- Consumer groups, topic partitioning
- Event-driven case processing pipeline
- Used in: RAVE Cloud, RAVE Classic

---

## Observability & Monitoring

### Prometheus 🔵 Advanced
- Custom metrics instrumentation in Go services
- Consolidated counters (mc_event_case_action_total with action labels)
- Histogram and gauge metrics for service health
- Used in: RAVE Cloud, monitoring infrastructure

### Grafana 🔵 Advanced
- Dashboard design for case creation rates, service health, CRM failures
- Alert rule configuration
- Designed AI-powered panel concept for natural language queries
- Used in: RAVE monitoring

### Humio/LogScale 🟡 Intermediate
- Centralized log aggregation and search
- Used in: HPE infrastructure (ccp-humio-resources-config)

### New Relic ⚪ Familiar
- Application performance monitoring
- Used in: HPE platform monitoring

### PagerDuty 🟡 Intermediate
- Incident notification integration
- Automated alerting from Grafana
- Used in: RAVE production monitoring

---

## Security

### JWT 🟢 Expert
- Multi-version auth middleware (v1.0 arubathena.com, v1.1+ greenlake.hpe.com)
- HPE claims extraction (hpe_principal_type, hpe_identity_type)
- Environment-based issuer validation
- Service token vs user token differentiation
- Cross-launch token support

### SOPS 🟡 Intermediate
- Secrets encryption for Kubernetes manifests
- Per-environment secret management (dev/intg/prod/stage)
- Used in: rave-sops-dev, rave-sops-prod, rave-sops-stage

### External Secrets Operator (ESO) 🔵 Advanced
- Migration from HashiCorp Vault to ESO
- SecretStore/ExternalSecret CRD configuration
- AWS Parameter Store integration
- Used in: RAVE infrastructure (3 services migrated)

### HashiCorp Vault ⚪ Familiar
- Legacy secret management (being migrated to ESO)
- Used in: RAVE infrastructure (historical)

---

## CI/CD

### GitHub Actions 🔵 Advanced
- Pipeline modularization (96% build time reduction as intern)
- Managed CI workflows for RAVE services
- Used in: All HPE repos, personal projects

### SonarQube 🟡 Intermediate
- Code quality and security scanning
- Used in: HPE CI pipelines

---

## Frontend & Desktop

### React 🟡 Intermediate
- Travel PWA frontend (Surmai)
- Used in: Surmai, GreenLake MFE

### SwiftUI 🔵 Advanced
- Native macOS desktop applications
- Menu bar apps, floating panels, system integration
- Used in: ClipStash, Kairo

### Tauri 🟡 Intermediate
- Cross-platform desktop apps with Rust backend
- Used in: Kairo (cross-platform port)

### PocketBase ⚪ Familiar
- Backend-as-a-service for Go applications
- Used in: Surmai

---

## Other Tools & Libraries

| Tool/Library         | Proficiency | Usage                                     |
|---------------------|-------------|-------------------------------------------|
| SendGrid            | 🟡 Intermediate | Email delivery for case creation pipeline |
| FastAPI             | 🔵 Advanced     | Python API framework (Sahaya, Genie Wellness Agent, Wellness Proxy) |
| LangGraph           | 🔵 Advanced     | Multi-agent LLM orchestration (Sahaya backend) |
| LangChain           | 🔵 Advanced     | LLM application framework (Sahaya backend) |
| React Native / Expo | 🔵 Advanced     | Cross-platform mobile development (Sahaya App) |
| SQLAlchemy + Alembic| 🟡 Intermediate | Python ORM + migrations (Sahaya backend)  |
| pgvector            | 🟡 Intermediate | PostgreSQL vector similarity search (Sahaya memory) |
| Redis               | 🟡 Intermediate | Caching and real-time features (Sahaya)   |
| Gemini              | 🟡 Intermediate | Google AI model for agent intelligence    |
| Cloudflare Tunnel   | 🟡 Intermediate | Secure service exposure (Sahaya deployment) |
| K3s                 | 🟡 Intermediate | Lightweight K8s for edge/ARM (Raspberry Pi) |
| SSE                 | 🟡 Intermediate | Server-Sent Events for LLM streaming      |
| GRDB                | 🔵 Advanced     | SQLite wrapper for Swift (ClipStash)      |
| Vaadin              | ⚪ Familiar      | Java web framework (Lucene Search tool)   |
| Colima              | 🔵 Advanced     | macOS container runtime (Docker alternative) |
| Git                 | 🟢 Expert       | Version control across all projects       |

---

## Proficiency Summary

| Level          | Technologies                                                          |
|----------------|-----------------------------------------------------------------------|
| 🟢 Expert      | Go, Python, MongoDB, Kubernetes, JWT, Git                            |
| 🔵 Advanced    | TypeScript, Swift, Docker, Helm, Istio, AWS, Kafka, Prometheus, Grafana, ESO, SwiftUI, SQLite, GRDB, GitHub Actions, FastAPI, LangGraph, LangChain, React Native/Expo |
| 🟡 Intermediate | Java, Rust, Terraform, ArgoCD, React, Humio, PagerDuty, SOPS, SonarQube, SendGrid, Tauri, Colima, SQLAlchemy, pgvector, Redis, Gemini, Cloudflare Tunnel, K3s, SSE |
| ⚪ Familiar     | C#, T-SQL, C++, SQL Server, Vault, New Relic, PocketBase, Vaadin     |

---

*Last updated: 2026-03-27*
*Source: Knowledge graph, codebase analysis, resume (tejasgajare)*
