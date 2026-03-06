# Scoring Rubrics — Master Evaluation Criteria

> Standardized scoring criteria for mock interview evaluation.
> Use these rubrics to score any answer consistently.

---

## Overall Scoring Scale (1-5)

| Score | Label | Description | Signal |
|-------|-------|-------------|--------|
| **1** | Poor | Vague, inaccurate, or unable to explain basic concepts. Likely did not do the work. | Cannot name specific technologies, services, or contributions. |
| **2** | Below Average | Remembers the outcome but cannot explain how or why. Missing critical details. | Knows what was built but not how. Uses "we" exclusively. |
| **3** | Acceptable | Covers the main points but lacks depth. Explains "what" but not "why." | Good overview, misses trade-offs, alternatives, and edge cases. |
| **4** | Strong | Clear, detailed, demonstrates ownership. Explains trade-offs and alternatives considered. | Names specific decisions, code patterns, Jira IDs. Discusses what-if scenarios. |
| **5** | Exceptional | Deep understanding, connects to broader patterns, articulates what they would do differently. Shows genuine ownership and reflection. | Connects across projects, discusses failures honestly, proposes improvements. |

---

## Category-Specific Rubrics

### Technical Depth

| Score | Indicators |
|-------|-----------|
| 1 | Cannot explain basic concepts (what is JWT, what is Kafka) |
| 2 | Knows concepts but not implementation details |
| 3 | Can explain implementation, missing edge cases and trade-offs |
| 4 | Code-level knowledge, explains trade-offs, knows specific libraries/versions |
| 5 | Understands internals (Go runtime, MongoDB wire protocol), connects patterns across projects |

**Key signals for Technical Depth:**
- Can they write code on the spot? (magic byte validation, error handling, concurrency pattern)
- Do they know specific values? (byte signatures: PDF = 0x25504446, PNG = 0x89504E47)
- Do they understand WHY, not just WHAT? (why MongoDB over SQL, why SQLite over Redis)
- Can they discuss limitations of their approach? (polyglot files, ZIP overlap, cache staleness)

### System Design

| Score | Indicators |
|-------|-----------|
| 1 | Cannot scope the problem or propose components |
| 2 | Proposes components but no clear data flow or interactions |
| 3 | Reasonable architecture, missing security/monitoring/failure handling |
| 4 | Complete design with trade-offs, failure modes, and monitoring |
| 5 | Connects to real implementation, proposes scaling strategy, discusses evolution |

**Key signals for System Design:**
- Do they clarify requirements before designing?
- Do they include monitoring and alerting in the design (not as afterthought)?
- Do they discuss failure modes? ("What if X is down?")
- Do they propose alternatives with trade-off analysis?
- Can they connect to real systems they've built?

### Communication

| Score | Indicators |
|-------|-----------|
| 1 | Unclear, rambling, cannot structure an answer |
| 2 | Somewhat clear but disorganized, hard to follow |
| 3 | Clear main points, missing structure or conciseness |
| 4 | Well-structured (STAR for behavioral, top-down for technical), clear and concise |
| 5 | Excellent teaching ability — could explain to junior and senior engineers |

**Key signals for Communication:**
- Do they structure answers (overview → detail → summary)?
- Can they adjust depth based on the question?
- Do they use concrete examples instead of abstract statements?
- Can they explain complex concepts simply?

### Ownership

| Score | Indicators |
|-------|-----------|
| 1 | Cannot specify personal contributions |
| 2 | Uses "we" exclusively, unclear what they personally did |
| 3 | Names personal contributions but limited scope |
| 4 | Clear ownership — "I designed X, I decided Y, I coded Z" |
| 5 | Ownership extends beyond code — includes documentation, monitoring, team enablement, process improvement |

**Key signals for Ownership:**
- "I" vs "we" ratio — do they clarify their personal role?
- Can they name specific files, functions, or Jira stories they worked on?
- Do they describe decisions THEY made, not just tasks assigned?
- Do they take ownership of failures, not just successes?

### Problem-Solving

