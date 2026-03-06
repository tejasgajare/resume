# Rabbit Holes — Deep Follow-Up Chains by Topic

> Progressive drill-down questions an interviewer might ask. Each chain goes 5-7 levels deep.
> Prepare to answer at any level — depth of answer reveals depth of understanding.

---

## 1. JWT / Authentication Chain

### Level 1: "How does authentication work in your system?"
**Expected Answer:**
Dual-layer auth: Istio RequestAuthentication validates JWT signatures at the mesh level, then our Go middleware extracts HPE-specific claims (workspace_id, principal_type, identity_type) for business logic. v3 APIs enforce environment-specific issuers.

### Level 2: "How does JWKS work?"
**Expected Answer:**
JSON Web Key Set — the identity provider (HPE SSO) exposes a `.well-known/jwks.json` endpoint containing public keys. Istio fetches these keys and uses them to verify JWT signatures. Each key has a `kid` (key ID) that matches the `kid` header in the JWT.

```json
{
  "keys": [{
    "kty": "RSA",
    "kid": "key-id-1",
    "n": "base64-modulus",
    "e": "AQAB",
    "use": "sig",
    "alg": "RS256"
  }]
}
```

### Level 3: "What's the caching strategy for JWKS?"
**Expected Answer:**
Istio caches the JWKS response. Default behavior:
- Fetches JWKS at startup and caches in memory
- Re-fetches when it encounters a JWT with an unknown `kid`
- Has a configurable refresh interval (default ~20 min for Istio's Envoy proxy)
- The JWKS endpoint should set Cache-Control headers

The Go middleware does NOT fetch JWKS — it relies on Istio for signature verification and uses `ParseUnverified` since Istio already validated the signature.

### Level 4: "How do you handle key rotation?"
**Expected Answer:**
The identity provider rotates keys by:
1. Publishing the new key in JWKS alongside the old key
2. Signing new tokens with the new key (new `kid`)
3. After a grace period, removing the old key from JWKS

Istio handles this transparently — when it sees a JWT with an unknown `kid`, it re-fetches JWKS. The overlap period ensures no tokens fail validation during rotation.

**Risk:** If rotation happens without overlap (old key removed before new key published), existing tokens fail. This is an IdP operational concern, not something we control.

### Level 5: "What if the JWKS endpoint is down?"
**Expected Answer:**
Istio has a cached copy of the last successful JWKS response. As long as the cache is valid:
- Tokens signed with known keys continue to work
- Tokens with new `kid` values will fail (can't fetch new keys)

If the cache is expired AND the JWKS endpoint is unreachable:
- Istio will reject all tokens (fail-closed is the default)
- This is correct from a security perspective — fail-open would be worse
- Mitigation: JWKS endpoint should be highly available (CDN, multiple regions)

### Level 6: "Why do you have both Istio auth and application-level auth?"
**Expected Answer:**
Defense in depth:
- **Istio**: Cryptographic verification (signature, expiry, issuer) — can't be bypassed by service code bugs
- **Application**: Business logic claims extraction (workspace_id, identity_type), version-specific validation, metrics on auth failures
- Istio can't do version-specific behavior (v1 tokens vs v3 tokens)
- Application auth also catches issues like cross-environment tokens that Istio's config might not restrict (depends on how Helm values are configured)

### Level 7: "If you were designing this from scratch, would you keep both layers?"
**Expected Answer:**
Yes, but I'd align them better from the start. The hardcoded-issuer bug happened because the two layers evolved independently. I'd:
- Make the Go middleware always read issuers from the same source as Istio (Helm values → env vars)
- Add integration tests that verify Istio config and middleware config match
- Consider moving claim extraction to an Envoy filter (Lua or WASM) to consolidate
- Keep application-level auth for version-specific logic that can't be expressed in Istio CRDs

---

## 2. Monitoring / Observability Chain

### Level 1: "How does your monitoring work?"
**Expected Answer:**
Prometheus scrapes metrics from all RAVE services via `/metrics` endpoint. Grafana visualizes dashboards. AlertManager routes alerts to Slack (warning) and PagerDuty (critical). Business metrics track case creation rates, CRM success/failure. Infrastructure metrics track pod health.

### Level 2: "How does Prometheus scraping work?"
**Expected Answer:**
Prometheus uses a pull model — it periodically HTTP GETs the `/metrics` endpoint of each target:

```yaml
scrape_configs:
  - job_name: 'wellness-producer'
    scrape_interval: 30s
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

Services expose metrics using `prometheus/client_golang`:
```go
var httpRequestsTotal = promauto.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total HTTP requests",
    },
    []string{"method", "path", "status"},
)
```

### Level 3: "What's the cardinality problem?"
**Expected Answer:**
Cardinality = number of unique time series. Each unique combination of metric name + label values = one time series.

**Bad example:**
```go
// This creates a time series per unique user ID — potentially millions!
httpRequestsTotal.WithLabelValues("GET", "/api/cases", "200", userID).Inc()
```

**Good example:**
```go
// Bounded cardinality: methods × paths × statuses
httpRequestsTotal.WithLabelValues("GET", "/api/cases", "200").Inc()
```

I specifically designed `mc_event_case_action_total` with only 4 bounded action values to avoid this.

### Level 4: "How do you handle metric explosion in practice?"
**Expected Answer:**
1. **Label hygiene**: Never put unbounded values (user IDs, request IDs, timestamps) in labels
2. **Recording rules**: Pre-compute expensive queries as new metrics with lower cardinality
3. **Metric lifecycle**: Remove deprecated metrics when dashboards are updated
4. **Alerting on cardinality**: Set alerts for total time series count exceeding thresholds
5. **In RAVE specifically**: The unified `mc_event_case_action_total` replaced 4 separate metrics, keeping cardinality bounded

### Level 5: "How do recording rules work?"
**Expected Answer:**
Recording rules pre-compute frequently used PromQL queries and store results as new metrics:

```yaml
groups:
  - name: rave_recording_rules
    rules:
      - record: rave:case_creation_rate:5m
        expr: rate(mc_event_case_action_total{action="created"}[5m])
      - record: rave:crm_failure_rate:5m
        expr: rate(crm_relay_failure_total[5m]) / rate(crm_relay_total[5m])
