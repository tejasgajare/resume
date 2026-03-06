---
name: work-context-refresh
description: >
  Incrementally builds and refreshes a comprehensive work portfolio, interview
  knowledge base, and knowledge graph. Mines brainstorming sessions from
  ~/.claude and ~/.copilot, scans local repos, queries Jira/Confluence/GitHub
  for new content, updates a SQLite knowledge graph, and generates structured
  markdown docs (work-context, STAR stories, interview prep). Designed for
  periodic runs — tracks state between invocations and only processes new content.
  Auto-commits and pushes to the resume repo.
---

# Work Context Refresh Skill

This skill incrementally discovers, processes, and documents all professional
and personal work into a structured knowledge base. It is designed to be run
periodically — each run discovers what's new since the last run and only
generates/updates content for changed items.

**Repository**: `~/gajaretejas/resume`
**Reference prompt**: `~/gajaretejas/resume/PROMPT.md` (the original generation spec — read it for the full output format)

When this skill is invoked, follow these 7 phases exactly.

---

## Phase 0: Environment Check

Verify required tools and paths exist:

```bash
cd ~/gajaretejas/resume && echo "REPO_OK"
test -f ~/gajaretejas/resume/PROMPT.md && echo "PROMPT_OK"
python3 -c "import sqlite3; print('SQLITE_OK')"
test -d ~/.claude && echo "CLAUDE_DIR_OK" || echo "CLAUDE_DIR_MISSING"
test -d ~/.copilot && echo "COPILOT_DIR_OK" || echo "COPILOT_DIR_MISSING"
```

If the resume repo doesn't exist, stop and tell the user.

---

## Phase 1: State Loading

### 1a. Load Manifest

Read the manifest file that tracks what has been processed:

```bash
cat ~/gajaretejas/resume/.manifest.json 2>/dev/null || echo "NO_MANIFEST"
```

If `NO_MANIFEST`, this is the first run. Create the initial manifest (see Phase 6 for the schema).
Save the manifest data in memory — you'll compare against it throughout.

Key fields to remember from the manifest:
- `last_run` — ISO timestamp of the last successful run
- `processed_sessions.claude` — list of session IDs already mined
- `processed_sessions.copilot` — list of session IDs already mined
- `processed_jira_keys` — list of Jira issue keys already processed
- `processed_repo_commits` — map of repo path → last processed commit SHA
- `processed_projects` — list of project directory paths already scanned

### 1b. Open Knowledge Database

Open (or create) the SQLite knowledge graph:

```bash
python3 -c "
import sqlite3, os
db_path = os.path.expanduser('~/gajaretejas/resume/knowledge.db')
conn = sqlite3.connect(db_path)
c = conn.cursor()

# Check if tables exist
c.execute(\"SELECT name FROM sqlite_master WHERE type='table'\")
tables = [r[0] for r in c.fetchall()]
print('EXISTING_TABLES:', tables)

if 'entities' not in tables:
    print('SCHEMA_MISSING — will be created in Phase 4')
else:
    c.execute('SELECT COUNT(*) FROM entities')
    print('ENTITY_COUNT:', c.fetchone()[0])
    c.execute('SELECT COUNT(*) FROM relationships')
    print('RELATIONSHIP_COUNT:', c.fetchone()[0])
    c.execute('SELECT COUNT(*) FROM memories')
    print('MEMORY_COUNT:', c.fetchone()[0])

conn.close()
"
```

If `SCHEMA_MISSING`, the schema will be created in Phase 4 (first run).

---

## Phase 2: Session Mining

This phase extracts insights from brainstorming sessions with Claude Code and
Copilot CLI. Focus on extracting: technical decisions, design rationale,
problems solved, architecture discussions, debugging insights, and patterns learned.

### 2a. Mine Claude Sessions

Scan `~/.claude/` for project knowledge and conversation history:

```bash
# List all Claude project directories with their memory and session files
find ~/.claude/projects/ -maxdepth 2 \( -name "MEMORY.md" -o -name "*.jsonl" \) -type f 2>/dev/null | sort
```

For each project directory found:

**Step 1: Read MEMORY.md files** (highest signal — these are curated project knowledge):
```bash
# Example: read a project's memory
cat ~/.claude/projects/{project-path}/memory/MEMORY.md 2>/dev/null
cat ~/.claude/projects/{project-path}/MEMORY.md 2>/dev/null
```

