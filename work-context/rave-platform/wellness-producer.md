# WellnessProducer — Deep-Dive

## Overview

WellnessProducer is the primary API gateway service in the RAVE platform. I've worked extensively on this service — it's where most of my contributions land. It receives wellness objects from GLCP clients, authenticates requests via JWT, validates payloads, routes to appropriate case creation backends, persists to MongoDB, and produces Kafka events for downstream services. It serves three API versions (v1, v2, v3) with backward compatibility.

---

## Role in RAVE Architecture

WellnessProducer sits at the edge of the RAVE platform. Every wellness object — automated or manual — enters through this service. It's the authentication boundary, the validation layer, and the routing decision point.

```
GLCP UI / External Clients
         │
         ▼
  ┌──────────────────┐
  │ wellnessProducer  │
  │                    │
  │  ├── Auth MW       │──▶ JWT validation (v1.0/v1.1+)
  │  ├── Validation    │──▶ Payload + ManualCaseData checks
  │  ├── Router        │──▶ v1/v2/v3 endpoint dispatch
  │  ├── Controller    │──▶ Business logic + routing
  │  ├── Kafka Prod    │──▶ Event publication
  │  └── MongoDB       │──▶ Persistence
  └──────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
[caseRelay] [crmRelay]
```

---

## REST API Endpoints

### v1 Endpoints (Legacy)

```
POST   /v1/wellness          — Create wellness object (automated alerts)
GET    /v1/wellness/:id      — Get wellness object by ID
GET    /v1/wellness           — List wellness objects (with filters)
PUT    /v1/wellness/:id      — Update wellness object
DELETE /v1/wellness/:id      — Delete wellness object
```

- **Auth**: v1.0 tokens (arubathena.com issuer) accepted
- **Validation**: Basic payload validation
- **Usage**: Legacy integrations, gradually being deprecated

### v2 Endpoints (Manual Case Support)

```
POST   /v2/wellness          — Create wellness object with manual case support
GET    /v2/wellness/:id      — Get wellness object
PUT    /v2/wellness/:id      — Update wellness object
POST   /v2/wellness/upload   — Upload attachment
```

- **Auth**: v1.0 and v1.1+ tokens accepted
- **New features**: ManualCaseData support, attachment uploads, enhanced validation
- **Contact handling**: Primary + alternate contacts

### v3 Endpoints (Current)

```
POST   /v3/wellness          — Create wellness object (v1.1+ tokens only)
GET    /v3/wellness/:id      — Get wellness object
PUT    /v3/wellness/:id      — Update wellness object
POST   /v3/wellness/upload   — Upload attachment
GET    /v3/wellness/categories — Get allowed categories
```

- **Auth**: v1.1+ tokens only (greenlake.hpe.com issuers), enforced by Istio EnvoyFilter + middleware
- **Dynamic issuers**: Environment-based issuer validation
- **Enhanced security**: Workspace isolation from token, not client request

See: [api-versioning.md](./api-versioning.md) for evolution details.

---

## Request Lifecycle

### 1. Receive

HTTP request arrives at wellnessProducer. Istio sidecar handles mTLS termination and initial JWT validation (RequestAuthentication). The request reaches the Go HTTP server (likely `gorilla/mux` or `chi` router).

### 2. Authenticate

The middleware chain processes the request:

```
Request → CORS → Logging → Auth Middleware → Rate Limiting → Handler
```

**Auth Middleware** (I significantly refactored this):

1. Extract `Authorization: Bearer <token>` header
2. Decode JWT header to determine version
3. Validate signature against known JWKS endpoints
4. Validate issuer against environment-specific allowed issuers
5. Extract HPE claims into structured `HPEClaims` object
6. Set claims in request context for downstream use

**Key fix I made**: The controller was previously re-parsing the token separately in `extractPlatformIdFromToken` — using the raw token without verification. I consolidated this so claims are extracted once in middleware (verified) and propagated via context. This eliminated a security gap where an attacker could craft a token with a valid signature but manipulated claims in the controller's separate parse path.

