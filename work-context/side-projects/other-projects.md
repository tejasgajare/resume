# Other Side Projects

> **Projects**: ChronoStore, ColimaUI, Lucene Search Analysis, Crono
> **Stack**: Go, Swift, Java/Spring Boot/Vaadin, Various

---

## ChronoStore — Time Series Database

### Overview

I built ChronoStore as a learning project to understand time series database internals. Time series data — metrics, IoT sensor readings, financial ticks — has unique access patterns (append-heavy, time-range queries, downsampling) that general-purpose databases handle poorly.

### Architecture

```
┌────────────────────────────────┐
│       Query Engine             │
│  (time-range, aggregation)     │
├────────────────────────────────┤
│       Write-Ahead Log (WAL)    │
├────────────────────────────────┤
│       In-Memory Buffer         │
│  (sorted by timestamp)         │
├────────────────────────────────┤
│       Segment Files            │
│  (compressed, time-partitioned)│
└────────────────────────────────┘
```

### Key Design Decisions

**Write-Ahead Log (WAL)**:
Every write goes to the WAL first, then to an in-memory buffer. The WAL ensures durability — if the process crashes, I can replay the log to recover uncommitted data.

**Time-Based Partitioning**:
Data is partitioned into segments by time window (e.g., 1-hour segments). This makes time-range queries efficient — I only need to scan segments that overlap the query range.

**Compression**:
Time series data compresses extremely well because:
- Timestamps are monotonically increasing → delta encoding
- Values often change slowly → XOR encoding (like Gorilla compression)

**Downsampling**:
Old data is automatically downsampled (e.g., per-second → per-minute → per-hour) to reduce storage without losing trend visibility.

### What I Learned

- LSM-tree inspired write path (WAL → memtable → flush to disk)
- Columnar storage for efficient aggregation queries
- Compression techniques specific to time series data
- How InfluxDB, TimescaleDB, and Prometheus approach the same problems differently

[KEY_POINTS]
- Time series databases optimize for append-heavy, time-range query patterns
- WAL + in-memory buffer provides durability without sacrificing write throughput
- Delta and XOR compression achieve 10-20x compression on typical time series data
- Time-based partitioning enables efficient range queries via segment pruning
[/KEY_POINTS]

---

## ColimaUI — Docker Desktop Alternative (macOS)

### Overview

ColimaUI is a native macOS application that provides a graphical interface for Colima, a lightweight Docker runtime for macOS. Colima is a CLI-only tool — my app wraps it in a native macOS menu bar interface.

### Stack
- **Swift / SwiftUI**: Native macOS UI
- **Colima CLI**: Underlying Docker runtime
- **Process management**: Launching and monitoring Colima from Swift

### Architecture

```
┌─────────────────────────────┐
│   Menu Bar UI (SwiftUI)     │
│   ├── Status indicator      │
│   ├── Container list        │
│   ├── Resource usage        │
│   └── Quick actions         │
├─────────────────────────────┤
│   Colima CLI Bridge         │
│   ├── Process.launch()      │
│   ├── Output parsing        │
│   └── Status polling        │
├─────────────────────────────┤
│   Colima (Lima + containerd)│
│   └── Docker-compatible API │
└─────────────────────────────┘
```

### Key Features
- **Menu bar status**: Green/red indicator for Colima running state
- **Container management**: List, start, stop, remove containers
- **Resource monitor**: CPU, memory, disk usage of the VM
- **Quick actions**: Start/stop Colima, prune images, restart Docker
- **Log viewer**: Stream Colima logs in the app

### Technical Challenges

**Process Management in Swift**:
Launching and monitoring CLI processes from a macOS app requires careful handling:

```swift
let process = Process()
process.executableURL = URL(fileURLWithPath: "/usr/local/bin/colima")
process.arguments = ["status", "--json"]

let pipe = Pipe()
process.standardOutput = pipe

try process.run()
process.waitUntilExit()

let data = pipe.fileHandleForReading.readDataToEndOfFile()
let status = try JSONDecoder().decode(ColimaStatus.self, from: data)
```

**Status Polling**: I poll Colima status every 5 seconds using a Timer. The challenge is handling the case where Colima is slow to respond (e.g., during VM startup) without blocking the UI.

[COMMON_MISTAKES]
- Blocking the main thread with `process.waitUntilExit()` — must run on a background queue
- Not handling Colima not being installed — need graceful fallback
- Assuming Colima's JSON output format is stable — it changes between versions
[/COMMON_MISTAKES]

---

## Lucene Search Analysis — Search Quality Tool

### Overview