Extract from MEMORY.md:
- Project overview and architecture
- Key patterns, conventions, and fixes
- Technology stack details
- Known gotchas and design decisions

**Step 2: Mine .jsonl conversation files** for sessions not in `processed_sessions.claude`:
```bash
# List sessions with metadata
python3 -c "
import json, os, glob

for jsonl_path in glob.glob(os.path.expanduser('~/.claude/projects/*/*.jsonl')):
    session_id = os.path.basename(jsonl_path).replace('.jsonl', '')
    line_count = sum(1 for _ in open(jsonl_path))
    # Get project name from parent dir
    project = os.path.basename(os.path.dirname(jsonl_path))
    print(f'{session_id} | {project} | {line_count} lines | {jsonl_path}')
"
```

For each **unprocessed** session .jsonl file, extract user messages and assistant responses that contain:
- Design decisions ("I chose X because...")
- Architecture discussions
- Problem-solving chains (debugging sessions)
- Trade-off analysis
- Code pattern explanations

```bash
# Extract key messages from a session (user questions + short assistant summaries)
python3 -c "
import json, sys

with open(sys.argv[1]) as f:
    for line in f:
        try:
            entry = json.loads(line)
            if entry.get('type') == 'user':
                msg = entry.get('message', {})
                content = msg.get('content', '')
                if isinstance(content, str) and len(content) > 20:
                    # Print first 500 chars of user message for context
                    print(f'USER [{entry.get(\"cwd\",\"\")}]: {content[:500]}')
                    print('---')
        except:
            pass
" ~/.claude/projects/{project-path}/{session-id}.jsonl
```

**Also scan** `~/.claude/history.jsonl` for the session index:
```bash
python3 -c "
import json
with open(os.path.expanduser('~/.claude/history.jsonl')) as f:
    for line in f:
        entry = json.loads(line)
        print(f'{entry.get(\"sessionId\",\"?\")} | {entry.get(\"project\",\"?\")} | {entry.get(\"display\",\"\")[:200]}')
" 2>/dev/null | tail -30
```

### 2b. Mine Copilot CLI Sessions

Scan `~/.copilot/` for session data:

```bash
# List Copilot sessions with metadata
for dir in ~/.copilot/session-state/*/; do
    if [ -f "$dir/workspace.yaml" ]; then
        echo "=== $(basename $dir) ==="
        cat "$dir/workspace.yaml" 2>/dev/null
        test -f "$dir/plan.md" && echo "HAS_PLAN"
    fi
done 2>/dev/null | head -100
```

For each **unprocessed** session:

**Step 1: Read plan.md files** (high signal — structured plans):
```bash
cat ~/.copilot/session-state/{session-id}/plan.md 2>/dev/null
```

**Step 2: Mine .jsonl event logs**:
```bash
python3 -c "
import json, sys, os

jsonl_path = sys.argv[1]
with open(jsonl_path) as f:
    for line in f:
        try:
            entry = json.loads(line)
            etype = entry.get('type', '')
            if etype == 'user.message':
                print(f'USER: {entry.get(\"data\",{}).get(\"content\",\"\")[:500]}')
                print('---')
            elif etype == 'assistant.message':
                print(f'ASSISTANT: {entry.get(\"data\",{}).get(\"content\",\"\")[:300]}')
                print('---')
        except:
            pass
" ~/.copilot/session-state/{session-id}.jsonl 2>/dev/null
```

### 2c. Compile Session Insights

After mining all sessions, compile a list of **insights** — each insight is a tuple of:
- **source**: session ID and agent (claude/copilot)
- **category**: architecture | debugging | design-decision | pattern | trade-off | brainstorming
- **project**: which project it relates to
- **content**: the extracted insight (1-3 sentences)
- **raw_context**: relevant excerpt from the conversation

Store these temporarily — they'll be written to the knowledge graph in Phase 4.

---

## Phase 3: Source Discovery

### 3a. Local Repository Scan

Check for new commits in known repositories since the last run:

```bash
# List all git repos to scan
for dir in ~/glcp/* ~/playground/* ~/kairo* ~/projects/* ~/surmai ~/dice; do
    if [ -d "$dir/.git" ]; then
        echo "$dir"
    fi
done 2>/dev/null
```

