# Repository Map — Tejas Gajare

A comprehensive map of all repositories I work with, organized by organization and purpose.

---

## HPE / GLCP Repositories (~/glcp/)

These are internal HPE GreenLake Cloud Platform repositories. I am a primary contributor
on the RAVE platform and Nimble Download systems.

### Core Platform

| Repository              | Path                          | Language  | Files | Description                                              |
|------------------------|-------------------------------|-----------|-------|----------------------------------------------------------|
| rave-cloud             | ~/glcp/rave-cloud             | Go        | 505+  | Primary RAVE cloud platform. 20+ microservices: wellnessProducer, caseRelay, crmRelay, and more. |
| rave-classic           | ~/glcp/rave-classic           | Go        | 219   | Legacy RAVE variant. Shopify/Sarama Kafka, PostgreSQL, Vertica, MongoDB. |
| rave-lvs               | ~/glcp/rave-lvs               | Go        | 500+  | On-prem/hybrid RAVE variant. ESO secrets, Helm charts, Istio config. |
| wellness-proxy         | ~/glcp/wellness-proxy         | Python    | 20    | FastAPI proxy for wellness API routing. Cross-launch tokens, Nimble/HPE endpoint routing. |
| nimbleDownloadUpdate   | ~/glcp/nimbleDownloadUpdate   | Go        | 52    | Global trade compliance: AutoOnboard, DownloadGatekeeper, GTCacheAPI, reference tools. |

### Infrastructure & DevOps

| Repository                    | Path                                   | Description                                     |
|------------------------------|----------------------------------------|-------------------------------------------------|
| rave-sops-dev                | ~/glcp/rave-sops-dev                   | SOPS-encrypted secrets for dev environment       |
| rave-sops-prod               | ~/glcp/rave-sops-prod                  | SOPS-encrypted secrets for prod environment      |
| rave-sops-stage              | ~/glcp/rave-sops-stage                 | SOPS-encrypted secrets for stage environment     |
| ccp-humio-resources-config   | ~/glcp/ccp-humio-resources-config      | Humio/LogScale logging configuration             |
| managed-ci-workflow-config   | ~/glcp/managed-ci-workflow-config      | CI/CD pipeline configurations                    |
| mcd-deploy-proj-rave         | ~/glcp/mcd-deploy-proj-rave            | ArgoCD deployment configurations for RAVE        |

### Supporting / Reference

| Repository              | Path                          | Language   | Description                                  |
|------------------------|-------------------------------|------------|----------------------------------------------|
| go-rest-api-example    | ~/glcp/go-rest-api-example    | Go (22)    | Reference implementation for Go REST patterns |
| homefleetRavePlugins   | ~/glcp/homefleetRavePlugins   | Go (10)    | RAVE plugins for HomeFleet integration        |
| mfe-greenlake-infra    | ~/glcp/mfe-greenlake-infra    | TypeScript (27) | Micro-frontend infrastructure for GreenLake |
| doc-portal             | ~/glcp/doc-portal             | —          | HPE GreenLake documentation portal            |

### My Role in HPE Repos

- **rave-cloud**: Primary contributor. Owned manual case creation pipeline, security fixes,
  v3 API design, monitoring. Hundreds of commits across wellnessProducer, caseRelay, crmRelay.
- **rave-lvs**: Active contributor. ESO migration, Helm chart updates, K8s configuration.
- **rave-classic**: Contributor. Monitoring metrics upgrades, Kafka consumer updates.
- **nimbleDownloadUpdate**: Primary contributor. Built 4-microservice global trade system. 50+ commits.
- **wellness-proxy**: Contributor. Cross-launch token support, routing updates.
- **Infrastructure repos**: Active contributor. SOPS secrets, CI/CD configs, deployment pipelines.

---

## Personal / Side Project Repositories

### GitHub: gajaretejas Organization

