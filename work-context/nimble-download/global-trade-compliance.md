# Global Trade Compliance — Nimble Download System

> **Role**: Software Engineer, HPE GreenLake Cloud Platform (GLCP)
> **Domain**: Export compliance for Nimble Storage software downloads
> **Stack**: Go microservices, MongoDB (Device Shadows), SQLite (IP caching), GTCaaS APIs
> **Team**: Nimble Download / Global Trade

---

## Overview

I built and maintained the Global Trade Compliance system for Nimble Storage software downloads within HPE's GreenLake Cloud Platform. This system ensures every software download complies with U.S. export control regulations — including Restricted Party List (RPL) screening at device onboarding time and Export License Checks (ELC) at download time. The system comprises four Go microservices that work together to gate downloads based on compliance screening results.

The core challenge: HPE ships storage software globally, but U.S. export law requires screening every customer and their IP geolocation against embargo lists, denied party lists, and export classification rules. My system automates this end-to-end, with sub-second latency requirements at download time.

---

## System Architecture

### High-Level Flow

```
Customer Device → AutoOnboard (RPL Screening) → Device Shadow (MongoDB)
                                                       ↓
Customer Download Request → DownloadGatekeeper (ELC Check) → Allow/Deny
                              ↓                    ↑
                         GTCacheAPI ←→ SQLite    GTCaaS APIs
```

### The Four Microservices

#### 1. AutoOnboard
- **Purpose**: Automated device onboarding with RPL (Restricted Party List) screening
- **Trigger**: When a Nimble device is registered to an HPE GreenLake account
- **Flow**: Receives device serial number → looks up customer account details → submits RPL screening request to GTCaaS → stores result in Device Shadow
- **Key Logic**: Serial number-based compliance checks (GLCP-294827). Each device gets a unique compliance determination based on the account it belongs to.

#### 2. DownloadGatekeeper
- **Purpose**: Real-time export license check at download time
- **Trigger**: Every software download request
- **Flow**: Receives download request → resolves IP to country → checks ELC via GTCaaS → caches result → allows or denies download
- **Integration**: GTCaaS APIs for ELC determination, IP geolocation for embargo checks
- **Caching**: SQLite-based IP cache for sub-second response times on repeat requests

#### 3. GTCacheAPI
- **Purpose**: Cache management layer for IP-to-country and license determination data
- **Storage**: SQLite for IP cache lookups (GLCP-242758), country code resolution (GLCP-242759)
- **Pattern**: Write-through cache with TTL-based expiration. Reduces GTCaaS API calls by 80%+ for repeat visitors.

#### 4. Reference Number Tools & Migration Scripts
- **Purpose**: Utilities for managing GTCaaS reference numbers and migrating device data
- **Key Tool**: Reference number script converted from Python to Go (GLCP-272882) for consistency with the rest of the stack
- **Migration**: Device shadow onboarding migration scripts (GLCP-292918) for bulk-processing existing Nimble devices

---

## Device Shadow State Machine

The Device Shadow is the central compliance record for each Nimble device. Stored in MongoDB, it tracks the device's compliance status through a well-defined state machine.

### States

```
PENDING ──→ ONBOARDED    (RPL screening passed)
   │
   ├──→ FAILED           (RPL screening error / transient failure)
   │
   └──→ BLOCKED          (RPL screening denied — entity on restricted list)
```

### State Transitions

| From | To | Trigger | Notes |
|------|----|---------|-------|
| (new) | PENDING | Device registered | Initial state on account add |
| PENDING | ONBOARDED | RPL clear | Device can proceed to downloads |
| PENDING | FAILED | GTCaaS error | Transient — retryable |
| PENDING | BLOCKED | RPL match | Entity on restricted party list |
| FAILED | PENDING | Retry triggered | Auto-retry with backoff |
| BLOCKED | PENDING | Account transition | Re-screen after account change (GLCP-289971) |

### Key Design Decisions

**Why MongoDB for Device Shadows?**
I chose MongoDB because Device Shadows are document-oriented — each shadow is a self-contained record with nested compliance metadata, account info, and screening history. The schema flexibility was critical because GTCaaS response formats evolved over time, and I needed to store arbitrary metadata without schema migrations.

**GTCaasID Persistence (GLCP-302655)**
A critical fix: I ensured the GTCaasID (the unique identifier from GTCaaS screening responses) is persisted in the Device Shadow. Previously, this ID was only used transiently, which made it impossible to correlate GTCaaS audit logs with our device records. This was a compliance audit requirement.

