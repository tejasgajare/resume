# JWT Authentication System — RAVE Platform

## Overview

I designed and implemented the multi-version JWT authentication system for RAVE's wellnessProducer service. This is one of my most impactful contributions — it handles the complexity of HPE's token ecosystem where two major token versions coexist across different GLCP environments, plus machine-to-machine service tokens. The system validates tokens at both the Istio service mesh level and the Go middleware level, with environment-specific issuer configuration that replaced a brittle hardcoded regex approach.

---

## Token Versions

### v1.0 Tokens (Legacy — arubathena.com)

```json
{
  "iss": "https://sso.common.cloud.hpe.com",
  "sub": "user-uuid-here",
  "aud": "https://arubathena.com",
  "exp": 1700000000,
  "iat": 1699996400,
  "customer_id": "cust-123",
  "platform_customer_id": "plat-cust-456"
}
```

- **Issuer**: HPE SSO, audience `arubathena.com`
- **Claims**: Flat structure, `customer_id` and `platform_customer_id` for tenant identification
- **Usage**: Legacy GLCP integrations, older UI versions
- **Accepted by**: v1 and v2 API endpoints

### v1.1+ Tokens (Current — greenlake.hpe.com)

```json
{
  "iss": "https://iss.greenlake.hpe.com/<env>",
  "sub": "user-uuid-here",
  "aud": "https://greenlake.hpe.com",
  "exp": 1700000000,
  "iat": 1699996400,
  "hpe_principal_type": "user",
  "hpe_identity_type": "hpe",
  "hpe_ccs_workspace_id": "workspace-uuid",
  "hpe_ccs_workspace_name": "Customer Workspace",
  "hpe_ccs_platform_customer_id": "plat-cust-456",
  "hpe_ccs_application_id": "app-uuid"
}
```

- **Issuer**: Environment-specific (`dev`, `intg`, `prod` each have different issuer URLs)
- **Claims**: Rich HPE claims structure with workspace context
- **Usage**: Current GLCP, all new integrations
- **Accepted by**: v1, v2, and v3 API endpoints (v3 **requires** v1.1+)

### Service Tokens (Machine-to-Machine)

```json
{
  "iss": "https://iss.greenlake.hpe.com/<env>",
  "sub": "service-account-uuid",
  "hpe_principal_type": "api-client",
  "hpe_ccs_workspace_id": "workspace-uuid"
}
```

- **Identification**: `hpe_principal_type == "api-client"`
- **Usage**: Internal service-to-service calls, automated pipelines
- **Handling**: Same middleware path but different authorization logic (service tokens may have broader permissions)

---

## HPE Claims Struct

```go
type HPEClaims struct {
    // Standard JWT claims
    Issuer    string `json:"iss"`
    Subject   string `json:"sub"`
    Audience  string `json:"aud"`
    ExpiresAt int64  `json:"exp"`
    IssuedAt  int64  `json:"iat"`
    
    // HPE-specific claims (v1.1+)
    HPEPrincipalType       string `json:"hpe_principal_type"`
    HPEIdentityType        string `json:"hpe_identity_type"`
    HPECCSWorkspaceID      string `json:"hpe_ccs_workspace_id"`
    HPECCSWorkspaceName    string `json:"hpe_ccs_workspace_name"`
    HPECCSPlatformCustID   string `json:"hpe_ccs_platform_customer_id"`
    HPECCSApplicationID    string `json:"hpe_ccs_application_id"`
    
    // Legacy claims (v1.0)
    CustomerID             string `json:"customer_id"`
    PlatformCustomerID     string `json:"platform_customer_id"`
}
```

**Why a single struct for both versions**: Rather than having separate v1 and v1.1 structs, I use one struct where HPE-specific fields are empty for v1.0 tokens. The middleware detects the version based on the issuer/audience and sets a `tokenVersion` flag in context. Downstream handlers check this flag to know which claims are available.

---

## Environment-Based Issuer Validation

### The Problem (Before My Fix)

The original implementation used hardcoded regex patterns to validate token issuers:

```go
// BEFORE: Brittle regex approach
var validIssuerPattern = regexp.MustCompile(`^https://(sso\.common\.cloud\.hpe\.com|iss\.greenlake\.hpe\.com/.*)$`)

func validateIssuer(issuer string) bool {
    return validIssuerPattern.MatchString(issuer)
}
```

**Problems with this approach**:
1. **Drift**: The Go middleware and Istio RequestAuthentication could disagree on valid issuers
2. **Rigidity**: Adding a new environment or issuer required code changes and redeployment
3. **No environment awareness**: Dev could accept prod tokens (and vice versa)
4. **Testing difficulty**: Couldn't test with environment-specific issuers

### My Solution: Per-Environment Configuration

```go
// AFTER: Environment-based configuration
type AuthConfig struct {
    AllowedIssuers []string `yaml:"allowedIssuers"`
    TokenVersion   string   `yaml:"minTokenVersion"`  // "v1.0" or "v1.1"
}