### 3. Validate

For manual case creation (`POST /v3/wellness` with ManualCaseData):

```go
// Validation pipeline (conceptual)
func validateManualCase(req WellnessModel) error {
    // 1. Required fields
    if req.ManualCaseData == nil {
        return ErrManualCaseDataRequired
    }
    
    // 2. Domain validation
    if req.ManualCaseData.Domain != "email" && req.ManualCaseData.Domain != "panhpe" {
        return ErrInvalidDomain
    }
    
    // 3. Category validation (allowlisted values)
    if !isAllowedCategory(req.ManualCaseData.Category) {
        return ErrInvalidCategory  // GLCP-332469
    }
    
    // 4. Contact validation
    if err := validateContacts(req.ManualCaseData); err != nil {
        return err  // GLCP-326100: primary email required
    }
    
    // 5. Attachment validation (if present)
    for _, att := range req.ManualCaseData.Attachments {
        if err := validateAttachment(att); err != nil {
            return err  // GLCP-333057: filename validation
        }
    }
    
    return nil
}
```

**Contact validation** (I fixed GLCP-326100):
- Primary contact: email is **required** (was missing this validation)
- Alternate contact: email is **optional**
- createdBy: **server-set** from JWT `sub` claim (not client-supplied) — security measure

### 4. Route

Based on `ManualCaseData.Domain`:

- `"email"` → Kafka topic for caseRelay → SendGrid email case
- `"panhpe"` → Kafka topic for crmRelay → DCM CRM API case

For automated alerts (no ManualCaseData), routing is based on device family and alert configuration.

### 5. Persist

Wellness object is persisted to MongoDB:

```go
// wellnessObject model (simplified)
type WellnessModel struct {
    ID              primitive.ObjectID  `bson:"_id,omitempty"`
    DeviceID        string              `bson:"deviceId"`
    PlatformID      string              `bson:"platformId"`
    WorkspaceID     string              `bson:"workspaceId"`
    AlertType       string              `bson:"alertType"`
    Severity        string              `bson:"severity"`
    Status          string              `bson:"status"`
    ManualCaseData  *ManualCaseData     `bson:"manualCaseData,omitempty"`
    Attachments     []Attachment        `bson:"attachments,omitempty"`
    CreatedAt       time.Time           `bson:"createdAt"`
    UpdatedAt       time.Time           `bson:"updatedAt"`
}
```

**BSON handling note**: I fixed a bug in `convertBsonDToMap` where `primitive.D` (ordered BSON documents from nested MongoDB queries) wasn't being handled in the type switch. MongoDB's Go driver v1.11.1 deserializes nested documents as `primitive.D`, not `map[string]interface{}`. The fix added a `primitive.D` case that converts to `map[string]interface{}` recursively. `primitive.A` (arrays) was already handled since it's a type definition over `[]interface{}`.

### 6. Respond

HTTP response with created wellness object ID, status, and any validation warnings.

---

## Middleware Chain

### Auth Middleware

See: [jwt-auth-system.md](./jwt-auth-system.md) for full details.

I implemented multi-version support:
- **v1.0 tokens**: `arubathena.com` issuer, older claims structure
- **v1.1+ tokens**: `greenlake.hpe.com` issuer, HPE claims with workspace context
- **Service tokens**: `hpe_principal_type: api-client` — machine-to-machine auth

### Logging Middleware

- Structured logging with request ID, method, path, status code, latency
- Correlation IDs propagated across Kafka messages for distributed tracing
- Sensitive data (tokens, emails) redacted in logs

### Rate Limiting

- Per-client rate limiting based on workspace ID
- Prevents abuse of manual case creation API
- Configurable limits per environment

---

## Kafka Event Production

WellnessProducer publishes events using `segmentio/kafka-go`:

```go
// Kafka producer configuration (conceptual)
writer := kafka.NewWriter(kafka.WriterConfig{
    Brokers:  cfg.KafkaBrokers,
    Topic:    cfg.CaseActionTopic,
    Balancer: &kafka.LeastBytes{},
})

// Event structure
type CaseActionEvent struct {
    WellnessObjectID string          `json:"wellnessObjectId"`
    Action           string          `json:"action"`
    Domain           string          `json:"domain"`
    ManualCaseData   *ManualCaseData `json:"manualCaseData,omitempty"`
    Timestamp        time.Time       `json:"timestamp"`
}
```

**Why Kafka**: Decouples wellnessProducer from case creation backends. If SendGrid is down, email cases queue in Kafka. If DCM CRM is slow, panhpe cases queue. No data loss, no blocking the API response.

---

## MongoDB Operations

### wellnessObject Model

Core CRUD operations on the wellnessObjects collection:

- **Create**: Insert new wellness object with generated ID
- **Read**: Find by ID, find by filters (device, platform, status, time range)
- **Update**: Status transitions, attachment status updates
- **Indexes**: Compound indexes on (platformId, createdAt), (deviceId, status), (workspaceId)

### Attachment Tracking

Each attachment goes through a state machine:

```
fileUploading ──▶ fileUploaded ──▶ fileAttachedToCase
       │                                    │
       └──── (upload failed) ────▶ fileFailed
```

Status updates happen atomically on the wellness object document.

---

## Error Handling

- **400 Bad Request**: Validation failures (missing fields, invalid domain, bad email format)
- **401 Unauthorized**: Invalid/expired JWT, wrong issuer for API version
- **403 Forbidden**: Cross-workspace access attempt (GLCP-333624)
- **413 Payload Too Large**: File exceeds 10 MiB limit
- **415 Unsupported Media Type**: File type not in allowlist
- **500 Internal Server Error**: MongoDB/Kafka failures (retried internally)

---

## Key Jira Issues I Worked On

| Issue | Description | Impact |
|-------|-------------|--------|
| GLCP-335836 | API modifications for manual cases | Enhanced validation pipeline |
| GLCP-334752 | v3 API implementation | v1.1+ auth enforcement |
| GLCP-333624 | Cross-workspace case creation fix | Tenant isolation security |
| GLCP-326100 | Email contact handling | Primary email validation |
| GLCP-325722 | Input validation gaps | Comprehensive request validation |
| GLCP-332469 | Category validation | Allowlisted categories |
| GLCP-336767 | Restrict manual cases | Authorization controls |
| GLCP-337148 | Wellness agent data access | Data access patterns |
| GLCP-330407 | Token acceptance | Multi-version token support |

---

## [KEY_POINTS]

- WellnessProducer is the API gateway — every wellness object enters through it
- Three API versions with backward compatibility and progressive security enforcement
- I fixed the `extractPlatformIdFromToken` security gap — claims now come from verified middleware context only
- Manual case validation is a multi-stage pipeline: domain → category → contacts → attachments
- Kafka decouples API response from case creation — no blocking on SendGrid/CRM

## [COMMON_MISTAKES]

- Thinking wellnessProducer creates cases directly — it routes to caseRelay/crmRelay via Kafka
- Assuming v3 just adds features — it also restricts (v1.1+ tokens only, stricter validation)
- Overlooking the controller's separate token parse — was a real security gap I identified and fixed
- Confusing BSON primitive.D with map[string]interface{} — they're different types in mongo-driver

## [FOLLOW_UP]

- "How do you handle a surge of automated alerts?" → Kafka buffering, spamGuard dedup, HPA scaling
- "What happens if MongoDB is down?" → Retry logic, Kafka provides durability, graceful degradation
- "How did you test the validation pipeline?" → Unit tests per validation stage, integration tests with test MongoDB
- "Walk me through a manual case creation request end-to-end" → Full lifecycle from HTTP to SendGrid/CRM
