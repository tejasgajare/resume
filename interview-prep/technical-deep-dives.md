# Technical Deep Dives — Code-Level Interview Q&A

> Tejas Gajare — Software Developer II, HPE GLCP Wellness/RAVE Team
> Each topic includes code-level expected answers with Go patterns, architecture decisions, and implementation details.

---

## 1. Go Concurrency Patterns in RAVE

### Q: "How do you handle concurrent operations in RAVE microservices?"

**Expected Answer:**

RAVE uses several concurrency patterns in Go:

**1. Goroutines for Async Processing**
The wellness automation pipeline processes events asynchronously. When a wellness event arrives via Kafka, it's dispatched to a goroutine pool:

```go
func (s *Service) processWellnessEvents(ctx context.Context) {
    for msg := range s.consumer.Messages() {
        go func(m *kafka.Message) {
            defer func() {
                if r := recover(); r != nil {
                    log.Errorf("panic in event processing: %v", r)
                }
            }()
            if err := s.handleEvent(ctx, m); err != nil {
                s.metrics.eventErrors.Inc()
                log.WithError(err).Error("failed to process event")
            }
        }(msg)
    }
}
```

**2. Context-Based Cancellation**
All database operations and external API calls use context with timeout:

```go
func (r *MongoRepo) FindWellnessObject(ctx context.Context, workspaceID, caseID string) (*WellnessModel, error) {
    ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
    defer cancel()
    
    filter := bson.M{
        "workspace_id": workspaceID,
        "case_id":      caseID,
    }
    var result WellnessModel
    err := r.collection.FindOne(ctx, filter).Decode(&result)
    return &result, err
}
```

**3. Mutex for Shared State**
The metrics registry and configuration hot-reload use sync.RWMutex:

```go
type MetricsCollector struct {
    mu      sync.RWMutex
    counters map[string]prometheus.Counter
}

func (mc *MetricsCollector) Increment(name string) {
    mc.mu.RLock()
    c, ok := mc.counters[name]
    mc.mu.RUnlock()
    if ok {
        c.Inc()
    }
}
```

**4. Channel-Based Pipeline (Kafka Consumer)**
In RAVE Classic (Sarama), the consumer uses channels:

```go
// Sarama consumer group handler
func (h *ConsumerHandler) ConsumeClaim(sess sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
    for msg := range claim.Messages() {
        if err := h.process(msg); err != nil {
            log.Error(err)
            continue
        }
        sess.MarkMessage(msg, "")
    }
    return nil
}
```

[KEY_POINTS] Context propagation, goroutine lifecycle management, panic recovery, mutex patterns
[COMMON_MISTAKES] Not mentioning context cancellation; forgetting panic recovery in goroutines
[FOLLOW_UP] "How do you prevent goroutine leaks?"
[FOLLOW_UP] "What's the difference between sync.Mutex and sync.RWMutex and when do you use each?"
[SCORING: 1-5]
- 1: "Go has goroutines"
- 3: Can explain goroutines and channels conceptually
- 5: Shows production patterns with context, cancellation, panic recovery, and explains trade-offs

---

## 2. MongoDB Data Modeling for Wellness Objects

### Q: "Walk me through the MongoDB data model for RAVE."

**Expected Answer:**

The primary collection is `wellness_objects` in MongoDB. Each document represents a wellness case:

```go
type WellnessModel struct {
    ID              primitive.ObjectID `bson:"_id,omitempty"`
    WorkspaceID     string            `bson:"workspace_id"`
    ApplicationID   string            `bson:"application_id"`
    CaseNumber      string            `bson:"case_number"`
    Status          string            `bson:"status"`
    Priority        Priority          `bson:"priority"`
    PrimaryContact  Contact           `bson:"primary_contact"`
    AltContact      *Contact          `bson:"alternate_contact,omitempty"`
    Attachments     []Attachment      `bson:"attachments"`
    CreatedBy       string            `bson:"created_by"`
    ManualCaseData  *ManualCaseData   `bson:"manual_case_data,omitempty"`
    CreatedAt       time.Time         `bson:"created_at"`
    UpdatedAt       time.Time         `bson:"updated_at"`
}

type Priority struct {
    Level  string `bson:"level"`  // critical, high, medium, low
    Source string `bson:"source"` // auto, manual
}

type Contact struct {
    Email string `bson:"email" validate:"required,email"`
    Phone string `bson:"phone"`
    Name  string `bson:"name"`
}
```

