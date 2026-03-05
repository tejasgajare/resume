# Manual Case Creation Pipeline — RAVE Platform

## Overview

I built and hardened the manual case creation pipeline in RAVE. This allows HPE GreenLake users to manually create support cases through the GLCP UI, as opposed to the automated wellness alert pipeline. Manual cases flow through wellnessProducer for validation, then route to either caseRelay (email via SendGrid) or crmRelay (PAN HPE / DCM CRM API) based on the case domain. I implemented the validation pipeline, contact handling, attachment integration, and several critical security fixes.

---

## Two Domains

### Email Domain (SendGrid)

```
Client → wellnessProducer → Kafka → caseRelay → SendGrid API → Support Team Inbox
```

- **Use case**: Cases routed to support teams via email
- **Backend**: SendGrid API for email delivery
- **Features**: HTML email templates, CC recipients, attachment support
- **Deferred attachments**: Initial email sent immediately, follow-up with attachments

### PAN HPE Domain (DCM CRM API)

```
Client → wellnessProducer → Kafka → crmRelay → DCM CRM API → HPE CRM System
```

- **Use case**: Cases created directly in HPE's CRM system
- **Backend**: PAN HPE / DCM (Data Center Management) CRM API
- **Features**: CRM case lifecycle, contact association, SLA tracking
- **Attachments**: Uploaded to S3, then attached to CRM case via DCM API

**Why two domains**: Different HPE customers and product lines use different support workflows. Some teams manage cases via email; others use the enterprise CRM. The domain field lets the UI route cases to the appropriate backend without the user knowing the underlying system.

---

## WellnessModel Payload Structure

### Full Request Payload

```json
{
  "deviceId": "device-serial-123",
  "platformId": "platform-uuid",
  "deviceType": "nimble-storage",
  "alertType": "manual",
  "severity": "high",
  "description": "Storage array reporting intermittent IO errors on drive bay 3",
  "manualCaseData": {
    "domain": "email",
    "category": "hardware-failure",
    "subcategory": "disk-error",
    "subject": "IO errors on Nimble array SN-12345",
    "description": "Customer reports intermittent read failures since 2024-01-15. Impact: degraded RAID performance.",
    "primaryContact": {
      "firstName": "John",
      "lastName": "Smith",
      "email": "john.smith@customer.com",
      "phone": "+1-555-0100"
    },
    "alternateContact": {
      "firstName": "Jane",
      "lastName": "Doe",
      "email": "jane.doe@customer.com",
      "phone": "+1-555-0101"
    },
    "attachments": [
      {
        "filename": "array-diagnostics.log",
        "contentType": "text/plain",
        "size": 245760
      }
    ]
  }
}
```

### WellnessModel Go Struct

```go
type WellnessModel struct {
    ID              primitive.ObjectID  `bson:"_id,omitempty" json:"id"`
    DeviceID        string              `bson:"deviceId" json:"deviceId" validate:"required"`
    PlatformID      string              `bson:"platformId" json:"platformId"`
    WorkspaceID     string              `bson:"workspaceId" json:"workspaceId"`
    DeviceType      string              `bson:"deviceType" json:"deviceType"`
    AlertType       string              `bson:"alertType" json:"alertType" validate:"required"`
    Severity        string              `bson:"severity" json:"severity"`
    Description     string              `bson:"description" json:"description"`
    Status          string              `bson:"status" json:"status"`
    ManualCaseData  *ManualCaseData     `bson:"manualCaseData,omitempty" json:"manualCaseData,omitempty"`
    Attachments     []Attachment        `bson:"attachments,omitempty" json:"attachments,omitempty"`
    CreatedBy       string              `bson:"createdBy" json:"createdBy"`
    CreatedAt       time.Time           `bson:"createdAt" json:"createdAt"`
    UpdatedAt       time.Time           `bson:"updatedAt" json:"updatedAt"`
}
```

---

## ManualCaseData Fields and Validation Rules

