# Utility Scripts & Playground Tools

> **Type**: Developer Tooling / Automation
> **Stack**: Python, Shell (Bash/Zsh), Go
> **Purpose**: Productivity scripts, playground environments, workflow automation

---

## Overview

I maintain a collection of utility scripts and playground tools that automate repetitive tasks in my development workflow. These range from simple shell one-liners to multi-file Python tools. The common theme: if I do something more than three times, I script it.

---

## Python Utilities

### Reference Number Script (Original)

Before converting to Go (GLCP-272882), the reference number management script was in Python. It handled:
- Generating GTCaaS reference numbers in the required format
- Validating reference number checksums
- Bulk-querying reference numbers against GTCaaS API
- Generating compliance reports from reference number data

```python
def generate_reference_number(prefix: str, entity_id: str) -> str:
    """Generate a GTCaaS-formatted reference number."""
    timestamp = datetime.utcnow().strftime("%Y%m%d%H%M%S")
    payload = f"{prefix}-{entity_id}-{timestamp}"
    checksum = hashlib.sha256(payload.encode()).hexdigest()[:8]
    return f"{payload}-{checksum}"
```

The Go conversion (see [Nimble Download → Global Trade Compliance](../nimble-download/global-trade-compliance.md)) maintained the same logic with added type safety.

### Jira Automation Scripts

I built Python scripts for Jira workflow automation:

- **Sprint assignment**: Batch-transition all stories in a sprint to "Assigned" status (now a Copilot CLI skill: `sprint-assign`)
- **Story point validation**: Check that all stories have points before sprint start
- **LaunchDarkly field checker**: Verify the LD field is set on stories that need it
- **Blocked issue reporter**: Daily summary of blocked issues with duration

### Data Processing Scripts

- **CSV to Confluence table**: Convert CSV exports to Confluence wiki markup or ADF
- **Log parser**: Extract structured data from Go microservice logs (JSON-line format)
- **MongoDB query builder**: Generate MongoDB aggregation pipelines from simple query descriptions

---

## Shell Utilities

### Git Workflow

```bash
# Quick branch cleanup — delete merged branches
alias git-clean='git branch --merged main | grep -v "main" | xargs git branch -d'

# Interactive rebase shortcut
alias gri='git rebase -i HEAD~'

# Show recent branches sorted by last commit
alias git-recent='git for-each-ref --sort=-committerdate refs/heads/ --format="%(committerdate:relative) %(refname:short)"'

# Diff stat for current branch vs main
alias git-stat='git diff --stat main...HEAD'
```

### Docker / Kubernetes

```bash
# Quick pod logs with follow
klog() {
    kubectl logs -f "$(kubectl get pods | grep "$1" | head -1 | awk '{print $1}')"
}

# Port-forward shortcut
kpf() {
    kubectl port-forward "svc/$1" "$2:$2"
}

# Docker cleanup
alias docker-clean='docker system prune -af --volumes'
```

### Go Development

```bash
# Run tests with coverage and open in browser
gocover() {
    go test -coverprofile=coverage.out ./... && go tool cover -html=coverage.out
}

# Lint + test in one command
gocheck() {
    golangci-lint run ./... && go test -race -v ./...
}

# Build all binaries in cmd/
gobuild() {
    for dir in cmd/*/; do
        echo "Building $(basename $dir)..."
        go build -o "bin/$(basename $dir)" "./$dir"
    done
}
```

---

## Playground Environments

### Go Playground

I maintain a local Go playground for quick experimentation:

```
playground/
├── go/
│   ├── concurrency/     # goroutine/channel experiments
│   ├── generics/        # Go 1.18+ generics patterns
│   ├── interfaces/      # interface design experiments
│   ├── benchmarks/      # micro-benchmarks
│   └── scratch/         # quick one-off experiments
```

Each experiment is a self-contained `main.go` with its own `go.mod`. I use this when learning new Go patterns or prototyping solutions before integrating into production code.

### Swift Playground

Similar structure for Swift experiments:

```
playground/
├── swift/
│   ├── concurrency/     # async/await, actors
│   ├── swiftui/         # view patterns, animations
│   ├── grdb/            # database patterns
│   └── appkit/          # NSPanel, pasteboard experiments
```

---

## Work Context Refresh Skill

I built the `work-context-refresh` Copilot CLI skill to automate maintaining my resume/interview prep documentation:

- Scans local repos for recent work
- Queries Jira/Confluence/GitHub for new content
- Updates structured markdown docs (work-context, STAR stories)
- Tracks state between invocations (only processes new content)
- Auto-commits and pushes to the resume repo

This is the skill that generates and maintains the document you're reading now.

[KEY_POINTS]
- Automate anything done more than 3 times
- Python for data processing and API scripts, Shell for workflow shortcuts
- Playground environments for rapid experimentation
- Copilot CLI skills for complex multi-step automation
[/KEY_POINTS]

---

## Sprint Assignment Skill

The `sprint-assign` skill automates sprint kickoff:

1. Fetch all issues in the current sprint
2. Validate story points on Stories (warn if missing)
3. Check LaunchDarkly field where required
4. Skip Blocked issues (leave in current status)
5. Transition all eligible issues to "Assigned" status

This replaced a manual 15-minute process of clicking through each Jira issue at sprint start.

[COMMON_MISTAKES]
- Not handling Jira transition guards: Some transitions require specific fields to be set. The script validates prerequisites before attempting transitions.
- Shell scripts that assume macOS: I use `#!/usr/bin/env bash` and check for tool availability for portability.
- Playground experiments that leak into production: I keep playgrounds in a separate repo, never in production codebases.
[/COMMON_MISTAKES]

---

## Interview Talking Points

**"How do you improve developer productivity?"**
I automate repetitive tasks aggressively. My sprint review generator saves 30-45 minutes per sprint. My sprint assignment skill eliminates 15 minutes of manual Jira clicking. My shell aliases and Go/Python utility scripts shave seconds off hundreds of daily operations.

**"Tell me about a script you're proud of."**
The reference number script — originally Python, later converted to Go. It generates, validates, and bulk-queries GTCaaS compliance reference numbers. The Go conversion added type safety for the strict reference number format and eliminated a Python runtime dependency on deployment targets.

[FOLLOW_UP]
- How do you decide when to automate vs. do manually?
- What's your testing strategy for utility scripts?
- How do you share scripts across a team?
[/FOLLOW_UP]

---

## Cross-References

- See [Tooling → Sprint Review Generator](sprint-review-generator.md) for the sprint review skill deep-dive
- See [Tooling → Search Analysis Tools](search-analysis-tools.md) for specialized search tooling
- See [Nimble Download → Global Trade Compliance](../nimble-download/global-trade-compliance.md) for the reference number script context
- See [Side Projects → ClipStash](../side-projects/clipstash.md) for Swift playground experiments applied to a real project

---

*Last updated: Auto-generated from work-context-refresh skill*
