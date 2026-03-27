# Sahaya — Personal AI Assistant

> **Type**: Personal / Side Project (Sole Developer)
> **Stack**: Python/FastAPI/LangGraph (backend), React Native/Expo (mobile), PostgreSQL/pgvector, Redis, Gemini
> **Platform**: iOS/Android (Expo), Raspberry Pi ARM64 (backend)
> **Repos**: `~/tejasgajare/sahaya` (backend), `~/tejasgajare/sahaya-app` (mobile)
> **Live**: api.tejasgajare.com (via Cloudflare Tunnel)

---

## A. Project Overview & Architecture

### What It Does

Sahaya is a **chat-first AI personal assistant** — a single mobile app that handles daily planning, task management, meals/pantry tracking, wellness monitoring, calendar, and general AI conversation. Think of it as a personal AI that knows your schedule, your pantry, your tasks, and your health data, all accessible through natural language.

The name "Sahaya" (सहाय) means "helper" in Marathi/Sanskrit.

### Architecture

```
┌──────────────────────────────────┐
│     Sahaya Mobile App            │  React Native / Expo
│  (Chat, Tasks, Meals, Wellness,  │  Zustand stores, Expo Router
│   Calendar, Home)                │  api.tejasgajare.com
└──────────────┬───────────────────┘
               │ HTTPS (Cloudflare Tunnel)
┌──────────────▼───────────────────┐
│     FastAPI Application Server   │  Python 3.11, Gunicorn
├──────────────────────────────────┤
│     LangGraph Orchestrator       │  Multi-agent state machine
│  ┌─────────┐  ┌─────────┐       │
│  │  Intent  │→│ Domain   │       │  5+ domain subgraphs
│  │  Router  │ │ Subgraph │       │  (chat, tasks, meals, calendar, wellness)
│  └─────────┘  └────┬────┘       │
│                     │            │
│  ┌──────────────────▼──────┐     │
│  │   Tool Execution Layer  │     │  Domain-specific tools per agent
│  └──────────────────┬──────┘     │
│                     │            │
│  ┌──────────────────▼──────┐     │
│  │   Memory Writer Agent   │     │  Background, updates pgvector
│  └─────────────────────────┘     │
├──────────────────────────────────┤
│  PostgreSQL + pgvector           │  18 SQLAlchemy models, Alembic
│  Redis                           │  Caching, real-time state
├──────────────────────────────────┤
│  8 Platform Connectors           │
│  Google Calendar │ Gmail │ Outlook│
│  Apple Health │ + 4 more         │
└──────────────────────────────────┘
         Raspberry Pi ARM64
         Docker Compose
```

### Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Mobile | React Native / Expo | Cross-platform app |
| State | Zustand | Client-side state management |
| Routing | Expo Router | File-based navigation |
| API Server | FastAPI + Gunicorn | Async Python HTTP server |
| Agent Framework | LangGraph + LangChain | Multi-agent orchestration |
| LLM | Gemini | Language model for all agents |
| ORM | SQLAlchemy | 18 database models |
| Migrations | Alembic | Schema evolution |
| Database | PostgreSQL + pgvector | Relational data + vector search |
| Cache | Redis | Session state, rate limiting |
| Streaming | SSE (Server-Sent Events) | Real-time LLM token streaming |
| Deployment | Docker Compose | Container orchestration |
| Networking | Cloudflare Tunnel | Secure external access |
| Infrastructure | Raspberry Pi 4 (ARM64) | Self-hosted, privacy-first |

### Scale

- **18 SQLAlchemy models** (users, teams, conversations, messages, tasks, recipes, ingredients, shopping items, calendar events, wellness entries, connectors, etc.)
- **87 REST endpoints** across 11 domain areas
- **5+ LangGraph domain subgraphs** with intent routing
- **8 platform connectors** (Google Calendar, Gmail, Outlook, Apple Health, + more)
- **9 build phases** completed over ~2 weeks
- Single user, self-hosted deployment

---

## B. Requirements & Problem Context

### The Problem

