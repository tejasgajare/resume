# Cross-Workspace Security Vulnerability Fix (GLCP-333624)

## STAR Narrative

### Situation
The RAVE Cloud Platform operates as a multi-tenant system serving HPE GreenLake customers. Each customer operates within an isolated workspace — a logical boundary that separates their cases, devices, and data from other customers. During a security review of the wellnessProducer service's v2 Create Case API, I discovered a critical tenant isolation vulnerability tracked as GLCP-333624. The v2 API accepted a `workspace_id` field in the request body and used it directly for case creation — the backend **trusted the client-supplied workspace identifier** without validating it against the authenticated user's actual workspace. This meant an authenticated user in Workspace A could create cases in Workspace B by simply changing the `workspace_id` in the request body. This was a critical security flaw: any authenticated GreenLake user could access and create cases in any other customer's workspace, potentially exposing confidential support case data, device information, and internal communications across tenant boundaries.

### Task
I was responsible for:
1. Identifying the root cause and full blast radius of the vulnerability.
2. Designing and implementing a fix that enforced server-side authorization for workspace context.
3. Ensuring the fix was backward-compatible (existing legitimate clients wouldn't break).
4. Applying the fix retroactively to v2 and natively in the new v3 API.
5. Testing the fix against the specific attack scenario and edge cases.
6. Coordinating with the security team on disclosure and remediation timeline.

### Action
**Step 1 — Root Cause Analysis:**
I traced the vulnerability through the code:
1. Client sends POST `/v2/cases` with body containing `workspace_id: "workspace-B"`.
2. Auth middleware validates the JWT (signature, expiration, issuer) — the token IS valid.
3. However, the middleware only sets generic user identity in context — it does NOT extract or validate `hpe_workspace_id` from the token.
4. The handler reads `workspace_id` from the request body and uses it for case creation.
5. No comparison between the token's workspace and the request's workspace.

The root cause: **the authorization model relied on client-supplied context rather than server-derived context from the authenticated token.** The token contained `hpe_workspace_id` — the user's actual workspace — but it was ignored.

**Step 2 — Blast Radius Assessment:**
I audited all endpoints that consumed workspace context:
- **Create Case** — vulnerable (workspace from body).
- **Update Case** — partially vulnerable (case lookup was by case ID, which is workspace-scoped in the database, but workspace filtering was inconsistent).
- **List Cases** — vulnerable if workspace filter came from request parameters.
- **Attachments** — indirectly vulnerable (attached to cases, inherits case workspace).

I documented each vulnerable endpoint, the specific code path, and the severity (how easy it was to exploit).

**Step 3 — Fix Design:**
The fix followed a principle: **all security-relevant context must be derived from the authenticated token, never from the client request.** Specifically:

1. **Auth middleware enhancement**: Parse `hpe_workspace_id` from the JWT claims and inject it into the Go `context.Context` via the `HPEClaims` struct.
2. **Handler modification**: Replace all reads of `workspace_id` from the request body with `GetHPEClaimsFromContext(ctx).WorkspaceID`.
3. **Request body workspace handling**: The `workspace_id` field in the request body is now ignored for authorization. It's not removed from the API schema (backward compatibility) but it's only used for informational logging — authorization always uses the token.
4. **Database query scoping**: All case queries include `workspace_id = {token_workspace}` as a mandatory filter, preventing cross-workspace data access.

**Step 4 — Implementation:**
```go
// Before (VULNERABLE):
func (h *Handler) CreateCase(w http.ResponseWriter, r *http.Request) {
    var req CreateCaseRequest
    json.NewDecoder(r.Body).Decode(&req)
    // workspace_id comes from the request body - ATTACKER CONTROLLED
    case := h.service.Create(req.WorkspaceID, req.CaseData)
}

// After (FIXED):
func (h *Handler) CreateCase(w http.ResponseWriter, r *http.Request) {
    var req CreateCaseRequest
    json.NewDecoder(r.Body).Decode(&req)
    // workspace_id derived from authenticated token - SERVER CONTROLLED
    claims := auth.GetHPEClaimsFromContext(r.Context())
    case := h.service.Create(claims.WorkspaceID, req.CaseData)
}
```

**Step 5 — Testing:**
I wrote targeted test cases:
1. **Attack scenario**: User with token for workspace A sends request with workspace B → case is created in workspace A (token workspace), NOT workspace B.
2. **Legitimate scenario**: User with token for workspace A sends request with workspace A (or no workspace) → case created in workspace A. No behavior change.
3. **Service token scenario**: Service tokens without workspace claim → appropriate error handling (service tokens are used for cross-workspace operations with explicit authorization).
4. **Edge cases**: Missing workspace in token, null workspace in body, workspace mismatch logging.

**Step 6 — Retroactive Fix and v3 Integration:**
- **v2 API**: Patched with the fix. This was a security fix, not a breaking change — legitimate clients were already sending the correct workspace, so the behavior was unchanged for them.
- **v3 API**: Natively designed with token-derived workspace from the start. The `workspace_id` field doesn't exist in the v3 request schema.
- Both versions now log a warning if the body workspace differs from the token workspace (for detecting potential attack attempts or misconfigured clients).

**Step 7 — Security Coordination:**
I worked with the security team to:
1. Classify the vulnerability severity (Critical — tenant isolation breach).
2. Expedite the fix through the deployment pipeline (bypassed normal sprint cadence).
3. Audit logs for any evidence of exploitation (none found).
4. Document the vulnerability and fix in the internal security knowledge base.

### Result
- **Critical tenant isolation vulnerability closed** — authenticated users can only operate within their own workspace.
- **Zero exploitation evidence** found in log audit — caught before any malicious use.
- **Fix applied to both v2 (retroactive) and v3 (native)** — all API versions are protected.
- **Authorization model strengthened** — server-derived context principle applied to all security-relevant fields, not just workspace.
- **Security audit improvement** — the vulnerability pattern (trusting client-supplied security context) was added to the team's code review checklist.
- **Deployed within 48 hours** of discovery, across dev → intg → prod with zero customer impact.
- This fix became a reference example for the team on how to handle trust boundaries in multi-tenant systems.

## Interview Delivery Tips

### How to Open This Story
"I discovered and fixed a critical tenant isolation vulnerability in our multi-tenant case management platform. The v2 API trusted client-supplied workspace identifiers — an authenticated user in workspace A could create cases in workspace B by changing a field in the request. I implemented server-side workspace derivation from the JWT token, assessed the blast radius across all endpoints, and coordinated with the security team for expedited deployment."

### Time Budget (5-minute answer)
- Situation: 1 minute (describe the vulnerability clearly — this IS the hook)
- Task: 30 seconds (discovery + fix + coordination)
- Action: 2.5 minutes (root cause, blast radius, fix pattern — before/after code)
- Result: 1 minute (zero exploitation, 48-hour deployment, team checklist)

### Pivot Points
- **If interviewer asks about security**: Trust boundaries, IDOR, OWASP, multi-tenancy strategies
- **If interviewer asks about process**: Security team coordination, log audit, disclosure timeline
- **If interviewer asks about testing**: Cross-workspace attack scenario, authorization matrix, property-based testing
- **If interviewer asks about architecture**: Middleware-enforced context, database-level RLS, defense-in-depth

---

## Rubric Annotations

### Key Phrases/Concepts That MUST Appear
- Tenant isolation / multi-tenancy security
- Client-supplied vs. server-derived security context
- `hpe_workspace_id` from JWT token claims
- Trust boundary violation
- Blast radius assessment across endpoints
- Retroactive fix on v2, native in v3
- Log audit for exploitation evidence

### Common Mistakes
- Describing the fix without explaining WHY the old approach was vulnerable
- Not assessing the blast radius (only fixing one endpoint without checking others)
- Treating this as a simple bug fix rather than a security vulnerability with disclosure implications
- Not explaining the trust boundary concept
- Ignoring the backward compatibility consideration

### Scoring Rubric [1-5]
- 1 — Poor: Cannot explain multi-tenancy security or why client-supplied workspace is dangerous
- 2 — Below Average: Describes the code fix but lacks security analysis, blast radius, and coordination
- 3 — Acceptable: Covers the vulnerability, fix, and testing but light on blast radius and security process
- 4 — Strong: Full narrative with root cause, blast radius, fix, testing, retroactive application, and coordination
- 5 — Exceptional: All of the above plus discusses the broader trust boundary principle, how to prevent similar vulnerabilities, and the security review process improvements

---

## Follow-Up Questions (Progressively Harder)

### Follow-Up 1: How do you ensure no future developer accidentally reintroduces this vulnerability?
**Expected Answer**: (1) **Code review checklist** — any endpoint that uses workspace/tenant context must derive it from the token, never from the request. This is now a mandatory review item. (2) **Automated linting/static analysis** — a custom linter rule that flags reads of `WorkspaceID` from request bodies in handler code. (3) **Integration tests** — the cross-workspace test case runs in CI; if a new endpoint allows cross-workspace access, it fails. (4) **Architecture enforcement** — the `GetHPEClaimsFromContext()` helper is the ONLY approved way to get workspace context. Direct JWT parsing in handlers is flagged in code review. (5) **Documentation** — the vulnerability is documented in the security knowledge base with the fix pattern.
**Red Flags**: "We just tell people not to do it" — no systematic prevention.
**Green Flags**: Multiple layers: linting, testing, code review, and documentation.

### Follow-Up 2: What are the common multi-tenancy isolation strategies, and which does RAVE use?
**Expected Answer**: (1) **Database-per-tenant** — complete isolation, highest cost. (2) **Schema-per-tenant** — moderate isolation, easier backup/restore per tenant. (3) **Row-level security** — shared tables with a tenant column, filtered by application or database-level policies. (4) **Application-level filtering** — shared database, application code adds `WHERE workspace_id = ?` to every query. RAVE uses application-level filtering — all case data is in shared MongoDB collections, and every query includes the workspace filter. The GLCP-333624 fix ensures this filter uses the token-derived workspace, not client-supplied. The risk with application-level filtering is exactly this type of bug — a missed filter means data leakage. Database-level row security (e.g., PostgreSQL RLS) would provide defense-in-depth.
**Red Flags**: Doesn't know multi-tenancy strategies beyond "separate databases."
**Green Flags**: Explains the spectrum from database-per-tenant to row-level security, identifies the risk of application-level filtering.

### Follow-Up 3: If you were implementing this from scratch, how would you prevent this class of vulnerability architecturally?
**Expected Answer**: (1) **Middleware-enforced tenant context** — the middleware sets the tenant context, and the database layer automatically applies the filter. Handlers never see or set the workspace directly. (2) **Database-level row security** — use PostgreSQL RLS or MongoDB field-level encryption to enforce tenant isolation at the data layer, independent of application code. (3) **Tenant-scoped database connections** — each request gets a database connection that's already scoped to the tenant. (4) **API gateway-level enforcement** — the gateway validates workspace authorization before the request reaches the service. (5) **Automated security testing** — fuzzing with cross-tenant payloads in CI/CD.
**Red Flags**: "Just be more careful in code reviews."
**Green Flags**: Proposes architectural controls (middleware, database-level, gateway) rather than relying on developer discipline.

---

## Grilling Chain (7 questions drilling deeper)

### Q1: What is IDOR, and how does this vulnerability relate to it?
**Why asked**: Tests knowledge of the vulnerability class.
**Expected answer**: IDOR (Insecure Direct Object Reference) is when an application exposes internal references (like database IDs or workspace IDs) and doesn't verify that the authenticated user is authorized to access the referenced object. GLCP-333624 is a textbook IDOR — the workspace ID is a direct reference to a tenant's workspace, and the API didn't verify the requester belonged to that workspace. IDOR is consistently in the OWASP Top 10 under "Broken Access Control." The fix pattern is always the same: validate authorization server-side before acting on any client-supplied reference.
**Red flags**: Doesn't know what IDOR stands for or how it applies.
**Green flags**: Connects to OWASP Top 10, explains the fix pattern, correctly classifies this as IDOR.

### Q2: How did you audit logs for exploitation evidence? What would exploitation look like?
**Why asked**: Tests incident response and forensic analysis skills.
**Expected answer**: I searched for patterns in access logs: (1) Requests where the body's `workspace_id` differed from the token's `hpe_workspace_id` — this is the attack signature. (2) Cases created where the case's workspace didn't match the creator's token workspace. (3) Unusual patterns: a single user creating cases across multiple workspaces. (4) I used Humio/LogScale to query structured logs, correlating JWT claims (logged by the auth middleware) with request body fields. (5) The audit found zero mismatches — the vulnerability existed but wasn't exploited (or wasn't exploited by anyone whose actions were logged). (6) I also checked for unusual access patterns from service tokens, which might not have workspace claims.
**Red flags**: "We checked and nothing happened" without explaining HOW they checked.
**Green flags**: Describes specific log queries, the attack signature, and tools used for analysis.

### Q3: Why was this vulnerability not caught during initial development or testing?
**Why asked**: Tests ability to analyze process failures without blame.
**Expected answer**: Several factors: (1) The original developer likely tested with their own token, which naturally matched the workspace they were targeting — the happy path works correctly. (2) Security testing requires adversarial thinking — testing what happens when inputs DON'T match, which requires explicit test cases. (3) No automated security tests for authorization (cross-tenant fuzzing). (4) Code review didn't flag it because the code was "correct" in the functional sense — it created cases in the requested workspace. (5) The Istio authentication layer gave a false sense of security — "the token is valid" doesn't mean "the user is authorized for this workspace." This is why defense-in-depth and security-focused code review checklists are essential.
**Red flags**: Blames a specific person; says "it was obvious and someone was careless."
**Green flags**: Analyzes systemic factors (testing gaps, false security assumptions, lack of adversarial testing), proposes process improvements.

### Q4: Explain the concept of trust boundaries. Where are the trust boundaries in this system?
**Why asked**: Tests security architecture knowledge.
**Expected answer**: A trust boundary is a line where the level of trust changes — data crossing the boundary must be validated. In our system: (1) **Client → API**: The API boundary — request data is untrusted. This is where workspace_id was incorrectly trusted. (2) **API → Token**: JWT claims are trusted (cryptographically signed by the identity provider). (3) **API → Database**: Database data is trusted (written by the application). (4) **API → External Services** (GTCaaS, CRM): Semi-trusted — responses should be validated. (5) **Service → Service**: Inside the mesh — trusted via mTLS, but still validated per defense-in-depth. The GLCP-333624 fix is about correctly placing the trust boundary: client request data is OUTSIDE the trust boundary; JWT claims are INSIDE.
**Red flags**: Can't define trust boundary; says "we trust everything inside our network."
**Green flags**: Identifies specific trust boundaries, explains which side each data source falls on, connects to the vulnerability.

### Q5: How would you implement automated security testing to catch this class of bug?
**Why asked**: Tests proactive security testing knowledge.
**Expected answer**: (1) **Authorization matrix testing**: Define a matrix of (user_workspace, target_workspace) pairs and verify that cross-workspace operations are rejected. Run in CI. (2) **Fuzzing**: Modify request fields while keeping the token unchanged — check that authorization is always based on the token. (3) **Property-based testing**: The property "workspace of created case == workspace of creating user's token" should always hold. Use a framework like go-fuzz or rapid to generate random workspace combinations. (4) **DAST (Dynamic Application Security Testing)**: Tools like OWASP ZAP or Burp Suite in CI that test live endpoints for IDOR. (5) **Custom linter**: Flag code that reads security-relevant fields from request bodies.
**Red flags**: "Just add more unit tests."
**Green flags**: Mentions authorization matrix, property-based testing, DAST tools, and custom linters.

### Q6: What is the difference between authentication and authorization? Where did each fail here?
**Why asked**: Fundamental security concept — must be crystal clear.
**Expected answer**: **Authentication**: Verifying identity — "who are you?" Authentication DID NOT fail — the JWT was valid, the user was correctly identified. **Authorization**: Verifying permissions — "are you allowed to do this?" Authorization FAILED — the system didn't check whether the authenticated user was allowed to create cases in the requested workspace. The fix was entirely on the authorization side: the authentication layer was already correct (valid JWT), but the authorization layer was missing (no workspace access check). This distinction is critical: a system can have perfect authentication and completely broken authorization.
**Red flags**: Conflates authentication and authorization; says "the authentication was broken."
**Green flags**: Clearly distinguishes the two, correctly identifies that auth-n was fine but auth-z was missing.

### Q7: How does this vulnerability compare to other well-known multi-tenant breaches? What lessons can you draw?
**Why asked**: Tests awareness of industry-wide security patterns and incidents.
**Expected answer**: Similar vulnerabilities have caused major breaches: (1) **Salesforce** community sites exposed data across tenants due to misconfigured sharing rules. (2) **Microsoft Power Apps** portals exposed 38M records due to misconfigured table permissions. (3) **SaaS platforms** frequently have IDOR vulnerabilities where tenant IDs in URLs or bodies aren't validated. Common lessons: (a) Never trust client-supplied tenant context. (b) Implement tenant isolation at the lowest possible layer (database, not just application). (c) Security testing must include cross-tenant scenarios. (d) The blast radius of a multi-tenant isolation failure is proportional to the number of tenants — it's not a single customer's data at risk, it's ALL customers. (e) Regular security audits specifically targeting tenant isolation.
**Red flags**: Not aware of any real-world examples; treats this as a minor bug.
**Green flags**: References real incidents, draws applicable lessons, understands the scale of multi-tenant failures.

---

## Tags & Cross-References

### Related STAR Stories
- [JWT Authentication](jwt-authentication.md) — the auth middleware redesign that enabled workspace extraction from tokens
- [API Versioning Evolution](api-versioning-evolution.md) — fix retroactively applied to v2, native in v3
- [File Upload Security](file-upload-security.md) — part of the same 13-issue security hardening initiative

### Interview Question Categories This Covers
- Security: IDOR, multi-tenancy, authorization, trust boundaries
- Incident Response: Root cause analysis, blast radius assessment, log auditing
- Process: Security coordination, expedited deployment, disclosure
- Architecture: Middleware-enforced context, database-level isolation, defense-in-depth

### Behavioral Dimensions Demonstrated
- **Security discovery**: Found the vulnerability through code review, not exploitation
- **Urgency management**: 48-hour fix deployment for a critical vulnerability
- **Systematic approach**: Blast radius assessment across all endpoints, not just the obvious one
- **Process improvement**: Added to code review checklist to prevent recurrence
