# Jira Appendix — Tejas Gajare

A comprehensive catalog of my 50+ Jira stories at HPE, organized by theme and impact area.
Each story includes a summary of the work, technologies involved, and significance.

---

## Overview

| Category                    | Story Count | Key Impact                                    |
|----------------------------|-------------|-----------------------------------------------|
| Security & Validation Fixes | 13+         | Tenant isolation, input validation, auth fixes |
| Manual Case Workflow        | 4+          | End-to-end case creation pipeline              |
| API Development             | 2+          | v3 API design, new endpoints                   |
| Monitoring & Observability  | 3+          | Prometheus, Grafana, alerting                  |
| Global Trade Compliance     | 5+          | Nimble download enforcement                    |
| Infrastructure              | 1+          | ESO migration, K8s operations                  |
| Other / Platform Work       | 20+         | Miscellaneous platform improvements            |

---

## Security & Validation Fixes

Critical security work hardening the multi-tenant RAVE platform. These stories addressed
vulnerabilities in case creation, tenant isolation, and input validation.

### GLCP-333624 — Cross-Workspace Case Creation Prevention
**Priority: Critical | Impact: High**

Fixed a vulnerability where the v2 Create Case API allowed cross-workspace case creation.
The backend trusted client-supplied workspace identifiers, enabling a user in Workspace A
to create cases in Workspace B.

- **Root cause**: Backend derived workspace context from client request body instead of
  authenticated token
- **Fix**: Enforced server-side authorization by extracting workspace context from the
  user's JWT token, not the client request
- **Impact**: Critical tenant isolation fix for production multi-tenant platform
- **Tech**: Go, JWT, MongoDB, REST API

### GLCP-325645 — Security Hardening (Auth/Validation)
**Priority: High | Impact: High**

Security hardening of authentication and validation logic across the case creation pipeline.

- Strengthened JWT validation rules
- Added missing authorization checks
- **Tech**: Go, JWT middleware, Istio

### GLCP-325722 — Input Validation Gaps
**Priority: High | Impact: High**

Addressed input validation gaps across case creation APIs. Covered edge cases in field
validation that could be exploited.

- Server-side input validation for all case creation fields
- Prevented injection and malformed data issues
- **Tech**: Go, REST API validation

### GLCP-333057 — Attachment Filename Validation
**Priority: High | Impact: High**

Implemented file upload security with multi-layer validation:

- Filename sanitization (path traversal prevention)
- Extension allowlist (60+ permitted file types)
- Content-type verification via magic bytes (not just extension-based)
- S3 upload pipeline with validated files only
- **Tech**: Go, AWS S3, file I/O

### GLCP-325650 — Email Format Validation
**Priority: Medium | Impact: Medium**

Fixed email format validation in case creation contacts.

- Added proper email format validation for primary and alternate contacts
- Ensured primary contact email is required (not just phone)
- Added createdBy field extraction from JWT sub claim
- **Tech**: Go, regex validation, JWT

### Additional Security Stories
<!-- TODO: List remaining 8+ security stories with GLCP IDs if available -->

Other security work included:
- Environment-based JWT issuer validation (replaced hardcoded regex accepting tokens from
  any environment)
- Per-environment issuer restrictions matching Istio RequestAuthentication helm values
- Cross-tenant data access prevention across multiple endpoints
- Attachment handling security improvements

---

## Manual Case Creation Workflow

End-to-end manual case creation pipeline: wellnessProducer API → caseRelay (email/SendGrid)
or crmRelay (CRM API).

### GLCP-326100 — Manual Case Creation Implementation
**Priority: High | Impact: High**

Core implementation of the manual case creation workflow.

- WellnessModel and ManualCaseData field design
- Routing logic: email domain cases → caseRelay/SendGrid, PAN HPE domain → crmRelay/DCM CRM
- Contact handling (primary, alternate, CC configuration)
- Input validation and payload normalization
- **Tech**: Go, MongoDB, SendGrid, REST API

### GLCP-329285 — Manual Case Workflow Improvements
**Priority: Medium | Impact: Medium**

Iterative improvements to the manual case creation workflow.

- Enhanced validation rules for case fields
- Improved error handling and response messages
- Contact email/phone field requirements refinement
- **Tech**: Go, MongoDB

### GLCP-332469 — Case Creation Pipeline Enhancement
**Priority: Medium | Impact: Medium**

Further enhancement of the case creation pipeline.

- CRM relay attachment retry logic for handling race conditions where attachments arrive
  before case number assignment