I wanted a **single AI assistant** that could:
1. Manage my daily schedule and tasks through natural language
2. Track meals, pantry inventory, and suggest recipes
3. Monitor wellness/health data from Apple Health
4. Handle calendar events across Google/Outlook
5. Remember context across conversations (semantic memory)
6. Run entirely self-hosted for privacy

No existing solution combined all of these with a local-first, privacy-respecting approach.

### Constraints

- **ARM64 deployment**: Raspberry Pi 4 — limited CPU/RAM, must be efficient
- **Single developer**: End-to-end from database schema to mobile UI
- **Real-time UX**: LLM responses are slow — need streaming for responsive feel
- **Privacy**: No cloud LLM hosting, but acceptable to use API calls (Gemini) since conversation data stays local
- **Multi-platform**: Mobile app must work on both iOS and Android

### Evolution

1. **V1**: Telegram bot with Go+gRPC backend + Python AI agents
2. **V2**: Dedicated React Native mobile app + pure Python/FastAPI/LangGraph backend
3. **Deployment**: K3s → Docker Compose (simplified after learning K3s was overkill)

---

## C. Challenges & Solution Exploration

### Challenge 1: Agent Architecture — Single vs Multi-Agent

**Problem**: With 87 endpoints across 11 domains, a single LLM agent with all tools would suffer from tool confusion (too many tools → poor selection accuracy).

**Options explored**:
| Approach | Pros | Cons |
|----------|------|------|
| Single agent, all tools | Simple, one prompt | Tool confusion at scale, slow |
| Multi-agent chat (AutoGen) | Built-in coordination | Heavy, overkill for mobile backend |
| Domain subgraphs (LangGraph) | Focused agents, clean routing | More code, routing overhead |
| Tool-specific chains (LangChain) | Simple chains | No multi-step reasoning |

**Decision**: LangGraph domain subgraphs with intent routing. Each domain (chat, tasks, meals, calendar, wellness) gets its own agent with a focused system prompt and limited tool set. An intent router classifies the user message and delegates to the right subgraph. Falls back to chat agent for ambiguous queries.

**Why this works**: Each agent sees only 5-15 tools instead of 87. System prompts are domain-specific. The intent router is lightweight (single LLM call with structured output).

### Challenge 2: Backend Language Migration (Go → Python)

**Problem**: The V1 backend was Go+gRPC+Python (Go HTTP server calling Python agent scripts). The LLM orchestration was painful in Go — LangChain/LangGraph ecosystem is Python-native.

**Options explored**:
| Approach | Pros | Cons |
|----------|------|------|
| Keep Go, call Python via gRPC | Go performance, existing code | Two processes, gRPC overhead, hard to debug |
| Rewrite in Python/FastAPI | Native LangGraph, single process | Lose Go performance |
| Use Go LLM libraries (go-llm) | Stay in Go | Immature ecosystem, fewer tools |

**Decision**: Full rewrite to Python/FastAPI/LangGraph. The LLM orchestration is the core of the app — fighting the ecosystem wasn't worth the marginal Go performance benefit for a single-user app.

### Challenge 3: Memory System

**Problem**: The AI needs to remember context across conversations — "You mentioned wanting to try Thai food" or "Last week you said you were low on eggs."

**Solution**: Three-layer memory system:
1. **pgvector semantic search**: Embed conversation turns, search by similarity
2. **Keyword fallback**: When vector match score is below threshold, fall back to text search
3. **Memory writer agent**: Background agent that processes conversations and extracts memorable facts

### Challenge 4: Raspberry Pi Deployment

**Problem**: K3s (lightweight Kubernetes) was my initial deployment target. It worked but was overkill.

**K3s issues**:
- Restarts took 2+ minutes (etcd, apiserver, controller-manager, scheduler all spinning up)
- Resource usage was high for a single-service deployment
- Debugging required `kubectl` → complex for simple log checks
- Helm charts added ceremony without multi-service benefit

**Decision**: Simplified to Docker Compose + Cloudflare Tunnel. Docker Compose starts in seconds, logs are `docker compose logs -f`, and Cloudflare Tunnel provides secure HTTPS without port forwarding or static IP.

### Challenge 5: Real-Time LLM Streaming

**Problem**: LLM responses take 2-10 seconds. Users expect instant feedback.