**Indexing Strategy:**
```javascript
// Compound index for tenant-scoped queries (most common access pattern)
db.wellness_objects.createIndex({ "workspace_id": 1, "case_number": 1 })

// Index for status-based queries (dashboard, monitoring)
db.wellness_objects.createIndex({ "workspace_id": 1, "status": 1 })

// TTL index for cleanup (optional)
db.wellness_objects.createIndex({ "created_at": 1 }, { expireAfterSeconds: 7776000 })
```

**BSON Deserialization Challenge:**
Nested documents like `priority` and `contacts` deserialize as `primitive.D` (which is `[]primitive.E`), not as `map[string]interface{}`. I fixed this in `convertBsonDToMap`:

```go
func convertBsonDToMap(d interface{}) interface{} {
    switch v := d.(type) {
    case primitive.D:
        m := make(map[string]interface{}, len(v))
        for _, e := range v {
            m[e.Key] = convertBsonDToMap(e.Value)
        }
        return m
    case []interface{}:
        // primitive.A is type []interface{}, so this catches it
        result := make([]interface{}, len(v))
        for i, elem := range v {
            result[i] = convertBsonDToMap(elem)
        }
        return result
    default:
        return v
    }
}
```

**Why MongoDB over SQL:**
- Wellness objects have varying shapes depending on source (automated vs manual, different device types)
- Embedded documents (contacts, attachments) avoid JOIN overhead
- Flexible schema supports evolving requirements without migrations
- Native Go driver (`go.mongodb.org/mongo-driver`) with strong BSON support

[KEY_POINTS] Compound indexes for multi-tenant queries, BSON deserialization gotchas, schema flexibility rationale
[COMMON_MISTAKES] Not mentioning indexes; not knowing about primitive.D vs map[string]interface{}
[FOLLOW_UP] "How do you handle schema evolution?"
[FOLLOW_UP] "What happens when the collection grows to billions of documents?"
[SCORING: 1-5]
- 1: "We use MongoDB"
- 3: Can explain the data model and basic indexing
- 5: BSON gotchas, index strategy rationale, primitive.D fix, and scaling considerations

---

## 3. JWT Token Validation Implementation

### Q: "Walk me through your JWT auth implementation."

**Expected Answer:**

The wellnessProducer has custom JWT middleware that evolved across three API versions:

**Token Flow:**
```
Client → Istio (RequestAuthentication) → wellnessProducer middleware → Handler
```

Istio does initial validation, but the Go middleware does additional checks because:
1. Istio can't do version-specific validation (v1.0 vs v1.1+ tokens)
2. We need to extract custom HPE claims for business logic
3. Version-specific issuer validation

**Middleware Implementation:**
```go
func JWTAuthMiddleware(allowedIssuers []string) gin.HandlerFunc {
    return func(c *gin.Context) {
        tokenStr := extractBearerToken(c.GetHeader("Authorization"))
        if tokenStr == "" {
            c.AbortWithStatusJSON(401, gin.H{"error": "missing authorization"})
            return
        }

        // Parse without verification first (Istio already verified signature)
        token, _, err := new(jwt.Parser).ParseUnverified(tokenStr, jwt.MapClaims{})
        if err != nil {
            c.AbortWithStatusJSON(401, gin.H{"error": "invalid token"})
            return
        }

        claims := token.Claims.(jwt.MapClaims)
        
        // Validate issuer against environment-specific list
        issuer, _ := claims["iss"].(string)
        if !isAllowedIssuer(issuer, allowedIssuers) {
            authFailureMetric.Inc()
            c.AbortWithStatusJSON(401, gin.H{"error": "invalid issuer"})
            return
        }

        // Extract HPE-specific claims
        hpeClaims := extractHPEClaims(claims)
        c.Set("hpe_claims", hpeClaims)
        c.Set("workspace_id", hpeClaims.WorkspaceID)
        c.Set("created_by", claims["sub"])
        
        c.Next()
    }
}

type HPEClaims struct {
    WorkspaceID   string `json:"hpe_workspace_id"`
    PrincipalType string `json:"hpe_principal_type"` // user, api-client
    IdentityType  string `json:"hpe_identity_type"`  // user-identity, service-identity
    TenantID      string `json:"hpe_tenant_id"`
}
```

