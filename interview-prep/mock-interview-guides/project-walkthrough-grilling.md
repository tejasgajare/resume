# Project Walkthrough Grilling — "Tell Me About X" Deep Chains

> 5-7 question chains for each major project walkthrough.
> These simulate the interviewer saying "Tell me about X" and then drilling down.

---

## Chain 1: RAVE Cloud Platform Walkthrough

### Q1: "Tell me about the RAVE Cloud platform."
**Why asked:** Can they give a clear overview?
**Expected:** Wellness automation engine for HPE GreenLake — monitors infrastructure, creates support cases. 20+ Go microservices on K8s. Event-driven (Kafka). Multi-tenant (workspace isolation).
**Red flags:** 🚩 Can't explain what the platform does at a business level
**Green flags:** ✅ Business purpose + technical architecture in 2-3 sentences

### Q2: "What's your role on the team?"
**Why asked:** Distinguish contributor from observer.
**Expected:** Primary developer on wellnessProducer. Owned JWT auth, file uploads, manual case workflow, monitoring. Fixed 13+ security vulnerabilities. Team: GLCP Wellness/RAVE.
**Red flags:** 🚩 "I'm part of the team that works on RAVE" — vague
**Green flags:** ✅ Names specific service, specific contributions, specific numbers

### Q3: "Walk me through a request flow — a user creates a support case."
**Why asked:** End-to-end system understanding.
**Expected:** Client → wellnessProducer (JWT auth → validate request → determine routing) → if email domain: caseRelay → SendGrid → email notification. If HPE domain: crmRelay → DCM CRM API → case created in CRM. Case metadata stored in MongoDB. Prometheus metrics incremented.
**Red flags:** 🚩 Can't trace a request end-to-end, or skips auth
**Green flags:** ✅ Full flow with auth, routing decision, storage, and monitoring

### Q4: "The CRM API returns an error. What happens?"
**Why asked:** Error handling and resilience.
**Expected:** crmRelay retries with exponential backoff. If retries exhausted, error logged, metric incremented (crm_relay_failure_total), alert fires in Grafana → PagerDuty/Slack. User gets an error response. Case may be in partial state in MongoDB.
**Red flags:** 🚩 "The user gets an error" without explaining the retry and alerting
**Green flags:** ✅ Retry logic, metrics, alerting, handling of partial state

### Q5: "You mentioned monitoring. What specifically would you see in Grafana right now?"
**Why asked:** Operational depth — do they actually use these tools?
**Expected:** mc_event_case_action_total counters by action type, CRM success/failure rates, service health (pod count, restart count), request latency percentiles, auth failure spikes.
**Red flags:** 🚩 "Dashboards showing service health" — too generic
**Green flags:** ✅ Names specific metrics, knows what normal looks like vs anomalous

### Q6: "What's the thing about this system that keeps you up at night?"
**Why asked:** Risk awareness and honesty.
**Expected:** Cross-tenant isolation depends on every query being scoped — one missed filter is a security breach. CRM dependency is fragile. No distributed tracing makes cross-service debugging hard. ESO migration is incomplete.
**Red flags:** 🚩 "Nothing, it's solid"
**Green flags:** ✅ Specific risks with reasoning, shows they think about failure modes

### Q7: "If you had 3 months with no feature pressure, what would you improve?"
**Why asked:** Technical vision and prioritization.
**Expected:** Add distributed tracing (Jaeger), complete ESO migration (17 remaining), add SLO framework, improve integration test coverage, implement the AI Grafana concept, refactor auth middleware into a shared library.
**Red flags:** 🚩 "Nothing comes to mind" or "rewrite everything"
**Green flags:** ✅ Prioritized list with rationale, practical improvements

---

## Chain 2: Nimble Download Walkthrough