- Improved reliability of the CRM integration
- **Tech**: Go, CRM API integration

### GLCP-334927 — Manual Case Documentation & Fixes
**Priority: Medium | Impact: Medium**

Documentation and additional fixes for manual case creation.

- Created comprehensive Manual Case Payload documentation
- Published to Confluence for cross-team reference
- Fixed edge cases discovered during documentation review
- **Tech**: Go, Confluence documentation

---

## API Development

### GLCP-334752 — v3 Wellness Producer API
**Priority: High | Impact: High**

Designed and built v3 Wellness Producer APIs with significant architectural improvements.

- Token-based auth middleware with multi-version JWT support
- Istio service mesh configuration (RequestAuthentication, EnvoyFilter)
- Workspace-based tenant isolation
- Backward compatibility maintained with v1 and v2
- HPE claims extraction (hpe_principal_type, hpe_identity_type)
- Service token vs user token differentiation
- Dynamic issuer support (*.greenlake.hpe.com for v1.1+ tokens)
- **Tech**: Go, JWT, Istio, Helm, REST API

### GLCP-336767 — API Endpoint Development
**Priority: Medium | Impact: Medium**

Additional API endpoint development and improvements.

- New endpoints for platform functionality
- API versioning consistency
- **Tech**: Go, REST API

---

## Monitoring & Observability

### GLCP-325062 — Rave Monitoring System
**Priority: High | Impact: High**

Built comprehensive cloud alerting system for RAVE platform.

- Prometheus metrics instrumentation in Go services
- Consolidated metrics: replaced separate MetricEventCaseCreated/Updated with unified
  `mc_event_case_action_total` counter with action label (created, updated, reopened, new_open)
- **Tech**: Go, Prometheus, Grafana

### GLCP-237935 — Monitoring & Alerting Configuration
**Priority: Medium | Impact: High**

Monitoring alerting and notification configuration.

- Grafana dashboard design for case creation rates, service health, CRM failures
- Alert categories: service stopped running (non-running pods warning, zero replicas critical),
  processing drops (case creation/automation rates below threshold)
- Automated Slack notifications for team awareness
- PagerDuty integration for production incident response
- **Tech**: Grafana, Prometheus, Slack API, PagerDuty

### GLCP-331718 — Monitoring Enhancements
**Priority: Medium | Impact: Medium**

Additional monitoring improvements and dashboard refinements.

- Dashboard query optimization
- New alert rules for edge cases
- **Tech**: Grafana, Prometheus

---

## Global Trade Compliance (Nimble Download)

4-microservice system enforcing export control regulations on HPE Nimble Storage software
downloads.

### GLCP-302655 — Download Gatekeeper Implementation
**Priority: High | Impact: High**

Built the DownloadGatekeeper service for enforcing Export License Classification (ELC)
checks at software download time.

- IP geolocation-based country resolution
- Account validation against GTCaaS
- SQLite caching layer reducing redundant API calls by 60%+
- Embargo detection and enforcement
- **Tech**: Go, SQLite, GTCaaS API, IP geolocation

### GLCP-289971 — Auto-Onboard Service
**Priority: High | Impact: High**

Built the AutoOnboard service for device onboarding via Restricted Party List (RPL) screening.

- Device shadow state machine in MongoDB (PENDING → ONBOARDED / FAILED / BLOCKED)
- Automated RPL screening via GTCaaS
- Serial number-based device-level onboarding
- **Tech**: Go, MongoDB, GTCaaS API

### GLCP-294827 — GT Cache API
**Priority: Medium | Impact: Medium**

Built GTCacheAPI service for IP/license cache management.

- Cache read/write operations for geolocation results
- Cache invalidation strategies
- API for cache management and diagnostics
- **Tech**: Go, SQLite

### GLCP-253683 — Global Trade System Architecture
**Priority: High | Impact: High**

Architectural design and initial implementation of the global trade compliance system.

- System design for 4-microservice architecture
- Integration patterns with GTCaaS (Global Trade Compliance as a Service)
- MongoDB schema design for device shadows
- **Tech**: Go, MongoDB, system architecture

### GLCP-254822 — Global Trade Additional Work
**Priority: Medium | Impact: Medium**

Additional global trade compliance implementation work.

- Reference number tools and utilities
- Additional screening integration work
- **Tech**: Go, GTCaaS API