| Score | Indicators |
|-------|-----------|
| 1 | Cannot describe a debugging or problem-solving approach |
| 2 | Ad hoc approach — "I tried things until it worked" |
| 3 | Structured approach but limited to a single strategy |
| 4 | Multi-signal debugging (metrics, logs, traces), systematic hypothesis testing |
| 5 | Prevention-oriented — not just solving problems but building systems to prevent them |

**Key signals for Problem-Solving:**
- Do they describe a systematic approach (hypothesis → test → verify)?
- Do they use multiple signals (metrics + logs + traces)?
- After fixing, do they add prevention (metrics, tests, documentation)?
- Can they debug across system boundaries (service A → Kafka → service B → MongoDB)?

---

## Behavioral STAR Scoring

| Component | Score 1-2 | Score 3 | Score 4-5 |
|-----------|----------|---------|-----------|
| **Situation** | Vague or missing context | Clear context but low stakes | Specific situation with clear stakes and constraints |
| **Task** | Unclear responsibility | Clear task but routine | Challenging task with personal ownership |
| **Action** | "We" actions, vague steps | Personal actions, basic steps | Detailed personal actions with decision rationale |
| **Result** | No measurable outcome | Outcome stated but not quantified | Quantified impact (metrics, time savings, risk reduction) |
| **Reflection** | None | Brief lesson learned | What they'd do differently, systemic improvements |

---

## Anti-Memorization Scoring

When checking for genuine understanding vs rehearsed answers:

| Signal | Likely Memorized | Likely Genuine |
|--------|-----------------|----------------|
| Follow-up depth | Falls apart after 1-2 follow-ups | Can go 5+ levels deep |
| "What if" variants | Can't adapt the answer | Reasons through the variant |
| Failure questions | Only has successes | Honest about mistakes with lessons |
| Code on demand | Cannot write related code | Can code it from understanding |
| Operational details | Describes design only | Describes debugging, monitoring, incidents |
| Emotion/frustration | Clinical, detached | "This bug annoyed me because..." |

---

## Cumulative Score Card Template

Use this to track scores during a mock interview:

```
Candidate: Tejas Gajare
Date: ____________
Position: Senior Software Engineer

WARM-UP
  Tell me about yourself:    [ ] / 5
  Current work summary:      [ ] / 5

PROJECT DEEP-DIVE
  Architecture overview:     [ ] / 5
  Personal ownership:        [ ] / 5
  Technical depth:           [ ] / 5
  Follow-up handling:        [ ] / 5

SYSTEM DESIGN
  Requirements clarification: [ ] / 5
  Component design:           [ ] / 5
  Trade-off analysis:         [ ] / 5
  Failure handling:           [ ] / 5
  Monitoring:                 [ ] / 5

BEHAVIORAL
  STAR structure:            [ ] / 5
  Ownership signal:          [ ] / 5
  Reflection/learning:       [ ] / 5

TECHNICAL GRILLING
  Code-level knowledge:      [ ] / 5
  Debugging approach:        [ ] / 5
  Cross-system patterns:     [ ] / 5

CATEGORY AVERAGES
  Technical Depth:           [ ] / 5
  System Design:             [ ] / 5
  Communication:             [ ] / 5
  Ownership:                 [ ] / 5
  Problem-Solving:           [ ] / 5

OVERALL:                     [ ] / 5

STRENGTHS:
1. _______________
2. _______________
3. _______________

AREAS TO IMPROVE:
1. _______________
2. _______________
3. _______________

HIRE RECOMMENDATION: [ ] Strong Yes  [ ] Yes  [ ] Lean Yes  [ ] Lean No  [ ] No
```

---

## Score Interpretation

| Average Score | Interpretation | Target Level |
|---------------|---------------|-------------|
| 4.5 - 5.0 | Staff Engineer ready | Senior SDE at FAANG |
| 3.5 - 4.4 | Strong Senior Engineer | Senior SDE at most companies |
| 2.5 - 3.4 | Solid Mid-Level | SDE II at most companies |
| 1.5 - 2.4 | Needs development | Not ready for Senior |
| 1.0 - 1.4 | Significant gaps | Re-evaluate target role |

**Target for Tejas:** Consistently score 3.5+ across all categories to be competitive for Senior SDE. Score 4+ in Technical Depth and Ownership (strongest areas based on actual contributions).
