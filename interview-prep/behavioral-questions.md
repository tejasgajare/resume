# Behavioral Questions — STAR Format Interview Prep

> Tejas Gajare — Software Developer II, HPE GLCP Wellness/RAVE Team
> Each answer follows STAR format with scoring rubric and follow-ups.

---

## 1. Leadership & Ownership

### Q: "Tell me about a time you owned a critical system."

**Situation:**
I was the primary developer on the wellnessProducer microservice in the RAVE Cloud platform at HPE. This Go-based service was the central API layer for HPE GreenLake's wellness automation — handling case creation, file uploads, JWT authentication, and multi-tenant data access for enterprise customers. It served as the gateway between the frontend UI and backend case processing services (caseRelay, crmRelay). Any outage directly impacted HPE customer support workflows.

**Task:**
I needed to evolve the service through three API versions (v1→v2→v3), fix critical security vulnerabilities, implement the entire manual case creation workflow, build the file upload system, and overhaul the monitoring stack — all while maintaining backward compatibility and zero-downtime deployments.

**Action:**
- Designed and implemented v3 APIs with environment-specific JWT issuer validation, replacing hardcoded regex that accepted tokens from any environment
- Built multi-layer file upload validation (filename sanitization, extension allowlist, magic byte verification)
- Fixed cross-workspace case creation vulnerability (GLCP-333624) by enforcing server-side workspace derivation from JWT claims
- Added comprehensive Prometheus metrics with unified `mc_event_case_action_total` counter
- Configured PagerDuty and Slack alerting for service health and processing drops
- Wrote Confluence documentation for manual case payload specifications

**Result:**
- 13+ security/validation fixes shipped across the platform
- v3 APIs deployed with zero breaking changes to existing v1/v2 consumers
- Monitoring reduced incident detection time — automated alerts caught issues before user reports
- Became the go-to person for wellnessProducer questions across the team

[KEY_POINTS] Specific service name, concrete versioning work, security fixes with Jira IDs, measurable outcomes
[COMMON_MISTAKES] Saying "we" without clarifying personal role; vague "I worked on the API"
[FOLLOW_UP] "What was the hardest bug you debugged in this service?"
[FOLLOW_UP] "How did you prioritize between security fixes and feature work?"
[FOLLOW_UP] "What would you do differently if you redesigned it?"
[SCORING: 1-5]
- 1: "I worked on a Go service at HPE"
- 2: "I maintained wellnessProducer and added some features"
- 3: "I owned the API versioning and some security fixes"
- 4: "I drove v1→v3 evolution, fixed cross-workspace auth, built file uploads, monitoring"
- 5: Above + explains trade-offs, connects to team impact, mentions what they'd improve

---

## 2. Conflict Resolution

### Q: "Describe a technical disagreement you had."

**Situation:**
During the design of the monitoring overhaul for RAVE Cloud, there was a discussion about how to structure Prometheus metrics for case creation events. The existing pattern had separate metrics for each action (caseCreated, caseUpdated, caseReopened), and some team members preferred keeping them separate for simplicity.

**Task:**
I needed to advocate for a better metrics architecture while respecting existing patterns and the team's familiarity with the current approach.

**Action:**
- I analyzed the existing Grafana dashboards and showed how separate metrics led to duplicated dashboard panels and more complex PromQL queries
- Proposed a unified `mc_event_case_action_total` counter with an `action` label (`created`, `updated`, `reopened`, `new_open`)
- Created a side-by-side comparison showing the PromQL queries before and after
- Demonstrated the cardinality was bounded (only 4 action values) so label-based approach wouldn't cause performance issues
- Built a prototype Grafana dashboard showing how the unified metric simplified visualization

**Result:**
- Team adopted the unified metrics approach
- Dashboard complexity reduced significantly — one panel with legend filtering vs four separate panels
- Pattern was adopted for other metrics in the service
- Improved alert rules — single query could catch drops across all action types

