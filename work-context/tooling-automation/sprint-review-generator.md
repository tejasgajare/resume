# Sprint Review Generator — Copilot CLI Skill

> **Type**: Developer Tooling / Automation
> **Stack**: GitHub Copilot CLI custom skill, Jira API, Confluence API, Python
> **Purpose**: Auto-generate sprint review Confluence pages from Jira data

---

## Overview

I built a custom GitHub Copilot CLI skill called `gen-sprint-review-doc` that automates the creation of sprint review/wellness Confluence pages. Instead of manually gathering sprint data from Jira, formatting it, and creating a Confluence page, I say "create a sprint review page" and the skill does everything end-to-end.

This is my most impactful developer productivity tool — it saves 30-45 minutes per sprint for the team lead (often me) and produces more consistent, data-rich documentation than manual creation.

---

## Architecture

### Flow

```
User: "Create sprint review page for Sprint 42"
            │
            ▼
┌───────────────────────────┐
│  Copilot CLI Skill Engine │
├───────────────────────────┤
│  1. Parse sprint identifier│
│  2. Fetch Jira sprint data │
│  3. Fetch all stories/tasks│
│  4. Generate burndown chart│
│  5. Format Confluence page │
│  6. Create page via API    │
└───────────────────────────┘
            │
            ▼
    Confluence Page Created
    (with Jira Smart Links,
     @mentions, status lozenges)
```

### Components

#### Jira Data Fetching
- Query sprint by name/ID from Jira API
- Fetch all issues in the sprint (Stories, Tasks, Bugs, Sub-tasks)
- Extract: story points, status, assignee, labels, sprint velocity
- Handle pagination for large sprints (50+ issues)

#### Burndown Chart Generation
- Calculate ideal burndown line from sprint capacity
- Plot actual burndown from daily status snapshots
- Generate chart as an image (matplotlib or chart.js)
- Embed in Confluence page

#### Confluence Page Formatting
- Rich ADF (Atlassian Document Format) content
- Jira Smart Links: clicking an issue key opens the Jira issue
- @mentions: team members tagged in their story sections
- Status lozenges: visual status indicators (Done ✅, In Progress 🔵, etc.)
- Structured sections: Summary, Burndown, Story Details, Risks, Action Items

#### Page Creation
- Create page in the correct Confluence space
- Set parent page (under Sprint Reviews folder)
- Add labels for discoverability
- Set page permissions if needed

---

## Skill Definition

The skill is defined in the Copilot CLI skill configuration:

```yaml
name: gen-sprint-review-doc
description: >
  Creates a sprint review/wellness Confluence page for the Rave team.
  Fetches stories from Jira, generates a burndown chart, and creates
  a formatted Confluence page with rich content.
triggers:
  - "create sprint review"
  - "sprint review page"
  - "sprint wellness"
  - "end of sprint doc"
```

### Rich Content Generation

The generated Confluence page includes:

1. **Sprint Summary Table**
   - Sprint dates, capacity, velocity
   - Committed vs. completed story points
   - Carry-over items

2. **Burndown Chart**
   - Visual progress indicator
   - Ideal vs. actual lines

3. **Story-by-Story Breakdown**
   - Each story with Jira Smart Link
   - Assignee with @mention
   - Status lozenge (visual badge)
   - Story points, labels, blockers

4. **Sprint Health Metrics**
   - Velocity trend (last 5 sprints)
   - Scope change percentage
   - Bug-to-feature ratio

5. **Risks & Action Items**
   - Carry-over items with context
   - Technical debt flagged
   - Action items from retro

### ADF (Atlassian Document Format)

Confluence v2 API requires ADF, not wiki markup. I generate structured JSON:

```json
{
  "type": "doc",
  "content": [
    {
      "type": "heading",
      "attrs": { "level": 1 },
      "content": [{ "type": "text", "text": "Sprint 42 Review" }]
    },
    {
      "type": "table",
      "content": [...]
    }
  ]
}
```

Jira Smart Links use the `inlineCard` node type with the Jira issue URL, and status lozenges use the `status` node with appropriate color attributes.

[KEY_POINTS]
- Automates 30-45 minutes of manual sprint documentation
- Generates rich Confluence pages with Jira Smart Links and @mentions
- ADF format for Confluence v2 API compatibility
- Reusable skill pattern for any Copilot CLI automation
[/KEY_POINTS]

---

## Implementation Details

### Sprint Data Aggregation

```python
def aggregate_sprint_data(sprint_issues):
    return {
        "total_points": sum(i.story_points for i in sprint_issues if i.story_points),
        "completed_points": sum(i.story_points for i in sprint_issues 
                               if i.status == "Done" and i.story_points),
        "stories": [i for i in sprint_issues if i.issue_type == "Story"],
        "bugs": [i for i in sprint_issues if i.issue_type == "Bug"],
        "carry_over": [i for i in sprint_issues if i.status != "Done"],
        "scope_changes": [i for i in sprint_issues if i.added_mid_sprint],
    }
```

### Error Handling

- Sprint not found → prompt user for correct sprint name
- Jira API rate limit → exponential backoff with retry
- Confluence space not accessible → clear error message with permission instructions
- Partial data → generate page with available data, flag missing sections

[COMMON_MISTAKES]
- Not handling sprints with no story points: Some teams use T-shirt sizes or don't estimate. The skill gracefully degrades to issue counts.
- ADF validation errors: Confluence rejects malformed ADF silently. I validate the JSON structure before submission.
- Smart Link format changes: Jira Smart Links require exact URL format — different Jira Cloud instances use different URL patterns.
[/COMMON_MISTAKES]

---

## Interview Talking Points

**"Tell me about a developer productivity tool you built."**
I built a Copilot CLI skill that auto-generates sprint review Confluence pages from Jira data. It fetches all sprint issues, generates a burndown chart, and creates a rich Confluence page with Jira Smart Links, @mentions, and status lozenges. It saves 30-45 minutes per sprint and produces more consistent documentation.

**"How do you approach automation?"**
I look for repetitive tasks that follow a pattern. Sprint reviews were perfect: same structure every sprint, same data sources (Jira), same destination (Confluence). The skill encapsulates the pattern and handles edge cases (missing story points, scope changes, carry-over items).

[FOLLOW_UP]
- How do you handle different sprint formats across teams?
- What's the testing strategy for a tool that creates Confluence pages?
- How would you extend this to generate retrospective documents?
[/FOLLOW_UP]

---

## Cross-References

- See [Nimble Download → Global Trade Compliance](../nimble-download/global-trade-compliance.md) for the team context where this tool is used
- See [Tooling → Utility Scripts](utility-scripts.md) for other automation tools I've built
- See [Side Projects → Surmai](../side-projects/surmai.md) for another project integrating with APIs

---

*Last updated: Auto-generated from work-context-refresh skill*
