# API Versioning Evolution: wellnessProducer v1 → v2 → v3

## STAR Narrative

### Situation
The wellnessProducer service — the core Go microservice for HPE GreenLake's RAVE case management platform — had evolved through multiple API versions as security requirements, authentication mechanisms, and client needs changed. The original v1 API was designed with basic authentication and minimal authorization. As HPE transitioned to a more sophisticated identity platform, v1.1 tokens introduced dynamic issuer domains (`*.greenlake.hpe.com`), and the v2 API added enhanced capabilities but also exposed the cross-workspace vulnerability (GLCP-333624). By the time I joined the API evolution effort, there were clients on v1, v1.1, and v2, and we needed a v3 that enforced stricter security while maintaining backward compatibility for existing integrations.

### Task
I was responsible for:
1. Designing and implementing the v3 API that enforced v1.1+ auth tokens (rejecting older token formats).
2. Configuring Istio EnvoyFilter to route requests to the correct API version based on the URL path.
3. Maintaining backward compatibility for v1 and v2 clients during the transition period.
4. Ensuring the v3 API incorporated all security fixes (GLCP-333624 workspace isolation, environment-based issuer validation).
5. Documenting the deprecation timeline and migration path for v1/v2 consumers.

### Action
**Step 1 — Version Strategy Selection:**
I evaluated three API versioning approaches:
- **URL path versioning** (`/v1/cases`, `/v2/cases`, `/v3/cases`) — selected.
- **Header versioning** (`Accept: application/vnd.hpe.rave.v3+json`) — rejected because it's harder to debug, route, and cache.
- **Content negotiation** — rejected due to complexity with existing Istio routing.

URL path versioning was already established (v1, v2 existed), so extending to v3 was natural and required no client SDK changes for version selection.

**Step 2 — v3 API Design:**
The v3 API enforced:
1. **v1.1+ token requirement**: Tokens must have the new HPE claims structure (`hpe_principal_type`, `hpe_workspace_id`, etc.). Tokens lacking these claims (old v1.0 format) are rejected with `401 Unauthorized` and a descriptive error message guiding the client to upgrade.
2. **Server-side workspace derivation**: Workspace context is extracted from `hpe_workspace_id` in the token — the request body's workspace field is ignored for authorization (GLCP-333624 fix).
3. **Environment-based issuer validation**: Only tokens from the deployment environment's identity provider are accepted.
4. **Service token support**: Machine-to-machine tokens with `hpe_principal_type: api-client` are supported for internal integrations.

**Step 3 — Istio EnvoyFilter Configuration:**
I configured Istio to route versioned API traffic:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: wellness-api-routing
spec:
  workloadSelector:
    labels:
      app: wellnessProducer
  configPatches:
    - applyTo: HTTP_ROUTE
      match:
        context: SIDECAR_INBOUND
      patch:
        operation: MERGE
        value:
          route:
            prefix_rewrite: /
