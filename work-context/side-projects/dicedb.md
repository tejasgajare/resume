# DiceDB — Open Source Contribution

> **Type**: Open Source Contribution
> **Stack**: Go
> **Project**: High-performance in-memory database
> **Repo**: Public (github.com/DiceDB/dice)

---

## Overview

I contributed to DiceDB, an open-source, high-performance in-memory database written in Go. DiceDB is designed as a Redis-compatible alternative with additional features like SQL-like query support and real-time reactivity. The codebase spans 109+ Go files with a focus on performance, concurrency, and protocol compatibility.

Contributing to DiceDB was my deep dive into database internals — command parsing, memory management, protocol handling, and concurrent data structure design in Go.

---

## DiceDB Architecture

### Core Components

```
┌─────────────────────────────────────┐
│          Client Connections          │
│   (RESP protocol / HTTP / WebSocket) │
├─────────────────────────────────────┤
│         Command Parser               │
│   (Redis-compatible command set)     │
├─────────────────────────────────────┤
│         Execution Engine             │
│   ├── String commands (GET, SET)     │
│   ├── Hash commands (HGET, HSET)     │
│   ├── List commands (LPUSH, RPUSH)   │
│   ├── Set commands (SADD, SMEMBERS)  │
│   └── Custom DiceDB commands         │
├─────────────────────────────────────┤
│         Storage Engine               │
│   ├── In-memory store (sync.Map)     │
│   ├── Expiry management              │
│   └── Eviction policies              │
├─────────────────────────────────────┤
│         Persistence (AOF/RDB)        │
└─────────────────────────────────────┘
```

### Key Design Principles

1. **Redis compatibility**: Wire protocol (RESP) and command semantics match Redis
2. **Single-threaded command execution**: Like Redis, commands execute sequentially to avoid locking
3. **Reactive queries**: DiceDB's unique feature — subscribe to query results that update in real-time
4. **Go-native**: Leverages Go's goroutines for connection handling while maintaining single-threaded execution semantics

---

## My Contributions

### Command Implementation

I implemented and enhanced Redis-compatible commands, following DiceDB's established patterns:

#### Command Structure Pattern

Every command in DiceDB follows this pattern:

```go
// Command registration
func init() {
    RegisterCommand("COMMANDNAME", commandNameHandler)
}

// Command handler
func commandNameHandler(args []string, store *Store) []byte {
    // 1. Validate arguments
    if len(args) < requiredArgs {
        return Encode(errors.New("ERR wrong number of arguments"), false)
    }

    // 2. Execute logic against the store
    result, err := store.Operation(args)
    if err != nil {
        return Encode(err, false)
    }

    // 3. Encode response in RESP format
    return Encode(result, false)
}
```

### RESP Protocol

DiceDB implements the Redis Serialization Protocol (RESP) for wire compatibility:

```go
// RESP encoding examples
// Simple String: +OK\r\n
// Error: -ERR message\r\n
// Integer: :1000\r\n
// Bulk String: $5\r\nhello\r\n
// Array: *2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n
```

I worked with the RESP encoder/decoder to ensure byte-level compatibility with Redis clients.

### Testing Patterns

DiceDB has a rigorous testing approach:

```go
func TestGETCommand(t *testing.T) {
    store := NewTestStore()

    tests := []struct {
        name     string
        setup    func()
        args     []string
        expected []byte
    }{
        {
            name:     "GET existing key",
            setup:    func() { store.Set("key1", "value1") },
            args:     []string{"key1"},
            expected: Encode("value1", false),
        },
        {
            name:     "GET non-existing key",
            args:     []string{"missing"},
            expected: RESP_NIL,
        },
        {
            name:     "GET with no arguments",
            args:     []string{},
            expected: Encode(errors.New("ERR wrong number of arguments for 'get' command"), false),
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            if tt.setup != nil {
                tt.setup()
            }
            result := getHandler(tt.args, store)
            assert.Equal(t, tt.expected, result)
        })
    }
}
```

[KEY_POINTS]
- Contributed to a Redis-compatible in-memory database in Go
- Learned database internals: command parsing, RESP protocol, memory management
- Table-driven tests are the standard pattern — every command has exhaustive test cases
- Single-threaded execution model with concurrent connection handling
[/KEY_POINTS]