// Loaded from Helm values — same source as Istio RequestAuthentication
func validateIssuer(issuer string, config AuthConfig) bool {
    for _, allowed := range config.AllowedIssuers {
        if issuer == allowed {
            return true
        }
    }
    return false
}
```

**Helm values** (shared between Go app and Istio):

```yaml
# values-prod.yaml
auth:
  allowedIssuers:
    - "https://sso.common.cloud.hpe.com"
    - "https://iss.greenlake.hpe.com/v1/prod"
  minTokenVersion: "v1.0"

# For v3 endpoints specifically
auth_v3:
  allowedIssuers:
    - "https://iss.greenlake.hpe.com/v1/prod"
  minTokenVersion: "v1.1"
```

**Why this matters**: The Go middleware and Istio RequestAuthentication now read from the same Helm values. They can never disagree on valid issuers. Adding a new environment or issuer is a Helm values change, not a code change.

---

## Middleware Architecture

### Full Processing Pipeline

```
HTTP Request
    │
    ▼
[1. Extract Token]
    │ Parse Authorization: Bearer <token>
    │ Handle missing/malformed header → 401
    │
    ▼
[2. Decode Header]
    │ Base64-decode JWT header (no verification yet)
    │ Extract algorithm (RS256) and key ID (kid)
    │
    ▼
[3. Signature Verification]
    │ Fetch JWKS from issuer's /.well-known/jwks.json
    │ Cache JWKS with TTL (avoid hitting JWKS endpoint every request)
    │ Verify signature using public key matching kid
    │ Handle expired tokens, invalid signatures → 401
    │
    ▼
[4. Issuer Validation]
    │ Extract issuer from verified claims
    │ Check against environment-specific AllowedIssuers
    │ For v3 endpoints: additionally enforce v1.1+ issuers only
    │ Handle invalid issuer → 401
    │
    ▼
[5. Claims Extraction]
    │ Unmarshal into HPEClaims struct
    │ Detect token version from issuer/audience
    │ For v1.1+: extract workspace_id, principal_type
    │ For v1.0: extract customer_id, platform_customer_id
    │
    ▼
[6. Context Propagation]
    │ Set HPEClaims in request context
    │ Set tokenVersion in context
    │ Set workspaceID in context (for tenant isolation)
    │
    ▼
[Handler] — accesses claims via context, never re-parses token
```

### The `extractPlatformIdFromToken` Fix

**Before** (security gap I identified):

```go
// Controller was re-parsing the token independently
func (c *Controller) extractPlatformIdFromToken(r *http.Request) string {
    tokenStr := r.Header.Get("Authorization")
    tokenStr = strings.TrimPrefix(tokenStr, "Bearer ")
    
    // PROBLEM: This parses without verification!
    token, _ := jwt.Parse(tokenStr, nil)  // nil keyfunc = no signature check
    claims := token.Claims.(jwt.MapClaims)
    
    return claims["hpe_ccs_platform_customer_id"].(string)
}
```

**After** (my fix — use middleware context):

```go
// Controller reads from verified middleware context
func (c *Controller) extractPlatformIdFromToken(r *http.Request) string {
    claims := r.Context().Value(hpeClaimsKey).(HPEClaims)
    return claims.HPECCSPlatformCustID
}
```

**Why this was critical**: An attacker could craft a token with a valid signature (from a legitimate JWKS) but include a different `platform_customer_id` in a separate, unsigned portion. The middleware would verify the real signature and extract the real claims, but the controller's separate parse (without verification) could read manipulated claims. By using the middleware's verified claims exclusively, we eliminate this attack vector.

---

## Security Fix: GLCP-333624 (Cross-Workspace Case Creation)

### The Vulnerability

Manual case creation was using the `workspaceId` from the client's request body, not from the JWT token:

```go
// VULNERABLE: Client supplies workspace ID
wellnessObj.WorkspaceID = req.ManualCaseData.WorkspaceID  // Client-controlled!
```

An attacker with a valid token for Workspace A could create cases that appear to belong to Workspace B by manipulating the request body.

### My Fix

```go
// SECURE: Workspace ID from verified JWT claims
claims := r.Context().Value(hpeClaimsKey).(HPEClaims)
wellnessObj.WorkspaceID = claims.HPECCSWorkspaceID  // Server-controlled from token
```

**Additional enforcement** (GLCP-325645 cross-tenant):
- If the request body contains a `workspaceId`, it must match the token's workspace
- If they don't match, return 403 Forbidden
- Audit log the mismatch for security review

---

## Service Token Handling

```go
func isServiceToken(claims HPEClaims) bool {
    return claims.HPEPrincipalType == "api-client"
}