I built a search analysis tool using Apache Lucene, Spring Boot, and Vaadin to visualize and debug full-text search behavior. The tool helps understand how Lucene tokenizes, analyzes, and scores documents — essential for tuning search relevance.

### Stack
- **Java / Spring Boot**: Backend and Lucene integration
- **Vaadin**: Java-based web UI framework (no JavaScript needed)
- **Apache Lucene**: Full-text search library
- **Maven**: Build system

### Architecture

```
┌────────────────────────────────┐
│   Vaadin UI (Java)             │
│   ├── Analyzer Comparison      │
│   ├── Token Stream Viewer      │
│   ├── Query Explanation         │
│   └── Score Breakdown          │
├────────────────────────────────┤
│   Spring Boot API              │
│   ├── Index Management         │
│   ├── Search Execution         │
│   └── Analysis Pipeline        │
├────────────────────────────────┤
│   Apache Lucene                │
│   ├── Analyzers                │
│   ├── Index Writer/Reader      │
│   └── Scoring (BM25)          │
└────────────────────────────────┘
```

### Features

**Analyzer Comparison**: Side-by-side view of how different analyzers tokenize the same text:
- StandardAnalyzer: "New York City" → ["new", "york", "city"]
- KeywordAnalyzer: "New York City" → ["New York City"]
- WhitespaceAnalyzer: "New York City" → ["New", "York", "City"]

**Query Explanation**: Visual breakdown of how Lucene scores a document for a query. Shows the BM25 formula components: term frequency, inverse document frequency, field length normalization.

**Token Stream Viewer**: Step-by-step visualization of the analysis pipeline: CharFilter → Tokenizer → TokenFilter chain.

### Why Vaadin?

Vaadin lets me build interactive web UIs in pure Java — no JavaScript, no REST APIs, no frontend build step. For an internal tool, this is ideal:
- Single codebase (Java)
- Server-side rendering (no client state management)
- Rich components (grids, charts) out of the box
- Fast development for data-heavy UIs

[KEY_POINTS]
- Understanding Lucene internals is crucial for search relevance tuning
- Analyzer choice dramatically affects search quality
- BM25 scoring is the industry standard — knowing its components matters
- Vaadin is underrated for internal tools: full-featured web UI in pure Java
[/KEY_POINTS]

---

## Crono — Time Tracking

### Overview

Crono is a lightweight time tracking utility I built for personal productivity. It focuses on simplicity — start/stop timers, categorize by project, and review weekly summaries.

### Key Design Principles
- **Minimal UI**: No complex project hierarchies. Just timer + tag.
- **Keyboard-first**: Global hotkey to start/stop tracking
- **Local-first**: All data stored locally, no cloud dependency
- **Weekly review**: Aggregate view of time spent per project/tag

### Technical Notes
- Built as a CLI tool with optional GUI
- SQLite for local persistence
- Export to CSV for analysis in spreadsheets

---

## Interview Talking Points

**"Tell me about a project where you explored a new domain."**
ChronoStore was my deep dive into time series database internals. I implemented a WAL, time-based partitioning, and delta/XOR compression. The project taught me how InfluxDB and Prometheus approach the same problems — write-optimized storage with efficient time-range queries.

**"What's an internal tool you built?"**
Lucene Search Analysis — a Spring Boot + Vaadin app that visualizes how Lucene tokenizes, analyzes, and scores documents. It shows analyzer comparison, token streams, and BM25 score breakdowns side-by-side. I chose Vaadin because it lets me build rich web UIs in pure Java with no frontend build step.

**"Tell me about a macOS app you built."**
ColimaUI wraps the Colima Docker runtime in a native macOS menu bar app. The interesting challenge was process management — launching CLI processes from Swift, parsing JSON output, and polling status without blocking the UI thread.

[FOLLOW_UP]
- How does ChronoStore's compression compare to Gorilla or Prometheus TSDB?
- Why Vaadin over React/Angular for the Lucene tool?
- How would you handle Colima version compatibility in ColimaUI?
[/FOLLOW_UP]

---

## Cross-References

- See [Side Projects → ClipStash](clipstash.md) for another macOS native app with similar patterns to ColimaUI
- See [Side Projects → DiceDB](dicedb.md) for another database internals project (in-memory vs. time series)
- See [Tooling → Search Analysis Tools](../tooling-automation/search-analysis-tools.md) for more on Lucene and search analysis
- See [Nimble Download → Global Trade Compliance](../nimble-download/global-trade-compliance.md) for production Go experience

---

*Last updated: Auto-generated from work-context-refresh skill*