[KEY_POINTS]
- Device Shadow is the single source of truth for compliance state
- State machine prevents illegal transitions (e.g., BLOCKED → ONBOARDED without re-screening)
- GTCaasID persistence enables end-to-end audit trail
- MongoDB document model allows flexible metadata storage
[/KEY_POINTS]

---

## RPL (Restricted Party List) Screening Flow

RPL screening happens at device onboarding time. It checks whether the customer (account holder) appears on any restricted party lists maintained by the U.S. government and other regulatory bodies.

### Flow

1. Device registers → AutoOnboard receives event
2. Look up account details (company name, address, country)
3. Submit screening request to GTCaaS RPL API
4. GTCaaS checks against:
   - Denied Persons List (DPL)
   - Entity List (EL)
   - Specially Designated Nationals (SDN)
   - Other consolidated screening lists
5. Receive result: CLEAR / MATCH / ERROR
6. Update Device Shadow state accordingly

### Serial Number-Based Checks (GLCP-294827)

I implemented serial number-based GT checks to enable per-device compliance tracking. Previously, compliance was at the account level only, but regulations require device-level granularity for certain export classifications. Each serial number now gets its own screening determination.

### Escalation for Blocked Devices (GLCP-253683, GLCP-250862)

When a device is RPL-blocked, the system:
1. Updates Device Shadow to BLOCKED
2. Sends Slack alert (GLCP-292971) to compliance team
3. Creates escalation ticket for manual review
4. Blocks all download attempts for that device

[COMMON_MISTAKES]
- Not handling account transitions: When a device moves between accounts, the compliance state must be re-evaluated (GLCP-289971). I initially missed this edge case — a device could remain BLOCKED even after moving to a compliant account.
- Treating RPL screening as a one-time check: Screening lists are updated regularly. I implemented periodic re-screening for ONBOARDED devices.
[/COMMON_MISTAKES]

---

## ELC (Export License Check) Flow

ELC happens at download time — every single download request is checked in real-time.

### Flow

1. Customer requests software download
2. DownloadGatekeeper intercepts request
3. Resolve customer IP to country code (via Neustar/MaxMind)
4. Check if country is embargoed
5. Check ELC determination via GTCaaS
6. If allowed → serve download via Apache (GLCP-246934)
7. If denied → return compliance denial response
8. Cache result in SQLite for subsequent requests

### IP-to-Country Resolution

**IP Cache Lookup (GLCP-242758)**:
SQLite stores IP-to-country mappings with TTL. Cache hit rate is ~85% for typical traffic patterns.

**Country Code from IP (GLCP-242759)**:
When cache misses, I call the geolocation provider (originally Neustar, migrating to MaxMind) to resolve the IP to an ISO country code.

### ELC Escalation (GLCP-243131)

For ambiguous ELC results (e.g., partial matches, classification edge cases), the system escalates to the compliance team rather than auto-denying. This reduces false denials while maintaining compliance.

[KEY_POINTS]
- ELC is a real-time check — must be sub-second for acceptable UX
- SQLite cache is critical for performance (85% cache hit rate)
- IP geolocation accuracy directly impacts compliance decisions
- Escalation path prevents over-blocking legitimate customers
[/KEY_POINTS]

---

## IP Caching Architecture

### Why SQLite?

I chose SQLite for the IP cache because:
1. **Embedded**: No external dependency. The cache lives in the same process as DownloadGatekeeper.
2. **Read-heavy workload**: IP cache is 85% reads, 15% writes. SQLite excels at this.
3. **Simplicity**: No need for Redis/Memcached infrastructure for what is essentially a key-value lookup with TTL.
4. **Durability**: Cache survives process restarts, unlike in-memory caches.

### Schema

```sql
CREATE TABLE ip_cache (
    ip_address TEXT PRIMARY KEY,
    country_code TEXT NOT NULL,
    resolved_at TIMESTAMP NOT NULL,
    ttl_seconds INTEGER NOT NULL DEFAULT 86400,
    source TEXT NOT NULL  -- 'neustar' or 'maxmind'
);

CREATE INDEX idx_ip_cache_expiry ON ip_cache(resolved_at);
```

### Cache Strategy

- **Write-through**: On cache miss, resolve IP → write to cache → return result
- **TTL-based expiration**: Default 24-hour TTL. IP-to-country mappings change infrequently.
- **Background cleanup**: Goroutine runs every hour to evict expired entries
- **Fallback**: If SQLite is unavailable, fall through to live API call (never block downloads due to cache failure)

[FOLLOW_UP]
- How would you handle cache stampede if many requests hit the same uncached IP simultaneously?
- What's the tradeoff between SQLite and Redis for this use case?
- How do you handle IPv6 addresses in the cache?
[/FOLLOW_UP]