**Version-Specific Behavior:**
- **v1.0**: Accepted `arubathena.com` tokens (legacy)
- **v1.1+**: Required `greenlake.hpe.com` tokens
- **v3**: Environment-specific issuers only (dev accepts dev tokens, prod accepts prod tokens)

**The Security Fix (Environment-Specific Issuers):**
```go
// Before (vulnerable): hardcoded regex accepting all environments
var validIssuer = regexp.MustCompile(`(arubathena\.com|common\.cloud\.hpe\.com)`)

// After (fixed): environment-configured allowed issuers
func NewAuthMiddleware() gin.HandlerFunc {
    issuers := strings.Split(os.Getenv("ALLOWED_JWT_ISSUERS"), ",")
    return JWTAuthMiddleware(issuers)
}
```

The `ALLOWED_JWT_ISSUERS` env var is set from Helm values, matching Istio's `RequestAuthentication` config.

[KEY_POINTS] Istio + application-level auth, environment-specific issuers, HPE claims extraction, the security fix
[COMMON_MISTAKES] Not explaining why both Istio and app-level auth exist; not knowing about JWKS
[FOLLOW_UP] "Why ParseUnverified instead of full verification?" → Istio already verified, we just need claims
[FOLLOW_UP] "What if someone bypasses Istio?" → Defense in depth; in production, traffic must go through mesh
[FOLLOW_UP] "How do you handle token refresh?"
[SCORING: 1-5]
- 1: "We use JWT"
- 3: Can explain the middleware and claims extraction
- 5: Security fix story, version evolution, Istio alignment, HPE claims, environment isolation

---

## 4. Kafka Consumer/Producer Patterns

### Q: "Compare the Kafka implementations in RAVE Classic vs Cloud."

**Expected Answer:**

**RAVE Classic — Shopify/Sarama:**
```go
import "github.com/Shopify/sarama"

// Consumer Group pattern
type ConsumerGroupHandler struct {
    processor EventProcessor
}

func (h *ConsumerGroupHandler) Setup(sarama.ConsumerGroupSession) error   { return nil }
func (h *ConsumerGroupHandler) Cleanup(sarama.ConsumerGroupSession) error { return nil }

func (h *ConsumerGroupHandler) ConsumeClaim(sess sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
    for msg := range claim.Messages() {
        if err := h.processor.Process(msg.Value); err != nil {
            log.WithError(err).WithField("topic", msg.Topic).Error("processing failed")
            // No ack — message will be redelivered
            continue
        }
        sess.MarkMessage(msg, "") // Commit offset
    }
    return nil
}

// Configuration
config := sarama.NewConfig()
config.Consumer.Group.Rebalance.Strategy = sarama.BalanceStrategyRoundRobin
config.Consumer.Offsets.Initial = sarama.OffsetNewest
config.Version = sarama.V2_6_0_0
```

