# Work Portfolio & Interview Knowledge Base — Generation Prompt

> **Usage**: Feed this entire prompt to a Claude Code session (or any capable AI agent with filesystem access) to generate a comprehensive work portfolio and interview prep knowledge base.

---

## Objective

Compile a comprehensive, deeply detailed documentation about all my professional and personal work. This document suite serves as a **complete data lake** covering every project I built, contributed to, or participated in. It must be thorough enough for two distinct use cases:

1. **Personal recall & interview prep** — I should be able to recall for any project: the requirements, challenges faced, solutions explored, why we chose the final solution, architecture, system design, deployment, service interactions, and my specific contributions.
2. **Mock interview agent consumption** — Any other AI agent should be able to load these documents and immediately act as a realistic interviewer/hiring manager who can grill me, validate my answers, rate them, and suggest improvements.

---

## Source Directories to Explore

### Primary Work Repositories
1. `~/glcp/rave-cloud` — Primary RAVE platform (explore all microservices)
2. `~/glcp/rave-classic` — Classic variant of RAVE
3. `~/glcp/rave-lvs` — LVS variant of RAVE
4. `~/glcp/wellness-proxy` — Python proxy for wellness API routing
5. `~/glcp/nimbleDownloadUpdate` — Global trade compliance system
6. `~/glcp/doc-portal` — HPE GreenLake documentation portal
7. `~/glcp/mcd-deploy-proj-rave` — ArgoCD deployment configs
8. *(Add any other repos you find under `~/glcp/` with my commits)*

### Infrastructure & DevOps Repos
- `~/workdir/aws-wellness_dev-config/`
- `~/workdir/aws-wellness_intg-config/`
- `~/workdir/aws-wellness_prod-config/`
- `~/workdir/ccp-terraform-aws-s3/`
- `~/workdir/ccp-rave-*-cluster/`
- `~/glcp/rave-sops-dev/`, `~/glcp/rave-sops-prod/`, `~/glcp/rave-sops-stage/`
- `~/glcp/ccp-humio-resources-config/`

### Reference & Context Sources
1. `~/playground/platform-docs` — Platform documentation
2. `~/playground/` — Side projects and experiments (explore all subdirectories)
3. `~/gajaretejas/resume/` — Current resume (read for existing work history and context, especially internship and Yardi/Grouped roles)

### Personal / Side Projects
- `~/kairo/` — macOS menu bar calendar (Swift/SwiftUI)
- `~/kairo-tauri/` — Cross-platform calendar port (Rust + Tauri)
- `~/ClipStash/` — macOS clipboard manager (Swift 6)
- `~/surmai/` — Travel planning PWA (React + Go/PocketBase)
- `~/dice/` — DiceDB open-source contribution (Go)
- *(Explore `~/playground/` for additional projects)*

### External Context Sources
- **Jira**: Explore all my work items, sprint history, and story details for richer context on requirements, scope, and decisions
- **Confluence**: Read my authored/edited pages for documentation, design decisions, API specs, and cross-team discussions

---

## What to Generate for Each Project

### A. Project Overview & Architecture
- What the project/system does (business purpose)
- Platform and technology stack with specific versions where relevant
- Architecture diagram description (service topology, data flow)
- System design: how microservices interact, communication patterns (REST, Kafka, gRPC), data stores
- Deployment: where and how it's deployed (cloud provider, Kubernetes, Helm, ArgoCD, environments)
- Scale: request volume, data sizes, number of services, number of environments

### B. Requirements & Problem Context
- What problem was being solved and why
- Who the stakeholders were (teams, customers, partners)
- Constraints (security, compliance, performance, backward compatibility)
- What existed before this work (the "before" state)

