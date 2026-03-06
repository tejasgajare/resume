# Manual Case Creation Workflow

## STAR Narrative

### Situation
The RAVE Cloud Platform supports automated case creation (triggered by device telemetry events) and manual case creation (initiated by support agents or customers). The manual case creation pipeline was incomplete and fragmented: it lacked input validation, contact management was inconsistent (CC recipients could be duplicated), the attachment workflow was disconnected from email delivery (emails were sent before attachments finished uploading), and there was no comprehensive documentation for the complex payload structure. The system needed to support two distinct communication domains — **email/SendGrid** (for email-based case communication) and **panhpe/DCM CRM** (for HPE's Device Case Management system) — each with different field requirements and integration patterns.

### Task
I was responsible for building the end-to-end manual case creation pipeline:
1. Implement comprehensive input validation for the manual case payload.
2. Build contact management with CC deduplication — ensure no duplicate contacts in email threads.
3. Implement the attachment workflow with deferred email send — emails only sent after all attachments are uploaded.
4. Support both communication domains (email/SendGrid and panhpe/DCM CRM) with domain-specific validation.
5. Create comprehensive field documentation for the complex payload structure.
6. Ensure the `createdBy` field is extracted from the JWT `sub` claim for audit purposes.

### Action
**Step 1 — Payload Design and Validation:**
I designed the manual case creation payload with strict validation rules. The WellnessModel contains core case fields, and the ManualCaseData contains manual-case-specific fields:

Validation rules included:
- **Required fields**: summary, description, severity, primary contact email (not just phone — added this requirement because email cases obviously need an email address).
- **Email format validation**: RFC 5322 compliant regex for all email fields.
- **Severity validation**: Must be one of the predefined levels (P1-P4).
- **Domain validation**: The communication domain must be either `email` or `panhpe`.
- **Field length limits**: Summary (200 chars), description (10,000 chars), to prevent abuse.
- **Cross-field validation**: If domain is `email`, primary contact must have a valid email address. If domain is `panhpe`, CRM-specific fields (account ID, site ID) are required.

**Step 2 — Contact Management with CC Deduplication:**
The contact management system handles three types of contacts:
1. **Primary contact** — the case requester. Email is required for email-domain cases.
2. **Alternate contacts** — additional contacts who should receive case updates.
3. **CC contacts** — explicitly CC'd recipients on email communications.

I implemented CC deduplication:
1. Collect all email addresses from primary, alternate, and CC contacts.
2. Normalize (lowercase, trim whitespace).
3. Deduplicate — if the primary contact's email appears in the CC list, remove it from CC (they're already included).
4. If the `createdBy` user (extracted from JWT `sub` claim) has an email, add them to CC if not already present — ensures the case creator is always in the loop.

**Step 3 — createdBy Field from JWT:**
For audit and accountability, every manually created case records who created it. I extracted the `createdBy` field from the JWT's `sub` claim in the auth middleware and injected it into the request context. The handler retrieves it and sets it on the case record. This replaces the previous approach where `createdBy` was a client-supplied field (easily spoofed).

