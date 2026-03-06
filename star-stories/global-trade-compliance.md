# Global Trade Compliance System for Nimble Download

## STAR Narrative

### Situation
HPE's Nimble Download Update service distributes firmware and software updates to HPE storage devices worldwide. Due to U.S. export control regulations (EAR — Export Administration Regulations), HPE is legally required to screen devices against the Restricted Party List (RPL), enforce embargo restrictions on downloads to sanctioned countries, and perform Export License Checks (ELC) before delivering controlled technology. The existing compliance checks were manual, inconsistent, and couldn't scale with the growing device fleet. The Nimble Download platform needed a fully automated, auditable compliance system that could handle device onboarding, ongoing screening, and real-time download gating — all without adding perceptible latency to the customer download experience.

### Task
I was responsible for building the Global Trade Compliance (GTC) system spanning 4 microservices:
1. **AutoOnboard** — Automated device onboarding with RPL screening via GTCaaS (Global Trade Compliance as a Service).
2. **DownloadGatekeeper** — Real-time ELC checks at download time with IP-based geolocation caching.
3. **GTCacheAPI** — Cache management API for IP geolocation data.
4. **Device Shadow** — MongoDB-based state machine tracking each device's compliance onboarding lifecycle.

Constraints: Zero tolerance for compliance violations (legal liability), sub-second latency for download gating, and auditability for every compliance decision. This was a 50+ commit effort across all 4 services.

### Action
**Step 1 — Device Shadow State Machine:**
I designed a state machine in MongoDB's Device Shadow collection to track each device's compliance onboarding status. States:
- **PENDING** — Device registered, awaiting RPL screening.
- **ONBOARDED** — Passed RPL screening, cleared for downloads.
- **FAILED** — RPL screening failed (temporary — can be retried).
- **BLOCKED** — Device appears on restricted party list (permanent until manual review).

Each state transition was recorded with a timestamp, the triggering event, and the GTCaaS response for auditability. The state machine enforced valid transitions — e.g., a BLOCKED device couldn't transition directly to ONBOARDED without going through a manual review process that resets it to PENDING.

**Step 2 — AutoOnboard Service (RPL Screening):**
AutoOnboard processes newly registered devices by:
1. Extracting device metadata (serial number, customer info, ship-to address).
2. Calling GTCaaS's Restricted Party Screening API to check the customer against RPL databases.
3. Based on the response: transitioning the Device Shadow to ONBOARDED (clean), FAILED (transient error), or BLOCKED (match found).
4. Implementing retry logic for FAILED states with exponential backoff.

The service runs as a batch processor, picking up PENDING devices periodically. Serial numbers are the primary key for device-level granularity.

**Step 3 — DownloadGatekeeper (ELC at Download Time):**
The DownloadGatekeeper intercepts every download request and performs real-time Export License Checks:
1. Extract the client IP from the request.
2. Check the IP cache (SQLite-based) for a cached geolocation result.
3. If cache miss: call the geolocation service (initially HPE Geolocation Service backed by Neustar, later GTCaaS backed by MaxMind) to resolve the IP to a country.
4. Check the country against the embargo list.
5. If embargoed: return 403 with the country code; if clear: allow the download.

The IP cache was critical for performance — most downloads come from a relatively small set of IP ranges, and caching eliminated redundant geolocation lookups. The cache stored IP → country mappings with a TTL.

**Step 4 — IP Caching Strategy:**
The GTCacheAPI managed the SQLite-based IP cache with:
- **geo_country column** — populated on 403 embargo responses (which include the country code). Non-embargoed 200 responses from the new MaxMind API don't include the country, so the column is only populated for blocked requests.
- **Cache invalidation** — TTL-based expiration to handle IP reassignment.
- **Slack alert deduplication** — first-time detection of an embargoed IP triggers a Slack notification; subsequent requests from the same IP are silently blocked using the cache.

**Step 5 — GTCaaS Migration (Neustar → MaxMind):**
Mid-project, HPE migrated from Neustar to MaxMind as the geolocation provider within GTCaaS. I adapted the integration:
- Old flow: Call HPE Geolocation Service (Neustar) → get country → call ELC API separately.
- New flow: Pass IP directly to GTCaaS License Screening API → it resolves country internally via MaxMind and performs ELC in one call.
- The new API requires API Key + mTLS (mutual TLS) authentication.
- RPL onboarding remained unchanged — only the ELC/geolocation path changed.
- I kept the IP cache but adapted the geo_country handling for the new response format.