### Q1: "Tell me about the Nimble Download system."
**Why asked:** Can they explain a compliance system clearly?
**Expected:** Enforces US export control on Nimble Storage software downloads. 4 microservices: AutoOnboard (RPL screening), DownloadGatekeeper (ELC at download time), GTCacheAPI (caching), reference tools. Integrates with GTCaaS for compliance decisions. MongoDB for device shadows, SQLite for IP cache.
**Red flags:** 🚩 "It checks if downloads are allowed"
**Green flags:** ✅ Regulatory context, architecture specifics, multiple components

### Q2: "Why four microservices instead of one?"
**Why asked:** Architecture justification.
**Expected:** Different scaling needs: AutoOnboard is batch (RPL feed processing), DownloadGatekeeper is real-time (per-download), GTCacheAPI needs low latency. Different SLAs: download blocking is user-facing, onboarding is background. Single responsibility principle.
**Red flags:** 🚩 "That's how it was designed" — no rationale
**Green flags:** ✅ Scaling/SLA rationale per service, single responsibility principle

### Q3: "Walk me through what happens when a new device is discovered."
**Why asked:** End-to-end flow understanding.
**Expected:** Device appears in RPL feed → AutoOnboard picks it up → creates device shadow in MongoDB (state: PENDING) → calls GTCaaS RPL Screening API → if clear: state → ONBOARDED → if match: state → BLOCKED. Periodic re-screening for ONBOARDED devices.
**Red flags:** 🚩 Can't explain the state machine transitions
**Green flags:** ✅ State machine with all transitions, re-screening concept

### Q4: "Now someone tries to download software. Walk me through that."
**Why asked:** Second flow — download-time enforcement.
**Expected:** Download request → DownloadGatekeeper intercepts → checks IP geolocation (GTCacheAPI, then GTCaaS if cache miss) → checks license/account status → checks device shadow state (must be ONBOARDED) → allow or deny. Cache hit saves GTCaaS call.
**Red flags:** 🚩 Skips IP check or device state check
**Green flags:** ✅ All three checks, cache-first strategy, fail-closed on uncertainty

### Q5: "Why SQLite for caching instead of Redis?"
**Why asked:** Trade-off reasoning.
**Expected:** Zero external dependencies (no Redis cluster to manage), embedded in the service, persistent (survives pod restarts), sufficient throughput for our volume. Redis would be needed if: multiple DownloadGatekeeper instances needed shared cache, or higher throughput required.
**Red flags:** 🚩 "SQLite was easier" without trade-off discussion
**Green flags:** ✅ Explicit trade-off analysis, knows when to upgrade to Redis

### Q6: "The GTCaaS team is switching from Neustar to MaxMind. How does that affect you?"
**Why asked:** Migration planning skills.
**Expected:** Designed cache to abstract over the provider. Key change: MaxMind inlines country in the screening response, so we don't need a separate geo API call. geo_country column in cache is kept but only populated on 403 embargo responses. Minimal code changes because of the abstraction.
**Red flags:** 🚩 "We'd need to rewrite the caching layer"
**Green flags:** ✅ Abstraction-based migration, specific API differences, minimal code changes

---

## Chain 3: Security Work Walkthrough

### Q1: "Tell me about the security work you did at HPE."
**Why asked:** Security awareness and contribution.
**Expected:** 13+ security/validation fixes. Major: cross-workspace case creation (GLCP-333624), environment-specific JWT issuers, file upload security (magic bytes), input validation, filename sanitization.
**Red flags:** 🚩 "I fixed some bugs" — doesn't frame as security
**Green flags:** ✅ Categorizes by severity, names specific vulnerabilities

### Q2: "Walk me through the cross-workspace vulnerability."
**Why asked:** Can they explain a real security fix?
**Expected:** v2 Create Case API accepted workspace_id from request body. Attacker could supply another tenant's workspace_id → case created in wrong namespace. Fix: derive workspace from JWT hpe_workspace_id claim (server-side), ignore client input. Added regression tests.
**Red flags:** 🚩 Describes the fix but not the vulnerability, or can't explain the attack vector
**Green flags:** ✅ Attack vector, root cause, fix, verification (tests), broader impact

