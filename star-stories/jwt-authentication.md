# JWT Authentication Middleware Redesign for wellnessProducer

## STAR Narrative

### Situation
The RAVE Cloud Platform's wellnessProducer service — the central Go microservice handling case management for HPE GreenLake — had a fundamentally flawed JWT authentication middleware. The existing implementation used hardcoded regex patterns (`arubathena.com`, `common.cloud.hpe.com`) to validate token issuers, which meant tokens from **any environment** (dev, integration, production) were accepted everywhere. This was a critical security gap: a token minted in the dev environment could authenticate requests against the production API. Additionally, the middleware lacked support for HPE service tokens (machine-to-machine), meaning automated internal services couldn't authenticate properly. The team was also evolving the API from v1 to v2/v3, and v1.1+ tokens introduced dynamic issuer domains (`*.greenlake.hpe.com`) that the old regex couldn't handle. GLCP-333624 exposed a cross-workspace vulnerability tied to how workspace context was derived, further pressuring the auth layer.

### Task
I was responsible for redesigning the JWT authentication middleware end-to-end:
1. Replace hardcoded issuer regex with environment-based issuer validation (matching Istio `RequestAuthentication` per-environment config via Helm values).
2. Add HPE claims extraction — parse custom claims like `hpe_principal_type`, `hpe_identity_type`, `hpe_workspace_id` from the token and inject them into the request context.
3. Support service tokens (`hpe_principal_type: api-client`, `hpe_identity_type: service-identity`) so internal services could authenticate without user context.
4. Ensure the middleware closely followed the official `access_token_validation.md` algorithm with documented deviations.
5. Fix the workspace isolation vulnerability (GLCP-333624) where v2 Create Case API trusted client-supplied workspace identifiers rather than deriving them from the token.

### Action
**Step 1 — Issuer Validation Overhaul:**
I replaced the hardcoded regex with an environment-driven allowlist. Each deployment environment (dev/intg/prod) gets its own set of trusted issuer domains injected via Helm values into the service config. This aligned with how Istio's `RequestAuthentication` already validated JWTs at the mesh level, creating defense-in-depth — even if Istio was misconfigured, the application-level middleware would reject cross-environment tokens.

**Step 2 — HPE Claims Struct and Context Helpers:**
I created a typed `HPEClaims` struct in Go to deserialize HPE-specific JWT claims:
```go
type HPEClaims struct {
    PrincipalType  string `json:"hpe_principal_type"`
    IdentityType   string `json:"hpe_identity_type"`
    WorkspaceID    string `json:"hpe_workspace_id"`
    TenantID       string `json:"hpe_tenant_id"`
}
```
I added context helper functions (`GetHPEClaimsFromContext`, `SetHPEClaimsInContext`) so downstream handlers could access structured claims without re-parsing the token. Previously, the controller was separately re-parsing the token (unverified) in `extractPlatformIdFromToken` — I eliminated this duplication.

**Step 3 — Service Token Support:**
For service-to-service authentication, I added a code path that recognizes tokens with `hpe_principal_type: api-client` and `hpe_identity_type: service-identity`. These tokens don't carry user-specific claims (no `sub` with a human identity), so I ensured handlers could check the caller type and branch logic accordingly.

**Step 4 — Dynamic Issuer Domains:**
For v1.1+ tokens, I added `*.greenlake.hpe.com` to the trusted issuer pattern, handling the new dynamic issuer domains that HPE's identity platform was rolling out.

**Step 5 — Workspace Isolation Fix (GLCP-333624):**
The v2 Create Case API accepted a client-supplied `workspace_id` in the request body and used it directly. I changed the API to derive workspace context exclusively from the authenticated token's `hpe_workspace_id` claim, making the client-supplied value ignored for authorization purposes. This was the critical tenant isolation fix.

**Step 6 — Testing and Validation:**
I wrote unit tests covering: valid tokens per environment, cross-environment rejection, service token authentication, missing claims handling, and the workspace derivation path.