```

Benefits:
- Dashboard queries are instantaneous (just query the pre-computed metric)
- Reduces Prometheus query load
- Enables alerting on derived metrics without complex AlertManager rules

### Level 6: "How do you debug a metric that's not appearing?"
**Expected Answer:**
1. `curl localhost:8080/metrics | grep metric_name` — is the service exposing it?
2. Check Prometheus targets page — is the service being scraped?
3. Check pod annotations — does it have `prometheus.io/scrape: "true"`?
4. Check relabel configs — is the metric being dropped?
5. Check Prometheus TSDB — has cardinality exceeded limits?
6. In RAVE: I've debugged missing metrics caused by the metric not being registered (forgot `promauto` or explicit `Register`)

### Level 7: "What would you do differently about RAVE monitoring?"
**Expected Answer:**
- Add distributed tracing (Jaeger/Zipkin) for cross-service request flows
- Implement SLOs (Service Level Objectives) with error budgets
- Add the AI-powered Grafana panel concept I designed — natural language queries for monitoring
- Better alert grouping — current alerts can be noisy during deployments
- Structured logging correlated with traces (trace ID in logs)

---

## 3. Kubernetes / Istio Chain

### Level 1: "How does Istio RequestAuthentication work?"
**Expected Answer:**
It's a CRD that configures Envoy sidecar proxies to validate JWTs:
- Specifies accepted issuers and JWKS URIs
- Envoy validates JWT signature, expiry, and issuer before the request reaches the service
- `forwardOriginalToken: true` passes the token to the service for claim extraction
- Invalid tokens get 401 before reaching the application

### Level 2: "What's the difference vs application-level auth?"
**Expected Answer:**
| Aspect | Istio (Envoy) | Application (Go middleware) |
|--------|--------------|----------------------------|
| What | Signature, expiry, issuer | Claims extraction, business logic |
| When | Before request reaches pod | Inside the pod |
| Bypass | Can't be bypassed by app code | Could be skipped with a code bug |
| Config | Helm/YAML (IaC) | Go code (needs deploy) |
| Granularity | Per-service selector | Per-route, per-version |

### Level 3: "How do you handle auth failures gracefully?"
**Expected Answer:**
Two types:
1. **Istio rejection**: Returns 401 with Envoy's default error body. We configured custom error responses via EnvoyFilter for better UX.
2. **App rejection**: Returns 401 with structured JSON error body and increments `auth_failure_total` metric.

The metric is crucial — a spike in auth failures often means either:
- A misconfiguration (wrong issuers after deployment)
- An attack (brute-force token guessing)
- A client bug (expired token not being refreshed)

### Level 4: "How does Istio service mesh routing work?"
**Expected Answer:**
Istio injects Envoy sidecar proxies into every pod. All traffic goes through the sidecar:
```
Client → Envoy (client sidecar) → Network → Envoy (server sidecar) → App
```

VirtualService and DestinationRule CRDs control routing:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: wellness-producer
spec:
  hosts:
  - wellness-producer
  http:
  - match:
    - headers:
        api-version:
          exact: "v3"
    route:
    - destination:
        host: wellness-producer
        subset: v3
  - route:
    - destination:
        host: wellness-producer
        subset: v2
```