**Step 6 — Auditability and Compliance:**
Every compliance decision was logged with: device serial number, timestamp, check type (RPL/ELC), decision (allow/block), source data (IP, country, GTCaaS response ID), and the service version that made the decision. This audit trail was essential for regulatory reviews.

### Result
- **4 microservices** built and deployed, handling the full compliance lifecycle from device onboarding to download gating.
- **50+ commits** across the system over multiple sprints.
- **Sub-100ms download gating** — IP cache hit rate >85%, meaning most ELC checks are served from cache.
- **Zero compliance violations** since deployment — every download is screened.
- **Automated RPL screening** replaced a manual process that previously took days per batch.
- **GTCaaS migration completed** seamlessly — no customer-facing impact during the Neustar → MaxMind cutover.
- **Full audit trail** for every compliance decision, passing regulatory review.

## Interview Delivery Tips

### How to Open This Story
"I built a 4-microservice global trade compliance system for HPE's firmware download platform. It automates device onboarding screening against restricted party lists and enforces export license checks at download time — with sub-100ms latency using an IP geolocation cache. This was a 50+ commit effort with zero tolerance for compliance violations."

### Time Budget (5-minute answer)
- Situation: 45 seconds (legal requirements, manual process, scaling challenge)
- Task: 30 seconds (4 services, constraints)
- Action: 2.5 minutes (state machine + caching are the stars)
- Result: 1 minute (sub-100ms, zero violations, successful migration)

### Pivot Points
- **If interviewer asks about system design**: Deep dive on state machine + caching
- **If interviewer asks about APIs**: Focus on GTCaaS integration, mTLS, migration
- **If interviewer asks about databases**: MongoDB for Device Shadow, SQLite for IP cache — explain choices
- **If interviewer asks about compliance**: EAR regulations, RPL vs ELC distinction, auditability

---

## Rubric Annotations

### Key Phrases/Concepts That MUST Appear
- Device Shadow state machine (PENDING → ONBOARDED/FAILED/BLOCKED)
- RPL (Restricted Party List) screening via GTCaaS
- ELC (Export License Check) at download time
- IP caching with SQLite for geolocation lookups
- GTCaaS migration from Neustar to MaxMind
- Audit trail for every compliance decision
- mTLS authentication for the new API

### Common Mistakes
- Conflating RPL screening (onboarding-time) with ELC (download-time) — they're different checks
- Not explaining the state machine transitions and why BLOCKED requires manual review
- Ignoring the caching strategy and its impact on latency
- Not mentioning the Neustar → MaxMind migration as a significant mid-project change
- Forgetting auditability — this is a legal requirement, not a nice-to-have

### Scoring Rubric [1-5]
- 1 — Poor: Cannot explain the compliance requirements or distinguish RPL from ELC
- 2 — Below Average: Describes the system at a high level but lacks state machine details and caching strategy
- 3 — Acceptable: Covers all 4 services and the state machine but light on caching, migration, and auditability
- 4 — Strong: Full narrative with state machine, caching, migration, and audit trail
- 5 — Exceptional: All of the above plus discusses trade-offs (cache TTL, false positives, mTLS complexity), legal implications, and how the system would scale

---

## Follow-Up Questions (Progressively Harder)

### Follow-Up 1: Why did you use a state machine for device onboarding instead of a simple pass/fail flag?
**Expected Answer**: A state machine captures the full lifecycle: a device might be PENDING (not yet screened), FAILED (transient error — should retry), BLOCKED (actual RPL match — needs manual review), or ONBOARDED (cleared). A boolean flag can't distinguish between "never checked" and "checked and failed" and "checked and permanently blocked." The state machine also enforces valid transitions — you can't go from BLOCKED to ONBOARDED without a deliberate manual review, which is a compliance requirement. Each transition is timestamped for the audit trail.
**Red Flags**: Says "a boolean would have been fine."
**Green Flags**: Explains the distinction between transient failure and permanent block, mentions audit requirements.