### Q3: "How did you discover this? Was it in a pen test or did you find it yourself?"
**Why asked:** Proactive vs reactive security awareness.
**Expected:** Discovered during code review / security audit of the case creation flow. Not a customer-reported incident or pen test finding.
**Red flags:** 🚩 "Someone told me about it"
**Green flags:** ✅ Proactive discovery through code review or security thinking

### Q4: "What other input validation issues did you find?"
**Why asked:** Breadth of security work.
**Expected:** Contact email validation (required email, not just phone), severity allowlist (reject invalid values), filename sanitization (path traversal, null bytes, special chars), attachment content-type vs extension mismatch, field length limits.
**Red flags:** 🚩 "Just the one big one"
**Green flags:** ✅ Multiple validation categories, specific examples with reasoning

### Q5: "If I were a malicious tenant, what's the next attack vector you'd worry about?"
**Why asked:** Threat modeling thinking.
**Expected:** IDOR (Insecure Direct Object Reference) — can a tenant enumerate other tenants' case IDs? Rate limiting — can a tenant flood the system? Attachment content — is a PDF actually safe? Kafka message injection — can a service impersonate another? Unscoped MongoDB queries in other services.
**Red flags:** 🚩 "I think we've covered everything"
**Green flags:** ✅ Multiple attack vectors prioritized by likelihood and impact

### Q6: "How do you bake security into the development process?"
**Why asked:** Process improvement vs firefighting.
**Expected:** Code review with security lens, input validation as a standard step, regression tests for security fixes, SonarQube in CI, security-focused sprint work when needed, threat modeling during design.
**Red flags:** 🚩 "We fix things as they come up"
**Green flags:** ✅ Systematic approach: design-time + build-time + test-time + deploy-time security

---

## Chain 4: Side Project Walkthrough (ClipStash)

### Q1: "Tell me about ClipStash."
**Why asked:** Personal project depth — shows passion and self-direction.
**Expected:** macOS clipboard history manager in Swift 6/SwiftUI. Features: clipboard monitoring, full-fidelity format preservation (all pasteboard representations), floating panel UI, global hotkey, SQLite via GRDB.
**Red flags:** 🚩 "It's a simple clipboard app"
**Green flags:** ✅ Full-fidelity preservation concept, specific technologies, architectural decisions

### Q2: "What's 'full-fidelity clipboard format preservation'?"
**Why asked:** Technical depth on a unique feature.
**Expected:** macOS pasteboard can contain multiple representations of the same data (plain text, RTF, HTML, images) simultaneously. ClipStash stores ALL representations, not just plain text. When you paste, it restores the original representations so the target app gets the richest format.
**Red flags:** 🚩 "It saves what you copy" — no understanding of multiple formats
**Green flags:** ✅ NSPasteboard representations, format fidelity, why it matters for user experience

### Q3: "How does the floating panel UI work?"
**Why asked:** Architecture depth on macOS-specific patterns.
**Expected:** NSPanel subclass (not NSWindow) — appears above other windows. ClipHistoryPanelController manages the panel lifecycle. Panel content view is recreated on each show() call (not cached) because SwiftUI's .onAppear only fires once if the panel is reused.
**Red flags:** 🚩 "SwiftUI view with a list"
**Green flags:** ✅ NSPanel vs NSWindow, lifecycle management, recreating content view gotcha

### Q4: "How do you test a SwiftUI app?"
**Why asked:** Testing practices for UI code.
**Expected:** Snapshot testing: renderInWindow() helper hosts views in real NSWindow with RunLoop pumping for .onAppear/state updates. Set window.appearance for consistent dark/light mode. Use bitmapImageRepForCachingDisplay for captures. Unit tests for business logic (DatabaseManager with inMemory mode).
**Red flags:** 🚩 "We don't test UI" or "Manual testing"
**Green flags:** ✅ Snapshot testing with specific technique, in-memory DB for unit tests