### Result
- **Eliminated cross-environment token acceptance** — dev tokens can no longer authenticate against production.
- **Fixed GLCP-333624** — closed the cross-workspace tenant isolation vulnerability, preventing unauthorized case creation across workspaces.
- **Enabled service token auth** — unblocked 3 internal services from integrating with wellnessProducer programmatically.
- **Removed token re-parsing** — eliminated the unverified `extractPlatformIdFromToken` code path, reducing attack surface.
- **Auth middleware now follows `access_token_validation.md`** algorithm with documented deviations, making security audits straightforward.
- Changes deployed across dev → intg → prod with zero authentication-related incidents post-deployment.

## Interview Delivery Tips

### How to Open This Story
"I redesigned the JWT authentication middleware for our core case management microservice. The existing system had a critical flaw — it accepted tokens from any environment because of hardcoded issuer validation. I also discovered and fixed a cross-workspace tenant isolation vulnerability where the API trusted client-supplied workspace identifiers."

### Time Budget (5-minute answer)
- Situation: 45 seconds (old system was insecure, why)
- Task: 30 seconds (5 responsibilities)
- Action: 2.5 minutes (focus on issuer validation + GLCP-333624 fix)
- Result: 1 minute (zero incidents, enabled service tokens, eliminated re-parsing)

### Pivot Points
- **If interviewer asks about security**: Lean into GLCP-333624, trust boundaries, defense-in-depth
- **If interviewer asks about Go**: Discuss context.Context pattern, typed claims struct, middleware chain
- **If interviewer asks about distributed systems**: Focus on Istio + app layer, JWKS caching, environment isolation
- **If interviewer asks about collaboration**: Mention security team coordination, documentation of `access_token_validation.md` adherence

---

## Rubric Annotations

### Key Phrases/Concepts That MUST Appear
- Environment-based issuer validation (not hardcoded regex)
- HPE claims extraction from JWT (`hpe_principal_type`, `hpe_workspace_id`)
- Service token support (machine-to-machine, `api-client` / `service-identity`)
- Workspace isolation from token (GLCP-333624)
- Defense-in-depth (Istio + application-level validation)
- Context injection pattern (Go `context.Context` with typed claims)
- Elimination of unverified token re-parsing

### Common Mistakes
- Describing JWT validation without mentioning JWKS or signature verification
- Conflating authentication (who are you?) with authorization (what can you do?)
- Not explaining WHY hardcoded issuers were dangerous (cross-environment acceptance)
- Skipping the tenant isolation aspect — this is the highest-impact part
- Forgetting to mention defense-in-depth (Istio mesh + app middleware)

### Scoring Rubric [1-5]
- 1 — Poor: Cannot explain JWT structure or why the old approach was insecure
- 2 — Below Average: Explains the change but lacks security reasoning; no mention of tenant isolation
- 3 — Acceptable: Covers issuer validation fix and service tokens but light on GLCP-333624 and defense-in-depth
- 4 — Strong: Full narrative covering all 5 action items, explains security implications, mentions Istio alignment
- 5 — Exceptional: All of the above plus articulates trade-offs (e.g., why not use Istio alone), discusses token refresh implications, explains the `access_token_validation.md` algorithm adherence

---

## Follow-Up Questions (Progressively Harder)

### Follow-Up 1: How does the JWT signature verification work in your middleware? What key material do you use?
**Expected Answer**: The middleware fetches the public key from the JWKS (JSON Web Key Set) endpoint exposed by the identity provider. The `kid` (key ID) in the JWT header is matched against the JWKS to find the correct public key. We use RS256 (RSA with SHA-256) for signature verification. The JWKS is cached with a TTL to avoid hitting the identity provider on every request. If a `kid` is not found in the cache, we refresh the JWKS once before rejecting.
**Red Flags**: Says "we decode the token and check the signature with a shared secret" (confuses RS256 with HS256); doesn't mention JWKS; says they "just trust the token if it's not expired."
**Green Flags**: Mentions JWKS caching, `kid` lookup, RS256, and the refresh-on-miss strategy.