### Follow-Up 2: How did you handle the IP cache invalidation? What if an IP is reassigned to a different country?
**Expected Answer**: TTL-based expiration — each cache entry expires after a configurable period (e.g., 24 hours). When the TTL expires, the next request for that IP triggers a fresh geolocation lookup. This handles IP reassignment within the TTL window. For the embargo use case, a false-negative (allowing a download from a now-embargoed IP because of stale cache) is the risk. We set the TTL conservatively short enough to limit exposure while still providing meaningful cache hit rates. IP reassignment across countries is relatively rare, and the TTL is tuned to balance freshness and performance.
**Red Flags**: "We never invalidate the cache" or "we just cache forever."
**Green Flags**: Discusses TTL tuning, explains the false-negative risk, mentions the balance between freshness and performance.

### Follow-Up 3: How would you scale this system to handle 10x the current download volume?
**Expected Answer**: (1) Replace SQLite IP cache with Redis for horizontal scalability — multiple DownloadGatekeeper instances share a cache. (2) Pre-warm the cache by batch-processing known customer IP ranges. (3) Shard the Device Shadow collection by serial number prefix for read/write scalability. (4) Add a CDN layer with edge-computed compliance checks for geographically distributed downloads. (5) Async RPL screening with a message queue (Kafka) instead of batch polling. (6) GTCaaS API rate limiting — implement circuit breaker and fallback to cached results during API degradation.
**Red Flags**: "Just add more instances" without addressing shared state (cache, database).
**Green Flags**: Addresses cache scaling (Redis), database sharding, and graceful degradation.

---

## Grilling Chain (7 questions drilling deeper)

### Q1: Explain the Device Shadow state machine. What are all the valid transitions and what triggers each?
**Why asked**: Tests detailed system design knowledge.
**Expected answer**: PENDING → ONBOARDED (RPL screening returns clean). PENDING → FAILED (RPL screening returns transient error — network timeout, API 500). PENDING → BLOCKED (RPL screening matches restricted party). FAILED → PENDING (retry timer triggers re-screening). BLOCKED → PENDING (manual review clears the device). Invalid transitions: BLOCKED → ONBOARDED (must go through PENDING), ONBOARDED → FAILED/BLOCKED (once onboarded, only re-screening on policy change would change status). Each transition records the triggering event, timestamp, and GTCaaS response ID.
**Red flags**: Can't enumerate all states; allows invalid transitions.
**Green flags**: Covers all transitions, explains the manual review requirement for BLOCKED → PENDING, mentions audit data per transition.

### Q2: Why SQLite for the IP cache instead of Redis or an in-memory map?
**Why asked**: Tests data store selection rationale.
**Expected answer**: SQLite was chosen because: (1) The DownloadGatekeeper runs as a single instance per environment — no need for distributed cache. (2) SQLite survives pod restarts (persistent volume), avoiding cold-start cache misses. (3) It supports complex queries for cache management (expiration, statistics, deduplication). (4) Zero operational overhead — no separate Redis cluster to manage. If we needed to scale to multiple Gatekeeper instances, Redis would be the next step. In-memory map was considered but wouldn't survive restarts and couldn't be queried for the Slack deduplication logic.
**Red flags**: "We just picked SQLite because it was easy."
**Green flags**: Explains the single-instance constraint, persistence advantage, and when to upgrade to Redis.

### Q3: How does mTLS work? Why does the GTCaaS API require it?
**Why asked**: Tests security knowledge beyond application-level auth.
**Expected answer**: mTLS (mutual TLS) means both the client and server present certificates during the TLS handshake. Standard TLS only verifies the server's identity; mTLS additionally verifies the client's identity. GTCaaS requires mTLS because: (1) It's a compliance-critical API — they need to authenticate and audit every caller. (2) API keys alone can be stolen; mTLS certificates are bound to specific infrastructure. (3) It prevents unauthorized services from calling the compliance API. In practice, our service presents a client certificate signed by HPE's internal CA, and GTCaaS verifies it against their trust store.
**Red flags**: Confuses mTLS with standard TLS; says "it's just HTTPS."
**Green flags**: Explains mutual verification, distinguishes from standard TLS, mentions CA trust chain.

### Q4: What happens if GTCaaS is down when a download request comes in?
**Why asked**: Tests resilience and failure mode design.
**Expected answer**: This is a critical design decision with compliance implications. Our approach: (1) Check IP cache first — if there's a cached result (even near-expiry), use it. (2) If cache miss AND GTCaaS is down: **block the download** (fail-closed). For compliance, a false block (denying a legitimate download) is acceptable; a false allow (permitting a download to an embargoed country) is not. (3) Implement a circuit breaker — after N consecutive GTCaaS failures, trip the circuit to avoid timeout accumulation, and return 503 with a retry-after header. (4) Alert immediately on circuit breaker trips.
**Red flags**: Says "just allow the download and check later" (fail-open — compliance violation).
**Green flags**: Explains fail-closed design for compliance, mentions circuit breaker, distinguishes between false-block and false-allow severity.