### Q5: "What's the hardest technical problem you solved in ClipStash?"
**Why asked:** Ownership verification — did they actually build it?
**Expected:** SwiftUI padding bug: .padding() on top of .formStyle(.grouped) causes double spacing and clips content. Took debugging to figure out that grouped form provides internal spacing. Or: panel lifecycle — .onAppear not firing on reused panels, requiring content view recreation.
**Red flags:** 🚩 Generic "getting it to work"
**Green flags:** ✅ Specific bug with specific cause and solution, shows real debugging

---

## Chain 5: Infrastructure & DevOps Walkthrough

### Q1: "Tell me about the infrastructure you manage."
**Why asked:** Breadth of infrastructure knowledge.
**Expected:** K8s clusters across 3 environments per RAVE variant. Helm charts for 20+ services. Istio service mesh (RequestAuthentication, VirtualService). ArgoCD for GitOps. ESO for secrets. GitHub Actions CI/CD. Terraform for AWS (S3, IAM, EKS).
**Red flags:** 🚩 "I deploy to Kubernetes" — too generic
**Green flags:** ✅ Specific tools with specific configurations, multi-environment management

### Q2: "Walk me through deploying a new version of wellnessProducer."
**Why asked:** End-to-end deployment knowledge.
**Expected:** Push code → GitHub Actions triggers (build, test, SonarQube, Docker build, push image) → Update Helm chart version → ArgoCD detects change → syncs to K8s → rolling update (maxUnavailable: 0, maxSurge: 1) → health check probes pass → traffic shifts.
**Red flags:** 🚩 "kubectl apply" or can't explain the pipeline
**Green flags:** ✅ Full pipeline with CI/CD, ArgoCD, rolling update details, health probes

### Q3: "Something went wrong in the deployment. How do you rollback?"
**Why asked:** Operational recovery.
**Expected:** ArgoCD: revert to previous sync (ArgoCD UI or CLI). Or: revert the Helm chart version in Git → ArgoCD re-syncs. Considerations: if the issue is a database migration, rollback may not be straightforward. Check if the new version changed data formats.
**Red flags:** 🚩 "kubectl rollout undo" without understanding the GitOps flow
**Green flags:** ✅ GitOps rollback (Git revert), ArgoCD sync, database migration awareness

### Q4: "Tell me about the ESO migration."
**Why asked:** Specific infrastructure contribution.
**Expected:** Migrated 3 services from Vault to ESO (caseRelay, crmRelay, wellnessProducer). ESO syncs secrets from AWS Parameter Store to K8s Secrets. Required: per-service ServiceAccount with IRSA IAM role annotation, SecretStore CRD, ExternalSecret CRDs. Maintained backward-compatible Secret names.
**Red flags:** 🚩 "We switched from Vault to ESO"
**Green flags:** ✅ Specific services, IAM setup, CRD details, backward compatibility

### Q5: "Why migrate from Vault to ESO?"
**Why asked:** Decision rationale.
**Expected:** Vault: requires dedicated server, operational overhead, license considerations. ESO: operator-based (runs in K8s), uses AWS managed services (Parameter Store), simpler for static secrets. Trade-off: Vault has dynamic secrets (DB credentials), ESO doesn't. Our use case is static config secrets — ESO is sufficient and simpler.
**Red flags:** 🚩 "We were told to migrate" — no understanding of why
**Green flags:** ✅ Trade-off analysis, understanding of both systems' strengths

### Q6: "You have 17 services still on Vault. What's the plan?"
**Why asked:** Planning and prioritization.
**Expected:** Documented the migration pattern from the first 3 services. Prioritized by risk (production-critical services first). Any team member can follow the documented pattern. Advocating for sprint capacity allocation. Not urgent (Vault still works) but represents operational debt.
**Red flags:** 🚩 "We'll get to it"
**Green flags:** ✅ Documented pattern, prioritization framework, realistic assessment of urgency