```go
type ManualCaseData struct {
    Domain           string    `bson:"domain" json:"domain" validate:"required,oneof=email panhpe"`
    Category         string    `bson:"category" json:"category" validate:"required"`
    Subcategory      string    `bson:"subcategory" json:"subcategory"`
    Subject          string    `bson:"subject" json:"subject" validate:"required,max=255"`
    Description      string    `bson:"description" json:"description" validate:"required,max=10000"`
    PrimaryContact   Contact   `bson:"primaryContact" json:"primaryContact" validate:"required"`
    AlternateContact *Contact  `bson:"alternateContact,omitempty" json:"alternateContact,omitempty"`
    CreatedBy        Contact   `bson:"createdBy" json:"createdBy"`  // Server-set from JWT
    Attachments      []AttachmentRef `bson:"attachments,omitempty" json:"attachments,omitempty"`
}

type Contact struct {
    FirstName string `bson:"firstName" json:"firstName" validate:"required"`
    LastName  string `bson:"lastName" json:"lastName" validate:"required"`
    Email     string `bson:"email" json:"email" validate:"required,email"`
    Phone     string `bson:"phone" json:"phone"`
}
```

### Validation Rules Table

| Field | Rule | Jira | Notes |
|-------|------|------|-------|
| `domain` | Required, must be "email" or "panhpe" | — | Determines routing backend |
| `category` | Required, must be in allowlist | GLCP-332469 | Prevents injection of arbitrary categories |
| `subcategory` | Optional | — | Free text within category |
| `subject` | Required, max 255 chars | GLCP-325722 | Truncated if over limit |
| `description` | Required, max 10000 chars | GLCP-325722 | Rich text support |
| `primaryContact.email` | **Required**, valid email format | GLCP-326100 | Was missing — I added this |
| `alternateContact.email` | Optional, but if present must be valid | GLCP-325650 | Email format validation |
| `createdBy` | **Server-set**, never from client | GLCP-333624 | From JWT `sub` claim |

---

## Validation Pipeline (Detailed)

### Step 1: Payload Parsing

```go
func (c *Controller) handleManualCase(w http.ResponseWriter, r *http.Request) {
    var model WellnessModel
    if err := json.NewDecoder(r.Body).Decode(&model); err != nil {
        respondError(w, 400, "Invalid JSON payload")
        return
    }
    // ...
}
```

### Step 2: ManualCaseData Required Check

```go
if model.ManualCaseData == nil {
    respondError(w, 400, "manualCaseData is required for manual cases")
    return
}
```

### Step 3: Domain Validation

```go
switch model.ManualCaseData.Domain {
case "email":
    // Route to caseRelay (SendGrid)
case "panhpe":
    // Route to crmRelay (DCM CRM API)
default:
    respondError(w, 400, fmt.Sprintf("Invalid domain: %s. Must be 'email' or 'panhpe'", 
        model.ManualCaseData.Domain))
    return
}
```

### Step 4: Category Validation (GLCP-332469)

```go
var allowedCategories = map[string]bool{
    "hardware-failure":    true,
    "software-issue":      true,
    "network-connectivity": true,
    "performance":         true,
    "configuration":       true,
    "security":            true,
    "firmware-update":     true,
    "data-protection":     true,
    "licensing":           true,
    "general-inquiry":     true,
    // ... more categories per product line
}

func validateCategory(category string) error {
    if !allowedCategories[category] {
        return fmt.Errorf("%w: %s", ErrInvalidCategory, category)
    }
    return nil
}
```

**Why allowlisted categories**: Categories map to CRM fields and email routing rules. Arbitrary categories would break downstream processing. The allowlist is maintained in configuration and can be updated per environment.

### Step 5: Contact Validation (GLCP-326100)

```go
func validateContacts(mcd *ManualCaseData) error {
    // Primary contact email is REQUIRED
    if mcd.PrimaryContact.Email == "" {
        return ErrPrimaryContactEmailRequired
    }
    
    // Validate email format
    if !isValidEmail(mcd.PrimaryContact.Email) {
        return fmt.Errorf("%w: %s", ErrInvalidEmailFormat, mcd.PrimaryContact.Email)
    }
    
    // Alternate contact email: optional, but if present must be valid
    if mcd.AlternateContact != nil && mcd.AlternateContact.Email != "" {
        if !isValidEmail(mcd.AlternateContact.Email) {
            return fmt.Errorf("%w: %s", ErrInvalidEmailFormat, mcd.AlternateContact.Email)
        }
    }
    
    // Primary contact name is required
    if mcd.PrimaryContact.FirstName == "" || mcd.PrimaryContact.LastName == "" {
        return ErrPrimaryContactNameRequired
    }
    
    return nil
}

func isValidEmail(email string) bool {
    // RFC 5322 compliant validation (simplified)
    // Uses regexp or net/mail.ParseAddress
    _, err := mail.ParseAddress(email)
    return err == nil
}
```