[KEY_POINTS] Technical evidence over opinion, prototype-driven persuasion, bounded cardinality awareness
[COMMON_MISTAKES] "I just told them my way was better" — no evidence; or "I gave in" — no conviction
[FOLLOW_UP] "How did you handle the transition period?"
[FOLLOW_UP] "What if your approach had caused a cardinality explosion?"
[SCORING: 1-5]
- 1: Can't recall a disagreement
- 2: "We disagreed and compromised"
- 3: Describes the disagreement with context
- 4: Shows evidence-based advocacy with specific technical details
- 5: Above + explains the trade-off analysis, how they'd handle if wrong, and broader impact

---

## 3. Handling Ambiguity

### Q: "Tell me about a project with unclear requirements."

**Situation:**
The Manual Case Creation workflow for RAVE was assigned with a high-level requirement: "users should be able to create support cases manually through the GreenLake portal." The Jira story had minimal acceptance criteria. Questions like which fields were required, how routing should work (email vs CRM), what validation rules applied, and how attachments should be handled were all undefined.

**Task:**
I needed to design and implement the complete workflow despite requirement gaps, while ensuring the system was secure and extensible for future needs.

**Action:**
- Started by reviewing the existing automated case creation flow to understand the data model and downstream service contracts (caseRelay, crmRelay)
- Identified the routing decision: email domain customers → SendGrid (via caseRelay) vs HPE domain customers → DCM CRM API (via crmRelay)
- Designed the ManualCaseData schema by examining what downstream systems needed and what the frontend team expected
- Built validation rules iteratively — started strict, relaxed based on frontend team feedback
- Discovered edge cases through implementation: attachments arriving before case numbers (crmRelay race condition), contacts needing email validation, createdBy needing JWT extraction
- Published comprehensive Confluence documentation for the payload specification so frontend and QA could align
- Created the attachment retry logic in crmRelay for the race condition

**Result:**
- Shipped a complete manual case creation pipeline with minimal requirement churn
- The Confluence doc became the source of truth for all teams
- Proactive edge case discovery prevented production bugs
- Pattern of "implement, discover, document" was reused for subsequent features

[KEY_POINTS] Proactive requirement discovery, documented decisions, iterative validation, edge case finding
[COMMON_MISTAKES] Waiting for complete requirements instead of iterating; not documenting decisions
[FOLLOW_UP] "How did you decide which fields to make required?"
[FOLLOW_UP] "What if you had gotten the data model wrong?"
[SCORING: 1-5]
- 1: "Requirements were unclear so I asked my manager"
- 2: "I figured out the requirements by asking people"
- 3: "I analyzed existing systems and designed the workflow"
- 4: "I reverse-engineered from downstream contracts, iterated, documented, found edge cases"
- 5: Above + discusses the decision framework, cross-team alignment, and what they'd systematize

---

## 4. Failure & Learning

### Q: "What's a mistake you made in production?"

**Situation:**
The JWT authentication middleware in wellnessProducer had a security gap I didn't initially catch. The auth validation used hardcoded regex patterns (`arubathena.com`, `common.cloud.hpe.com`) for JWT issuer validation. This meant tokens from ANY environment (dev, staging, prod) were accepted in ALL environments — a dev token could access production APIs.

**Task:**
When I discovered this during a security review, I needed to fix it without breaking existing integrations, coordinate with the infrastructure team's Istio configuration, and ensure no environment accepted tokens from another.

**Action:**
- Analyzed how Istio RequestAuthentication was configured in Helm values — each environment already had correct issuers
- Realized the Go middleware was a separate layer that wasn't aligned with Istio's config
- Designed environment-specific issuer validation: the middleware reads allowed issuers from environment variables matching the Helm-configured values
- Implemented the fix with backward compatibility — existing tokens continued working in their correct environment
- Added auth failure metrics (`wellness_producer_auth_failure_total`) to detect misconfiguration
- Tested across all three environments (dev/intg/prod) before rollout

**Result:**
- Eliminated cross-environment token acceptance vulnerability
- Auth middleware now aligned with Istio RequestAuthentication configuration
- Auth failure metrics provided early warning for token issues
- Led to a broader review of auth patterns across other RAVE services