For each repo, check for new commits:
```bash
# Get commits since last run (or all if first run)
cd {repo_path} && git --no-pager log --oneline --since="{last_run}" --author="gajaret\|Tejas" -- 2>/dev/null | head -50
```

If there are new commits, the repo needs (re)processing. Record the latest commit SHA.

Also check the repo's structure for context:
```bash
cd {repo_path} && find . -maxdepth 2 -name "*.go" -o -name "*.py" -o -name "*.swift" -o -name "*.rs" -o -name "*.java" -o -name "*.ts" -o -name "*.js" | head -30
```

### 3b. Jira Discovery

Query Jira for recently updated issues assigned to the user:

Use the `searchJiraIssuesUsingJql` MCP tool with:
- **cloudId**: `b26ad273-0621-4dd6-8915-78cfbe11048e`
- **jql**: `assignee = currentUser() AND updated >= "{last_run_date}" ORDER BY updated DESC`
  - If first run, use: `assignee = currentUser() ORDER BY updated DESC`
- **fields**: `["summary", "status", "issuetype", "priority", "labels", "description", "created", "updated"]`
- **maxResults**: `50`

Paginate if needed. Filter out any issue keys already in `processed_jira_keys`.

For each **new** issue, also fetch full details if needed:
- Use `getJiraIssue` MCP tool for issues that look significant (Stories, important Bugs)
- Check for linked issues, comments, and remote links

### 3c. Confluence Discovery

Query Confluence for recently edited pages by the user:

Use the `searchConfluenceUsingCql` MCP tool with:
- **cloudId**: `b26ad273-0621-4dd6-8915-78cfbe11048e`
- **cql**: `contributor = currentUser() AND lastModified >= "{last_run_date}" ORDER BY lastModified DESC`
  - If first run: `contributor = currentUser() ORDER BY lastModified DESC`
- **limit**: `25`

For relevant pages (design docs, API specs, architecture decisions), read the full content:
- Use `getConfluencePage` MCP tool with `contentFormat: "markdown"`

### 3d. GitHub Discovery

Query GitHub for repository activity:

Use `github-mcp-server-search_repositories` with `query: "user:gajaretejas"` to get all repos.

For each repo with recent activity, use `github-mcp-server-list_commits` to find new commits.

Also check for any new repos not yet in the manifest.

### 3e. Compile Discovery Results

Create a summary of all new content found:
- New/changed repos with commit counts
- New Jira issues with summaries
- New Confluence pages
- New GitHub repos or activity

Tell the user what was discovered and confirm before proceeding to generation.

---

## Phase 4: Knowledge Graph Update

### 4a. Create/Verify Schema

```bash
python3 << 'PYEOF'
import sqlite3, os

db_path = os.path.expanduser('~/gajaretejas/resume/knowledge.db')
conn = sqlite3.connect(db_path)
c = conn.cursor()

c.executescript("""
CREATE TABLE IF NOT EXISTS entities (
    id TEXT PRIMARY KEY,
    type TEXT NOT NULL CHECK(type IN ('project', 'technology', 'contribution', 'story', 'concept', 'role', 'team', 'person')),
    name TEXT NOT NULL,
    description TEXT,
    metadata_json TEXT DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS relationships (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    source_id TEXT NOT NULL REFERENCES entities(id),
    target_id TEXT NOT NULL REFERENCES entities(id),
    rel_type TEXT NOT NULL CHECK(rel_type IN (
        'uses', 'built_with', 'contributed_to', 'related_to',
        'depends_on', 'part_of', 'led', 'designed', 'debugged',
        'owns', 'works_on', 'member_of', 'evolved_to'
    )),
    weight REAL DEFAULT 1.0,
    metadata_json TEXT DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(source_id, target_id, rel_type)
);

CREATE TABLE IF NOT EXISTS memories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    source TEXT NOT NULL,
    source_id TEXT,
    category TEXT NOT NULL CHECK(category IN (
        'architecture', 'debugging', 'design_decision', 'pattern',
        'trade_off', 'brainstorming', 'lesson_learned', 'gotcha',
        'interview_insight', 'metric', 'general'
    )),
    project_id TEXT REFERENCES entities(id),
    content TEXT NOT NULL,
    raw_context TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS documents (
    id TEXT PRIMARY KEY,
    file_path TEXT NOT NULL UNIQUE,
    entity_id TEXT REFERENCES entities(id),
    doc_type TEXT CHECK(doc_type IN (
        'overview', 'architecture', 'deep_dive', 'star_story',
        'interview_qa', 'mock_guide', 'rubric', 'index'
    )),
    content_hash TEXT,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS processing_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    source_type TEXT NOT NULL CHECK(source_type IN (
        'claude_session', 'copilot_session', 'git_repo',
        'jira_issue', 'confluence_page', 'github_repo', 'claude_memory'
    )),
    source_id TEXT NOT NULL,
    source_path TEXT,
    processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    notes TEXT,
    UNIQUE(source_type, source_id)
);

-- Indexes for efficient querying
CREATE INDEX IF NOT EXISTS idx_entities_type ON entities(type);
CREATE INDEX IF NOT EXISTS idx_relationships_source ON relationships(source_id);
CREATE INDEX IF NOT EXISTS idx_relationships_target ON relationships(target_id);
CREATE INDEX IF NOT EXISTS idx_relationships_type ON relationships(rel_type);
CREATE INDEX IF NOT EXISTS idx_memories_category ON memories(category);
CREATE INDEX IF NOT EXISTS idx_memories_project ON memories(project_id);
CREATE INDEX IF NOT EXISTS idx_documents_entity ON documents(entity_id);
CREATE INDEX IF NOT EXISTS idx_processing_log_source ON processing_log(source_type, source_id);
""")

conn.commit()
print("SCHEMA_OK")

# Print stats
for table in ['entities', 'relationships', 'memories', 'documents', 'processing_log']:
    c.execute(f'SELECT COUNT(*) FROM {table}')
    print(f'{table}: {c.fetchone()[0]} rows')

conn.close()
PYEOF
```