**SSE vs WebSockets**:
| Aspect | SSE | WebSockets |
|--------|-----|------------|
| Direction | Server → Client only | Bidirectional |
| Reconnection | Automatic | Manual |
| Proxy support | Works through all proxies | Can be blocked |
| Complexity | Simple | Connection management |

**Decision**: SSE for streaming. User messages go via POST, responses stream back via SSE. Unidirectional is all we need for LLM token streaming.

---

## D. My Specific Contributions

**I designed and built the entire system end-to-end as the sole developer.** There is no "team" — every line of code, every design decision, every deployment configuration is mine.

### Backend (Python/FastAPI/LangGraph)

- **Designed the data model**: 18 SQLAlchemy models covering users, teams, conversations, messages, tasks, recipes, ingredients, shopping items, calendar events, wellness entries, connectors, device tokens, data access logs, follow-ups, and more
- **Built the LangGraph orchestrator**: Multi-agent state machine with intent routing and domain subgraphs
- **Implemented SSE streaming**: Token-by-token LLM response streaming via FastAPI StreamingResponse
- **Built the memory system**: pgvector semantic search with keyword fallback and background memory writer agent
- **Created 8 platform connectors**: Google Calendar, Gmail, Outlook, Apple Health integration
- **Set up infrastructure**: Docker Compose, Helm charts (K3s era), Cloudflare Tunnel, Alembic migrations
- **Built in 9 phases** (each committed separately):
  1. Foundation (scaffolding, app factory, models, repos, schemas)
  2. LangGraph orchestrator (single agent with all domain tools)
  3. Domain agent subgraphs (intent routing, per-domain agents)
  4. Memory system (pgvector + keyword fallback + memory writer)
  5. SSE streaming + human-in-the-loop interrupts
  6-7. Background intelligence + web search + admin + push + privacy
  8. Connectors (Google Calendar, Gmail, Outlook, Apple Health)
  9. K3s + Helm deployment infrastructure

### Mobile App (React Native/Expo)

- **Built 6 main screens**: Home, Chat, Tasks, Meals, Wellness, Calendar
- **Rich chat responses**: Action cards with icons, checkboxes, radio buttons, lists
- **Calendar day view**: Custom timeline grid with 15-min slots, proper event spanning, overlapping event columns; later migrated to @howljs/calendar-kit
- **Wellness tiles**: Apple Fitness-inspired — activity graphs, sleep stages (REM/Deep/Core)
- **API integration**: Wired all screens from mock Zustand stores to real backend APIs
- **Conversation lifecycle**: Auto-split, auto-title, new chat creation, message persistence

### Infrastructure & Deployment

- **Raspberry Pi ARM64 deployment**: Built Docker images for ARM, set up Docker Compose
- **K3s → Docker Compose migration**: Recognized K3s was overkill, simplified
- **Cloudflare Tunnel**: Secure HTTPS exposure at api.tejasgajare.com
- **OpenAPI 3.1 spec**: Full API documentation + Postman collection

---

## E. STAR Stories

### STAR 1: Multi-Agent Architecture Design

**Situation**: I was building a personal AI assistant that needed to handle 11 different domains (chat, tasks, meals, pantry, wellness, calendar, etc.) with 87+ REST endpoints. A single LLM agent with all tools would suffer from tool confusion — experiments showed accuracy dropping as tool count increased.

**Task**: Design an agent architecture that scales to many domains while keeping each agent focused and accurate. The architecture needed to support real-time streaming, conversation persistence, and human-in-the-loop for destructive actions.

**Action**:
1. Evaluated four architectures: single agent, AutoGen multi-agent chat, LangGraph domain subgraphs, and LangChain tool chains
2. Chose LangGraph domain subgraphs — each domain gets its own agent with a focused system prompt and 5-15 tools
3. Built an intent router that classifies user messages into domains using a lightweight LLM call with structured output
4. Implemented checkpointing for conversation state persistence — each turn is checkpointed so conversations survive restarts
5. Added human-in-the-loop interrupts — for destructive actions (delete task, cancel event), the graph pauses and sends a confirmation request via SSE
6. Built in 9 incremental phases, each independently testable