[KEY_POINTS] Honest about the gap, systematic fix, testing across environments, metrics for detection
[COMMON_MISTAKES] Blaming others; minimizing the issue; not explaining the fix clearly
[FOLLOW_UP] "How did you discover this issue?"
[FOLLOW_UP] "How do you prevent similar issues in the future?"
[FOLLOW_UP] "What was the blast radius if this had been exploited?"
[SCORING: 1-5]
- 1: "We had a bug and fixed it"
- 2: Describes the issue without ownership
- 3: Takes ownership, explains the fix
- 4: Detailed explanation with root cause analysis and prevention
- 5: Above + connects to systemic improvements, metrics, and team learning

---

## 5. Cross-Team Collaboration

### Q: "Describe working with another team on a shared system."

**Situation:**
The Nimble Download Global Trade Compliance system required collaboration between our RAVE/Wellness team and the Nimble Storage team. The system needed to integrate with GTCaaS (Global Trade Compliance as a Service), which was maintained by a separate compliance team. My services needed to call their RPL and ELC screening APIs, and they were planning a migration from Neustar to MaxMind for IP geolocation.

**Task:**
I needed to design and build the DownloadGatekeeper and AutoOnboard services that would call GTCaaS APIs, while coordinating the upcoming MaxMind migration that would change the API contract.