**What I fixed (GLCP-326100)**: The primary contact email validation was completely missing. A user could create a case with no email on the primary contact, which meant caseRelay would try to CC them on the case email and fail silently. I added the required check and format validation.

### Step 6: createdBy Server-Set from JWT (GLCP-333624)

```go
func (c *Controller) setCreatedBy(r *http.Request, mcd *ManualCaseData) {
    claims := r.Context().Value(hpeClaimsKey).(HPEClaims)
    
    // createdBy comes from the authenticated user, NEVER from the request
    mcd.CreatedBy = Contact{
        Email: claims.Subject,  // JWT sub claim = user email/ID
        // FirstName and LastName populated from user profile if available
    }
}
```

**Why server-set**: If `createdBy` came from the client request, an attacker could impersonate another user as the case creator. By extracting from the JWT `sub` claim, we ensure the case creator is always the authenticated user.

### Step 7: Workspace ID from JWT (GLCP-333624)

```go
func (c *Controller) setWorkspaceID(r *http.Request, model *WellnessModel) error {
    claims := r.Context().Value(hpeClaimsKey).(HPEClaims)
    
    // Workspace ID comes from token, not request body
    model.WorkspaceID = claims.HPECCSWorkspaceID
    
    // If request body included workspaceId, verify it matches
    if model.WorkspaceID != "" && model.WorkspaceID != claims.HPECCSWorkspaceID {
        // Log security event
        log.Warn("workspace ID mismatch: body=%s, token=%s", 
            model.WorkspaceID, claims.HPECCSWorkspaceID)
        return ErrWorkspaceMismatch  // 403 Forbidden
    }
    
    return nil
}
```

---

## Contact Handling and Email CC Logic

### CC Recipients for Email Domain Cases

When a case is created via the email domain (SendGrid), the following contacts are CC'd:

```go
func buildCCList(mcd *ManualCaseData) []string {
    ccSet := make(map[string]bool)  // Deduplication
    
    // 1. Primary contact — always CC'd
    if mcd.PrimaryContact.Email != "" {
        ccSet[strings.ToLower(mcd.PrimaryContact.Email)] = true
    }
    
    // 2. Alternate contact — CC'd if email provided
    if mcd.AlternateContact != nil && mcd.AlternateContact.Email != "" {
        ccSet[strings.ToLower(mcd.AlternateContact.Email)] = true
    }
    
    // 3. CreatedBy (from JWT) — always CC'd
    if mcd.CreatedBy.Email != "" {
        ccSet[strings.ToLower(mcd.CreatedBy.Email)] = true
    }
    
    // Convert set to slice
    var ccList []string
    for email := range ccSet {
        ccList = append(ccList, email)
    }
    
    return ccList
}
```

### Deduplication

**Why dedup**: The same person might be both the primary contact and the case creator. Without deduplication, they'd get multiple copies of the same email. The `ccSet` map ensures each email appears only once.

**Case-insensitive**: Email addresses are case-insensitive per RFC 5321. `John@Example.com` and `john@example.com` are the same recipient. I normalize to lowercase before dedup.

---

## Attachment Integration

### Upload Flow

```
1. Client POSTs /v3/wellness with manualCaseData (attachment metadata only)
   → Wellness object created with attachment status: "pending"

2. Client POSTs /v3/wellness/upload with file binary
   → File validated (6-layer pipeline — see file-upload-system.md)
   → File uploaded to S3
   → Attachment status: fileUploading → fileUploaded

3. All files uploaded:
   → Email domain: caseRelay sends follow-up email with attachments
   → PAN HPE domain: crmRelay attaches files to CRM case via DCM API
   → Attachment status: fileAttachedToCase
```

### Deferred Email Send (Email Domain)

```go
func (cr *CaseRelay) processCase(event CaseActionEvent) error {
    // 1. Send initial case email (no attachments)
    if err := cr.sendInitialEmail(event); err != nil {
        return err
    }
    
    // 2. Check if attachments are pending
    if len(event.ManualCaseData.Attachments) == 0 {
        return nil  // No attachments, we're done
    }
    
    // 3. Wait for all attachments to reach "fileUploaded" status
    // (this is event-driven via Kafka, not polling)
    // When all attachments are uploaded, a follow-up event triggers:
    cr.waitForAttachments(event.WellnessObjectID)
    
    // 4. Send follow-up email with attachments
    return cr.sendAttachmentEmail(event)
}
```

**Why deferred**: The case creation API response shouldn't be blocked by file uploads. The user gets immediate feedback ("Case created!"), and attachments flow in asynchronously. The support team gets the case info right away and attachments follow.