**Result**: Clean separation of concerns — each agent handles its domain effectively. Intent routing accuracy is high because the classifier only needs to pick a domain (not a specific tool). The system handles multi-step conversations naturally through LangGraph's state management. Completed the full backend in ~2 weeks.

`[KEY_POINTS]` LangGraph domain subgraphs, intent routing, tool confusion avoidance, checkpointing, human-in-the-loop, SSE streaming
`[COMMON_MISTAKES]` Putting all tools in one agent, not handling conversation state, using WebSockets when SSE suffices
`[FOLLOW_UP]` "What happens when the intent router misclassifies? How do you handle cross-domain queries?"
`[SCORING: 1-5]`
- 1: Can't explain the architecture
- 3: Describes multi-agent setup but can't explain why or discuss alternatives
- 5: Explains routing decisions, tool confusion problem, state management, and trade-offs with alternatives

### STAR 2: Backend Language Migration (Go → Python)

**Situation**: The V1 Sahaya backend was Go+gRPC+Python — a Go HTTP server calling Python AI agent scripts via gRPC. The LLM orchestration was the core feature, but the Go ecosystem for LLM tooling was immature. Every LangChain/LangGraph feature required either porting to Go or adding gRPC round-trips.

**Task**: Decide whether to keep the polyglot architecture or rewrite. The backend had 8 SQLite tables, 8 HTTP endpoints, and a working Telegram bot integration.

**Action**:
1. Benchmarked three options: keep Go+gRPC, rewrite to Python, use Go LLM libraries
2. Analyzed the pain points: gRPC serialization overhead for streaming, two separate processes to debug, no native tool-calling support in Go
3. Made the decision: full rewrite to Python/FastAPI/LangGraph
4. Migrated from SQLite to PostgreSQL+pgvector (needed vector search for memory)
5. Replaced gRPC with native Python function calls — eliminated inter-process overhead
6. Used SQLAlchemy+Alembic instead of raw SQL — better for the 18-model data model

**Result**: Single-process Python backend with native LangGraph integration. Development velocity increased dramatically — features that took days in Go+gRPC took hours in Python. The trade-off (Go performance → Python) was irrelevant for a single-user app where the bottleneck is LLM API latency, not server processing.

`[KEY_POINTS]` Pragmatic language choice, LLM ecosystem maturity, single-process simplification, right-sizing for scale
`[COMMON_MISTAKES]` Choosing a language for performance when the bottleneck is external API latency
`[FOLLOW_UP]` "At what scale would you switch back to Go? What about Go+Python hybrid?"
`[SCORING: 1-5]`
- 1: Doesn't understand the trade-offs
- 3: Can describe the migration but not the reasoning
- 5: Articulates ecosystem maturity, bottleneck analysis, and when the trade-off would reverse

### STAR 3: Raspberry Pi Deployment Journey

**Situation**: I wanted to self-host the Sahaya backend on a Raspberry Pi 4 at home — privacy-first, no cloud dependency (except Gemini API calls). I initially chose K3s (lightweight Kubernetes) because I was comfortable with K8s from work.

**Task**: Deploy a FastAPI+PostgreSQL+Redis stack on ARM64 with secure external access.

**Action**:
1. Set up K3s on the Pi with Helm charts, built the deployment manifests
2. K3s worked but had issues: 2+ minute restarts, high idle resource usage, kubectl overhead for simple debugging
3. Recognized I was cargo-culting enterprise patterns for a single-service deployment
4. Migrated to Docker Compose — same containers, 10x simpler operations
5. Set up Cloudflare Tunnel for HTTPS exposure (no port forwarding, no static IP)
6. Configured api.tejasgajare.com DNS A record through Cloudflare

**Result**: Docker Compose starts in seconds (vs 2+ minutes for K3s). Logs are `docker compose logs -f` (vs `kubectl logs -n sahaya pod/...`). Cloudflare Tunnel provides production-grade HTTPS. The lesson: match infrastructure complexity to actual requirements. K3s is great for multi-service orchestration but overkill when Docker Compose + Cloudflare achieves the same result.

