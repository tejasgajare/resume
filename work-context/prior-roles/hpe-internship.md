# HPE Internship — Software Engineer Intern

**Company:** Hewlett Packard Enterprise
**Title:** Software Engineer Intern
**Location:** San Jose, CA (<!-- TODO: confirm if remote/hybrid -->)
**Duration:** May 2022 — August 2022
**Team:** Wellness/RAVE Team, GreenLake Cloud Platform

---

## Overview

My summer internship at HPE was my first exposure to the GreenLake Cloud Platform and the
Wellness/RAVE team I would later join full-time. This internship gave me hands-on experience
with enterprise-grade Go microservices, CI/CD pipelines, and cloud-native development patterns
at scale.

---

## What I Worked On

### 1. REST API Enhancement

I enhanced customer-facing Go-based REST APIs to conform with HPE internal standards.

**What this involved:**
- Aligning API request/response formats with HPE's internal API guidelines
- Ensuring consistent error handling, status codes, and response structures
- Improving interoperability between RAVE microservices and other GLCP services
- The changes accelerated user adoption by making APIs more predictable and consistent

**Technologies:** Go, REST APIs, HTTP middleware, JSON serialization

### 2. CI/CD Pipeline Modularization

I modularized the CI/CD pipeline for the HPE RAVE platform, achieving a **96% reduction in
build and deployment time** (down to under 2 minutes).

**What this involved:**
- Analyzed the existing monolithic CI/CD pipeline to identify bottlenecks
- Broke the pipeline into modular stages that could run independently or in parallel
- Optimized build caching and dependency resolution
- Reduced feedback loop for developers from long wait times to near-instant builds

**Technologies:** GitHub Actions, Docker, CI/CD tooling

**Impact:** This was a significant productivity multiplier for the entire team. A build that
previously took ~50 minutes was reduced to under 2 minutes, dramatically accelerating the
development cycle.

---

## What I Learned

### Technical Skills
- **Go at scale**: First experience with a large Go codebase (500+ files, 20+ microservices)
- **Microservices architecture**: Understanding service boundaries, inter-service communication
- **Kubernetes & cloud-native patterns**: Exposure to K8s deployments, service mesh (Istio)
- **CI/CD best practices**: Pipeline optimization, build caching, modular pipeline design
- **Enterprise API standards**: HPE's internal API guidelines and patterns

### Professional Growth
- Working within a mature engineering team on production-critical systems
- Code review processes and engineering standards at a Fortune 500 company
- Understanding the full lifecycle of enterprise software (development → testing → deployment)
- Cross-team collaboration within the larger GLCP organization

---

## How It Connected to My Full-Time Role

The internship was directly formative for my full-time position:

| Internship Experience                 | Full-Time Application                                   |
|--------------------------------------|--------------------------------------------------------|
| Go REST API standardization          | Designed v3 Wellness Producer APIs from scratch         |
| CI/CD pipeline optimization          | Managed deployment pipelines across dev/intg/prod       |
| RAVE platform familiarity            | Became primary owner of case creation pipeline          |
| Team relationships                   | Smooth onboarding; immediate productivity               |
| K8s/cloud-native exposure            | Infrastructure owner for K8s cluster operations         |

The internship reduced my onboarding time as a full-time engineer to near-zero. I already
understood the codebase architecture, team processes, and the platform's role within GLCP.
This allowed me to take on significant ownership (security fixes, new API versions,
monitoring systems) much faster than a typical new hire.

---

## Interview Talking Points

**Why this matters:**
- Demonstrates ability to deliver measurable impact (96% build time reduction) even as an intern
- Shows I can work effectively in large, enterprise codebases
- The internship-to-full-time pipeline shows the team valued my contributions enough to bring me back
- CI/CD optimization is a universally valued skill that shows systems-level thinking

**STAR-ready summary:**
- **Situation**: Joined HPE RAVE team for summer internship; build pipeline was slow
- **Task**: Enhance APIs to standards and improve CI/CD pipeline efficiency
- **Action**: Modularized monolithic pipeline into parallel stages with build caching
- **Result**: 96% build time reduction (50 min → 2 min), improved API interoperability

---

<!-- TODO: Add specific details about:
  - Exact APIs enhanced (which microservice endpoints?)
  - Pipeline tool specifics (was it GitHub Actions from the start, or migrated?)
  - Mentor/manager name if comfortable sharing
  - Any presentations or demos given at end of internship
  - Number of PRs submitted / code reviews done
-->

*Source: Resume, knowledge graph*