**RAVE Cloud — segmentio/kafka-go:**
```go
import "github.com/segmentio/kafka-go"

reader := kafka.NewReader(kafka.ReaderConfig{
    Brokers:     []string{"kafka:9092"},
    Topic:       "wellness-events",
    GroupID:     "wellness-consumer",
    MinBytes:    1,
    MaxBytes:    10e6,
    StartOffset: kafka.LastOffset,
})

func (s *Service) consumeLoop(ctx context.Context) {
    for {
        msg, err := reader.ReadMessage(ctx)
        if err != nil {
            if errors.Is(err, context.Canceled) {
                return
            }
            log.WithError(err).Error("read failed")
            continue
        }
        if err := s.process(ctx, msg.Value); err != nil {
            log.WithError(err).Error("processing failed")
            // Message committed on read (auto-commit)
        }
    }
}
```

**Key Differences:**

| Aspect | Sarama (Classic) | kafka-go (Cloud) |
|--------|-----------------|------------------|
| API Style | Callback-based (ConsumerGroupHandler) | Imperative (ReadMessage loop) |
| Offset Commit | Manual (MarkMessage) | Auto-commit on read |
| Error Handling | Skip + no-ack → redelivery | Message committed, error logged |
| Configuration | Verbose Config struct | Simpler ReaderConfig |
| Maintenance | Shopify archived the repo | Actively maintained |
| Reliability | More control, more complex | Simpler, less control |

**Why Cloud uses kafka-go:**
- Simpler API reduces boilerplate
- Better Go idioms (context-based)
- Shopify/Sarama was being deprecated (archived)
- Cloud is a newer codebase, no legacy constraints

[KEY_POINTS] Code-level comparison, trade-offs between libraries, understanding of offset management
[COMMON_MISTAKES] Not knowing the difference between the libraries; saying "we just used Kafka"
[FOLLOW_UP] "What happens if processing fails in each model?"
[FOLLOW_UP] "How do you handle exactly-once semantics?"
[SCORING: 1-5]
- 1: "Kafka is a message queue"
- 3: Can explain producer/consumer patterns
- 5: Library comparison, offset management, reliability trade-offs, migration rationale

---

## 5. Kubernetes Deployment Strategies

### Q: "How are RAVE services deployed?"

**Expected Answer:**

**Deployment Pipeline:**
```
[Git Push] → [GitHub Actions CI]
    → Build Docker image
    → Run tests + SonarQube analysis
    → Push to container registry
    → Update ArgoCD app manifest
    → [ArgoCD] syncs to K8s cluster
```

**Kubernetes Resources per Service:**
```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wellness-producer
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    spec:
      serviceAccountName: wellness-producer-sa  # Custom SA for ESO
      containers:
      - name: wellness-producer
        image: registry/wellness-producer:v3.2.0
        ports:
        - containerPort: 8080
        env:
        - name: ALLOWED_JWT_ISSUERS
          valueFrom:
            secretKeyRef:
              name: wellness-producer-secrets
              key: jwt-issuers
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
```

**Istio Configuration:**
```yaml
# RequestAuthentication — validates JWT at mesh level
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: wellness-producer-auth
spec:
  selector:
    matchLabels:
      app: wellness-producer
  jwtRules:
  - issuer: "https://sso.common.cloud.hpe.com"
    jwksUri: "https://sso.common.cloud.hpe.com/.well-known/jwks.json"
    forwardOriginalToken: true
```

**ESO (External Secrets Operator):**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: wellness-producer-secrets
spec:
  secretStoreRef:
    name: aws-parameter-store
    kind: SecretStore
  target:
    name: wellness-producer-secrets
  data:
  - secretKey: jwt-issuers
    remoteRef:
      key: /rave/prod/wellness-producer/jwt-issuers
```

**Environments:**
- **dev**: Single replica, relaxed resource limits, rapid iteration
- **intg**: Production-like config, integration testing
- **prod**: Multi-replica, HPA, PDB, full monitoring

**Custom Service Account Provisioning:**
Each service gets a dedicated ServiceAccount for least-privilege access to ESO secrets. I provisioned these during the ESO migration:

```bash
kubectl create serviceaccount wellness-producer-sa -n rave
kubectl annotate serviceaccount wellness-producer-sa \
  eks.amazonaws.com/role-arn=arn:aws:iam::role/wellness-producer-eso
