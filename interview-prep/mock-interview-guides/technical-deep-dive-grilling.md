# Technical Deep-Dive Grilling — Code-Level Question Chains

> 5-7 question chains per topic, requiring code-level knowledge.
> Each question has expected answer, red flags, and green flags.

---

## Chain 1: Go Concurrency & Runtime

### Q1: "How do you handle concurrent requests in your Go services?"
**Why asked:** Basic concurrency understanding.
**Expected:** Each HTTP request is a goroutine (Go's net/http does this). Kafka consumers use goroutines per message. Context propagation for cancellation.
**Red flags:** 🚩 "We use threads" or doesn't know Go handles concurrency automatically for HTTP
**Green flags:** ✅ Understands goroutine-per-request model, context propagation

### Q2: "Write a pattern for processing Kafka messages concurrently with a bounded worker pool."
**Why asked:** Can they implement concurrency correctly?
**Expected:**
```go
func processMessages(ctx context.Context, msgs <-chan Message, workers int) {
    sem := make(chan struct{}, workers)
    for msg := range msgs {
        sem <- struct{}{} // Acquire slot
        go func(m Message) {
            defer func() { <-sem }() // Release slot
            if err := handle(ctx, m); err != nil {
                log.Error(err)
            }
        }(msg)
    }
}
```
**Red flags:** 🚩 Unbounded goroutine spawning, no error handling, missing defer for semaphore release
**Green flags:** ✅ Bounded concurrency, proper closure capture (loop variable), error handling, defer

### Q3: "How do you prevent goroutine leaks?"
**Why asked:** Operational concern in long-running services.
**Expected:** Context cancellation, timeouts on all I/O, select with done channels, goroutine count metrics (runtime.NumGoroutine()), pprof for leak detection.
**Red flags:** 🚩 "Goroutines are lightweight, leaks don't matter"
**Green flags:** ✅ Context-based cancellation, timeout patterns, pprof for diagnosis

### Q4: "Explain the difference between sync.Mutex and sync.RWMutex. When do you use each?"
**Why asked:** Synchronization primitive knowledge.
**Expected:** Mutex: exclusive lock for read AND write. RWMutex: multiple readers OR one writer. Use RWMutex when reads >> writes (e.g., metrics registry, configuration). Use Mutex when write frequency is high or critical section is short (overhead of RWMutex not justified).
**Red flags:** 🚩 Can't explain the difference
**Green flags:** ✅ Explains read/write asymmetry, when RWMutex overhead isn't justified

### Q5: "How does Go's garbage collector work and how does it affect your services?"
**Why asked:** Runtime understanding — important for latency-sensitive services.
**Expected:** Concurrent tri-color mark-and-sweep. Low pause times (<1ms typically). Can be tuned via GOGC. For RAVE, GC pauses are negligible compared to network I/O (MongoDB, CRM API calls). For DiceDB (in-memory), GC matters more.
**Red flags:** 🚩 "Go has GC, it's fine"
**Green flags:** ✅ Tri-color algorithm, GOGC tuning, understands when GC matters (latency-sensitive vs I/O-bound)

### Q6: "How would you detect and fix a goroutine leak in production?"
**Why asked:** Debugging skills under production constraints.
**Expected:** Expose pprof endpoint (net/http/pprof), check goroutine count over time, take goroutine dump (`/debug/pprof/goroutine`), find blocked goroutines, trace back to missing cancellation or unbounded channel.
**Red flags:** 🚩 "Restart the pod"
**Green flags:** ✅ pprof, goroutine dump analysis, root cause identification, prevention patterns

---

## Chain 2: JWT & Cryptographic Auth

### Q1: "Walk me through JWT validation from token to authorized request."
**Why asked:** Can they explain the full auth flow?
**Expected:** Bearer token → extract from Authorization header → base64 decode header/payload → verify signature with public key (from JWKS) → check exp/nbf/iss/aud claims → extract custom claims → set context.
**Red flags:** 🚩 Skips signature verification or doesn't understand JWKS
**Green flags:** ✅ Full flow, mentions JWKS, explains header.payload.signature structure

### Q2: "Why does your middleware use ParseUnverified instead of full verification?"
**Why asked:** Specific implementation decision — tests ownership.
**Expected:** Istio's Envoy sidecar already verified the signature using JWKS from the SSO endpoint. The Go middleware only needs to extract claims for business logic. Double verification would add latency and require JWKS key management in the app.
**Red flags:** 🚩 "I don't know" or "We should verify" (doesn't understand the Istio layer)
**Green flags:** ✅ Explains defense-in-depth architecture, Istio responsibility vs app responsibility

### Q3: "What's the RS256 signing algorithm and how does it work?"
**Why asked:** Cryptographic understanding depth.
**Expected:** RSA PKCS#1 v1.5 signature with SHA-256 hash. Private key signs the hash of `header.payload`, public key verifies. Asymmetric — issuer has private key, verifiers have public key (via JWKS).
**Red flags:** 🚩 Confuses RS256 with HS256 (symmetric)
**Green flags:** ✅ Asymmetric key pair, hash-then-sign, public key distribution via JWKS

### Q4: "An attacker captures a valid JWT. What can they do?"
**Why asked:** Security threat modeling.
**Expected:** Replay attack — use the token until it expires. Mitigation: short expiry, audience restriction, IP binding (rare), token revocation (hard with JWTs since they're stateless).
**Red flags:** 🚩 "Nothing, the token is signed" (misunderstands the threat)
**Green flags:** ✅ Replay attack awareness, stateless nature of JWT = no native revocation, discusses mitigations

### Q5: "How would you implement token revocation for JWTs?"
**Why asked:** Design thinking for a known JWT limitation.
**Expected:** Options: (1) Short expiry + refresh tokens, (2) Token blacklist/denylist in Redis, (3) Token versioning per user (version in JWT, check against user's current version). Each has trade-offs: short expiry annoys users, blacklist needs storage, versioning needs DB lookup.
**Red flags:** 🚩 "JWTs can't be revoked" (technically true but there are patterns)
**Green flags:** ✅ Multiple approaches with trade-offs, understands the fundamental tension (stateless vs revocable)

### Q6: "Your JWT middleware has a bug that accepts expired tokens. How do you detect this?"
**Why asked:** Debugging auth issues.
**Expected:** Auth failure metrics would NOT spike (tokens are accepted). Detection: security audit, penetration testing, or noticing requests with old timestamps succeeding. Add metric for token age distribution.
**Red flags:** 🚩 "Check the logs" without understanding that accepted tokens don't log errors
**Green flags:** ✅ Understands that bugs accepting bad tokens are harder to detect than bugs rejecting good tokens

---

## Chain 3: MongoDB Operations

### Q1: "Explain your MongoDB indexing strategy."
**Why asked:** Database performance fundamentals.
**Expected:** Compound index `{workspace_id: 1, case_number: 1}` for the primary access pattern. Workspace-first because all queries are tenant-scoped. Secondary indexes for status and time-range queries.
**Red flags:** 🚩 "We index every field" or no indexing strategy
**Green flags:** ✅ Compound index design following query patterns, understands prefix property

### Q2: "You have a slow query. Walk me through diagnosing it."
**Why asked:** Database debugging skills.
**Expected:** `db.collection.explain("executionStats")` → check if it's COLLSCAN (no index) or IXSCAN (index used) → check nReturned vs totalDocsExamined → add or modify index → re-explain → verify.
**Red flags:** 🚩 "Add an index and see if it helps" without explain()
**Green flags:** ✅ Uses explain(), knows COLLSCAN vs IXSCAN, examines document ratio

### Q3: "What's the difference between `findOne` and `find().limit(1)`?"
**Why asked:** MongoDB API nuance.
**Expected:** Functionally similar but `findOne` returns a single document directly, `find().limit(1)` returns a cursor. `findOne` is slightly more efficient for single-document lookups. In Go driver: `FindOne()` vs `Find()` with options.
**Red flags:** 🚩 "They're the same"
**Green flags:** ✅ Cursor vs document return type, performance nuance

### Q4: "Explain the primitive.D deserialization issue you fixed."
**Why asked:** Ownership verification — specific to their contribution.
**Expected:** MongoDB Go driver deserializes nested BSON documents as `primitive.D` (`[]primitive.E`), not `map[string]interface{}`. Code using type switches on `interface{}` won't match `primitive.D`. Fixed by adding explicit `primitive.D` case in `convertBsonDToMap`. `primitive.A` is `type []interface{}` — Go type switch on `[]interface{}` matches it (type definition, not alias behavior).
**Red flags:** 🚩 Can't explain the difference between primitive.D and map
**Green flags:** ✅ Type system knowledge, Go's type switch behavior for defined types, recursive conversion

### Q5: "How do you handle MongoDB connection failures in production?"
**Why asked:** Operational resilience.
**Expected:** Retry with `mongo.IsNetworkError()`, connection pool configuration (MaxPoolSize, ConnectTimeout, SocketTimeout), replica set failover handling (10s election), read preference (primary for writes, secondaryPreferred for reads during failover).
**Red flags:** 🚩 "Let it crash and restart"
**Green flags:** ✅ Retry logic, connection pool tuning, replica set awareness, read preference

### Q6: "You need to add a field to every document in a large collection. How?"
**Why asked:** Migration skills at scale.
**Expected:** Background `updateMany` with `$set` and a filter that only matches documents missing the field. Batch processing (not one giant update). Monitor progress. Consider doing it during low-traffic window. Add the field with `omitempty` in Go struct first (backward compatible).
**Red flags:** 🚩 Single `updateMany({}, {$set: ...})` without batching or consideration of scale
**Green flags:** ✅ Batched updates, backward-compatible struct changes first, monitoring

---

## Chain 4: Kafka Patterns

### Q1: "Compare Sarama and kafka-go."
**Why asked:** Library comparison from real experience.
**Expected:** Sarama: callback-based (ConsumerGroupHandler), manual offset commit (MarkMessage), verbose config. kafka-go: imperative ReadMessage loop, auto-commit by default, context-native, simpler API. Sarama archived by Shopify.
**Red flags:** 🚩 "They're both Kafka libraries"
**Green flags:** ✅ Code-level comparison, offset management difference, maintenance status

### Q2: "What happens if a consumer crashes mid-processing with each library?"
**Why asked:** Reliability implications.
**Expected:** Sarama: if MarkMessage wasn't called, the message is redelivered on rebalance. kafka-go (auto-commit): message may be committed before processing, so it's lost. kafka-go (manual): similar to Sarama.
**Red flags:** 🚩 "Kafka guarantees delivery" without understanding at-least-once vs exactly-once
**Green flags:** ✅ Explains at-least-once semantics, auto-commit risk, manual commit safety

### Q3: "How would you implement exactly-once processing?"
**Why asked:** Distributed systems depth.
**Expected:** True exactly-once is very hard. Options: (1) Kafka transactions (write to multiple topics atomically), (2) Idempotent consumers (deduplication by message ID), (3) Outbox pattern (DB + Kafka in same transaction). RAVE uses at-least-once with idempotent handlers.
**Red flags:** 🚩 "Set acks=all and it's exactly-once" (confuses producer durability with consumer processing)
**Green flags:** ✅ Understands the distributed systems challenge, idempotency pattern, outbox pattern

### Q4: "A Kafka consumer is falling behind (consumer lag is growing). Diagnose and fix."
**Why asked:** Operational Kafka skills.
**Expected:** Check consumer lag metric, check processing time per message, check for slow downstream (MongoDB, CRM). Fix: increase partitions + consumers, optimize processing, batch processing, or add more consumer group instances.
**Red flags:** 🚩 "Increase the poll interval"
**Green flags:** ✅ Metrics-driven diagnosis, multiple remediation options, understands partition/consumer relationship

### Q5: "How does Kafka partitioning affect your system?"
**Why asked:** Architecture implications of Kafka internals.
**Expected:** Messages with same key go to same partition → ordering guarantee per key. In RAVE: key by workspace_id or case_id ensures case events are processed in order. Partition count limits consumer parallelism.
**Red flags:** 🚩 Doesn't understand partition/key relationship
**Green flags:** ✅ Key-based routing, ordering guarantee scope, parallelism limits

---

## Chain 5: API Design & Versioning

### Q1: "How did you evolve the wellnessProducer API through 3 versions?"
**Why asked:** API design skills from real experience.
**Expected:** v1: initial API with legacy auth. v2: added manual case creation, but had security gaps (trusted client workspace). v3: environment-specific auth, stricter validation, backward compatible.
**Red flags:** 🚩 Can't explain what changed between versions
**Green flags:** ✅ Version-by-version evolution with rationale, security improvements

### Q2: "How did you maintain backward compatibility?"
**Why asked:** Practical API versioning challenges.
**Expected:** v1 and v2 endpoints still work. New validation in v3 only. Version routing via Istio VirtualService (header-based) or URL path (/v1/, /v2/, /v3/). Old clients unaffected.
**Red flags:** 🚩 "We just deprecated old versions"
**Green flags:** ✅ Multiple versions coexist, routing mechanism explained, migration path for clients

### Q3: "A client sends a v2 request to the v3 endpoint. What happens?"
**Why asked:** Edge case handling in versioned APIs.
**Expected:** Depends on the difference. If auth token is v1.0 format and v3 requires v1.1+, the auth middleware rejects it (401). If the request body has extra/missing fields, validation handles it. Backward compatibility means v2 requests should generally work on v3 (additive changes only).
**Red flags:** 🚩 "It would fail" without explaining why or how
**Green flags:** ✅ Specific failure modes, auth version vs request version distinction

### Q4: "When would you introduce v4?"
**Why asked:** API lifecycle thinking.
**Expected:** Breaking changes that can't be backward compatible: field removal, type changes, fundamentally different auth model. Would also consider: is the maintenance burden of 4 versions worth it? Can we sunset v1/v2?
**Red flags:** 🚩 "Whenever we have new features" — versions aren't for features, they're for breaking changes
**Green flags:** ✅ Breaking change criteria, version lifecycle, sunset planning

### Q5: "How do you deprecate an old API version?"
**Why asked:** Practical deprecation process.
**Expected:** Announce timeline, add deprecation headers, monitor usage, disable for new clients, keep for existing clients during migration, final removal. Metrics on per-version usage are critical.
**Red flags:** 🚩 "Just turn it off"
**Green flags:** ✅ Communication plan, usage metrics, migration support, gradual removal

---

## Chain 6: Error Handling & Debugging

### Q1: "How do you handle errors in your Go services?"
**Why asked:** Go-specific patterns.
**Expected:** Explicit error returns, wrapping with %w, sentinel errors, errors.Is/As for type checking, handler-level HTTP status mapping.
**Red flags:** 🚩 Uses panic for business logic, or ignores errors
**Green flags:** ✅ Wrapping, sentinel errors, proper mapping to HTTP codes

### Q2: "Write error handling for a function that calls MongoDB, validates data, and calls an external API."
**Why asked:** Practical error handling composition.
**Expected:**
```go
func (s *Service) ProcessCase(ctx context.Context, id string) error {
    model, err := s.repo.FindByID(ctx, id)
    if err != nil {
        if errors.Is(err, mongo.ErrNoDocuments) {
            return ErrNotFound
        }
        return fmt.Errorf("find case %s: %w", id, err)
    }
    if err := validate(model); err != nil {
        return fmt.Errorf("validation: %w", err)
    }
    if err := s.crmClient.Submit(ctx, model); err != nil {
        return fmt.Errorf("submit to CRM: %w", err)
    }
    return nil
}
```
**Red flags:** 🚩 Generic error returns without context, no wrapping
**Green flags:** ✅ Error wrapping with context at each level, specific sentinel error handling

### Q3: "A production service is returning 500 errors intermittently. Diagnose it."
**Why asked:** Production debugging skills.
**Expected:** Check error logs (structured logging with request ID), check metrics (which endpoint, error rate pattern), check dependencies (MongoDB connection, CRM availability, Kafka lag), check recent deployments, check resource limits (OOM, CPU throttle).
**Red flags:** 🚩 "Read the logs" without structure
**Green flags:** ✅ Multi-signal approach (logs + metrics + dependencies), structured investigation

### Q4: "How do you add observability to help debug these issues proactively?"
**Why asked:** Prevention over reaction.
**Expected:** Structured logging (JSON with request ID, workspace ID, service name), Prometheus metrics per endpoint, distributed tracing, error categorization metrics, health check endpoints.
**Red flags:** 🚩 "Add more log.Println"
**Green flags:** ✅ Structured logging, correlation IDs, metrics by error type, tracing