### C. Challenges & Solution Exploration
- Key technical challenges encountered
- Alternative solutions considered with trade-off analysis (why each was or wasn't chosen)
- The final solution and the reasoning behind it
- Unexpected problems that came up during implementation and how they were resolved

### D. My Specific Contributions
- What I personally designed, built, owned, and maintained (be precise — avoid "we" without clarifying my role)
- Code-level details: specific files, functions, patterns, and design decisions I made
- Cross-team interactions I led or facilitated
- Documentation I wrote
- Reviews, mentoring, or process improvements I drove

### E. STAR Stories (Situation, Task, Action, Result)
For each significant contribution, generate a detailed STAR narrative:
- **Situation**: Full context — what was happening, why it mattered, what the stakes were
- **Task**: What specifically I was responsible for, and any constraints
- **Action**: Step-by-step what I did (technical details, design choices, trade-offs, collaboration)
- **Result**: Measurable outcomes, impact on the team/product/business, what changed after

### F. Interview Rabbit Holes & Deep-Dives
For each major technical area, anticipate the follow-up questions an interviewer would ask and prepare content for them:
- "How does X work under the hood?"
- "Why did you choose X over Y?"
- "What would you do differently?"
- "How would this scale to 10x/100x?"
- "What are the failure modes?"
- "Walk me through debugging a production issue in this system"
- Edge cases, failure scenarios, and limitations I should be ready to discuss

### G. Mock Interview Grilling Material

For each major project/contribution, generate the following material designed for consumption by an AI mock-interview agent:

#### G.1 Rubric-Annotated Answers
For every STAR story and technical question, include:
- **Gold standard answer**: The ideal response with full context, covering everything a perfect candidate would say
- **Key phrases/concepts**: Specific terms, patterns, or details that MUST appear in a good answer (e.g., "singleflight deduplication", "magic byte verification", "tenant isolation")
- **Common mistakes**: What candidates typically get wrong, forget to mention, or over-simplify
- **Scoring rubric** `[1-5]`:
  - **1 — Poor**: Vague, inaccurate, can't explain basic concepts; likely didn't do the work
  - **2 — Below Average**: Remembers the outcome but can't explain how or why; missing critical details
  - **3 — Acceptable**: Covers the main points but lacks depth; could explain "what" but not "why"
  - **4 — Strong**: Clear, detailed, demonstrates ownership; explains trade-offs and alternatives
  - **5 — Exceptional**: Deep understanding, connects to broader patterns, articulates what they'd do differently, shows genuine ownership
- **3 progressively harder follow-ups** with expected answers for each

#### G.2 Grilling Chains
Sequences of 5-7 questions that drill progressively deeper into a single topic, designed to test whether I truly understand vs. memorized. Each question in the chain must include:
- The question itself
- **Why this question is being asked** (what skill/knowledge it tests)
- **Expected answer** with key points
- **Red flags** in the answer that indicate shallow understanding (e.g., "We just used the library" without knowing what it does)
- **Green flags** that indicate deep understanding (e.g., explaining trade-offs, edge cases, or why alternatives were rejected)

#### G.3 Cross-Project Connection Questions
Questions that require connecting knowledge across multiple projects to demonstrate breadth and pattern recognition. Examples:
- "You used Kafka in RAVE and also built a Kafka producer script — compare the reliability guarantees in each"
- "You've worked in Go, Python, Swift, and Rust — for a new microservice, how do you decide which language to use?"
- "Compare your approach to authentication in WellnessProducer vs. the Nimble Download Gatekeeper"

#### G.4 Gotcha / Ownership Verification Questions
At least one "gotcha" question per major contribution designed to test if I truly did the work vs. just observed it:
- "What was the hardest bug you personally debugged in this system and how did you find it?"
- "If I looked at the git blame for [specific file], what would I see?"
- "Walk me through a production incident you handled for this service"
- "What's one thing about this system that still bothers you?"

#### G.5 Answer Calibration Examples
For the top 10-15 most likely interview questions, include example answers at three quality levels:
- **Weak answer**: What a candidate who observed but didn't contribute would say
- **Acceptable answer**: What a solid contributor would say
- **Strong answer**: What the actual owner/designer would say
These calibration examples help the evaluating agent distinguish between quality levels.

---

## Output Directory Structure

Organize all content into the git repo at `~/gajaretejas/resume` with the following structure:

```
~/gajaretejas/resume/
├── README.md                                    # Index of all generated content with navigation links
├── PROMPT.md                                    # This prompt (keep for reproducibility)
│
├── work-context/
│   ├── overview.md                              # Role summary, team context, high-level contributions
│   ├── rave-platform/
│   │   ├── architecture.md                      # RAVE system architecture, service topology, data flow
│   │   ├── wellness-producer.md                 # WellnessProducer deep-dive (your primary service)
│   │   ├── jwt-auth-system.md                   # JWT authentication system design & implementation
│   │   ├── file-upload-system.md                # File upload & attachment system
│   │   ├── manual-case-workflow.md              # Manual case creation pipeline
│   │   ├── api-versioning.md                    # v1 → v2 → v3 API evolution
│   │   ├── monitoring-observability.md          # Prometheus, Grafana, alerting
│   │   └── cross-launch-tokens.md               # Wellness proxy token support
│   ├── infrastructure/
│   │   ├── aws-terraform.md                     # AWS infrastructure (API Gateway, S3, IAM, EKS)
│   │   ├── kubernetes-helm.md                   # K8s, Helm charts, Istio configuration
│   │   ├── cicd-pipelines.md                    # GitHub Actions, SonarQube, ArgoCD
│   │   ├── secrets-management.md                # SOPS, ESO, Vault migration
│   │   └── logging.md                           # Humio/LogScale configuration
│   ├── nimble-download/
│   │   └── global-trade-compliance.md           # Nimble Download system architecture & contributions
│   ├── tooling-automation/
│   │   ├── sprint-review-generator.md           # Claude Code skill for sprint docs
│   │   ├── utility-scripts.md                   # Python/Shell utilities
│   │   └── search-analysis-tools.md             # Lucene, Atlas Search tools
│   ├── prior-roles/
│   │   ├── hpe-internship.md                    # HPE internship (Summer 2022)
│   │   ├── grouped.md                           # Grouped — Full Stack Developer
│   │   └── yardi.md                             # Yardi — Software Engineer
│   └── side-projects/
│       ├── clipstash.md                         # macOS clipboard manager
│       ├── kairo.md                             # macOS menu bar calendar
│       ├── surmai.md                            # Travel planning PWA
│       ├── dicedb.md                            # Open source contribution
│       └── other-projects.md                    # Any additional projects found
│
├── star-stories/
│   ├── jwt-authentication.md                    # STAR + rubric + follow-ups
│   ├── monitoring-overhaul.md                   # STAR + rubric + follow-ups
│   ├── multi-environment-infra.md               # STAR + rubric + follow-ups
│   ├── global-trade-compliance.md               # STAR + rubric + follow-ups
│   ├── file-upload-security.md                  # STAR + rubric + follow-ups
│   ├── api-versioning-evolution.md              # STAR + rubric + follow-ups
│   ├── manual-case-workflow.md                  # STAR + rubric + follow-ups
│   └── [additional stories as discovered].md
│
├── interview-prep/
│   ├── system-design-questions.md               # System design Q&A with depth
│   ├── behavioral-questions.md                  # Behavioral Q&A (leadership, conflict, etc.)
│   ├── technical-deep-dives.md                  # Code-level technical Q&A
│   ├── rabbit-holes.md                          # Deep follow-up chains by topic
│   ├── resume-bullets.md                        # Polished resume bullet points
│   ├── metrics-and-impact.md                    # Quantifiable achievements
│   │
│   └── mock-interview-guides/
│       ├── interviewer-system-prompt.md          # Self-contained system prompt for an AI interviewer
│       ├── behavioral-grilling.md               # Behavioral Q&A with full rubrics
│       ├── system-design-grilling.md            # System design Q&A with full rubrics
│       ├── technical-deep-dive-grilling.md      # Code-level Q&A with full rubrics
│       ├── project-walkthrough-grilling.md      # "Walk me through X" with full rubrics
│       ├── cross-project-questions.md           # Questions connecting multiple projects
│       ├── scoring-rubrics.md                   # Master scoring criteria & calibration
│       └── answer-calibration.md                # Weak/acceptable/strong example answers
│
├── technology-stack.md                          # Complete tech stack with proficiency levels
├── repository-map.md                            # Map of all repos with commit counts & roles
└── jira-appendix.md                             # Key JIRA stories & PR references
```

---

## Mock Interview Agent: `interviewer-system-prompt.md` Specification

The `interviewer-system-prompt.md` file is the most critical output for the mock interview use case. It must be a **self-contained system prompt** that any AI agent can load with zero prior context and immediately conduct a full, realistic mock interview. It must contain:

### Role Definition
- The agent acts as a **senior engineering interviewer and hiring manager** evaluating a candidate for a Senior Software Engineer role
- It should be demanding but fair — simulating a real FAANG/top-tier company interview bar
- It should never accept vague answers like "we built X" without asking for the candidate's specific role, decisions, and contributions

### Candidate Context (Embedded Summaries)
- Concise summaries of all projects, contributions, and side projects (not the full docs — just enough for the agent to know what to ask about)
- References to the full doc files (e.g., "For full details, see `work-context/rave-platform/jwt-auth-system.md`") so the agent can load more context if needed
- The candidate's tech stack, experience level, and target role

### Interview Flow Instructions
The agent should follow this structure:
1. **Warm-up** (2-3 min): Brief intro, "tell me about yourself", resume walkthrough
2. **Project Deep-Dive** (15-20 min): Pick a major project, ask progressively deeper questions using grilling chains
3. **System Design** (15-20 min): Either a new design problem OR "redesign a system you built" using the candidate's actual work
4. **Behavioral** (10-15 min): STAR-format behavioral questions with follow-ups
5. **Technical Grilling** (10-15 min): Code-level questions, debugging scenarios, trade-off discussions
6. **Wrap-up**: Overall assessment and feedback

### Evaluation Methodology
- After each answer, internally score it [1-5] using the rubrics
- Track cumulative scores across categories (technical depth, communication, ownership, problem-solving)
- Adapt difficulty based on answer quality: if the candidate scores 4-5, escalate to harder follow-ups; if 1-2, probe to understand the gap

### Feedback Protocol
After the interview (or after each section, if the candidate requests), the agent should:
1. **Rate** the answer (1-5 with justification)
2. **Explain gaps** (what was missing from the ideal answer)
3. **Suggest an improved answer** (rewrite the candidate's answer to be stronger)
4. **Re-ask** the question to let the candidate try again with the feedback

### Anti-Memorization Checks
- If an answer sounds rehearsed or too polished, ask a tangential follow-up that requires genuine understanding
- Use "what if" variants: "What if you couldn't use Kafka?", "What if the team was 3x larger?", "What if latency requirements were 10x stricter?"
- Ask about failures, mistakes, and things the candidate would do differently

---

## Important Instructions

1. **Depth over breadth** — For major contributions (JWT auth, file uploads, monitoring, API versioning, infrastructure), go extremely deep. Include code patterns, specific function names, configuration details, and design rationale. Surface-level summaries are not useful for interview prep.

2. **First person, ownership-focused** — Write from my perspective. Distinguish clearly between what I did vs. what the team did. Interviewers care about individual contributions.

3. **Include the "why"** — For every technical decision, explain WHY that approach was chosen, what alternatives existed, and what trade-offs were made. This is what separates a 3/5 interview answer from a 5/5.

4. **Anticipate follow-ups** — After every major point, ask yourself "what would a skeptical interviewer ask next?" and include that content.

5. **Quantify everything possible** — Commits, response times, error rate reductions, number of services, environments, team size, sprint velocity. Numbers make answers concrete.

6. **Cross-reference across projects** — Link related concepts across different projects (e.g., Kafka usage in RAVE vs. utility scripts, Go patterns across services, auth approaches across systems).

7. **Include failure stories** — Not just successes. Production bugs, design mistakes, things I'd do differently. These demonstrate genuine experience and humility.

8. **Interview-ready format** — Every STAR story and technical answer must include inline metadata tags: `[KEY_POINTS]`, `[COMMON_MISTAKES]`, `[FOLLOW_UP]`, `[SCORING: 1-5 rubric]` so that another AI agent can parse and use these to evaluate mock interview answers.

9. **Adversarial depth** — For each major contribution, include at least one "gotcha" question designed to test if the candidate truly did the work vs. just observed it (e.g., "What was the hardest bug you personally debugged in this system and how did you find it?").

10. **Answer calibration** — Include example weak, acceptable, and strong answers for the top 10-15 most likely interview questions, so the evaluating agent has calibration points for scoring.

11. **Explore external sources thoroughly** — Don't just read code. Check Jira for requirements/context, Confluence for design docs, git logs for chronology, PR descriptions for rationale, and comments for discussion context.

12. **Organize for both humans and agents** — The directory structure should be navigable by a human skimming for interview prep AND parseable by an agent loading context for mock interviews. Use consistent heading structures and metadata tags.

---

## Verification Criteria

The generated content is complete when:

- [ ] Every project in the source directories has a corresponding deep-dive document
- [ ] Every significant contribution has a STAR story with rubric annotations
- [ ] The `interviewer-system-prompt.md` is self-contained — a fresh Claude session can load it and immediately start a mock interview with no other context
- [ ] Every technical question has enough ground truth that an agent can distinguish between a memorized answer and genuine understanding
- [ ] Grilling chains exist for at least: JWT auth, file uploads, API versioning, monitoring, infrastructure/K8s, the Nimble Download system, and at least 2 side projects
- [ ] Cross-project connection questions are generated covering at least 5 different project pairs
- [ ] Answer calibration examples exist for the top 10 most likely interview questions
- [ ] All content is committed to the git repo at `~/gajaretejas/resume`