### Follow-Up 1b: What is the difference between a JWT and an opaque token? When would you use each?
**Expected Answer**: A JWT is self-contained — the token itself contains all the claims needed for authorization. The server can validate it locally using the public key (no round-trip). An opaque token is a random string that must be sent to the authorization server's introspection endpoint to get the associated claims. JWTs are faster (no network call) but can't be revoked instantly (valid until expiration). Opaque tokens can be revoked immediately but require an introspection call on every request. We use JWTs because our service needs to validate thousands of requests per second, and the introspection latency would be unacceptable. The trade-off is that compromised tokens remain valid until they expire.
**Red Flags**: Doesn't know what an opaque token is; says JWTs can be revoked.
**Green Flags**: Explains the trade-off between stateless validation (JWT) and centralized validation (opaque), mentions revocation limitations.

### Follow-Up 2: Why not just rely on Istio's RequestAuthentication for JWT validation? Why add application-level validation too?
**Expected Answer**: Istio `RequestAuthentication` provides mesh-level JWT validation — it rejects tokens with invalid signatures or untrusted issuers before they reach the pod. However, Istio operates at the transport layer and doesn't inject parsed claims into the application context. The app needs to extract HPE-specific claims (`hpe_workspace_id`, `hpe_principal_type`) for business logic. Additionally, defense-in-depth means if Istio is misconfigured or bypassed (e.g., during local development, or a mesh bug), the application still validates. The two layers serve different purposes: Istio gates access, the middleware enriches context.
**Red Flags**: Says Istio handles everything, no need for app-level validation.
**Green Flags**: Articulates defense-in-depth, explains that Istio doesn't provide claim extraction for business logic.

### Follow-Up 3: How did you handle the transition from hardcoded issuers to environment-based ones without breaking existing deployments?
**Expected Answer**: The Helm values were structured so each environment's deployment got the correct issuer allowlist. During the rollout, I maintained backward compatibility by temporarily including both the old issuers and the new environment-specific ones, then removed the old ones after confirming all tokens in each environment used the correct issuer. The Istio `RequestAuthentication` was already per-environment, so I aligned the application config to match. Deployment was staged: dev first, then intg, then prod, with monitoring for 401 spikes at each stage.
**Red Flags**: Doesn't consider migration strategy; says "we just changed it everywhere at once."
**Green Flags**: Staged rollout, backward compatibility period, monitoring for auth failures.

---

## Grilling Chain (7 questions drilling deeper)

### Q1: Walk me through the three parts of a JWT. What's in each?
**Why asked**: Tests fundamental JWT knowledge — do they actually understand the token format?
**Expected answer**: Header (algorithm `alg` and key ID `kid`, base64url-encoded JSON), Payload (claims — `iss`, `sub`, `exp`, `iat`, plus custom claims like `hpe_workspace_id`), Signature (cryptographic signature over header + payload using the algorithm specified in the header).
**Red flags**: Can't name all three parts; confuses encoding with encryption; says JWT is encrypted.
**Green flags**: Mentions base64url (not base64), distinguishes encoding from signing, names standard claims.

### Q2: Explain the difference between RS256 and HS256. Why does it matter which one you use?
**Why asked**: Tests understanding of asymmetric vs symmetric signing — critical for security design.
**Expected answer**: HS256 uses a shared secret (HMAC-SHA256) — both the issuer and verifier must have the same secret. RS256 uses asymmetric RSA keys — the issuer signs with a private key, and verifiers use the public key. RS256 is preferred in distributed systems because the verifier never needs access to the private key. HS256 requires secure secret distribution and is risky if any verifier is compromised.
**Red flags**: Can't explain the difference; says "RS256 is just more secure" without explaining why.
**Green flags**: Explains asymmetric key distribution advantage, mentions JWKS as the public key distribution mechanism.

### Q3: What is a JWKS endpoint and how does key rotation work?
**Why asked**: Tests operational understanding of JWT infrastructure.
**Expected answer**: JWKS (JSON Web Key Set) is an endpoint (usually `/.well-known/jwks.json`) that publishes the identity provider's current public keys. Each key has a `kid` (key ID). During rotation, the provider adds a new key to the JWKS before removing the old one (overlap period). Tokens signed with the old key remain valid until they expire. The `kid` in the JWT header tells the verifier which key to use. Verifiers should cache JWKS but refresh on cache miss (unknown `kid`).
**Red flags**: Doesn't know what JWKS is; says "we just hardcode the public key."
**Green flags**: Explains rotation overlap, cache-and-refresh strategy, `kid` matching.