### 4b. Insert/Update Entities

For each discovered project, technology, and contribution, insert or update entities:

```bash
python3 << 'PYEOF'
import sqlite3, json, os

db_path = os.path.expanduser('~/gajaretejas/resume/knowledge.db')
conn = sqlite3.connect(db_path)
c = conn.cursor()

def upsert_entity(eid, etype, name, description, metadata=None):
    metadata_json = json.dumps(metadata or {})
    c.execute("""
        INSERT INTO entities (id, type, name, description, metadata_json)
        VALUES (?, ?, ?, ?, ?)
        ON CONFLICT(id) DO UPDATE SET
            description = excluded.description,
            metadata_json = excluded.metadata_json,
            updated_at = CURRENT_TIMESTAMP
    """, (eid, etype, name, description, metadata_json))

def add_relationship(source, target, rel_type, weight=1.0, metadata=None):
    metadata_json = json.dumps(metadata or {})
    c.execute("""
        INSERT OR IGNORE INTO relationships (source_id, target_id, rel_type, weight, metadata_json)
        VALUES (?, ?, ?, ?, ?)
    """, (source, target, rel_type, weight, metadata_json))

def add_memory(source, source_id, category, content, project_id=None, raw_context=None):
    c.execute("""
        INSERT INTO memories (source, source_id, category, project_id, content, raw_context)
        VALUES (?, ?, ?, ?, ?, ?)
    """, (source, source_id, category, project_id, content, raw_context))

# === INSERT YOUR DISCOVERED ENTITIES HERE ===
# Example patterns:

# Projects
# upsert_entity('proj-rave-cloud', 'project', 'RAVE Cloud Platform', 'HPE GreenLake RAVE platform...', {'repo': '~/glcp/rave-cloud'})
# upsert_entity('proj-kairo', 'project', 'Kairo', 'macOS menu bar calendar', {'repo': '~/kairo', 'lang': 'Swift'})

# Technologies
# upsert_entity('tech-go', 'technology', 'Go', 'Primary backend language')
# upsert_entity('tech-istio', 'technology', 'Istio', 'Service mesh for K8s')

# Contributions
# upsert_entity('contrib-case-creation', 'contribution', 'Case Creation System', 'Multi-tenant case creation...')

# Relationships
# add_relationship('proj-rave-cloud', 'tech-go', 'built_with')
# add_relationship('contrib-case-creation', 'proj-rave-cloud', 'part_of')

# Memories from sessions
# add_memory('claude_session', 'session-abc', 'design_decision', 'Chose singleflight for dedup...', 'proj-rave-cloud')

conn.commit()
conn.close()
print("GRAPH_UPDATED")
PYEOF
```