---

## Go Patterns I Learned

### 1. Interface-Based Command Dispatch

Commands are registered in a map and dispatched dynamically:

```go
type CommandHandler func(args []string, store *Store) []byte

var commandTable = map[string]CommandHandler{}

func RegisterCommand(name string, handler CommandHandler) {
    commandTable[strings.ToUpper(name)] = handler
}

func ExecuteCommand(cmd string, args []string, store *Store) []byte {
    handler, ok := commandTable[strings.ToUpper(cmd)]
    if !ok {
        return Encode(fmt.Errorf("ERR unknown command '%s'", cmd), false)
    }
    return handler(args, store)
}
```

### 2. Byte-Level Encoding

Working with RESP required careful byte manipulation — no JSON marshaling, just raw bytes:

```go
func EncodeBulkString(s string) []byte {
    return []byte(fmt.Sprintf("$%d\r\n%s\r\n", len(s), s))
}
```

### 3. Expiry Management

TTL-based key expiration using a priority queue (min-heap) of expiry times:

```go
type ExpiryEntry struct {
    Key       string
    ExpiresAt time.Time
}

// Background goroutine checks heap and evicts expired keys
```

### 4. Connection Handling

Each client connection gets its own goroutine, but commands funnel into a single execution goroutine via channels:

```go
// Per-connection goroutine
go func(conn net.Conn) {
    for {
        cmd, args := parseCommand(conn)
        resultCh := make(chan []byte)
        executionQueue <- ExecutionRequest{cmd, args, resultCh}
        result := <-resultCh
        conn.Write(result)
    }
}(conn)
```

This pattern gives concurrent connection handling with sequential command execution — the same model Redis uses.

[COMMON_MISTAKES]
- Forgetting RESP \r\n terminators: Off-by-one in protocol encoding breaks all Redis clients
- Not handling nil/null values correctly in RESP: Redis nil is "$-1\r\n", not an empty string
- Assuming Go maps are concurrent-safe: The store uses sync.Map or a mutex-protected map
- Not testing edge cases: Empty strings, binary data in keys, very large values
[/COMMON_MISTAKES]

---

## What I Gained

### Technical Skills
- **Database internals**: Command parsing, protocol handling, storage engines
- **Go performance**: Profiling, allocation reduction, zero-copy patterns
- **Protocol implementation**: RESP wire protocol at the byte level
- **Open source workflow**: PR reviews, issue triage, community guidelines

### Interview Value
- Demonstrates ability to work in large Go codebases (109+ files)
- Shows understanding of database fundamentals
- Evidence of open-source collaboration skills
- Redis internals knowledge applicable to system design interviews

[FOLLOW_UP]
- How does DiceDB's single-threaded model compare to Redis 7's multi-threaded I/O?
- What are the tradeoffs of RESP vs. a binary protocol like gRPC?
- How would you implement transactions (MULTI/EXEC) in this architecture?
- What eviction policy would you choose for a cache use case?
[/FOLLOW_UP]

---

## Interview Talking Points

**"Tell me about an open-source contribution."**
I contributed to DiceDB, a Redis-compatible in-memory database written in Go. I implemented and enhanced command handlers following the RESP protocol, wrote table-driven tests, and worked with the single-threaded execution model. The project has 109+ Go files, so I learned to navigate and contribute to a large unfamiliar codebase.

**"What did you learn about database internals?"**
Working on DiceDB taught me how an in-memory database actually works at the protocol level — parsing RESP wire format, dispatching commands, managing key expiry with priority queues, and the single-threaded execution model that avoids locking overhead.

---

## Cross-References

- See [Nimble Download → Global Trade Compliance](../nimble-download/global-trade-compliance.md) for production Go microservices experience
- See [Side Projects → ClipStash](clipstash.md) for another project using embedded databases (SQLite via GRDB)
- See [Tooling → Search Analysis Tools](../tooling-automation/search-analysis-tools.md) for database query analysis experience

---

*Last updated: Auto-generated from work-context-refresh skill*