### Q5: How did you ensure the Neustar → MaxMind migration didn't introduce compliance gaps?
**Why asked**: Tests migration safety for a zero-tolerance system.
**Expected answer**: (1) Ran both systems in parallel for 2 weeks — every download triggered both the old Neustar path and the new MaxMind path, logging both results without acting on the new one (shadow testing). (2) Compared results to identify discrepancies — MaxMind and Neustar occasionally disagree on IP geolocation for edge cases (VPNs, satellite internet, mobile carriers). (3) Reviewed discrepancies with the compliance team to determine which source was more accurate. (4) Gradual cutover: 10% of traffic to MaxMind → 50% → 100%, monitoring for any blocked IPs that were previously allowed. (5) Kept Neustar as a fallback for 30 days post-migration.
**Red flags**: "We just switched over on a specific date."
**Green flags**: Describes shadow testing, discrepancy analysis, gradual rollout, and fallback period.

### Q6: How do you handle IP geolocation accuracy limitations? VPNs? IPv6?
**Why asked**: Tests awareness of real-world edge cases in geolocation.
**Expected answer**: Geolocation databases have known limitations: (1) VPNs — the IP resolves to the VPN exit node's country, not the user's actual country. This is actually acceptable for compliance: if the VPN exits in an embargoed country, the download should be blocked regardless of the user's true location. (2) IPv6 — MaxMind supports IPv6, but coverage is lower than IPv4. If geolocation fails (unknown IP), we fail-closed (block). (3) Mobile carriers — IP pools may be geolocated to the carrier's headquarters rather than the device's location. (4) CDN/proxy IPs — we need the client's real IP, so we handle `X-Forwarded-For` headers carefully, trusting only known proxy hops. These edge cases are documented and accepted by the compliance team with compensating controls.
**Red flags**: "Geolocation is always accurate" or doesn't consider VPN scenarios.
**Green flags**: Addresses VPN, IPv6, mobile, and proxy scenarios; mentions fail-closed for unknowns; notes compliance team acceptance.

### Q7: If you were building this system today with no constraints, what would you change?
**Why asked**: Tests architectural vision and awareness of modern approaches.
**Expected answer**: (1) Use event-driven architecture with Kafka — publish device registration events, AutoOnboard consumes and processes them instead of polling. (2) Replace SQLite cache with Redis + a pre-warming pipeline that processes known customer IP ranges nightly. (3) Build the state machine on a proper workflow engine (Temporal, AWS Step Functions) for better observability and retry management. (4) Use feature flags (LaunchDarkly) for the provider migration instead of manual config changes. (5) Add compliance-as-code — define embargo lists and RPL rules as versioned config, not hardcoded. (6) Implement real-time streaming analytics (Kafka Streams) for compliance dashboards. (7) Consider edge computing for download gating — push compliance checks to CDN edge nodes for lower latency.
**Red flags**: "Nothing, the design is optimal."
**Green flags**: Mentions event-driven architecture, workflow engines, edge computing, and compliance-as-code.

---

## Tags & Cross-References

### Related STAR Stories
- [Multi-Environment Infra](multi-environment-infra.md) — GTC services deployed across the same dev/intg/prod environments
- [Monitoring Overhaul](monitoring-overhaul.md) — GTC metrics (blocked downloads, RPL screenings) feed into the monitoring dashboards
- [File Upload Security](file-upload-security.md) — similar defense-in-depth philosophy applied to different domain

### Interview Question Categories This Covers
- System Design: State machines, caching strategies, microservice architecture
- Distributed Systems: Service coordination, data consistency, API integration
- Compliance/Legal: Export control, restricted party screening, auditability
- Databases: MongoDB (state machine), SQLite (caching), data modeling choices

### Behavioral Dimensions Demonstrated
- **Scale of impact**: 4 microservices, 50+ commits, zero compliance violations
- **Domain expertise**: Deep understanding of export control regulations
- **Adaptability**: GTCaaS provider migration mid-project
- **Performance engineering**: Sub-100ms latency via intelligent caching