**Important**: Use the actual discovered data from Phases 2 and 3. The examples above are templates — replace them with real entities, relationships, and memories extracted from:
- Session mining (MEMORY.md files, conversation .jsonl files)
- Jira issues (each significant story/bug → contribution entity)
- Repo structure (each repo → project entity, each language → technology entity)
- Confluence pages (design decisions → memory entries)

### 4c. Log Processing

After inserting, log what was processed:
```bash
python3 -c "
import sqlite3, os
conn = sqlite3.connect(os.path.expanduser('~/gajaretejas/resume/knowledge.db'))
c = conn.cursor()
# Log each processed source — examples:
# c.execute('INSERT OR REPLACE INTO processing_log (source_type, source_id, source_path, notes) VALUES (?, ?, ?, ?)',
#     ('claude_session', '{session-id}', '{path}', 'Extracted 5 memories'))
conn.commit()
conn.close()
"
```

---

## Phase 5: Document Generation

Read `~/gajaretejas/resume/PROMPT.md` for the full specification of what to generate for
each project. The directory structure and content format are defined there in detail.

### 5a. Query Knowledge Graph for Generation Targets

```bash
python3 << 'PYEOF'
import sqlite3, os, json

conn = sqlite3.connect(os.path.expanduser('~/gajaretejas/resume/knowledge.db'))
c = conn.cursor()

# Find entities that have been updated but whose documents are stale or missing
c.execute("""
    SELECT e.id, e.type, e.name, e.description, d.file_path, d.content_hash
    FROM entities e
    LEFT JOIN documents d ON d.entity_id = e.id
    WHERE e.type IN ('project', 'contribution')
    ORDER BY e.updated_at DESC
""")

for row in c.fetchall():
    eid, etype, name, desc, fpath, chash = row
    status = 'EXISTS' if fpath else 'NEEDS_GENERATION'
    print(f'{status} | {eid} | {etype} | {name} | {fpath or "NO_DOC"}')

# Also get all memories for context
c.execute("SELECT category, content, project_id FROM memories ORDER BY created_at DESC LIMIT 50")
memories = c.fetchall()
print(f"\n{len(memories)} memories available for enriching documents")

conn.close()
PYEOF
```

### 5b. Create Directory Structure

```bash
cd ~/gajaretejas/resume
mkdir -p work-context/rave-platform
mkdir -p work-context/infrastructure
mkdir -p work-context/nimble-download
mkdir -p work-context/tooling-automation
mkdir -p work-context/prior-roles
mkdir -p work-context/side-projects
mkdir -p star-stories
mkdir -p interview-prep/mock-interview-guides
```

### 5c. Generate/Update Documents

For each entity that needs a document (from 5a), generate the content following
the PROMPT.md specification. Use the knowledge graph to:

1. **Query related entities**: Find technologies, contributions, and concepts related to the project
2. **Query memories**: Find relevant memories for context, decisions, and insights
3. **Query relationships**: Understand how projects connect to each other

**For each project**, generate per PROMPT.md sections A-G:
- A. Project Overview & Architecture
- B. Requirements & Problem Context
- C. Challenges & Solution Exploration
- D. My Specific Contributions
- E. STAR Stories (with rubric annotations)
- F. Interview Rabbit Holes & Deep-Dives
- G. Mock Interview Grilling Material

**Use this pattern for generating each document**:

```bash
python3 << 'PYEOF'
import sqlite3, os, json

conn = sqlite3.connect(os.path.expanduser('~/gajaretejas/resume/knowledge.db'))
c = conn.cursor()

entity_id = '{entity-id}'  # Replace with actual

# Get entity details
c.execute("SELECT * FROM entities WHERE id = ?", (entity_id,))
entity = c.fetchone()

# Get related technologies
c.execute("""
    SELECT e.name, r.rel_type FROM relationships r
    JOIN entities e ON e.id = r.target_id
    WHERE r.source_id = ? AND e.type = 'technology'
""", (entity_id,))
technologies = c.fetchall()

# Get memories about this project
c.execute("""
    SELECT category, content, raw_context FROM memories
    WHERE project_id = ?
    ORDER BY category, created_at
""", (entity_id,))
memories = c.fetchall()

# Get related contributions
c.execute("""
    SELECT e.name, e.description FROM relationships r
    JOIN entities e ON e.id = r.source_id
    WHERE r.target_id = ? AND r.rel_type = 'part_of'
""", (entity_id,))
contributions = c.fetchall()

print(json.dumps({
    'entity': entity,
    'technologies': technologies,
    'memories': [{'cat': m[0], 'content': m[1]} for m in memories],
    'contributions': [{'name': c[0], 'desc': c[1]} for c in contributions]
}, indent=2, default=str))

conn.close()
PYEOF
```