### Level 5: "How does the ESO integration work?"
**Expected Answer:**
External Secrets Operator syncs secrets from AWS Parameter Store into Kubernetes Secrets:

```
AWS Parameter Store → SecretStore (CRD) → ExternalSecret (CRD) → K8s Secret → Pod env/volume
```

Each service needs:
1. A ServiceAccount with IAM role annotation (IRSA)
2. A SecretStore referencing the AWS region and auth method
3. ExternalSecret resources mapping parameter paths to secret keys

I migrated 3 services (caseRelay, crmRelay, wellnessProducer) from Vault to ESO, with 17 remaining.

### Level 6: "What problems did you encounter during the ESO migration?"
**Expected Answer:**
- **IAM permissions**: Each service's IRSA role needed specific Parameter Store read access — had to create per-service IAM policies
- **Secret naming**: Had to maintain backward compatibility — same K8s Secret names as Vault-managed secrets so pods didn't need config changes
- **Rotation**: ESO polls for changes, but Parameter Store doesn't have native rotation — added TTL on ExternalSecret to re-sync periodically
- **Ordering**: ExternalSecret CRD must be applied AFTER SecretStore — Helm hook ordering matters

### Level 7: "Compare Vault vs ESO — what are the trade-offs?"
**Expected Answer:**
| Aspect | Vault | ESO + Parameter Store |
|--------|-------|----------------------|
| Complexity | Full server to manage | Operator + AWS managed |
| Dynamic secrets | Yes (DB creds, etc.) | No (static) |
| Audit | Built-in | AWS CloudTrail |
| Cost | Self-hosted compute | AWS charges per parameter |
| Secret rotation | Native | Manual or Lambda-based |
| K8s integration | CSI driver or injector | Native operator |

ESO is simpler for our use case (static config secrets). Vault is overkill when you don't need dynamic secret generation.

---

## 4. MongoDB / Data Modeling Chain

### Level 1: "How does your data model handle multi-tenancy?"
**Expected Answer:**
Shared collection with query-level scoping. Every document has a `workspace_id` field. Every query includes `workspace_id` in the filter — typically extracted from the JWT middleware and injected automatically.

### Level 2: "What's the indexing strategy?"
**Expected Answer:**
Compound index `{workspace_id: 1, case_number: 1}` is the primary access pattern. This makes tenant-scoped lookups O(log n) and ensures the index supports both "get case by number within workspace" and "list all cases for workspace" (prefix of compound index).

Secondary indexes:
- `{workspace_id: 1, status: 1}` for dashboard queries
- `{created_at: 1}` for TTL or time-range queries

