# API Versioning: v1 → v2 → v3 Evolution — RAVE Platform

## Overview

I drove the API versioning evolution in RAVE's wellnessProducer service from v1 through v3. Each version added capabilities while maintaining backward compatibility with existing integrations. The v3 release was particularly significant — it enforced v1.1+ authentication tokens with dynamic issuers, marking the transition from legacy arubathena.com-based auth to HPE GreenLake's native token ecosystem. This required coordination between Go middleware, Istio EnvoyFilter configuration, and Helm-based environment management.

---

## Version Overview

| Version | Auth | Key Features | Status |
|---------|------|-------------|--------|
| **v1** | v1.0 + v1.1+ tokens | Basic wellness CRUD, automated alerts | Legacy (maintained) |
| **v2** | v1.0 + v1.1+ tokens | Manual case support, attachments, enhanced validation | Active |
| **v3** | **v1.1+ tokens only** | Dynamic issuers, strict validation, workspace isolation | Current |

---

## v1: Legacy Endpoints

### Endpoints

```
POST   /v1/wellness          — Create wellness object
GET    /v1/wellness/:id      — Get by ID
GET    /v1/wellness           — List with filters
PUT    /v1/wellness/:id      — Update
DELETE /v1/wellness/:id      — Delete
```

### Characteristics

- **Auth**: Accepts both v1.0 (`arubathena.com`) and v1.1+ (`greenlake.hpe.com`) tokens
- **Use case**: Automated alert pipeline — dataParser and alertProcessor create wellness objects
- **Validation**: Basic required field checks (deviceId, alertType)
- **Tenant identification**: `customer_id` from v1.0 tokens or `hpe_ccs_platform_customer_id` from v1.1+

### Why v1 Still Exists

Legacy integrations and internal services still use v1.0 tokens. Forcing a migration to v3 for automated alert ingestion would require updating every data source and internal service simultaneously — a risky big-bang migration. Instead, v1 continues to serve automated flows while v2/v3 handle user-facing manual cases.

---

## v2: Manual Case Support

### New Endpoints

```
POST   /v2/wellness          — Create wellness object (with ManualCaseData support)
GET    /v2/wellness/:id      — Get by ID (enhanced response)
PUT    /v2/wellness/:id      — Update (with case status transitions)
POST   /v2/wellness/upload   — Upload attachment (NEW)
```

### What Changed from v1

1. **ManualCaseData field**: New optional nested object for manual case creation
2. **Attachment uploads**: New `/upload` endpoint with file validation pipeline
3. **Contact handling**: Primary and alternate contact support
4. **Domain routing**: Cases routed to email (SendGrid) or panhpe (DCM CRM) based on domain field
5. **Enhanced validation**: Category validation, email format checking

### Auth Behavior

- Still accepts both v1.0 and v1.1+ tokens
- For v1.0 tokens: Manual case creation works but `workspaceId` derived from `customer_id` mapping
- For v1.1+ tokens: Full HPE claims available, workspace ID from `hpe_ccs_workspace_id`

### Migration from v1 → v2

```
v1 clients: No changes required, continue using /v1/ endpoints
v2 clients: Use /v2/ endpoints for manual case features
```

No breaking changes — v2 is a superset of v1 functionality.

---

## v3: Current — v1.1+ Auth Enforcement (GLCP-334752)

### Endpoints

```
POST   /v3/wellness          — Create wellness object (v1.1+ tokens REQUIRED)
GET    /v3/wellness/:id      — Get by ID
PUT    /v3/wellness/:id      — Update
POST   /v3/wellness/upload   — Upload attachment
GET    /v3/wellness/categories — Get allowed categories (NEW)
```

### What Changed from v2

1. **Auth enforcement**: v1.1+ tokens ONLY — v1.0 tokens rejected with 401
2. **Dynamic issuers**: Environment-specific issuer validation (not hardcoded regex)
3. **Workspace isolation**: `workspaceId` always from JWT, never from request body (GLCP-333624)
4. **createdBy from JWT**: `sub` claim sets the case creator (GLCP-333624)
5. **Category endpoint**: New endpoint to fetch allowed categories dynamically
6. **Stricter validation**: All ManualCaseData fields validated more thoroughly

### Why v3 Requires v1.1+ Tokens

1. **Security**: v1.1+ tokens have `hpe_ccs_workspace_id` which enables server-side tenant isolation
2. **Claims richness**: v1.1+ provides `hpe_principal_type`, `hpe_identity_type` for authorization
3. **Issuer trust**: v1.1+ issuers are environment-specific, preventing token replay across environments
4. **Platform alignment**: HPE is migrating all GLCP services to v1.1+ — v3 aligns with this direction

---

## Dynamic Issuer Validation (v3)

### The Problem