Use the query results to enrich the generated document content. Write each document
using the `create` or `edit` file tools.

**After creating each document**, register it in the knowledge graph:
```bash
python3 -c "
import sqlite3, hashlib, os
conn = sqlite3.connect(os.path.expanduser('~/gajaretejas/resume/knowledge.db'))
c = conn.cursor()
# Read file content for hash
with open(os.path.expanduser('~/gajaretejas/resume/{file-path}')) as f:
    content_hash = hashlib.sha256(f.read().encode()).hexdigest()
c.execute('''
    INSERT INTO documents (id, file_path, entity_id, doc_type, content_hash)
    VALUES (?, ?, ?, ?, ?)
    ON CONFLICT(id) DO UPDATE SET content_hash = excluded.content_hash, last_updated = CURRENT_TIMESTAMP
''', ('{doc-id}', '{file-path}', '{entity-id}', '{doc-type}', content_hash))
conn.commit()
conn.close()
"
```

### 5d. Generate Cross-Cutting Documents

After individual project docs, generate:
- `interview-prep/mock-interview-guides/interviewer-system-prompt.md` — Self-contained system prompt (see PROMPT.md for full spec)
- `interview-prep/mock-interview-guides/cross-project-questions.md` — Questions connecting multiple projects
- `interview-prep/mock-interview-guides/answer-calibration.md` — Weak/acceptable/strong examples
- `technology-stack.md` — Complete tech stack from knowledge graph
- `repository-map.md` — Map of all repos with roles
- `README.md` — Index of all content with navigation

Query the knowledge graph to find cross-project connections:
```sql
-- Find technologies shared across multiple projects
SELECT t.name, GROUP_CONCAT(p.name, ', ') as projects, COUNT(*) as project_count
FROM relationships r
JOIN entities t ON t.id = r.target_id AND t.type = 'technology'
JOIN entities p ON p.id = r.source_id AND p.type = 'project'
WHERE r.rel_type = 'built_with'
GROUP BY t.id
HAVING COUNT(*) > 1
ORDER BY project_count DESC;

-- Find pattern connections across projects
SELECT m1.project_id, m2.project_id, m1.content, m2.content
FROM memories m1
JOIN memories m2 ON m1.category = m2.category AND m1.project_id != m2.project_id
WHERE m1.category IN ('pattern', 'design_decision')
LIMIT 20;
```

---

## Phase 6: State Save

Update the manifest with everything processed in this run:

```bash
python3 << 'PYEOF'
import json, os
from datetime import datetime, timezone

manifest_path = os.path.expanduser('~/gajaretejas/resume/.manifest.json')

# Load existing or create new
try:
    with open(manifest_path) as f:
        manifest = json.load(f)
except (FileNotFoundError, json.JSONDecodeError):
    manifest = {
        "version": 1,
        "created_at": datetime.now(timezone.utc).isoformat(),
        "processed_sessions": {"claude": [], "copilot": []},
        "processed_jira_keys": [],
        "processed_repo_commits": {},
        "processed_projects": [],
        "processed_confluence_pages": [],
        "run_history": []
    }

# Update with this run's data
manifest["last_run"] = datetime.now(timezone.utc).isoformat()

# Append newly processed session IDs
# manifest["processed_sessions"]["claude"].extend([...new session IDs...])
# manifest["processed_sessions"]["copilot"].extend([...new session IDs...])

# Append newly processed Jira keys
# manifest["processed_jira_keys"].extend([...new keys...])

# Update repo commit SHAs
# manifest["processed_repo_commits"]["~/glcp/rave-cloud"] = "abc123..."

# Log run
manifest["run_history"].append({
    "timestamp": datetime.now(timezone.utc).isoformat(),
    "new_sessions_mined": 0,  # Replace with actual count
    "new_jira_issues": 0,     # Replace with actual count
    "docs_generated": 0,       # Replace with actual count
    "docs_updated": 0,         # Replace with actual count
    "memories_added": 0        # Replace with actual count
})

# Keep only last 50 runs in history
manifest["run_history"] = manifest["run_history"][-50:]

with open(manifest_path, 'w') as f:
    json.dump(manifest, f, indent=2)

print(f"Manifest saved. Last run: {manifest['last_run']}")
PYEOF
```