### Level 3: "How do you handle schema evolution?"
**Expected Answer:**
MongoDB's flexible schema is both a blessing and a curse:
- New fields: Add to struct with `omitempty` tag — old documents without the field return zero value
- Field rename: Dual-read (check both old and new field name) during migration period
- Nested structure changes: More complex — need a migration script or handle both shapes in code

```go
// Backward-compatible field addition
type WellnessModel struct {
    // Old field
    Status string `bson:"status"`
    // New field (v3) — old documents will have nil
    ManualCaseData *ManualCaseData `bson:"manual_case_data,omitempty"`
}
```

### Level 4: "What about the BSON deserialization issues?"
**Expected Answer:**
MongoDB Go driver v1.11.1 deserializes nested documents as `primitive.D` (which is `[]primitive.E`), not as `map[string]interface{}`. This caused bugs when our code expected maps. I fixed it with a recursive `convertBsonDToMap` function.

The subtlety: `primitive.A` is defined as `type A []interface{}` — it's a type definition over `[]interface{}`, so Go's type switch on `[]interface{}` correctly matches it. No need for a separate `primitive.A` case.

### Level 5: "How would you shard this collection?"
**Expected Answer:**
Shard key: `workspace_id` (hashed).

Why hashed:
- Distributes writes evenly (no hot spots from sequential workspace IDs)
- Tenant-scoped queries still efficient (single shard for a given workspace)
- Trade-off: range queries across workspaces are scatter-gather (acceptable — we rarely do this)

Alternative: `{workspace_id: 1, created_at: 1}` range shard key if time-series access pattern is dominant.

### Level 6: "What are the failure modes of your MongoDB setup?"
**Expected Answer:**
- **Primary failover**: MongoDB replica set elects new primary in ~10s. During election, writes fail. Our Go code has retry logic with `mongo.IsNetworkError()` check.
- **Read concern**: We use `majority` read concern for critical reads (case status) and `local` for dashboards (eventual consistency is fine).
- **Write concern**: `w:majority` for data integrity — acknowledged only when majority of replicas confirm.
- **Connection pool exhaustion**: Can happen under load — configured `MaxPoolSize` and `ConnectTimeout` appropriately.

### Level 7: "How would you migrate from MongoDB to a different database?"
**Expected Answer:**
I'd use the repository pattern we already have — all MongoDB access goes through a `Repository` interface:

```go
type WellnessRepository interface {
    FindByID(ctx context.Context, id string) (*WellnessModel, error)
    Insert(ctx context.Context, model *WellnessModel) error
    Update(ctx context.Context, model *WellnessModel) error
}
```

To migrate:
1. Implement new repository (e.g., `PostgresWellnessRepository`)
2. Dual-write period: write to both, read from old
3. Backfill: migrate existing data
4. Validation: compare reads from both
5. Cutover: read from new, stop writing to old
6. Cleanup: remove old repository code

---

## 5. File Upload Security Chain

### Level 1: "How do file uploads work in your system?"
**Expected Answer:**
Multi-layer validation: filename sanitization → extension allowlist (60+ types) → content-type header check → magic byte verification → size limits → store in MongoDB → forward to CRM.

### Level 2: "What are magic bytes and why do you check them?"
**Expected Answer:**
Magic bytes are the first few bytes of a file that identify its format. A PDF starts with `%PDF` (0x25504446), a PNG starts with `0x89504E47`. We check these because an attacker could rename `malware.exe` to `report.pdf` — the extension passes, but the magic bytes reveal the truth.

### Level 3: "What about polyglot files?"
**Expected Answer:**
A polyglot file is valid in multiple formats simultaneously. For example, a file can be both a valid PDF and a valid ZIP. Magic bytes only check the beginning of the file, so a file with valid PDF magic bytes could contain embedded malicious content after the PDF header.

Mitigation: magic bytes are one layer in defense-in-depth. Combined with extension allowlist and content-type check, the attack surface is reduced. For full protection, you'd need content scanning (ClamAV).

### Level 4: "How do you handle the ZIP-based formats problem?"
**Expected Answer:**
DOCX, XLSX, PPTX are all ZIP files with specific internal structure. They share the same magic bytes (`PK` / `0x504B0304`). We can't distinguish between them at the byte level alone.