See: [file-upload-system.md](./file-upload-system.md) for complete upload validation details.

---

## Error Handling

### Validation Errors (400)

```json
{
  "error": "validation_error",
  "message": "Primary contact email is required",
  "field": "manualCaseData.primaryContact.email",
  "code": "REQUIRED_FIELD"
}
```

### Authorization Errors (403)

```json
{
  "error": "forbidden",
  "message": "Workspace ID in request does not match authenticated workspace",
  "code": "WORKSPACE_MISMATCH"
}
```

### Processing Errors

If case creation fails downstream (SendGrid/DCM):
- Event stays in Kafka for retry
- Wellness object status: `caseFailed`
- Alert raised in monitoring
- Manual intervention possible via admin API

---

## End-to-End Flow Example

### Email Domain Case with Attachments

```
1. User fills case form in GLCP UI
2. UI sends POST /v3/wellness with JWT (v1.1+)

3. wellnessProducer:
   a. Auth middleware validates JWT, extracts HPEClaims
   b. Validate ManualCaseData:
      - domain: "email" ✓
      - category: "hardware-failure" (in allowlist) ✓
      - primaryContact.email: "john@customer.com" (valid) ✓
      - createdBy: set from JWT sub = "tejas.gajare@hpe.com"
   c. workspaceID: set from JWT hpe_ccs_workspace_id
   d. Persist wellness object to MongoDB
   e. Publish case-action event to Kafka

4. caseRelay (consumes from Kafka):
   a. Receive case-action event
   b. Build email from template (device type, severity, description)
   c. CC list: ["john@customer.com", "tejas.gajare@hpe.com"] (deduped)
   d. Send initial email via SendGrid (no attachments yet)

5. User uploads attachment via POST /v3/wellness/upload:
   a. Size check: 245 KB < 10 MiB ✓
   b. Filename: "array-diagnostics.log" (sanitized, no path traversal) ✓
   c. Extension: ".log" (in allowlist) ✓
   d. Content-type: text file (skip magic bytes) ✓
   e. Upload to S3: workspace-id/wellness-obj-id/uuid/array-diagnostics.log
   f. Status: fileUploading → fileUploaded

6. caseRelay (attachment complete event):
   a. All attachments uploaded
   b. Download from S3
   c. Send follow-up email with attachment via SendGrid
   d. Status: fileAttachedToCase

7. Support team receives:
   - Initial email: Case details, severity, customer info
   - Follow-up email: Attachment (array-diagnostics.log)
```

---

## Related Jira Issues

| Issue | Description | My Contribution |
|-------|-------------|-----------------|
| GLCP-335836 | API modifications for manual cases | Built validation pipeline |
| GLCP-333624 | Cross-workspace case creation | Workspace ID from JWT |
| GLCP-326100 | Email contact handling | Primary email required validation |
| GLCP-325722 | Input validation gaps | Comprehensive field validation |
| GLCP-332469 | Category validation | Allowlisted categories |
| GLCP-325650 | Email format validation | RFC-compliant email checks |
| GLCP-336767 | Restrict manual cases | Authorization controls |
| GLCP-323886 | Attachment retry logic | Deferred email flow |

---

## [KEY_POINTS]

- Two domains (email/panhpe) route to different backends — same validation pipeline
- Primary contact email was missing validation — I added it (GLCP-326100)
- createdBy is server-set from JWT sub, never from client — prevents impersonation
- workspaceID from JWT, not request body — prevents cross-workspace case creation (GLCP-333624)
- Deferred email pattern: case email first, attachments follow asynchronously
- Contact CC list is deduplicated and case-insensitive

## [COMMON_MISTAKES]

- Thinking "email" and "panhpe" are just different email addresses — they're entirely different backends
- Assuming createdBy is client-supplied — it's a server-controlled field from JWT
- Forgetting contact deduplication — same person as primary + creator shouldn't get duplicate emails
- Overlooking the deferred attachment flow — attachments don't block case creation

## [FOLLOW_UP]

- "Why not validate all fields at once and return all errors?" → We do return first error for UX clarity, but could batch
- "What if SendGrid is down?" → Kafka retries, case stays in queue, monitoring alerts
- "How do you handle concurrent case creation from the same workspace?" → MongoDB atomicity, no distributed lock needed
- "Walk me through the contact CC logic" → Primary + alternate + createdBy, deduplicated, case-insensitive