### Future: GTCaaS MaxMind Migration
Planning and documentation for migration from HPE Geolocation Service (Neustar) to MaxMind
within GTCaaS. IP-based country resolution changes, embargo handling updates, API key auth
addition. Target: 2026.

---

## Infrastructure

### GLCP-325064 — ESO Migration (Vault → ESO)
**Priority: Medium | Impact: High**

Migrated RAVE services from HashiCorp Vault to External Secrets Operator (ESO).

- Completed migration for 3 services: caseRelay, crmRelay, wellnessProducer
- Planned migration for remaining 17 services
- ESO SecretStore/ExternalSecret CRD configuration
- AWS Parameter Store via ESO parameterStorePath
- Custom ServiceAccount setup for pod-level AWS IAM role binding
- **Tech**: Kubernetes, ESO, AWS IAM, Helm

---

## Other Notable Work

### BSON/MongoDB Data Conversion Fix
Fixed primitive.D handling in `convertBsonDToMap` for nested MongoDB documents. Priority,
primary_contact, and alternate_contact fields were failing conversion when data came from
MongoDB (primitive.D as []primitive.E) vs direct JSON.

- **Root cause**: Go type switch on `[]interface{}` doesn't match `primitive.D` (which is
  `[]primitive.E`), requiring explicit case
- **Tech**: Go, MongoDB driver v1.11.1, BSON

### CRM Relay Attachment Retry Logic
Implemented retry logic for handling race conditions where CRM attachments arrive before
case number assignment. Ensures attachments are properly linked to cases even with
asynchronous CRM processing.

### Cross-Launch Token Support (Wellness Proxy)
Added cross-launch token handling to the Python-based Wellness Proxy for Nimble routes.
Token validation and routing logic for cross-service authentication.

### Comprehensive Manual Case Payload Documentation
Created detailed documentation covering WellnessModel fields, ManualCaseData fields,
validation rules, email configuration, contact fields, and attachment workflow. Published
to Confluence for cross-team reference.

---

## Story Map by Sprint/Timeline

```
2023 H2:  Global Trade (GLCP-253683, GLCP-254822, GLCP-289971, GLCP-294827, GLCP-302655)
2024 H1:  Manual Case (GLCP-326100, GLCP-329285) + API Dev (GLCP-334752)
2024 H2:  Security (GLCP-333624, GLCP-325645, GLCP-325722, GLCP-333057, GLCP-325650)
          Monitoring (GLCP-325062, GLCP-237935, GLCP-331718)
          Infrastructure (GLCP-325064)
2025 H1:  Case Enhancements (GLCP-332469, GLCP-334927) + API (GLCP-336767)
          GTCaaS MaxMind migration planning
2026 Q1:  Performance Investigation (GLCP-325825) — Case creation P90 degradation
          Kafka Partition Scaling (GLCP-325825) — crmRelay horizontal scaling
          Office XML Validation (GLCP-340302, GLCP-339197) — .xlsx/.docx/.pptx fix
          Prometheus Metrics (GLCP-331718) — crmService instrumentation
          Case Throttling Alerts (GLCP-330413) — Actionable alert replacement
          OpenAPI Spec Update (GLCP-335836) — Wellness API v2 spec
          MetricCollector Removal (GLCP-335930) — Deprecated component cleanup
          Test Coverage Push — wellnessManagement 76→80%, healthchecker 73→80%
          JWT Issuer Validation Fix — JWKS cache scoping by issuer+kid
          v2beta1 glp:manual Exclusion — DB-level origin filter
          OData in Operator — caseNumber multi-ID filter
          Docker Compose Consolidation — Unified local dev environment
```
<!-- TODO: Verify timeline accuracy against actual sprint dates -->

---

## Themes & Patterns

Looking across all 50+ stories, several patterns emerge:

1. **Security-first development**: ~25% of stories involve security hardening, tenant
   isolation, or input validation
2. **End-to-end ownership**: I consistently own features from API design through
   infrastructure deployment
3. **Compliance focus**: Both data privacy (GLCP security fixes) and trade compliance
   (Nimble Download) are recurring themes
4. **Observability investment**: Monitoring work ensures the systems I build are
   maintainable in production
5. **Infrastructure fluency**: I work across the full stack — Go code, K8s manifests,
   Helm charts, CI/CD configs, Grafana dashboards

---

*Last updated: 2026-03-27*
*Source: Knowledge graph (jira-bulk), Jira project GLCP, codebase analysis*
*Note: Story descriptions are based on knowledge graph memories and may be approximate.
Refer to actual Jira tickets for authoritative details.*