| Repository   | GitHub URL                              | Language       | Description                                       |
|-------------|----------------------------------------|----------------|---------------------------------------------------|
| crono       | github.com/gajaretejas/crono           | Go             | Time series vector database with custom storage engine |
| chronostore | github.com/gajaretejas/chronostore     | Go             | Time series database exploration project           |
| colimaui    | github.com/gajaretejas/colimaui        | Swift/SwiftUI  | macOS UI for Colima container runtime              |
| kairo       | github.com/gajaretejas/kairo           | Swift/SwiftUI  | macOS menu bar calendar with EventKit + weather    |
| resume      | github.com/gajaretejas/resume          | LaTeX/Markdown | This repository — resume, portfolio, interview prep |
| localdev    | github.com/gajaretejas/localdev        | Python         | Local development environment setup tools          |

### Local Projects (~/projects/)

| Repository              | Path                              | Language        | Description                                    |
|------------------------|-----------------------------------|-----------------|------------------------------------------------|
| ClipStash              | ~/projects/ClipStash              | Swift (23)      | macOS clipboard history manager. SwiftUI + AppKit + GRDB. |
| surmai                 | ~/projects/surmai                 | Go/TypeScript (34) | Travel planning PWA. React frontend + Go/PocketBase backend. |
| dice                   | ~/projects/dice                   | Go (109+)       | Open-source contribution to DiceDB (high-perf in-memory DB). |
| localdev               | ~/projects/localdev               | Python          | Local MongoDB and PortalDB provisioning scripts. |
| lucene-search-analysis | ~/projects/lucene-search-analysis | Java            | Lucene/Atlas Search analysis tool (Spring Boot + Vaadin). |

### Other Local

| Repository   | Path        | Language       | Description                                       |
|-------------|-------------|----------------|---------------------------------------------------|
| kairo       | ~/kairo     | Swift/SwiftUI  | macOS menu bar calendar (primary dev location)     |

---

## Repository Activity Summary

### By Commit Volume (estimated)

| Tier         | Repositories                                              |
|-------------|-----------------------------------------------------------|
| Very High   | rave-cloud, nimbleDownloadUpdate                          |
| High        | rave-lvs, ClipStash, kairo                                |
| Medium      | rave-classic, wellness-proxy, infrastructure repos, surmai |
| Growing     | crono, chronostore, colimaui, dice                        |
| Reference   | go-rest-api-example, homefleetRavePlugins, doc-portal     |

### By Language

| Language    | Repositories                                                              |
|------------|---------------------------------------------------------------------------|
| Go         | rave-cloud, rave-classic, rave-lvs, nimbleDownloadUpdate, crono, chronostore, surmai, dice, go-rest-api-example, homefleetRavePlugins |
| Python     | wellness-proxy, localdev                                                  |
| Swift      | ClipStash, kairo, colimaui                                                |
| TypeScript | mfe-greenlake-infra, surmai (frontend)                                    |
| Java       | lucene-search-analysis                                                    |
| Helm/YAML  | rave-sops-*, mcd-deploy-proj-rave, managed-ci-workflow-config             |
| LaTeX/MD   | resume                                                                    |

---

## Architecture: How Repos Relate

```
HPE GreenLake Cloud Platform
├── RAVE Platform
│   ├── rave-cloud (primary)  ──── evolved from ──── rave-classic (legacy)
│   ├── rave-lvs (on-prem/hybrid variant)
│   └── wellness-proxy (API routing layer)
│       └── Routes to: rave-cloud endpoints, Nimble endpoints
│
├── Nimble Download
│   └── nimbleDownloadUpdate (global trade compliance)
│       └── Uses: GTCaaS, MongoDB, SQLite cache
│
├── Infrastructure
│   ├── rave-sops-{dev,prod,stage} (secrets)
│   ├── mcd-deploy-proj-rave (ArgoCD deployments)
│   ├── managed-ci-workflow-config (CI/CD)
│   └── ccp-humio-resources-config (logging)
│
└── Supporting
    ├── go-rest-api-example (reference patterns)
    ├── homefleetRavePlugins (HomeFleet integration)
    ├── mfe-greenlake-infra (micro-frontend)
    └── doc-portal (documentation)

Personal Projects
├── macOS Apps: ClipStash, Kairo, ColimaUI
├── Databases: Crono (TSDB), ChronoStore (TSDB), DiceDB (OSS contrib)
├── Web: Surmai (travel PWA)
├── Tooling: LocalDev, Lucene Search Analysis
└── Portfolio: Resume
```

---

*Last updated: 2025*
*Source: Knowledge graph, local filesystem, GitHub*