### Q4: Your middleware extracts `hpe_workspace_id` from the token for authorization. What if an attacker crafts a token with a different `hpe_workspace_id`?
**Why asked**: Tests understanding of token integrity — can claims be tampered with?
**Expected answer**: They can't. The JWT signature covers the entire payload, including all claims. If an attacker modifies `hpe_workspace_id`, the signature verification will fail because the signature was computed over the original payload. The attacker would need the identity provider's private key to forge a valid signature.
**Red flags**: Suggests additional validation of claims against an external source; doesn't understand that signature protects claim integrity.
**Green flags**: Immediately points to signature integrity; explains why claim tampering is impossible without the private key.

### Q5: How do you handle token expiration and refresh? What happens when a token expires mid-session?
**Why asked**: Tests understanding of token lifecycle in practice.
**Expected answer**: The middleware checks the `exp` claim and rejects expired tokens with 401. Token refresh is handled client-side — the client uses a refresh token to obtain a new access token from the identity provider before the current one expires. The middleware doesn't handle refresh itself; it's stateless and only validates the presented token. For long-running operations, the client should proactively refresh. We include a small clock skew tolerance (e.g., 30 seconds) to handle minor time drift between servers.
**Red flags**: Says the middleware refreshes tokens; doesn't know about refresh tokens.
**Green flags**: Mentions clock skew tolerance, stateless middleware design, client-side refresh responsibility.

### Q6: You mentioned service tokens. How do you prevent a service token from being used by an unauthorized service?
**Why asked**: Tests understanding of service identity and authorization scope.
**Expected answer**: Service tokens are issued to specific service accounts registered with the identity provider. Each service has its own client credentials (client ID + secret). The token's `sub` or `client_id` claim identifies the service. We can add scope-based or audience-based restrictions — the token's `aud` (audience) claim must match the target service. Additionally, network-level controls (Istio `AuthorizationPolicy`, mTLS between services) restrict which services can reach which endpoints. The service token itself is just authentication; authorization is enforced separately.
**Red flags**: Says "any service with a valid token can call any API."
**Green flags**: Mentions audience claims, scope restrictions, network-level controls (mTLS, Istio policies).

### Q7: If you were designing this system from scratch today, what would you do differently?
**Why asked**: Tests ability to reflect, identify improvements, and think beyond the current implementation.
**Expected answer**: I would: (1) Use an OPA (Open Policy Agent) sidecar for policy-as-code authorization instead of embedding auth logic in the middleware. (2) Implement a token introspection cache rather than validating JWTs directly, giving the identity provider the ability to revoke tokens. (3) Add structured audit logging for every auth decision (who, what, when, result) for compliance. (4) Consider PASETO as an alternative to JWT for its simpler security model. (5) Design the claims schema upfront with versioning, rather than evolving it reactively.
**Red flags**: Says "nothing, it's perfect"; can't think of improvements.
**Green flags**: Mentions policy-as-code, token revocation challenges, audit logging, alternative token formats.

---

## Tags & Cross-References

### Related STAR Stories
- [Cross-Workspace Security](cross-workspace-security.md) — GLCP-333624 is the vulnerability this auth work enabled fixing
- [API Versioning Evolution](api-versioning-evolution.md) — v3 API enforces the v1.1+ tokens designed here
- [Multi-Environment Infra](multi-environment-infra.md) — environment-based issuer validation uses the same Helm values

### Interview Question Categories This Covers
- System Design: Middleware architecture, defense-in-depth
- Security: JWT, JWKS, token validation, tenant isolation
- Go Programming: context.Context patterns, typed structs, middleware chains
- DevOps: Istio integration, Helm-driven configuration, staged deployment

### Behavioral Dimensions Demonstrated
- **Technical depth**: JWT internals, JWKS, RS256, claims extraction
- **Security mindset**: Identified cross-environment risk, workspace isolation gap
- **Ownership**: End-to-end redesign, not just a patch
- **Collaboration**: Aligned with Istio config, security team, access_token_validation.md