---

## GTCaaS Integration

GTCaaS (Global Trade Compliance as a Service) is HPE's internal compliance API platform. My microservices integrate with it for both RPL screening and ELC determination.

### API Integration Points

| API | Used By | Purpose |
|-----|---------|---------|
| RPL Screening API | AutoOnboard | Submit entity for restricted party screening |
| ELC Determination API | DownloadGatekeeper | Check export license requirements |
| Reference Number API | Reference tools | Manage compliance reference numbers |

### API Client Logging (GLCP-272532)

I added comprehensive logging to the GTCaaS API client, including:
- Request/response correlation IDs
- Latency metrics per API call
- Error categorization (transient vs. permanent)
- GTCaasID tracking for audit trail

### Vault Credentials (GLCP-258323)

GTCaaS API credentials are stored in HashiCorp Vault. I implemented automatic credential rotation and secure retrieval at service startup.

### Geo Production URLs (GLCP-254822, GLCP-254820)

Configured production and geo-specific URLs for GTCaaS endpoints. Different regions route to different GTCaaS instances for latency optimization.

### UAT Environment (GLCP-242760)

Set up the UAT (User Acceptance Testing) environment for compliance testing with sanitized data.

### Test URL Configuration (GLCP-257467)

Configured test URLs for integration testing against GTCaaS sandbox environments.

---

## MaxMind Migration

### Background

The system originally used Neustar for IP-to-country geolocation. We planned a migration to MaxMind due to:
1. **Cost**: MaxMind is significantly cheaper at our volume
2. **Accuracy**: MaxMind GeoIP2 has comparable or better accuracy for country-level resolution
3. **Self-hosted option**: MaxMind databases can be hosted locally, eliminating API latency

### Migration Plan

1. **Dual-resolution phase**: Run both Neustar and MaxMind in parallel, log discrepancies
2. **Accuracy validation**: Compare country code results for 30 days of production traffic
3. **Embargo edge cases**: Special attention to IP ranges near embargoed countries (border regions, VPNs, satellite ISPs)
4. **Cache migration**: Update cache entries to include source provider, allowing gradual rollover
5. **Cutover**: Switch primary to MaxMind, keep Neustar as fallback for 90 days

### Embargo Handling Concerns

The most critical aspect of the migration: ensuring no embargoed country IPs get misclassified. MaxMind and Neustar can disagree on country classification for:
- Border region IPs
- Satellite internet providers
- Mobile carrier IPs that span countries
- VPN exit nodes in unusual locations

I designed the validation to flag any discrepancy involving an embargoed country for manual review.

[COMMON_MISTAKES]
- Assuming IP geolocation is deterministic: Different providers return different countries for the same IP, especially near borders.
- Not considering IPv6: MaxMind handles IPv6 differently than Neustar. The cache schema needed updating.
- Ignoring cache warming: After migration, the entire SQLite cache is cold. I planned a pre-warming script to avoid latency spikes.
[/COMMON_MISTAKES]

---

## Onboarding Existing Devices (GLCP-250036)

When the compliance system launched, thousands of Nimble devices were already in production without compliance screening. I built a bulk onboarding pipeline:

1. **Extract**: Query all existing Nimble devices from the device registry
2. **Filter**: Skip devices already with valid Device Shadows
3. **Batch**: Group into batches of 100 for GTCaaS rate limiting
4. **Screen**: Submit each batch for RPL screening
5. **Update**: Create Device Shadows with screening results
6. **Report**: Generate summary report of outcomes (onboarded/blocked/failed)

### Migration Scripts (GLCP-292918)

I wrote migration scripts to handle Device Shadow schema changes during the onboarding process. These scripts:
- Backfill GTCaasID for devices screened before the field was added
- Normalize country codes to ISO 3166-1 alpha-2
- Reconcile duplicate Device Shadows from parallel onboarding attempts

---

## Reference Number Script Conversion (GLCP-272882)

Converted the reference number management script from Python to Go. Motivations:
- **Consistency**: All other tooling is in Go
- **Single binary**: No Python dependency on deployment targets
- **Type safety**: Reference number formats are strict; Go's type system catches format errors at compile time

The script manages GTCaaS reference numbers used in compliance tracking and audit trails.

---

## Slack Alerting (GLCP-292971)

I implemented Slack alerting for compliance events:
- **RPL blocks**: Immediate alert when a device is blocked
- **ELC denials**: Alert with download details for denied requests
- **System errors**: Alert when GTCaaS APIs are unreachable
- **Migration progress**: Daily summary during bulk onboarding

Alerts include deep links to the relevant Device Shadow in our admin UI and the GTCaaS screening result.