**Step 4 — Attachment Workflow with Deferred Email Send:**
The manual case attachment workflow:
1. Client creates the case with attachment metadata (filename, size, type — but not the file content).
2. Server validates the case, creates it in the database with status `PENDING_ATTACHMENTS`.
3. Server returns pre-signed S3 upload URLs for each attachment.
4. Client uploads files directly to S3 using the pre-signed URLs (server doesn't proxy file content).
5. S3 upload completion triggers a notification (via S3 event notification or client callback).
6. After all attachments are confirmed uploaded, the system transitions the case to `CREATED` status.
7. **Only now** is the notification email sent via SendGrid — with attachment links included.

This deferred send pattern ensures:
- Emails never contain broken attachment links.
- Large file uploads don't block the case creation response.
- The email includes the complete case context.

**Step 5 — Domain-Specific Integration:**
For the **email/SendGrid** domain:
- Case communication flows through email.
- SendGrid handles delivery, tracking, and bounce management.
- Email templates are configured per case type and severity.
- Attachments are included as S3 pre-signed URL links (not inline) to avoid email size limits.

For the **panhpe/DCM CRM** domain:
- Cases are synced to HPE's DCM (Device Case Management) CRM.
- The CRM integration uses its own API with specific field mappings.
- Contact management maps RAVE contacts to CRM contact records.
- Status updates are bidirectional — CRM status changes sync back to RAVE.

**Step 6 — Comprehensive Field Documentation:**
I created and published comprehensive Manual Case Payload documentation to Confluence covering:
- All WellnessModel fields with types, requirements, and constraints.
- All ManualCaseData fields with domain-specific requirements.
- Validation rules matrix — which fields are required per domain.
- Email configuration options (template selection, CC behavior).
- Contact field mappings between RAVE and CRM.
- Attachment workflow sequence diagram.
- Example payloads for each domain.

This documentation became the cross-team reference for any service integrating with the manual case API.

### Result
- **End-to-end manual case pipeline** serving both email and CRM communication domains.
- **CC deduplication** eliminated duplicate email recipients — reduced customer confusion from duplicate notifications.
- **Deferred email send** ensured 100% of emails contain valid attachment links.
- **Server-derived createdBy** from JWT replaced client-supplied value, closing an audit gap.
- **Input validation** caught malformed payloads at the API boundary — reduced downstream errors by 80%.
- **Confluence documentation** became the authoritative reference, reducing cross-team Slack questions about the payload format.
- Delivered across multiple Jira issues in the manual case workflow category.

## Interview Delivery Tips

### How to Open This Story
"I built the end-to-end manual case creation pipeline supporting two communication domains — email via SendGrid and CRM via HPE's DCM system. Key challenges were contact management with CC deduplication, a deferred email send pattern that waits for attachment uploads, and ensuring the createdBy field comes from the JWT rather than the client."

### Time Budget (5-minute answer)
- Situation: 30 seconds (fragmented pipeline, two domains, attachment issues)
- Task: 30 seconds (6 responsibilities)
- Action: 3 minutes (deferred email send + CC deduplication are the highlights)
- Result: 1 minute (100% valid attachment links, reduced customer confusion)

### Pivot Points
- **If interviewer asks about workflow design**: Deferred send pattern, status tracking, event-driven redesign
- **If interviewer asks about email**: SendGrid integration, bounce management, CC deduplication
- **If interviewer asks about APIs**: Pre-signed S3 URLs, domain-specific validation, payload documentation
- **If interviewer asks about security**: createdBy from JWT, trust boundaries, audit trail

---

## Rubric Annotations

### Key Phrases/Concepts That MUST Appear
- Two communication domains: email/SendGrid and panhpe/DCM CRM
- CC deduplication (normalize, deduplicate, include createdBy)
- Deferred email send (attachments must be uploaded before email)
- createdBy from JWT `sub` claim (not client-supplied)
- Cross-field validation (domain-specific required fields)
- Pre-signed S3 upload URLs (client-direct upload)
- Comprehensive payload documentation on Confluence

### Common Mistakes
- Describing the workflow without distinguishing the two domains
- Not explaining WHY deferred email send is necessary
- Forgetting the CC deduplication problem — seems trivial but causes customer confusion
- Not mentioning the createdBy security fix
- Describing validation without cross-field rules

### Scoring Rubric [1-5]
- 1 — Poor: Cannot describe the manual case workflow or its domains
- 2 — Below Average: Describes case creation but misses contact management and deferred email
- 3 — Acceptable: Covers the workflow, contact dedup, and deferred email but light on domain differences
- 4 — Strong: Full narrative with both domains, deferred send, validation, and documentation
- 5 — Exceptional: All of the above plus discusses trade-offs (pre-signed URLs vs. server proxy), CRM sync challenges, and documentation-driven development

---

## Follow-Up Questions (Progressively Harder)

### Follow-Up 1: Why use pre-signed S3 URLs instead of proxying file uploads through your API server?
**Expected Answer**: Pre-signed URLs allow clients to upload directly to S3, which: (1) eliminates the API server as a bottleneck for large files — the server doesn't need memory or bandwidth for file content. (2) Reduces latency — direct S3 upload is faster than server proxy. (3) Scales independently — S3 handles any upload volume. (4) Reduces attack surface — the API server never handles file bytes (though it still validates metadata). Trade-offs: (a) You lose the ability to validate file content before it reaches S3 (magic byte checks happen as a post-upload step). (b) Pre-signed URLs have expiration — if the client is slow, the URL expires. (c) CORS must be configured on the S3 bucket for browser-based uploads.
**Red Flags**: "Pre-signed URLs are always better" without mentioning content validation trade-off.
**Green Flags**: Discusses the content validation trade-off, mentions CORS, explains the performance benefits.

### Follow-Up 2: How does the bidirectional CRM sync work? What happens when both systems update the case simultaneously?
**Expected Answer**: This is the classic distributed state synchronization problem. Our approach: (1) CRM-to-RAVE sync uses webhooks — CRM sends a notification on status change, RAVE updates its record. (2) RAVE-to-CRM sync is event-driven — case updates in RAVE push to CRM via its API. (3) Conflict resolution: last-write-wins with timestamp comparison. Each system records the last known state of the other. If a conflict is detected (both updated since last sync), we prefer the CRM state for business-process fields (status, priority) and RAVE state for technical fields (attachments, contacts). (4) Idempotency: each sync operation uses an idempotency key to prevent duplicate updates.
**Red Flags**: "We just update both systems and it works out."
**Green Flags**: Addresses conflict resolution, mentions last-write-wins with field-level preferences, idempotency keys.

### Follow-Up 3: How would you redesign this as an event-driven architecture?
**Expected Answer**: (1) Case creation publishes a `CaseCreated` event to Kafka. (2) Domain-specific consumers subscribe: the SendGrid consumer sends emails, the CRM consumer syncs to DCM. (3) Attachment upload completion publishes `AttachmentUploaded` events. (4) A saga orchestrator (or choreography) manages the workflow: waits for all `AttachmentUploaded` events before publishing `CaseReady` which triggers the email. (5) Benefits: decoupled domains, retryable consumers, event sourcing for audit trail. (6) Challenges: eventual consistency (case might be visible before email is sent), complex failure handling, and the need for a dead letter queue.
**Red Flags**: Doesn't understand event-driven patterns.
**Green Flags**: Describes saga pattern, mentions eventual consistency trade-off, discusses dead letter queues.

---

## Grilling Chain (6 questions drilling deeper)

### Q1: Walk me through the exact sequence of API calls for creating a manual case with 2 attachments via the email domain.
**Why asked**: Tests detailed system knowledge and ability to explain a complex workflow.
**Expected answer**: (1) POST `/v3/cases` with `domain: email`, case metadata, contact info, attachment metadata (2 items with filenames/sizes). (2) Server validates payload, creates case in DB with status `PENDING_ATTACHMENTS`, generates 2 pre-signed S3 upload URLs. (3) Response: 201 with case ID, 2 pre-signed URLs, and attachment IDs. (4) Client PUTs file 1 to S3 pre-signed URL 1. (5) Client PUTs file 2 to S3 pre-signed URL 2. (6) S3 events (or client callbacks) notify server that uploads are complete. (7) Server verifies both attachments uploaded (ETags match), transitions case to `CREATED`. (8) Server sends email via SendGrid API with case details and S3 pre-signed download URLs for the 2 attachments. (9) Primary contact, alternate contacts, CC contacts (deduplicated), and createdBy all receive the email.
**Red flags**: Can't describe the sequence; confuses upload and download pre-signed URLs.
**Green flags**: Correct sequence, distinguishes upload vs. download URLs, mentions the deferred email trigger.

### Q2: How did you handle email bounce management with SendGrid?
**Why asked**: Tests real-world email delivery knowledge.
**Expected answer**: SendGrid provides webhooks for delivery events: delivered, bounced, dropped, spam report. For bounces: (1) Soft bounces (temporary — mailbox full) are retried by SendGrid automatically. (2) Hard bounces (permanent — invalid address) are reported via webhook. (3) We log bounce events against the case and contact. (4) For hard bounces, the contact's email is flagged, and subsequent cases avoid sending to that address. (5) Suppression lists — SendGrid maintains a suppression list of bounced addresses; we sync this to prevent sending to known-bad addresses. (6) Monitoring: we track bounce rate as a metric and alert if it spikes (could indicate email reputation issues).
**Red flags**: "SendGrid handles everything automatically."
**Green flags**: Distinguishes soft/hard bounces, mentions suppression lists and bounce rate monitoring.

### Q3: Explain the CC deduplication algorithm in detail. What edge cases did you handle?
**Why asked**: Tests attention to detail in seemingly simple logic.
**Expected answer**: (1) Collect all emails: primary.email, each alternate[].email, each cc[].email, createdBy email. (2) Normalize: lowercase, trim whitespace, remove dots in Gmail addresses (optional — we didn't do this). (3) Build a set of unique normalized emails. (4) Assign roles: primary is always TO, others are CC. (5) Edge cases: (a) Primary email appears in CC list — remove from CC (already in TO). (b) CreatedBy email matches primary — don't add duplicate. (c) Alternate contact has same email as another alternate — deduplicate. (d) Empty/null emails — skip. (e) Case insensitivity: `User@HPE.com` == `user@hpe.com`. (f) Plus addressing: `user+tag@hpe.com` — treated as different address (intentional routing).
**Red flags**: Doesn't consider case insensitivity or null handling.
**Green flags**: Handles case normalization, null emails, and explains the role assignment logic.

### Q4: Why extract createdBy from the JWT sub claim instead of trusting the client?
**Why asked**: Tests security mindset.
**Expected answer**: Client-supplied `createdBy` is a trust boundary violation. A client can claim to be anyone. The JWT `sub` claim is cryptographically signed by the identity provider — it can't be forged without the private key. Using the JWT claim ensures: (1) Audit trail integrity — the recorded creator is definitively the authenticated user. (2) Non-repudiation — the user can't deny creating the case. (3) Authorization consistency — the creator's identity drives permission checks. This is the same principle as the GLCP-333624 workspace fix: derive security-relevant context from the token, not the client.
**Red flags**: "The client wouldn't lie about who they are."
**Green flags**: Mentions trust boundary, non-repudiation, parallels to GLCP-333624.

### Q5: How would you handle email delivery failures for critical cases?
**Why asked**: Tests resilience design for a critical workflow.
**Expected answer**: (1) Retry with exponential backoff — SendGrid API returns 5xx or network error, retry 3 times with increasing delay. (2) Dead letter queue — after retries exhausted, push to a DLQ for manual review. (3) Fallback notification — if email fails, send a Slack/webhook notification to the support team with the case details. (4) Case status reflects delivery: `EMAIL_SENT`, `EMAIL_FAILED`. (5) Monitoring: alert on email delivery failure rate > threshold. (6) For P1 cases, implement a parallel notification path (e.g., SMS via Twilio) so the recipient is reached even if email fails.
**Red flags**: "If the email fails, the case still exists, so it's fine."
**Green flags**: Describes retry strategy, dead letter queue, fallback channels, and case status tracking.

### Q6: If you were to add a third communication domain (e.g., Slack-based cases), what would you change?
**Why asked**: Tests extensibility thinking and architecture flexibility.
**Expected answer**: (1) Abstract the domain-specific logic behind a `CommunicationDomain` interface: `SendNotification(case, contacts)`, `SyncStatus(case, newStatus)`, `ValidatePayload(payload)`. (2) Each domain (email, CRM, Slack) implements this interface. (3) Domain selection is driven by the `domain` field in the request — a factory pattern creates the correct implementation. (4) Validation rules become configurable per domain (Slack might need a channel ID instead of email addresses). (5) The core case creation logic remains unchanged — only the notification and sync paths differ. (6) For Slack specifically: use Slack API for notifications, threaded messages for case updates, reactions for quick status changes.
**Red flags**: Describes copy-pasting the entire workflow for Slack.
**Green flags**: Proposes an interface/strategy pattern, explains what changes vs. what stays the same.

---

## Tags & Cross-References

### Related STAR Stories
- [File Upload Security](file-upload-security.md) — attachment security pipeline used by this workflow
- [Cross-Workspace Security](cross-workspace-security.md) — createdBy from JWT follows the same trust principle
- [API Versioning Evolution](api-versioning-evolution.md) — manual case endpoints versioned through the API evolution

### Interview Question Categories This Covers
- System Design: Workflow orchestration, deferred operations, multi-domain support
- API Design: Input validation, cross-field rules, domain-specific payloads
- Integration: SendGrid email, CRM sync, S3 pre-signed URLs
- Software Engineering: Interface/strategy pattern, DRY, documentation-driven development

### Behavioral Dimensions Demonstrated
- **Attention to detail**: CC deduplication, validation edge cases
- **User empathy**: Eliminated duplicate email confusion for customers
- **Documentation**: Comprehensive Confluence payload docs reduced Slack noise
- **End-to-end ownership**: From validation to email delivery to CRM sync