v1 and v2 used a hardcoded regex to validate issuers:

```go
// Hard to maintain, error-prone, environment-unaware
var issuerRegex = regexp.MustCompile(`^https://(sso\.common\.cloud\.hpe\.com|iss\.greenlake\.hpe\.com/.*)$`)
```

### My Solution

Environment-specific configuration loaded from Helm values:

```yaml
# values-dev.yaml
auth:
  v3_allowed_issuers:
    - "https://iss.greenlake.hpe.com/v1/dev"
    
# values-intg.yaml
auth:
  v3_allowed_issuers:
    - "https://iss.greenlake.hpe.com/v1/intg"

# values-prod.yaml
auth:
  v3_allowed_issuers:
    - "https://iss.greenlake.hpe.com/v1/prod"
    - "https://iss.greenlake.hpe.com/v1/prod-us-west"  # Multi-region
```

```go
type V3AuthConfig struct {
    AllowedIssuers []string
}

func (mw *AuthMiddleware) validateV3Token(claims HPEClaims) error {
    // Must be v1.1+ (has HPE claims)
    if claims.HPECCSWorkspaceID == "" {
        return ErrV3RequiresV11Token
    }
    
    // Issuer must be in environment-specific allowlist
    for _, allowed := range mw.v3Config.AllowedIssuers {
        if claims.Issuer == allowed {
            return nil
        }
    }
    
    return fmt.Errorf("%w: issuer %s not in allowed list", ErrInvalidIssuer, claims.Issuer)
}
```

**Why this matters**: These Helm values are the same source of truth used by Istio RequestAuthentication. The Go middleware and Istio mesh never disagree on which issuers are valid. Adding a new region or environment is a Helm values change, not a code change.

---

## Istio EnvoyFilter Configuration

### v3 Route Enforcement

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: v3-auth-enforcement
spec:
  workloadSelector:
    labels:
      app: wellness-producer
  configPatches:
    - applyTo: HTTP_ROUTE
      match:
        context: SIDECAR_INBOUND
        routeConfiguration:
          vhost:
            route:
              name: "v3-routes"
      patch:
        operation: MERGE
        value:
          route:
            metadata:
              filter_metadata:
                envoy.filters.http.jwt_authn:
                  require_any:
                    requirements:
                      - provider_name: "hpe-glcp-v11"
                        # Only v1.1+ provider, not the legacy v1.0 provider
```

### How v3 Auth Works End-to-End

```
Client sends request with JWT
    │
    ▼
[Istio Sidecar]
    │ EnvoyFilter: /v3/* routes require v1.1+ JWT provider
    │ RequestAuthentication: Validates signature against v1.1+ JWKS
    │ If v1.0 token on v3 route → 401 at mesh level (never reaches app)
    │
    ▼
[Go Middleware]
    │ Additional validation: workspace claims present, issuer in allowlist
    │ Extract HPEClaims with full workspace context
    │
    ▼
[Handler]
    │ Uses verified claims from context
```

**Why both Istio and Go validation**: Istio rejects obviously wrong tokens (v1.0 on v3) before they consume application resources. Go middleware handles the nuanced checks (specific issuer validation, claims extraction) that Istio can't express.

---

## Backward Compatibility Strategy

### URL-Based Versioning

```
/v1/wellness    → v1 handler chain (v1.0 + v1.1+ auth)
/v2/wellness    → v2 handler chain (v1.0 + v1.1+ auth, manual case support)
/v3/wellness    → v3 handler chain (v1.1+ only, strict validation)
```

**Why URL versioning over header versioning**: URL-based versioning is explicit, cacheable, and debuggable. You can see the version in access logs, load balancer rules, and Istio routing. Header-based versioning (`Accept: application/vnd.hpe.v3+json`) is harder to debug and route at the infrastructure level.

### Shared Business Logic

```go
// Internal handler shared across versions
func (c *Controller) createWellnessObject(ctx context.Context, model WellnessModel, version APIVersion) error {
    // Common logic: persist, publish to Kafka
    
    // Version-specific behavior
    switch version {
    case V1:
        // Minimal validation, no manual case support
    case V2:
        // Manual case validation, contact handling
    case V3:
        // Strict validation, workspace from JWT, createdBy from JWT
        model.WorkspaceID = ctx.Value(hpeClaimsKey).(HPEClaims).HPECCSWorkspaceID
        model.CreatedBy = ctx.Value(hpeClaimsKey).(HPEClaims).Subject
    }
    
    return c.repo.Create(ctx, model)
}
```

**Why shared logic**: Prevents divergence between versions. Core persistence and Kafka publishing logic is the same across all versions. Only auth and validation differ.

### Deprecation Path