**Action:**
- Met with the GTCaaS team to understand their API contract, rate limits, and migration timeline
- Designed the IP cache (SQLite) to abstract over the geolocation provider — when MaxMind replaced Neustar, only the cache population logic needed to change
- Planned the migration: geo_country column kept in cache but only populated on 403 embargo responses (MaxMind's API returns country inline with screening response, not separate)
- Built the DownloadGatekeeper with a clean separation between compliance logic and provider-specific details
- Documented the migration plan and shared with both teams
- Coordinated testing across environments with the GTCaaS team

**Result:**
- DownloadGatekeeper shipped with 60%+ reduction in redundant API calls via caching
- MaxMind migration plan minimized code changes — cache abstraction proved its value
- Both teams had clear documentation and shared test plans
- The abstraction pattern was appreciated by the GTCaaS team for their own clients

[KEY_POINTS] Cross-team coordination, abstraction for provider migration, shared documentation, concrete metrics
[COMMON_MISTAKES] Taking sole credit or not explaining the collaboration dynamic
[FOLLOW_UP] "How did you handle disagreements with the other team?"
[FOLLOW_UP] "What was the communication cadence?"
[SCORING: 1-5]
- 1: "We worked with another team"
- 2: "I called their APIs"
- 3: "I coordinated the integration and tested together"
- 4: "I designed abstractions anticipating their migration, documented, and coordinated testing"
- 5: Above + explains the communication process, how risks were managed, and lessons learned

---

## 6. Delivering Under Pressure

### Q: "Tell me about a time you had to deliver something quickly."

**Situation:**
A security vulnerability (GLCP-333624) was discovered in the v2 Create Case API — it allowed cross-workspace case creation because the backend trusted client-supplied workspace identifiers. This was a P1 security issue that could lead to data leakage between tenants.

**Task:**
Fix the vulnerability with minimal disruption, test across environments, and deploy before the next security review deadline.

**Action:**
- Identified the root cause: the API handler used workspace_id from the request body instead of extracting it from the JWT token's `hpe_workspace_id` claim
- Implemented server-side workspace derivation — the handler now ignores client-supplied workspace and extracts it from the authenticated JWT
- Added input validation for all other fields in the same pass (GLCP-325722)
- Wrote regression tests verifying that client-supplied workspace_id is ignored
- Coordinated with the frontend team to confirm they didn't depend on the old behavior
- Deployed through dev → intg → prod with verification at each stage

**Result:**
- Vulnerability patched before the security deadline
- No customer-facing disruption — the fix was transparent to well-behaved clients
- Broader security review identified 12+ additional validation improvements
- Established pattern: all tenant-scoped data must derive from JWT, never from request body

[KEY_POINTS] P1 urgency, root cause clarity, regression tests, coordinated deployment
[COMMON_MISTAKES] Not quantifying urgency; skipping the testing/verification part
[FOLLOW_UP] "How do you prevent similar vulnerabilities in new APIs?"
[SCORING: 1-5]
- 3: Fixes the bug, mentions testing
- 5: Root cause, systemic fix, regression tests, cross-team coordination, prevention pattern

---

## 7. Innovation / Going Above and Beyond

### Q: "Tell me about something you built that wasn't asked of you."

**Situation:**
Our team spent significant time manually creating sprint review documents in Confluence — gathering Jira story statuses, writing summaries, formatting burndown charts. This was a 2-3 hour task every sprint.

**Task:**
I identified an opportunity to automate this entirely using AI tooling.

**Action:**
- Built a Copilot CLI skill (`gen-sprint-review-doc`) that:
  - Fetches all stories from the current Jira sprint via API
  - Generates burndown chart data
  - Creates a formatted Confluence page with Jira Smart Links, @mentions, and status lozenges
  - Produces rich ADF (Atlassian Document Format) content, not just markdown
- Also built a `sprint-assign` skill that transitions all sprint issues to "Assigned" status with validation (checks story points on Stories, handles LaunchDarkly field, skips Blocked issues)
- Both skills are self-contained — any team member with Copilot CLI can run them

**Result:**
- Sprint review generation reduced from 2-3 hours to under 2 minutes
- Consistent formatting across all sprint reviews
- Team adopted both skills as standard workflow tools
- Demonstrated practical AI automation value to the team

[KEY_POINTS] Self-initiated, practical automation, time savings quantified, team adoption
[COMMON_MISTAKES] Describing the tool without explaining the problem it solved
[FOLLOW_UP] "How did you decide which parts to automate?"
[FOLLOW_UP] "What's the maintenance burden?"
[SCORING: 1-5]
- 3: Built a useful tool
- 5: Identified problem, built solution, quantified impact, drove adoption

---

## 8. Mentoring / Knowledge Sharing

### Q: "How do you share knowledge with your team?"

**Situation:**
As I built out complex systems (JWT auth, file uploads, manual case workflow), the team needed to understand these systems for code reviews, on-call, and future development.

**Task:**
Make my work accessible and understandable to the team.

**Action:**
- Published comprehensive Confluence documentation for every major feature:
  - Manual Case Payload documentation covering all fields, validation rules, and routing logic
  - JWT auth flow documentation explaining token validation algorithm
  - Monitoring runbooks linked from PagerDuty alerts
- Added inline code documentation for complex logic (magic byte validation, BSON conversion)
- Created Grafana dashboards that served as both monitoring AND documentation of system behavior
- Participated actively in code reviews with explanatory comments on complex patterns

**Result:**
- On-call engineers could handle wellnessProducer incidents using runbooks
- New team members onboarded faster with comprehensive docs
- Code review quality improved — reviewers understood the context

[KEY_POINTS] Multiple knowledge-sharing channels (docs, code comments, dashboards, reviews)
[COMMON_MISTAKES] "I documented everything" without specifics
[FOLLOW_UP] "How do you keep documentation up to date?"
[SCORING: 1-5]
- 3: Writes documentation
- 5: Multi-channel knowledge sharing with measurable team impact

---

## General Behavioral Tips

### Framework: STAR + So What
Every answer should end with "So what?" — the broader impact or lesson learned.

### Red Flags to Avoid
- Using "we" exclusively without clarifying personal contribution
- No specific details (Jira IDs, service names, metrics)
- No acknowledgment of challenges or trade-offs
- Rehearsed-sounding answers without depth

### Green Flags to Hit
- Named specific systems, technologies, and outcomes
- Explained the "why" behind decisions
- Acknowledged failures and learnings
- Connected individual work to team/business impact
- Quantified results where possible

[KEY_POINTS] Always have 3 behavioral stories ready: ownership, failure, collaboration
[COMMON_MISTAKES] Preparing only success stories; not practicing the follow-up questions
[SCORING: 1-5] Score yourself after each practice answer. Aim for 4+ consistently.