```

[KEY_POINTS] Full deployment pipeline, Istio auth config, ESO integration, environment differences
[COMMON_MISTAKES] Not mentioning health probes, rolling update strategy, or service accounts
[FOLLOW_UP] "How do you handle rollback?"
[FOLLOW_UP] "What's the difference between Istio auth and your middleware auth?"
[SCORING: 1-5]
- 1: "We deploy to Kubernetes"
- 3: Can explain deployments and basic K8s concepts
- 5: Full pipeline with Istio, ESO, ArgoCD, environment strategy, and service accounts

---

## 6. File Content Validation via Magic Bytes

### Q: "How does your file upload validation work at the byte level?"

**Expected Answer:**

File uploads go through a multi-layer validation pipeline. The magic byte check is the deepest layer:

**Magic Bytes Registry:**
```go
// magicByteSignatures maps file extensions to their expected magic bytes
var magicByteSignatures = map[string][]byte{
    // Documents
    "pdf":  {0x25, 0x50, 0x44, 0x46},       // %PDF
    "docx": {0x50, 0x4B, 0x03, 0x04},       // PK (ZIP-based)
    "xlsx": {0x50, 0x4B, 0x03, 0x04},       // PK (ZIP-based)
    
    // Images
    "png":  {0x89, 0x50, 0x4E, 0x47},       // .PNG
    "jpg":  {0xFF, 0xD8, 0xFF},              // JPEG SOI marker
    "gif":  {0x47, 0x49, 0x46, 0x38},       // GIF8
    "bmp":  {0x42, 0x4D},                    // BM
    
    // Archives
    "zip":  {0x50, 0x4B, 0x03, 0x04},       // PK
    "gz":   {0x1F, 0x8B},                    // gzip magic
    "tar":  {0x75, 0x73, 0x74, 0x61, 0x72}, // ustar (offset 257)
    
    // Text-based (no magic bytes — validated by extension only)
    // "txt", "csv", "log", "json", "xml" — no byte check
}
```

**Validation Function:**
```go
func validateFileContent(data []byte, filename string) error {
    ext := strings.ToLower(filepath.Ext(filename))
    ext = strings.TrimPrefix(ext, ".")
    
    // Check extension allowlist first
    if !isAllowedExtension(ext) {
        return fmt.Errorf("file extension %q not allowed", ext)
    }
    
    // Magic byte check (skip for text-based formats)
    expected, hasMagic := magicByteSignatures[ext]
    if !hasMagic {
        return nil // Text-based format, no byte signature to check
    }
    
    if len(data) < len(expected) {
        return fmt.Errorf("file too small for declared type %q", ext)
    }
    
    if !bytes.HasPrefix(data, expected) {
        return fmt.Errorf("content mismatch: file claims to be %q but magic bytes don't match", ext)
    }
    
    return nil
}
```

**Why This Matters:**
An attacker could rename `malware.exe` to `malware.pdf`. The extension check passes, but magic byte verification catches the mismatch — a real PDF starts with `%PDF` (0x25504446), not the PE header of an executable.

**Limitations I'm Aware Of:**
- ZIP-based formats (docx, xlsx, pptx) all share the same magic bytes — can't distinguish between them at the byte level
- Polyglot files can have valid magic bytes for one type while containing payloads for another
- Text-based formats (CSV, JSON, XML) have no reliable magic bytes — rely on extension only
- A full content scanning solution (ClamAV) would be more thorough but adds latency

[KEY_POINTS] Specific byte values, handling of ZIP-based formats, text format exceptions, known limitations
[COMMON_MISTAKES] Not knowing actual magic byte values; not mentioning the ZIP overlap problem
[FOLLOW_UP] "How would you handle a ZIP bomb?" → Check compressed vs uncompressed ratio before extraction
[FOLLOW_UP] "What about EICAR test files?" → Could add signature detection for common test patterns
[SCORING: 1-5]
- 1: "We check file extensions"
- 3: Knows about magic bytes conceptually
- 5: Specific byte values, validation code, ZIP overlap awareness, polyglot file discussion

---

## 7. CRM Relay Attachment Race Condition

### Q: "Describe a concurrency bug you fixed in production."

**Expected Answer:**

In crmRelay, there was a race condition with attachments:

**The Problem:**
When a manual case is created with attachments, two things happen:
1. Case creation request goes to DCM CRM API → returns case number
2. Attachments are sent to CRM API referencing the case number

But sometimes attachments arrived at crmRelay BEFORE the case number was assigned (the CRM API was slow). The attachment handler looked up the case by number and failed.

**The Fix — Retry with Backoff:**
```go
func (r *CRMRelay) uploadAttachment(ctx context.Context, caseNum string, attachment Attachment) error {
    var lastErr error
    for attempt := 0; attempt < maxRetries; attempt++ {
        if attempt > 0 {
            backoff := time.Duration(attempt) * 2 * time.Second
            select {
            case <-time.After(backoff):
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        
        // Check if case exists yet
        exists, err := r.caseExists(ctx, caseNum)
        if err != nil {
            lastErr = err
            continue
        }
        if !exists {
            lastErr = fmt.Errorf("case %s not yet created", caseNum)
            r.metrics.attachmentRetryTotal.Inc()
            continue
        }
        
        // Case exists — upload attachment
        return r.crmClient.UploadAttachment(ctx, caseNum, attachment)
    }
    return fmt.Errorf("attachment upload failed after %d retries: %w", maxRetries, lastErr)
}
```

**Metrics:** Added `crm_relay_attachment_retry_total` counter to track how often this happens.

[KEY_POINTS] Specific race condition, retry with backoff pattern, context cancellation, metrics
[COMMON_MISTAKES] Not explaining the root cause clearly; generic "we added retries"
[FOLLOW_UP] "Why not use a message queue instead of retries?" → Could, but adds complexity for a rare race condition
[SCORING: 1-5]
- 3: Describes the bug and the fix
- 5: Root cause, code-level solution, metrics, context-aware retry, trade-off discussion

---

## 8. Go Error Handling Patterns

### Q: "How do you handle errors in your Go services?"

**Expected Answer:**

I follow Go's explicit error handling with some patterns specific to RAVE:

**Wrapped Errors with Context:**
```go
func (s *Service) CreateCase(ctx context.Context, req CreateCaseRequest) (*WellnessModel, error) {
    if err := validate(req); err != nil {
        return nil, fmt.Errorf("validation failed: %w", err)
    }
    
    model, err := s.repo.Insert(ctx, req.ToModel())
    if err != nil {
        return nil, fmt.Errorf("insert case for workspace %s: %w", req.WorkspaceID, err)
    }
    
    return model, nil
}
```

**Sentinel Errors for API Responses:**
```go
var (
    ErrNotFound        = errors.New("not found")
    ErrUnauthorized    = errors.New("unauthorized")
    ErrForbidden       = errors.New("forbidden")
    ErrContentMismatch = errors.New("content type mismatch")
)

// In handler
func (h *Handler) GetCase(c *gin.Context) {
    result, err := h.service.GetCase(ctx, id)
    if err != nil {
        switch {
        case errors.Is(err, ErrNotFound):
            c.JSON(404, gin.H{"error": "case not found"})
        case errors.Is(err, ErrForbidden):
            c.JSON(403, gin.H{"error": "access denied"})
        default:
            c.JSON(500, gin.H{"error": "internal server error"})
            log.WithError(err).Error("unexpected error")
        }
        return
    }
    c.JSON(200, result)
}
```

[KEY_POINTS] Error wrapping with %w, sentinel errors, errors.Is for type checking, no panic in handlers
[COMMON_MISTAKES] Using panic for business logic errors; not wrapping errors with context
[FOLLOW_UP] "When would you use panic vs returning an error?"
[SCORING: 1-5]
- 3: Basic error handling
- 5: Wrapped errors, sentinel patterns, errors.Is, handler-level error mapping