---

## Phase 7: Commit & Push

```bash
cd ~/gajaretejas/resume

# Stage all changes (including knowledge.db and .manifest.json)
git add -A

# Check if there are changes to commit
if git diff --cached --quiet; then
    echo "NO_CHANGES — nothing to commit"
else
    # Generate commit message based on what changed
    CHANGED_FILES=$(git diff --cached --name-only | head -20)
    CHANGE_COUNT=$(git diff --cached --name-only | wc -l | tr -d ' ')

    git commit -m "work-context-refresh: update ${CHANGE_COUNT} files

Auto-generated by work-context-refresh skill.
$(date -u +%Y-%m-%dT%H:%M:%SZ)

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"

    git push origin main
    echo "COMMITTED_AND_PUSHED"
fi
```

---

## Useful Knowledge Graph Queries

These queries help agents navigate the knowledge base efficiently. Include them
when other agents need to understand the data:

### Find all projects and their tech stacks
```sql
SELECT p.name, GROUP_CONCAT(t.name, ', ') as technologies
FROM entities p
JOIN relationships r ON r.source_id = p.id AND r.rel_type = 'built_with'
JOIN entities t ON t.id = r.target_id
WHERE p.type = 'project'
GROUP BY p.id;
```

### Find all contributions for a project
```sql
SELECT c.name, c.description
FROM entities c
JOIN relationships r ON r.source_id = c.id AND r.rel_type = 'part_of'
WHERE r.target_id = '{project-id}' AND c.type = 'contribution';
```

### Find memories by category for interview prep
```sql
SELECT m.content, m.raw_context, e.name as project
FROM memories m
LEFT JOIN entities e ON e.id = m.project_id
WHERE m.category = '{category}'
ORDER BY m.created_at DESC;
```

### Find cross-project technology connections
```sql
SELECT t.name as technology,
       GROUP_CONCAT(p.name, ' | ') as used_in,
       COUNT(DISTINCT p.id) as project_count
FROM relationships r
JOIN entities t ON t.id = r.target_id AND t.type = 'technology'
JOIN entities p ON p.id = r.source_id AND p.type = 'project'
WHERE r.rel_type IN ('built_with', 'uses')
GROUP BY t.id
HAVING project_count > 1
ORDER BY project_count DESC;
```

### Get document coverage (what's missing)
```sql
SELECT e.id, e.name, e.type,
       CASE WHEN d.id IS NOT NULL THEN 'documented' ELSE 'MISSING' END as status,
       d.file_path, d.last_updated
FROM entities e
LEFT JOIN documents d ON d.entity_id = e.id
WHERE e.type IN ('project', 'contribution')
ORDER BY status DESC, e.name;
```

### Get all memories for enriching a document
```sql
SELECT m.category, m.content, m.source, m.raw_context
FROM memories m
WHERE m.project_id = '{entity-id}'
ORDER BY
  CASE m.category
    WHEN 'architecture' THEN 1
    WHEN 'design_decision' THEN 2
    WHEN 'trade_off' THEN 3
    WHEN 'debugging' THEN 4
    WHEN 'pattern' THEN 5
    WHEN 'gotcha' THEN 6
    ELSE 7
  END;
```

---

## Important Notes

1. **Incremental by design** — Always check the manifest and processing_log before processing anything. Skip already-processed items.
2. **Session data is private** — Never commit raw session .jsonl content. Only commit distilled insights and memories.
3. **Knowledge graph is the backbone** — All document generation should query the graph for context. This ensures cross-referencing and consistency.
4. **Quality over quantity** — When mining sessions, focus on high-signal content: design decisions, trade-offs, architecture discussions, debugging breakthroughs. Skip casual/trivial exchanges.
5. **Respect PROMPT.md format** — The document format (sections A-G, STAR stories, rubrics, grilling chains) is defined in PROMPT.md. Read it before generating content.
6. **Idempotent** — Running twice with no new data should produce no changes.
7. **Tell the user** — At key milestones (discovery complete, generation starting, commit), inform the user of progress and let them guide if needed.