Additional validation: For Office formats, we could unzip and check for the expected internal files (e.g., `word/document.xml` for DOCX, `xl/workbook.xml` for XLSX). We don't do this currently — trade-off between security and performance.

### Level 5: "What about zip bombs?"
**Expected Answer:**
A zip bomb is a small compressed file that expands to enormous size (e.g., 42.zip is 42KB compressed, 4.5PB uncompressed). Mitigation:
- Don't decompress uploads on the server (we forward as-is to CRM)
- If decompression is needed: limit uncompressed size, limit nesting depth, check compression ratio
- Our current system doesn't decompress — it validates the file as-is and forwards the bytes

### Level 6: "What would a complete security audit of your file upload system find?"
**Expected Answer:**
Potential findings:
1. **No virus scanning**: Magic bytes don't catch actual malware — need ClamAV or similar
2. **Text file injection**: CSV/JSON/XML files aren't content-validated — could contain formula injection (CSV), XXE (XML), or code
3. **No content length verification**: If the Content-Length header differs from actual body size
4. **Race condition**: Multiple concurrent uploads to the same case — need mutex or optimistic concurrency
5. **Storage DoS**: No per-tenant rate limiting on uploads

### Level 7: "How would you redesign the file upload system?"
**Expected Answer:**
1. Pre-signed upload URLs (S3) to avoid file transit through the API server
2. Async processing pipeline: upload → S3 → Lambda trigger → ClamAV scan → if clean → notify CRM
3. Content-disposition headers to force download (prevent browser execution)
4. Per-tenant quotas and rate limiting
5. Deferred forwarding to CRM (instead of synchronous) for resilience

---

## 6. Go Language / Patterns Chain

### Level 1: "Why Go for RAVE services?"
**Expected Answer:**
Existing codebase was Go. But also: fast compilation, static binary deployment (great for Docker), built-in concurrency, strong stdlib for HTTP/JSON, good MongoDB driver support. The team had Go expertise.

### Level 2: "What Go patterns do you use most?"
**Expected Answer:**
Interface-based design, error wrapping, context propagation, goroutine-with-recovery, functional options for configuration, repository pattern for data access.

### Level 3: "How do you handle dependency injection in Go?"
**Expected Answer:**
Constructor injection with interfaces:
```go
type Service struct {
    repo  Repository
    kafka Producer
    auth  AuthValidator
}

func NewService(repo Repository, kafka Producer, auth AuthValidator) *Service {
    return &Service{repo: repo, kafka: kafka, auth: auth}
}
```
No framework (like Wire) — explicit wiring in `main.go`. Keeps it simple and testable.

### Level 4: "How do you test services with external dependencies?"
**Expected Answer:**
Interface-based mocking. Define interfaces for all external calls, implement mocks for tests:
```go
type MockRepository struct {
    FindFunc func(ctx context.Context, id string) (*WellnessModel, error)
}
func (m *MockRepository) FindByID(ctx context.Context, id string) (*WellnessModel, error) {
    return m.FindFunc(ctx, id)
}
```
Integration tests use testcontainers or actual MongoDB in CI.

### Level 5: "What's one Go gotcha that bit you in production?"
**Expected Answer:**
The `primitive.D` vs `map[string]interface{}` issue. Go's type system is nominal — `primitive.D` is `[]primitive.E`, not `map[string]interface{}`. A type switch on `interface{}` won't match `primitive.D` as a map. I had to add explicit cases for MongoDB's BSON types.

Another: `primitive.A` is `type A []interface{}` — a type definition, not alias. But the `[]interface{}` case in a type switch DOES match it because Go type switches check the underlying type for defined types based on `[]interface{}`.

[KEY_POINTS] Each rabbit hole should go at least 5 levels deep. Practice answering at each level.
[COMMON_MISTAKES] Memorizing Level 1-2 answers but being unable to go deeper.
[SCORING: 1-5] Level reached indicates depth: L1-2 = score 2, L3-4 = score 3, L5-6 = score 4, L7 = score 5