func (mw *AuthMiddleware) handleServiceToken(claims HPEClaims, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if isServiceToken(claims) {
            // Service tokens may not have all user claims
            // Set a service context flag for downstream handlers
            ctx := context.WithValue(r.Context(), isServiceKey, true)
            r = r.WithContext(ctx)
        }
        next.ServeHTTP(w, r)
    })
}
```

**Why service tokens matter**: Internal services (like schedulerService or alertProcessor) need to call wellnessProducer APIs. They use service tokens (`hpe_principal_type: api-client`) which may not have user-specific claims like `hpe_identity_type`. The middleware handles this gracefully rather than rejecting service-to-service calls.

---

## JWKS Caching

```go
type JWKSCache struct {
    keys      map[string]*rsa.PublicKey  // kid → public key
    fetchedAt time.Time
    ttl       time.Duration
    mu        sync.RWMutex
}

func (c *JWKSCache) GetKey(kid string) (*rsa.PublicKey, error) {
    c.mu.RLock()
    if time.Since(c.fetchedAt) < c.ttl {
        if key, ok := c.keys[kid]; ok {
            c.mu.RUnlock()
            return key, nil
        }
    }
    c.mu.RUnlock()
    
    // Cache miss or expired — refetch
    return c.refresh(kid)
}
```

**Why caching**: JWKS endpoints are external (HPE SSO). Without caching, every request would make an HTTP call to fetch public keys. With caching (TTL-based), we only refetch when keys rotate or cache expires. This also provides resilience — if the JWKS endpoint is temporarily down, cached keys still work.

---

## Integration with Istio

### RequestAuthentication (Mesh Level)

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: wellness-producer-auth
spec:
  selector:
    matchLabels:
      app: wellness-producer
  jwtRules:
    - issuer: "https://iss.greenlake.hpe.com/v1/{{ .Values.env }}"
      jwksUri: "https://iss.greenlake.hpe.com/v1/{{ .Values.env }}/.well-known/jwks.json"
      forwardOriginalToken: true
    - issuer: "https://sso.common.cloud.hpe.com"
      jwksUri: "https://sso.common.cloud.hpe.com/.well-known/jwks.json"
      forwardOriginalToken: true
```

### Two-Layer Validation

1. **Istio**: Validates JWT signature and issuer at the mesh level. Rejects obviously invalid tokens before they reach the application.
2. **Go Middleware**: Re-validates and extracts claims. Provides fine-grained control (v3 endpoint restrictions, claims extraction, context propagation).

**Why two layers**: Defense in depth. Istio catches invalid tokens before they consume application resources. Go middleware handles application-specific logic (version restrictions, claims mapping, context propagation) that Istio can't do.

---

## Testing Strategy

### Unit Tests

```go
func TestAuthMiddleware_V10Token(t *testing.T) {
    // Generate test JWT with v1.0 claims
    token := generateTestJWT(v10Claims, v10Issuer)
    req := httptest.NewRequest("POST", "/v1/wellness", nil)
    req.Header.Set("Authorization", "Bearer "+token)
    
    // Verify middleware accepts and extracts claims
    rr := httptest.NewRecorder()
    handler := authMiddleware(testHandler)
    handler.ServeHTTP(rr, req)
    
    assert.Equal(t, 200, rr.Code)
}

func TestAuthMiddleware_V11TokenOnV3Endpoint(t *testing.T) {
    // v1.1+ token on v3 endpoint — should work
    token := generateTestJWT(v11Claims, v11Issuer)
    // ...
}

func TestAuthMiddleware_V10TokenOnV3Endpoint(t *testing.T) {
    // v1.0 token on v3 endpoint — should be rejected
    token := generateTestJWT(v10Claims, v10Issuer)
    // Should return 401
}

func TestAuthMiddleware_CrossWorkspace(t *testing.T) {
    // Token for workspace A, request body claims workspace B
    // Should return 403
}
```

---

## [KEY_POINTS]

- Multi-version JWT support: v1.0 (arubathena.com) and v1.1+ (greenlake.hpe.com) tokens coexist
- Environment-based issuer validation replaced brittle regex — Helm values shared with Istio
- I fixed extractPlatformIdFromToken — controller was re-parsing tokens without verification
- GLCP-333624: workspace ID now comes from verified JWT, not client request body
- Two-layer validation (Istio + Go middleware) provides defense in depth

## [COMMON_MISTAKES]

- Thinking v1.0 tokens are "insecure" — they're valid tokens, just older with fewer claims
- Assuming Istio handles all auth — it validates signatures but doesn't extract/propagate HPE claims
- Overlooking service tokens — `api-client` principal type needs different handling than user tokens
- Using regex for issuer validation — exact match from config is more secure and maintainable

## [FOLLOW_UP]

- "How do you handle token rotation?" → JWKS caching with TTL, automatic refetch on cache miss
- "What if someone bypasses Istio?" → Go middleware is the authoritative auth layer, Istio is defense in depth
- "How do you test JWT auth without real HPE tokens?" → Test JWT generation with known keys, mock JWKS endpoint
- "Walk me through the GLCP-333624 fix" → Client-supplied workspace → JWT workspace extraction, 403 on mismatch
