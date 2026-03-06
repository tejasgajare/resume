# Resume Bullets — Polished & Quantified

> Ready-to-use resume bullet points for Tejas Gajare.
> Each bullet follows the format: Action Verb + What + How + Impact/Metric.

---

## RAVE Cloud Platform (HPE GreenLake)

### Core API Development
- Architected and implemented **v3 Wellness Producer APIs** with token-based auth middleware, environment-specific JWT issuer validation, and workspace-based tenant isolation for HPE GreenLake Cloud Platform
- Designed and built end-to-end **manual case creation workflow** enabling tenants to create support cases via API with automated routing to CRM (HPE domains) or email (SendGrid) based on domain classification
- Evolved REST APIs through **three major versions** (v1→v2→v3) maintaining full backward compatibility while progressively tightening authentication and adding multi-tenant isolation

### Security & Validation
- Fixed **13+ security vulnerabilities** across the RAVE platform including cross-workspace case creation (tenant isolation), input validation gaps, attachment filename sanitization, and cross-environment token acceptance
- Implemented **multi-layer file upload security**: filename sanitization, 60+ type extension allowlist, content-type verification via magic byte analysis, and file size limits — preventing path traversal, extension spoofing, and content-type mismatch attacks
- Resolved critical **cross-tenant case creation vulnerability** (GLCP-333624) by enforcing server-side workspace derivation from JWT claims, eliminating client-supplied workspace trust

### Monitoring & Observability
- Built comprehensive **Prometheus/Grafana monitoring system** with unified `mc_event_case_action_total` metric, service health alerts, processing drop detection, and automated Slack/PagerDuty notifications for production incident response
- Implemented **auth failure metrics** and environment-specific issuer validation, aligning Go middleware with Istio RequestAuthentication configuration across 3 environments (dev/intg/prod)

### Data & Integration
- Fixed **MongoDB BSON deserialization bug** in CRM relay — nested documents (priority, contacts) were incorrectly handled as primitive.D instead of map types, causing silent data corruption in CRM API payloads
- Implemented **CRM attachment retry logic** resolving race condition where file attachments arrived before case number assignment, using exponential backoff with context-aware cancellation

---

## Infrastructure & DevOps

### Kubernetes & Service Mesh
- Managed production **Kubernetes cluster operations** including custom ServiceAccount provisioning, Istio configuration (RequestAuthentication, EnvoyFilters, DestinationRules), and deployment pipelines across 20+ microservices
- Migrated **3 services from HashiCorp Vault to External Secrets Operator (ESO)** using AWS Parameter Store, with per-service IAM roles (IRSA) and backward-compatible K8s Secret naming

### CI/CD & Automation
- Modularized **CI/CD pipeline** reducing build and deployment time by **96% (to under 2 minutes)** during internship at HPE
- Configured **ArgoCD GitOps deployments** with Helm chart management for 3 RAVE platform variants (Cloud, Classic, LVS) across dev/intg/prod environments
- Integrated **SonarQube** code quality analysis into GitHub Actions pipelines with quality gate enforcement

### Secrets & Configuration
- Led **ESO migration initiative**: completed 3 services, documented migration plan and patterns for remaining 17, provisioned custom ServiceAccounts with AWS IRSA for least-privilege access

---

## Nimble Download — Global Trade Compliance

- Developed **Go-based download middleware** enforcing global trade compliance on Nimble Storage software downloads, integrating IP geolocation and account validation with **SQLite caching reducing redundant API calls by 60%+**
- Built **4-microservice compliance system**: AutoOnboard (RPL screening), DownloadGatekeeper (ELC enforcement at download time), GTCacheAPI (IP/license caching), and reference number tools
- Designed **device shadow state machine** in MongoDB tracking device lifecycle from discovery through RPL screening to onboarded/blocked status
- Planned and documented **GTCaaS MaxMind migration** from Neustar geolocation, designing cache abstraction layer to minimize code changes when provider switches

---

## Tooling & Automation

- Created **AI-powered sprint review generator** (Copilot CLI skill) automating Confluence page creation with Jira Smart Links, @mentions, status lozenges, and burndown charts — reducing sprint documentation from **2-3 hours to under 2 minutes**
- Built **sprint assignment automation** skill transitioning all sprint issues to "Assigned" status with story point validation, LaunchDarkly field handling, and blocked issue detection
- Designed **AI-powered Grafana observability concept** accepting natural language prompts for real-time monitoring insights using Prometheus data

---

## Side Projects

### ClipStash — macOS Clipboard Manager
- Built native macOS clipboard history manager in **Swift 6 / SwiftUI** with full-fidelity clipboard format preservation, floating panel UI, global hotkey activation, and SQLite persistence via GRDB
- Implemented **snapshot testing framework** for SwiftUI using `renderInWindow()` helper with NSWindow hosting for accurate visual regression testing

### Kairo — macOS Menu Bar Calendar
- Built native macOS menu bar calendar widget using **SwiftUI** with EventKit integration for calendar events and real-time weather display via Open-Meteo API
- Implemented clean **MVVM architecture** with dedicated service and model layers

### Crono — Time Series Vector Database
- Building a time series database in **Go** with custom storage engine, vector indexing, state management, and REST API layer
- Designed modular architecture with separate packages for API server, indexing, state management, storage engine, and vector operations

### Surmai — Travel Planning PWA
- Built full-stack travel planning app with **React frontend** and **Go/PocketBase backend** as a Progressive Web App

### DiceDB — Open Source Contribution
- Contributed to **DiceDB**, a high-performance in-memory database written in Go

---

## Prior Roles

### Grouped — Full Stack Developer (2022)
- Led development of core microservices and cloud infrastructure for a social media platform
- Designed **three-tier, multi-AZ infrastructure** running Spring Boot and Flask microservices on AWS ECS with 5 horizontally scaled Fargate instances behind a load balancer
- Utilized **server-side rendering** to decrease page load time by **35%**

### Yardi — Software Engineer (2019-2021)
- Designed **CCPA & GDPR compliance data scrubbing tool** handling complex hierarchical database relationships using T-SQL; added feature to identify sensitive fields, reducing field selection time by **50%**
- Developed **Harbor Management System** using ASP.NET and Microsoft SQL Server for lease tracking via dashboard

---

## Education Highlights

- **M.S. Computer Science**, Syracuse University (2021-2023)
  - Coursework: NLP, Data Mining, Cryptography, Blockchain, Algorithms
- **B.E. Computer Engineering**, Savitribai Phule Pune University (2015-2019)
  - Finalist, **ACM ICPC** Asia Regionals 2019
  - Ranked **6th nationally** in ABU Robocon 2017