---

## Download Serving (GLCP-246934)

After compliance checks pass, downloads are served via Apache. I integrated the Go microservice with Apache's download infrastructure:
- DownloadGatekeeper returns a signed redirect URL
- Apache serves the actual file with proper content headers
- Download tokens are single-use and time-limited (15-minute expiry)

---

## Account Transition Handling (GLCP-289971)

A critical bug fix: when a device transitions between accounts (e.g., customer sells hardware, device is re-registered), the compliance state must be re-evaluated. Previously, a device blocked under Account A would remain blocked even after moving to compliant Account B.

### Fix

1. Detect account transition events
2. Reset Device Shadow state to PENDING
3. Trigger re-screening with new account details
4. Update Device Shadow with new screening result

This required careful coordination with the device lifecycle events to avoid race conditions between the transition and any in-flight download requests.

[KEY_POINTS]
- Account transitions require full compliance re-screening
- Race condition between transition event and in-flight downloads
- Cannot assume previous compliance state is valid after any account change
- This was a production bug that blocked legitimate customers
[/KEY_POINTS]

---

## Complete Jira Story Reference

### Core Compliance Engine
| Story | Title | Area |
|-------|-------|------|
| GLCP-302655 | Persist GTCaasID in Device Shadow | AutoOnboard |
| GLCP-289971 | Download not denied on account transition | DownloadGatekeeper |
| GLCP-294827 | Serial number-based GT checks | AutoOnboard |
| GLCP-292918 | Migration scripts for device shadow onboarding | Migration |
| GLCP-250036 | Onboard existing nimble devices | Migration |
| GLCP-272882 | Convert reference number script to Go | Tooling |

### GTCaaS Integration
| Story | Title | Area |
|-------|-------|------|
| GLCP-253683 | Escalation for blocked devices | Escalation |
| GLCP-250862 | Escalation for RPL blocked | Escalation |
| GLCP-243131 | ELC escalation | Escalation |
| GLCP-272532 | API client logs | Observability |
| GLCP-258323 | Vault credentials | Security |
| GLCP-254822 | Geo production URLs | Infrastructure |
| GLCP-254820 | Geo production URLs | Infrastructure |
| GLCP-257467 | Test URL configuration | Testing |
| GLCP-242760 | UAT environment | Testing |

### IP Geolocation & Caching
| Story | Title | Area |
|-------|-------|------|
| GLCP-242759 | Country code from IP | DownloadGatekeeper |
| GLCP-242758 | IP cache lookup | GTCacheAPI |

### Operational
| Story | Title | Area |
|-------|-------|------|
| GLCP-292971 | Slack alerting | Observability |
| GLCP-246934 | Serve download via Apache | Infrastructure |

---

## Interview Talking Points

### System Design Questions

**"Tell me about a compliance system you built."**
I built a 4-microservice system in Go that enforces U.S. export compliance for every Nimble Storage software download. It performs RPL screening at device onboarding (checking restricted party lists) and ELC checks at download time (verifying export license requirements). I used MongoDB for device compliance state tracking and SQLite for IP-to-country caching to keep download latency under 200ms.

**"How did you handle state management?"**
The Device Shadow state machine in MongoDB tracks each device through PENDING → ONBOARDED / FAILED / BLOCKED states. State transitions are atomic and validated — you can't go from BLOCKED to ONBOARDED without re-screening. I persisted the GTCaasID for end-to-end audit trails.

**"How did you handle caching?"**
SQLite for IP-to-country resolution with 24-hour TTL and 85% cache hit rate. Write-through pattern — on miss, resolve and cache before returning. The cache falls through gracefully if SQLite is unavailable, so downloads are never blocked by cache failures.

### Behavioral Questions

**"Tell me about a production bug you fixed."**
The account transition bug (GLCP-289971): devices remained blocked after moving to compliant accounts. I redesigned the transition handler to reset compliance state and trigger re-screening, handling race conditions with in-flight downloads.

**"Tell me about a migration you planned."**
The MaxMind migration from Neustar for IP geolocation. I designed a dual-resolution validation phase, special handling for embargoed country edge cases, and a cache warming strategy to prevent latency spikes at cutover.

---

## Cross-References

- See [Side Projects → ClipStash](../side-projects/clipstash.md) for my SQLite expertise applied differently
- See [Tooling → Utility Scripts](../tooling-automation/utility-scripts.md) for the reference number script context
- See [Tooling → Sprint Review Generator](../tooling-automation/sprint-review-generator.md) for Jira automation I built for this team

---

*Last updated: Auto-generated from work-context-refresh skill*