```
v1: Active (automated flows) → Future: Migrate to v2/v3 when internal services update tokens
v2: Active (manual cases with v1.0 tokens) → Future: Deprecate when v1.0 tokens sunset
v3: Current (all new development) → Standard version going forward
```

No version is removed while clients still use it. Monitoring tracks traffic per version to inform deprecation timelines.

---

## Migration Path

### For Automated Alert Producers (v1 → v2)

1. No auth changes required (both accept v1.0 + v1.1+)
2. Update URL from `/v1/wellness` to `/v2/wellness`
3. Optional: Add ManualCaseData for enhanced features
4. Test in dev/intg environments

### For GLCP UI (v2 → v3)

1. **Auth migration**: Ensure UI uses v1.1+ tokens (greenlake.hpe.com SSO)
2. Update URL from `/v2/wellness` to `/v3/wellness`
3. Remove `workspaceId` from request body (now from JWT)
4. Remove `createdBy` from request body (now from JWT)
5. Test workspace isolation (GLCP-333624 fix)

### Version Detection in Middleware

```go
func detectAPIVersion(r *http.Request) APIVersion {
    path := r.URL.Path
    switch {
    case strings.HasPrefix(path, "/v3/"):
        return V3
    case strings.HasPrefix(path, "/v2/"):
        return V2
    default:
        return V1
    }
}
```

---

## Version-Specific Response Differences

### v1 Response

```json
{
  "id": "wellness-obj-id",
  "deviceId": "device-123",
  "alertType": "disk-error",
  "status": "created"
}
```

### v2 Response (includes manual case data)

```json
{
  "id": "wellness-obj-id",
  "deviceId": "device-123",
  "alertType": "manual",
  "status": "caseCreating",
  "manualCaseData": {
    "domain": "email",
    "category": "hardware-failure",
    "caseId": "pending"
  }
}
```

### v3 Response (includes workspace context)

```json
{
  "id": "wellness-obj-id",
  "deviceId": "device-123",
  "alertType": "manual",
  "status": "caseCreating",
  "workspaceId": "workspace-uuid",
  "createdBy": "user@hpe.com",
  "manualCaseData": {
    "domain": "email",
    "category": "hardware-failure",
    "caseId": "pending"
  }
}
```

---

## Related Jira Issues

| Issue | Description | Version Impact |
|-------|-------------|---------------|
| GLCP-334752 | v3 API implementation | v3 creation |
| GLCP-335836 | API modifications | v2/v3 validation |
| GLCP-333624 | Cross-workspace fix | v3 workspace from JWT |
| GLCP-330407 | Token acceptance | v1/v2/v3 auth matrix |
| GLCP-325722 | Input validation | v2/v3 enhanced checks |

---

### v2beta1 glp:manual Event Exclusion

**Requirement**: Documents with `origin: "glp:manual"` must be excluded from all v2beta1 wellness API responses, without affecting v2 or v2beta2.

**Approach**: DB-level filter injection in v2beta1-exclusive methods.
- Key insight: all 6 DB methods called by v2beta1 read endpoints are exclusive to v2beta1 — none shared with v2/v2beta2
- Added `origin != "glp:manual"` filter directly in these DB methods
- Safe modification with zero cross-version impact

### OData `in` Operator for caseNumber Filter

**Requirement**: Platform Co-pilot needs to fetch groups of support cases by multiple case IDs in one request.
- `in` operator was already fully supported in OData infrastructure (`tree.In` in `supportedOperators`)
- Maps to MongoDB `$in` via `inOperatorFilter()` which handles collections
- Only needed to add the mapping in the filter configuration — no new code required
- Updated OpenAPI documentation accordingly

---

## [KEY_POINTS]

- URL-based versioning: /v1/, /v2/, /v3/ with explicit routing
- v3 is the security milestone — v1.1+ tokens only, workspace from JWT, dynamic issuers
- Backward compatibility maintained — v1 and v2 continue serving existing clients
- Istio EnvoyFilter enforces v3 auth at mesh level before request reaches application
- Shared business logic across versions prevents code divergence

## [COMMON_MISTAKES]

- Thinking v3 "replaces" v1/v2 — all versions coexist, each serving different client needs
- Assuming version differences are just feature additions — v3 also restricts (removes v1.0 token support)
- Overlooking Istio's role in v3 auth — EnvoyFilter rejects v1.0 tokens at the mesh level
- Forgetting the dynamic issuer change — this was a major architectural shift from regex to config

## [FOLLOW_UP]

- "Why not just force everyone to v3?" → Internal services still use v1.0 tokens, big-bang migration is risky
- "How do you monitor version adoption?" → Per-version traffic metrics in Prometheus/Grafana
- "What happens when a v1.0 token hits a v3 endpoint?" → 401 from Istio (never reaches Go app)
- "How do you add a new API version?" → New route prefix, version-specific middleware chain, shared controller logic