```

The EnvoyFilter handles path-based routing, ensuring that `/v3/cases` requests reach the v3 handler while `/v1/cases` and `/v2/cases` continue reaching their respective handlers. Istio's `RequestAuthentication` resource was configured per-version — v3 only accepts issuers from the current deployment environment.

**Step 4 — Backward Compatibility:**
I maintained backward compatibility through:
1. **v1 API**: Unchanged — continues to work with original token format. Marked as deprecated.
2. **v2 API**: Unchanged but patched with the GLCP-333624 workspace fix — this was a security fix, not a breaking change, so it was applied retroactively.
3. **v3 API**: New strict requirements. Clients must migrate to v1.1+ tokens.
4. **Shared business logic**: All three versions share the same underlying service layer — only the middleware chain (auth, validation) differs. This prevents logic divergence.

**Step 5 — Deprecation Communication:**
I created a deprecation plan:
- v1: Deprecated immediately, sunset in 6 months. Responses include `Sunset` header and `Deprecation` header per RFC 8594.
- v2: Supported with security patches only. New features are v3-only.
- v3: Current active version, all new development.

I documented the migration path in Confluence: what changes between versions, how to obtain v1.1+ tokens, and a checklist for API consumers.

**Step 6 — Testing Strategy:**
I wrote integration tests covering:
- v3 endpoint with v1.0 token → 401 with upgrade message.
- v3 endpoint with v1.1+ token → 200 with correct workspace derivation.
- v2 endpoint with any valid token → 200 (backward compatible).
- v3 endpoint with service token → 200 with service identity context.
- Cross-version regression: creating a case on v3 and retrieving it on v2 (shared data layer).

### Result
- **v3 API deployed** with enforced v1.1+ tokens, closing the authentication gap.
- **GLCP-333624 fix** retroactively applied to v2 and natively incorporated in v3.
- **Zero breaking changes** for v1/v2 clients — they continued operating during the transition.
- **Istio routing** cleanly separates version traffic at the mesh level.
- **3 internal services** migrated to v3 within the first sprint; external consumers given 6-month deprecation window.
- **Shared service layer** prevented logic divergence — a bug fixed in the service layer is fixed for all versions.

## Interview Delivery Tips

### How to Open This Story
"I evolved our case management API from v1 through v3, where each version introduced stricter security requirements. v3 enforced modern HPE token formats and incorporated the workspace isolation fix. I configured Istio EnvoyFilter for version routing while maintaining backward compatibility."

### Time Budget (5-minute answer)
- Situation: 30 seconds (evolving auth requirements, cross-version clients)
- Task: 30 seconds (5 responsibilities)
- Action: 2.5 minutes (version strategy + Istio routing + shared service layer)
- Result: 1.5 minutes (zero breaking changes, 3 services migrated first sprint)

### Pivot Points
- **If interviewer asks about API design**: Versioning strategies, backward compatibility, deprecation
- **If interviewer asks about infrastructure**: Istio EnvoyFilter, RequestAuthentication, service mesh
- **If interviewer asks about code architecture**: Handler/middleware per version, shared service layer
- **If interviewer asks about process**: Deprecation headers (RFC 8594), sunset timeline, client outreach

---

## Rubric Annotations

### Key Phrases/Concepts That MUST Appear
- URL path versioning (`/v1/`, `/v2/`, `/v3/`)
- v1.1+ token enforcement on v3
- Istio EnvoyFilter for version routing
- Backward compatibility with shared service layer
- Deprecation headers (Sunset, Deprecation per RFC 8594)
- Server-side workspace derivation (GLCP-333624)
- Middleware chain per version, shared business logic

### Common Mistakes
- Describing versioning without explaining the trade-offs of different strategies
- Not explaining how backward compatibility is maintained (shared service layer)
- Forgetting the Istio routing layer — this is infrastructure, not just code
- Treating versioning as purely a routing problem, ignoring authentication differences
- Not having a deprecation strategy

### Scoring Rubric [1-5]
- 1 — Poor: Cannot explain API versioning strategies or why versioning was needed
- 2 — Below Average: Describes URL versioning but lacks backward compatibility and Istio details
- 3 — Acceptable: Covers versioning, backward compatibility, and basic routing but light on deprecation and testing
- 4 — Strong: Full narrative with Istio config, shared service layer, deprecation plan, and testing
- 5 — Exceptional: All of the above plus discusses trade-offs between versioning strategies, RFC 8594, and long-term API evolution philosophy

---

## Follow-Up Questions (Progressively Harder)

### Follow-Up 1: What are the pros and cons of URL path versioning vs. header-based versioning?
**Expected Answer**: URL path: **Pros** — simple, visible, cacheable (CDN can cache per path), easy to route (Istio, nginx), easy to test (just change the URL). **Cons** — pollutes the URL space, harder to express minor versions, each version looks like a different resource. Header-based: **Pros** — clean URLs, more REST-pure (same resource, different representation), supports fine-grained versioning. **Cons** — harder to test (need to set headers), harder to route at infrastructure level, breaks simple browser testing, caching is more complex (Vary header). For our system, URL path won because Istio routing is path-based and our clients are programmatic (not browser-based).
**Red Flags**: Says one approach is always better; doesn't mention caching or routing implications.
**Green Flags**: Discusses both approaches with concrete trade-offs, explains why URL path was chosen for this specific context.

### Follow-Up 2: How do you handle a breaking change within a version? For example, what if v3 needs an incompatible schema change?
**Expected Answer**: Options: (1) Create v4 — if the change is significant. (2) Use additive changes — add new fields but don't remove or rename old ones. Make new fields optional. (3) Feature flags — use a request header or query parameter to opt into the new behavior within v3. (4) Content negotiation within the version — `Accept: application/vnd.hpe.rave.v3.1+json`. (5) For our system, we follow the "expand and contract" pattern: add the new field, support both old and new for a migration period, then remove the old field in the next major version.
**Red Flags**: "Just change it and tell clients to update."
**Green Flags**: Mentions expand-and-contract, additive changes, and migration periods.

### Follow-Up 3: How does Istio's EnvoyFilter work under the hood? What's actually happening at the proxy level?
**Expected Answer**: Istio injects an Envoy sidecar proxy into every pod. EnvoyFilter patches the Envoy configuration at specific points in the filter chain. For our routing: the Envoy listener receives the inbound HTTP request, the HTTP Connection Manager parses the URL path, and our EnvoyFilter patches the route configuration to rewrite or match paths based on the version prefix. Under the hood, Envoy uses a trie-based route matching structure. The filter applies at `SIDECAR_INBOUND` context, meaning it intercepts traffic arriving at the pod's sidecar. The config is pushed to Envoy via xDS (Envoy's discovery service) from Istio's control plane (istiod).
**Red Flags**: "EnvoyFilter is just nginx config for Kubernetes."
**Green Flags**: Mentions Envoy sidecar, xDS protocol, filter chain, SIDECAR_INBOUND context.

---

## Grilling Chain (6 questions drilling deeper)

### Q1: Why did you need both Istio RequestAuthentication and application-level auth middleware for versioned APIs?
**Why asked**: Tests understanding of defense-in-depth in a service mesh.
**Expected answer**: Istio `RequestAuthentication` validates JWT signatures and issuers at the mesh level — it rejects obviously invalid tokens before they reach the application. But it operates uniformly: it doesn't know about v1 vs. v3 API requirements (e.g., v3 requiring v1.1+ claims). The application middleware enforces version-specific requirements: checking for HPE claims, extracting workspace, validating token version. They serve complementary purposes — Istio is the perimeter guard, the middleware is the application-aware gatekeeper.
**Red flags**: Says one layer is sufficient.
**Green flags**: Articulates complementary roles, explains version-specific requirements need app-level logic.

### Q2: How do you prevent clients from using v1 indefinitely and never migrating?
**Why asked**: Tests practical API lifecycle management.
**Expected answer**: (1) Deprecation headers (`Sunset`, `Deprecation`) in every v1 response. (2) Rate limiting on deprecated versions — gradually reduce allowed request rates. (3) Logging and dashboards tracking v1 usage — identify specific clients. (4) Direct outreach to high-volume v1 clients with migration support. (5) Feature freeze — no new capabilities on v1/v2. (6) Hard sunset date — after the deprecation period, v1 returns `410 Gone`. (7) Monitoring for 410 responses post-sunset to confirm no critical integrations break.
**Red flags**: "We just turn it off on the sunset date" without outreach or monitoring.
**Green flags**: Graduated approach with metrics, outreach, rate limiting, and monitored sunset.

### Q3: How does the shared service layer work? How do you avoid version-specific business logic leaking into shared code?
**Why asked**: Tests code architecture for multi-version APIs.
**Expected answer**: The service layer exposes version-agnostic methods (e.g., `CreateCase(ctx, params)`) that operate on domain objects. Each API version has its own handler/controller layer that: (1) Parses version-specific request formats. (2) Runs version-specific middleware (auth, validation). (3) Transforms the request into the common domain model. (4) Calls the shared service layer. (5) Transforms the response into the version-specific format. If a version needs genuinely different business logic, it's handled via feature flags or strategy pattern in the service layer, not by forking the code.
**Red flags**: Describes copy-pasting business logic per version.
**Green flags**: Clean separation of concerns — handler (version-specific) → service (shared) → repository (shared).

### Q4: What is the difference between Istio RequestAuthentication and AuthorizationPolicy?
**Why asked**: Tests Istio security model knowledge.
**Expected answer**: `RequestAuthentication` defines HOW to authenticate — which JWTs to accept, which issuers and audiences are valid. It validates the token. `AuthorizationPolicy` defines WHO can do WHAT — RBAC rules based on authenticated identity. RequestAuthentication says "accept tokens from issuer X"; AuthorizationPolicy says "allow requests from principal Y to path /v3/cases with method POST." They work in sequence: RequestAuthentication validates the token and sets the identity, then AuthorizationPolicy checks if that identity is allowed to perform the action.
**Red flags**: Confuses the two; says they're interchangeable.
**Green flags**: Explains authentication vs. authorization distinction, describes the sequential flow.

### Q5: How do you handle API versioning in documentation and SDKs?
**Why asked**: Tests awareness of the full API lifecycle beyond code.
**Expected answer**: (1) Versioned OpenAPI specs — each version has its own spec file. (2) API documentation site shows all supported versions with a prominent notice on deprecated ones. (3) SDK versioning — the client SDK accepts a version parameter and routes requests accordingly. (4) Changelog between versions — what changed, what's new, what's deprecated, migration examples. (5) Automated compatibility tests — generate client stubs from OpenAPI specs and verify they work against the live API. (6) For our Confluence documentation, I maintained a migration guide with before/after request examples for each endpoint.
**Red flags**: "Documentation updates happen whenever someone remembers."
**Green flags**: Structured approach with versioned specs, changelogs, and automated testing.

### Q6: If a client sends a v3 request with both a workspace in the body and a different workspace in the token, what happens?
**Why asked**: Tests the specific security design and edge case handling.
**Expected answer**: The v3 API ignores the workspace in the request body for authorization purposes. The workspace is derived exclusively from the token's `hpe_workspace_id` claim. Specifically: (1) Middleware extracts `hpe_workspace_id` from the token and sets it in the request context. (2) The handler uses the context workspace, not the body workspace. (3) If the body workspace differs from the token workspace, the handler could either: (a) silently use the token workspace (our approach — least disruptive), or (b) return a `403 Forbidden` if the body workspace doesn't match the token (strict mode — considered for future). (4) This mismatch is logged as a security event for auditing. This is the core GLCP-333624 fix.
**Red flags**: Says "we use the body workspace" or "we reject the request."
**Green flags**: Explains token-derived workspace, mentions logging the mismatch, references GLCP-333624.

---

## Tags & Cross-References

### Related STAR Stories
- [JWT Authentication](jwt-authentication.md) — auth middleware that powers version-specific token validation
- [Cross-Workspace Security](cross-workspace-security.md) — GLCP-333624 fix retroactively applied to v2, native in v3
- [Manual Case Workflow](manual-case-workflow.md) — manual case endpoints versioned through this evolution

### Interview Question Categories This Covers
- API Design: Versioning strategies, backward compatibility, deprecation
- Infrastructure: Istio EnvoyFilter, RequestAuthentication, service mesh routing
- Architecture: Shared service layer, middleware chains, handler isolation
- Process: RFC 8594 deprecation, sunset planning, client migration

### Behavioral Dimensions Demonstrated
- **Pragmatic decision-making**: URL versioning chosen for practical reasons, not dogma
- **Backward compatibility**: Zero breaking changes for existing clients
- **Standards awareness**: RFC 8594 deprecation headers
- **Cross-cutting concern**: Versioning touches auth, routing, business logic, and docs
