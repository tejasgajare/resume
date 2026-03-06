# Behavioral Grilling — Deep Question Chains

> 5-7 question chains drilling progressively deeper into behavioral competencies.
> Each question has expected answer, red flags, and green flags.

---

## Chain 1: Ownership & Accountability

### Q1: "Tell me about a system you own."
**Why asked:** Baseline — can the candidate articulate ownership?
**Expected:** Names wellnessProducer, explains what it does, their specific responsibilities.
**Red flags:** 🚩 Uses "we" exclusively, can't name the service, vague responsibilities
**Green flags:** ✅ Names specific service, API versions, personal decisions made

### Q2: "What does 'owning' a service mean to you in practice?"
**Why asked:** Tests understanding of ownership beyond code — includes on-call, monitoring, documentation.
**Expected:** Code, monitoring, documentation, on-call, security, API evolution, team enablement.
**Red flags:** 🚩 "I write the code" — ownership is only about code
**Green flags:** ✅ Mentions monitoring/alerting, documentation, making others productive

### Q3: "Walk me through the last production issue you handled for this service."
**Why asked:** Tests operational ownership — do they actually handle incidents?
**Expected:** CRM attachment race condition or JWT issuer bug. Specific debugging steps, metrics used.
**Red flags:** 🚩 Can't recall a specific incident, generic "we looked at logs"
**Green flags:** ✅ Describes specific metrics checked, root cause, fix, and prevention

### Q4: "What's the worst thing about this service that you haven't fixed yet?"
**Why asked:** Tests honesty and prioritization. Every system has debt.
**Expected:** Honest assessment — maybe lack of distributed tracing, alert noise, remaining ESO migrations, or incomplete test coverage.
**Red flags:** 🚩 "It's in pretty good shape" — either dishonest or not deeply involved
**Green flags:** ✅ Names specific tech debt, explains why it's not yet fixed (priorities), has a plan

### Q5: "If you left tomorrow, would the team be able to maintain this service?"
**Why asked:** Tests knowledge sharing and bus factor awareness.
**Expected:** Yes — documentation on Confluence, monitoring runbooks, code comments on complex logic. Acknowledges areas where tribal knowledge exists.
**Red flags:** 🚩 "They'd be fine" without explaining why, or "They'd struggle" without having tried to address it
**Green flags:** ✅ Concrete documentation, runbooks, dashboard walkthroughs, areas they'd want to improve

### Q6: "How do you decide what to work on when there are competing priorities?"
**Why asked:** Tests judgment and communication with stakeholders.
**Expected:** Security first (P1 vulnerability fixed before feature work), then customer impact, then tech debt. Communicates priority decisions to team/manager.
**Red flags:** 🚩 "My manager tells me what to work on" — no agency
**Green flags:** ✅ Framework for prioritization, example of pushing back or reprioritizing

[SCORING: 1-5]
- 1-2: Can't articulate ownership or give specifics
- 3: Describes responsibilities but lacks depth
- 4: Clear ownership with incidents, documentation, and priorities
- 5: Holistic ownership including team enablement, honest about gaps, proactive prioritization

---

## Chain 2: Failure & Growth

### Q1: "What's a technical mistake you made?"
**Why asked:** Baseline honesty check.
**Expected:** JWT hardcoded issuer regex, or BSON deserialization oversight, or missing input validation.
**Red flags:** 🚩 "I can't think of any mistakes" or trivially small mistakes
**Green flags:** ✅ Significant mistake, clearly explained, shows self-awareness

### Q2: "How did you discover it?"
**Why asked:** Proactive discovery vs reactive (user-reported, incident).
**Expected:** Security review for JWT, debugging CRM failures for BSON, or test failure.
**Red flags:** 🚩 Customer reported it, never would have found it themselves
**Green flags:** ✅ Proactive discovery through review, testing, or monitoring

### Q3: "What was the blast radius if this hadn't been caught?"
**Why asked:** Risk assessment skills — understanding impact.
**Expected:** Cross-environment token acceptance → prod data accessible with dev tokens. Cross-workspace → tenant data leakage.
**Red flags:** 🚩 "It wasn't that serious" when it clearly was
**Green flags:** ✅ Honest about severity, can articulate the worst-case scenario

### Q4: "What systemic change did you make to prevent similar issues?"
**Why asked:** Growth beyond fixing the bug — improving the system.
**Expected:** Environment-specific issuers, auth metrics, regression tests, input validation framework, documentation.
**Red flags:** 🚩 Just fixed the bug and moved on
**Green flags:** ✅ Systemic improvements — tests, metrics, documentation, process changes

### Q5: "What's a mistake you'd make differently if you could go back?"
**Why asked:** Reflection and learning depth.
**Expected:** Would have added integration tests earlier, would have aligned Istio and middleware auth from day one, would have questioned the original design.
**Red flags:** 🚩 "I wouldn't change anything" — no growth mindset
**Green flags:** ✅ Specific alternative approach, explains why it would be better

### Q6: "How do you balance thoroughness with speed?"
**Why asked:** Practical judgment in engineering trade-offs.
**Expected:** Example — security fixes get thorough testing, feature work gets MVP then iterate. P1 vulnerability was fixed quickly but with regression tests.
**Red flags:** 🚩 "I always prioritize quality" (unrealistic) or "Ship fast, fix later" (irresponsible)
**Green flags:** ✅ Context-dependent, examples of both fast and thorough, knows when each is appropriate

---

## Chain 3: Collaboration & Communication