`[KEY_POINTS]` Right-sizing infrastructure, K3s vs Docker Compose trade-offs, Cloudflare Tunnel, ARM64 considerations
`[COMMON_MISTAKES]` Cargo-culting enterprise patterns, not evaluating simpler alternatives
`[FOLLOW_UP]` "At what point would K8s make sense for a personal project? What if you had 5 services?"
`[SCORING: 1-5]`
- 1: Can't explain the deployment
- 3: Describes the setup but not why K3s was abandoned
- 5: Shows judgment about infrastructure complexity matching requirements

---

## F. Interview Rabbit Holes & Deep-Dives

### "How does LangGraph's multi-agent orchestration work under the hood?"

LangGraph models applications as **directed graphs** where:
- **Nodes** are functions or agents that transform state
- **Edges** are transitions (can be conditional based on state)
- **State** is a typed dictionary that flows through the graph

The orchestrator runs the graph step-by-step. At each node, it executes the function, updates state, and follows edges to the next node. Conditional edges enable routing (e.g., intent classification determines which domain subgraph runs).

**Checkpointing**: After each node execution, state is checkpointed (to PostgreSQL in Sahaya's case). If the app crashes mid-conversation, it resumes from the last checkpoint.

**Streaming**: LangGraph supports streaming at multiple levels — token-level (each LLM token), event-level (tool calls, state updates), and custom events. Sahaya uses token-level streaming piped through SSE.

### "Why pgvector over Pinecone or Weaviate?"

1. **Single database**: Postgres serves both relational data (18 models) and vectors. One database to backup, migrate, and manage.
2. **Self-hosted**: Running on a Pi — can't depend on cloud vector DBs. Pinecone/Weaviate would add external dependency and latency.
3. **Scale-appropriate**: pgvector handles thousands of vectors efficiently. At personal scale, we're nowhere near the millions where dedicated vector DBs shine.
4. **Trade-off**: Less sophisticated ANN indexing than Pinecone. But for <100K vectors, brute-force search with IVFFlat index is fast enough.

### "How do you handle state management between agents?"

LangGraph's state is a **shared typed dictionary**. Each agent can read and write to specific keys. The orchestrator manages:
- `messages`: Conversation history (append-only)
- `current_domain`: Which domain subgraph is active
- `pending_action`: For human-in-the-loop (action awaiting confirmation)
- `memory_context`: Retrieved memories for the current turn
- Domain-specific state (e.g., `selected_recipe`, `task_list`)

Agents can't interfere with each other because they only run when the intent router selects them, and each subgraph has its own state scope within the shared dictionary.

### "What would you do differently?"

1. **Test infrastructure**: I built fast without comprehensive tests. Would add integration tests for each agent subgraph with mocked LLM responses.
2. **Error boundaries**: If an agent fails mid-response, the user gets a broken SSE stream. Would add graceful degradation (retry, fallback to chat agent).
3. **Observability**: No Prometheus/Grafana on the Pi. Would add basic metrics for agent latency, tool call success rates, memory hit rates.
4. **Offline mode**: Currently requires internet for Gemini API. Would explore local LLM (Ollama + Llama 3) as fallback.

---

## G. Mock Interview Grilling Material

### G.1 Grilling Chain: Multi-Agent Architecture

**Q1**: "Tell me about the AI assistant you built."
- *Tests*: Can they summarize a complex system concisely?
- *Expected*: End-to-end description: mobile app → FastAPI → LangGraph → domain agents → connectors
- *Red flag*: Can't explain the agent architecture beyond "I used LangGraph"
- *Green flag*: Explains the problem (tool confusion) that drove the multi-agent design

**Q2**: "Why multiple agents instead of one?"
- *Tests*: Understanding of LLM tool-calling limitations
- *Expected*: Tool confusion at scale, focused system prompts, 5-15 tools per agent vs 87 combined
- *Red flag*: "Because multi-agent is the trend"
- *Green flag*: Cites specific experiments showing accuracy drop with more tools

**Q3**: "How does intent routing work? What's the failure mode?"
- *Tests*: System design thinking, error handling
- *Expected*: Lightweight LLM classifier with structured output, fallback to chat agent for ambiguous queries
- *Red flag*: No fallback strategy
- *Green flag*: Discusses misclassification rates and how to measure/improve them

**Q4**: "Walk me through what happens when I say 'Add eggs to my shopping list and schedule grocery shopping for tomorrow.'"
- *Tests*: Multi-step, cross-domain understanding
- *Expected*: Intent router detects multi-domain → either sequential routing (task → calendar) or the chat agent handles both with tool calls to both domains
- *Red flag*: Claims each message can only go to one domain
- *Green flag*: Discusses the cross-domain coordination challenge and how they handle it

**Q5**: "How would you scale this to 1000 users?"
- *Tests*: Systems thinking, scaling awareness
- *Expected*: Horizontal FastAPI instances, connection pooling, per-user state isolation in pgvector, Redis for shared state, queue for background tasks, GPU server for local LLM
- *Red flag*: "Just add more Pis"
- *Green flag*: Identifies the LLM API as the bottleneck and discusses batching/caching strategies

**Q6**: "What's the hardest bug you hit?"
- *Tests*: Debugging ability, genuine ownership
- *Expected*: Specific bug with debugging steps — e.g., SSE stream corruption when agent errors mid-response, state serialization issues with complex types, Expo SDK version conflicts
- *Red flag*: Can't recall a specific bug
- *Green flag*: Describes the debugging process, not just the fix

### G.2 Ownership Verification Questions

**"If I looked at git blame for the LangGraph orchestrator, what would I see?"**
Expected: "Every line is mine — I'm the sole developer. The orchestrator is in `app/agents/orchestrator.py`. It defines the graph with `StateGraph`, adds nodes for each domain, and uses conditional edges from the intent router."

**"What's one thing about the system that still bothers you?"**
Expected: Honest answer about a genuine limitation — e.g., "The intent router is a full LLM call which adds 500ms latency. I'd like to replace it with a fine-tuned classifier or even a simple keyword/regex router for common patterns."

**"Walk me through deploying a change to the Pi."**
Expected: "`make ship` — builds Docker image, pushes to registry, SSH to Pi, `docker compose pull && docker compose up -d`. Takes about 3 minutes total."

### G.3 Answer Calibration

**Question**: "Tell me about your personal AI assistant project."

**Weak answer** (1-2): "I built a chatbot using LangGraph and React Native. It can handle tasks and calendar events. I deployed it on a Raspberry Pi."
— No architecture depth, no trade-offs, no challenges

**Acceptable answer** (3): "I built Sahaya, a full-stack AI assistant with a Python/FastAPI backend using LangGraph for multi-agent orchestration and a React Native mobile app. It has 5 domain agents for chat, tasks, meals, calendar, and wellness. I deployed it on a Raspberry Pi with Docker Compose and Cloudflare Tunnel."
— Good overview but missing the 'why' and trade-offs

**Strong answer** (4-5): "I built Sahaya end-to-end as the sole developer. The key design challenge was agent architecture — with 87 endpoints across 11 domains, a single agent would suffer tool confusion. I chose LangGraph domain subgraphs with intent routing, where each agent sees only 5-15 domain-specific tools. The backend started as Go+gRPC+Python but I migrated to pure Python because the LLM ecosystem in Go was immature — the bottleneck is LLM latency, not server processing, so Go's performance advantage was irrelevant. For deployment, I initially used K3s on a Raspberry Pi but realized I was cargo-culting enterprise patterns for a single-service app. Simplified to Docker Compose + Cloudflare Tunnel — starts in seconds versus 2+ minutes for K3s."
— Shows design reasoning, trade-off analysis, and pragmatic engineering judgment

---

## Cross-References

- See [RAVE Platform → Architecture](../rave-platform/architecture.md) for comparison: enterprise K8s vs Pi Docker Compose
- See [Technology Stack](../../technology-stack.md) for LangGraph, FastAPI, React Native proficiency
- See [Interview Prep → Technical Deep-Dives](../../interview-prep/technical-deep-dives.md) for LangGraph deep-dive
- See [Cross-Project Questions](../../interview-prep/mock-interview-guides/cross-project-questions.md) for Sahaya vs RAVE comparisons

---

*Last updated: 2026-03-27*
*Source: Claude sessions (sahaya, sahaya-app), Copilot sessions, git history*