### Q1: "Describe working with the GTCaaS team on Nimble."
**Why asked:** Can they collaborate cross-team?
**Expected:** API contract discussions, migration planning, shared documentation, coordinated testing.
**Red flags:** 🚩 "They gave us an API and we called it" — no collaboration
**Green flags:** ✅ Proactive communication, abstraction design anticipating their migration

### Q2: "What was the hardest part of that collaboration?"
**Why asked:** Realistic assessment of cross-team challenges.
**Expected:** Different timelines, API contract changes, testing across environments, ownership boundaries.
**Red flags:** 🚩 "It was easy" — either not involved or not honest
**Green flags:** ✅ Specific challenge with how they resolved it

### Q3: "How do you communicate technical decisions to non-technical stakeholders?"
**Why asked:** Communication breadth — not just engineers.
**Expected:** Confluence documentation, security impact framed in business terms, risk summaries.
**Red flags:** 🚩 "I only talk to engineers"
**Green flags:** ✅ Translates technical risk to business impact, written documentation

### Q4: "How do you handle code review disagreements?"
**Why asked:** Constructive feedback skills.
**Expected:** Data-driven discussions (metrics comparison for monitoring change), prototype-based persuasion, willingness to be wrong.
**Red flags:** 🚩 "I just approve if they insist" or "My way is usually right"
**Green flags:** ✅ Specific example with evidence-based resolution

### Q5: "How do you onboard a new team member to your service?"
**Why asked:** Knowledge sharing and team building.
**Expected:** Point them to Confluence docs, walk through architecture in a pairing session, assign a starter task, review their first PR thoroughly.
**Red flags:** 🚩 "They can read the code"
**Green flags:** ✅ Structured onboarding, documentation, mentoring approach

---

## Chain 4: Initiative & Impact

### Q1: "Tell me about something you built that wasn't on the roadmap."
**Why asked:** Self-direction and initiative.
**Expected:** Sprint review generator skill, sprint assignment automation. Identified problem, built solution, drove adoption.
**Red flags:** 🚩 Only works on assigned tasks
**Green flags:** ✅ Identifies problems proactively, builds solutions, measures impact

### Q2: "How did you get buy-in for it?"
**Why asked:** Influence without authority.
**Expected:** Demonstrated time savings (2-3 hours → 2 minutes), showed working prototype, low risk (optional tool, doesn't affect production).
**Red flags:** 🚩 "I just built it and told people" — no persuasion
**Green flags:** ✅ Demonstrated value, addressed concerns, got team adoption

### Q3: "What's the maintenance burden?"
**Why asked:** Practical thinking — tools need maintenance.
**Expected:** Honest — Jira API changes could break it, Confluence ADF format changes, but it's self-contained and easy to update.
**Red flags:** 🚩 "It just works" — unrealistic
**Green flags:** ✅ Acknowledges maintenance, has mitigation plan

### Q4: "What's the biggest impact you've had beyond your assigned work?"
**Why asked:** Holistic impact assessment.
**Expected:** Security fixes (13+ vulnerabilities found proactively), automation tools, monitoring overhaul, documentation.
**Red flags:** 🚩 Impact limited to assigned stories
**Green flags:** ✅ Proactive improvements across security, tooling, documentation, and team productivity

### Q5: "How do you decide when to stop improving something?"
**Why asked:** Pragmatism and diminishing returns awareness.
**Expected:** When the improvement's marginal benefit is less than the opportunity cost. Example: monitoring is "good enough" to catch critical issues; perfecting dashboards yields diminishing returns vs. building the next feature.
**Red flags:** 🚩 "I always want it perfect" or "Whatever passes code review"
**Green flags:** ✅ Explicit trade-off reasoning, examples of stopping and examples of pushing further

---

## Chain 5: Adaptability & Learning

### Q1: "How did you ramp up on the RAVE codebase?"
**Why asked:** Learning approach for complex systems.
**Expected:** Read code, traced request flows, built small features, asked questions, read Confluence, reviewed PRs.
**Red flags:** 🚩 "Someone trained me" — passive learning
**Green flags:** ✅ Self-directed exploration, multiple learning channels, asks good questions

### Q2: "What's a technology you learned on the job?"
**Why asked:** Learning agility.
**Expected:** Istio, Kafka, Prometheus/Grafana, Helm, ESO — not just used but understood deeply.
**Red flags:** 🚩 "I used it but I don't really understand how it works"
**Green flags:** ✅ Can explain internals, not just surface-level usage

### Q3: "How do you stay current with Go ecosystem changes?"
**Why asked:** Continuous learning habits.
**Expected:** Go blog, release notes, open-source contributions (DiceDB), side projects, community.
**Red flags:** 🚩 "I don't really follow changes"
**Green flags:** ✅ Specific examples of adopting new patterns, contributing to open source

### Q4: "Tell me about a time you had to learn something quickly to solve a problem."
**Why asked:** Under-pressure learning.
**Expected:** Learning magic byte validation for file uploads, BSON primitive types for the deserialization fix, Istio CRDs for the auth alignment.
**Red flags:** 🚩 Generic "I googled it"
**Green flags:** ✅ Specific technical topic, how they learned it, the outcome

### Q5: "What are you learning right now?"
**Why asked:** Growth mindset and curiosity.
**Expected:** Rust (Kairo Tauri port), TSDB internals (Crono), or Swift 6 concurrency model.
**Red flags:** 🚩 Nothing specific
**Green flags:** ✅ Active project demonstrating learning, can explain what they're struggling with
