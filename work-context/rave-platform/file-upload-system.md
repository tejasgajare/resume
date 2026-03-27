# File Upload & Attachment System — RAVE Platform

## Overview

I built the multi-layer file upload validation and attachment pipeline for RAVE's manual case creation workflow. This system handles user-uploaded files (logs, screenshots, archives) that accompany support cases. Security was the primary design driver — file uploads are a classic attack surface, so I implemented defense-in-depth with filename sanitization, extension allowlisting, magic byte content verification, path traversal prevention, and size limits. Files flow through validation into S3 storage, with status tracking integrated into the wellness object lifecycle.

---

## Multi-Layer Validation Pipeline

```
File Upload Request
    │
    ▼
[1. Size Check]
    │ Reject > 10 MiB → 413 Payload Too Large
    │
    ▼
[2. Filename Sanitization]
    │ Strip path components
    │ Reject ../ sequences → 400
    │ Reject null bytes (\x00) → 400
    │ Normalize filename
    │
    ▼
[3. Extension Allowlist]
    │ Extract extension from sanitized filename
    │ Check against 60+ allowed extensions
    │ Reject unlisted extensions → 415
    │
    ▼
[4. Content-Type Verification]
    │ Read first 512 bytes (magic bytes)
    │ Detect actual content type
    │ Forward check: magic bytes → expected content type
    │ Reverse check: claimed extension → expected magic bytes
    │ Reject mismatch → 415
    │
    ▼
[5. S3 Upload]
    │ Generate unique key
    │ Upload to S3 bucket
    │ Record status: fileUploading → fileUploaded
    │
    ▼
[6. Attachment Tracking]
    │ Update wellness object with attachment metadata
    │ Track lifecycle: fileUploaded → fileAttachedToCase
```

**Why multi-layer**: Each layer catches different attack vectors. Size check prevents DoS. Filename sanitization prevents path traversal. Extension allowlist prevents executable uploads. Content-type verification prevents extension spoofing (renaming `malware.exe` to `malware.log`). No single layer is sufficient — it's defense in depth.

---

## Layer 1: Size Check

```go
const MaxFileSize = 10 * 1024 * 1024  // 10 MiB

func validateFileSize(r *http.Request) error {
    r.Body = http.MaxBytesReader(nil, r.Body, MaxFileSize)
    // MaxBytesReader will return an error if the body exceeds the limit
    // This also protects against slow-loris style attacks by limiting total bytes
    return nil
}
```

**Why 10 MiB**: Balances usability (log files, screenshots, small archives) with resource protection. Most support-relevant files fit within this limit. Larger files (core dumps, full system logs) are handled through separate channels.

---

## Layer 2: Filename Sanitization (GLCP-333057)

```go
func sanitizeFilename(filename string) (string, error) {
    // 1. Reject null bytes (can cause string truncation in C-based systems)
    if strings.ContainsRune(filename, '\x00') {
        return "", ErrNullByteInFilename
    }
    
    // 2. Reject path traversal sequences
    if strings.Contains(filename, "..") {
        return "", ErrPathTraversal
    }
    if strings.Contains(filename, "../") || strings.Contains(filename, "..\\") {
        return "", ErrPathTraversal
    }
    
    // 3. Strip directory components — only keep the base filename
    filename = filepath.Base(filename)
    
    // 4. Reject if filepath.Base returned "." or "/" (empty/root)
    if filename == "." || filename == "/" || filename == "\\" {
        return "", ErrInvalidFilename
    }
    
    // 5. Strip leading dots (hidden files on Unix)
    filename = strings.TrimLeft(filename, ".")
    if filename == "" {
        return "", ErrInvalidFilename
    }
    
    // 6. Replace problematic characters
    // Spaces, special chars that could cause issues in S3 keys or email attachments
    replacer := strings.NewReplacer(
        " ", "_",
        "#", "_",
        "%", "_",
        "&", "_",
        "{", "_",
        "}", "_",
    )
    filename = replacer.Replace(filename)
    
    return filename, nil
}
```

### Attack Vectors Prevented

| Attack | Example | Prevention |
|--------|---------|------------|
| **Path traversal** | `../../../etc/passwd` | `..` rejection + `filepath.Base` |
| **Null byte injection** | `file.txt\x00.exe` | Null byte check |
| **Hidden file creation** | `.bashrc` | Leading dot stripping |
| **Directory escape** | `foo/../../bar` | `filepath.Base` extracts only filename |
| **Windows traversal** | `..\..\..\windows\system32\` | `..` and `..\\` rejection |

---

## Layer 3: Extension Allowlist

```go
var AllowedExtensions = map[string]bool{
    // Log files
    ".log": true, ".txt": true, ".csv": true, ".json": true,
    ".xml": true, ".yaml": true, ".yml": true, ".conf": true,
    ".cfg": true, ".ini": true, ".properties": true,
    
    // Archive files
    ".zip": true, ".tar": true, ".gz": true, ".tgz": true,
    ".bz2": true, ".xz": true, ".7z": true, ".rar": true,
    
    // Document files
    ".pdf": true, ".doc": true, ".docx": true, ".xls": true,
    ".xlsx": true, ".ppt": true, ".pptx": true, ".rtf": true,
    ".odt": true, ".ods": true, ".odp": true,
    
    // Image files
    ".png": true, ".jpg": true, ".jpeg": true, ".gif": true,
    ".bmp": true, ".tiff": true, ".tif": true, ".svg": true,
    ".ico": true, ".webp": true,
    
    // Video files
    ".mp4": true, ".avi": true, ".mov": true, ".wmv": true,
    ".mkv": true, ".webm": true,
    
    // Web files
    ".html": true, ".htm": true, ".css": true, ".js": true,
    
    // HPE-specific log formats
    ".ilo": true, ".ssacli": true, ".hplog": true,
    ".supportdump": true, ".diag": true,
    
    // System logs
    ".dmesg": true, ".syslog": true, ".kern": true,
    ".messages": true, ".dump": true, ".core": true,
}

func validateExtension(filename string) error {
    ext := strings.ToLower(filepath.Ext(filename))
    if ext == "" {
        return ErrNoExtension
    }
    if !AllowedExtensions[ext] {
        return fmt.Errorf("%w: %s", ErrDisallowedExtension, ext)
    }
    return nil
}
```

**Why allowlist over denylist**: A denylist (blocking `.exe`, `.bat`, `.sh`) will always miss new dangerous extensions. An allowlist explicitly permits only known-safe types. If a new file type is needed, it's a conscious decision to add it.

**Why 60+ extensions**: Support cases for HPE infrastructure require diverse file types. Network engineers send packet captures, sysadmins send log files, storage engineers send diagnostic dumps. Restricting to just `.txt` and `.pdf` would cripple the feature's utility.

---

## Layer 4: Content-Type Verification (Magic Bytes)

### Forward Check: Magic Bytes → Expected Content Type

```go
// magicBytes maps file signatures to expected content types
var magicBytes = map[string][]byte{
    "application/pdf":  {0x25, 0x50, 0x44, 0x46},           // %PDF
    "image/png":        {0x89, 0x50, 0x4E, 0x47},           // .PNG
    "image/jpeg":       {0xFF, 0xD8, 0xFF},                  // JFIF/EXIF
    "image/gif":        {0x47, 0x49, 0x46, 0x38},           // GIF8
    "application/zip":  {0x50, 0x4B, 0x03, 0x04},           // PK..
    "application/gzip": {0x1F, 0x8B},                         // gzip
    "application/x-7z": {0x37, 0x7A, 0xBC, 0xAF, 0x27, 0x1C}, // 7z
    "application/x-rar": {0x52, 0x61, 0x72, 0x21, 0x1A, 0x07}, // Rar!
    "video/mp4":        {0x00, 0x00, 0x00},                  // ftyp (offset varies)
    // ... more entries
}

func detectContentType(data []byte) string {
    // 1. Check Go's built-in detection (reads first 512 bytes)
    detected := http.DetectContentType(data)
    
    // 2. If generic "application/octet-stream", try our magic bytes
    if detected == "application/octet-stream" {
        for contentType, magic := range magicBytes {
            if bytes.HasPrefix(data, magic) {
                return contentType
            }
        }
    }
    
    return detected
}
```

### Reverse Check: Claimed Extension → Expected Magic Bytes

```go
// extensionToContentType maps extensions to expected content types
var extensionToContentType = map[string]string{
    ".pdf":  "application/pdf",
    ".png":  "image/png",
    ".jpg":  "image/jpeg",
    ".jpeg": "image/jpeg",
    ".gif":  "image/gif",
    ".zip":  "application/zip",
    ".gz":   "application/gzip",
    // ... more entries
}

func verifyContentType(filename string, data []byte) error {
    ext := strings.ToLower(filepath.Ext(filename))
    
    // For text-based formats, skip magic byte check
    // (text files don't have reliable magic bytes)
    if isTextExtension(ext) {
        return nil
    }
    
    expectedType, hasMapping := extensionToContentType[ext]
    if !hasMapping {
        // Extension allowed but no content type mapping — permit
        // (HPE-specific formats may not have magic bytes)
        return nil
    }
    
    detectedType := detectContentType(data)
    if detectedType != expectedType {
        return fmt.Errorf("%w: expected %s, got %s", 
            ErrContentTypeMismatch, expectedType, detectedType)
    }
    
    return nil
}
```

### Why Both Forward and Reverse Checks

| Check | What it catches | Example |
|-------|----------------|---------|
| **Forward** (bytes → type) | File with wrong extension | `virus.exe` renamed to `virus.log` — magic bytes reveal it's an executable |
| **Reverse** (ext → expected bytes) | Extension that doesn't match content | `report.pdf` that's actually a ZIP — extension says PDF but bytes say ZIP |

**Why text files are exempt**: Text-based formats (`.log`, `.txt`, `.csv`, `.json`, `.xml`, `.yaml`) don't have reliable magic bytes. A JSON file and a plain text file are both UTF-8 text. We rely on the extension allowlist for these — they're inherently safe since they're not executable.

---

## Layer 5: S3 Upload Pipeline

```go
func (s *S3Uploader) Upload(ctx context.Context, file UploadedFile) (*UploadResult, error) {
    // 1. Generate unique S3 key
    key := fmt.Sprintf("%s/%s/%s/%s", 
        file.WorkspaceID,
        file.WellnessObjectID,
        uuid.New().String(),
        file.SanitizedFilename,
    )
    
    // 2. Update attachment status: fileUploading
    if err := s.updateAttachmentStatus(ctx, file.WellnessObjectID, 
        file.AttachmentID, StatusFileUploading); err != nil {
        return nil, err
    }
    
    // 3. Upload to S3
    input := &s3.PutObjectInput{
        Bucket:      aws.String(s.bucket),
        Key:         aws.String(key),
        Body:        file.Reader,
        ContentType: aws.String(file.ContentType),
        Metadata: map[string]*string{
            "workspace-id":      aws.String(file.WorkspaceID),
            "wellness-object-id": aws.String(file.WellnessObjectID),
            "original-filename": aws.String(file.OriginalFilename),
            "uploaded-by":       aws.String(file.UploadedBy),  // from JWT sub
        },
    }
    
    result, err := s.client.PutObject(input)
    if err != nil {
        // 4a. Upload failed — mark as fileFailed
        s.updateAttachmentStatus(ctx, file.WellnessObjectID, 
            file.AttachmentID, StatusFileFailed)
        return nil, fmt.Errorf("S3 upload failed: %w", err)
    }
    
    // 4b. Upload succeeded — mark as fileUploaded
    s.updateAttachmentStatus(ctx, file.WellnessObjectID, 
        file.AttachmentID, StatusFileUploaded)
    
    return &UploadResult{
        Key:      key,
        Bucket:   s.bucket,
        ETag:     *result.ETag,
        Location: fmt.Sprintf("s3://%s/%s", s.bucket, key),
    }, nil
}
```

### S3 Key Structure

```
{workspaceId}/{wellnessObjectId}/{uuid}/{sanitizedFilename}
```

- **workspaceId**: Tenant isolation at the S3 level
- **wellnessObjectId**: Groups attachments with their case
- **uuid**: Prevents filename collisions
- **sanitizedFilename**: Human-readable for debugging

---

## Layer 6: Attachment Lifecycle Tracking

### Status State Machine

```
                    ┌──────────────┐
                    │ fileUploading │
                    └──────┬───────┘
                           │
                    ┌──────┴──────┐
                    │              │
              (success)       (failure)
                    │              │
                    ▼              ▼
            ┌──────────────┐  ┌──────────┐
            │ fileUploaded  │  │ fileFailed│
            └──────┬───────┘  └──────────┘
                   │
                   │ (case creation includes attachment)
                   ▼
         ┌───────────────────┐
         │ fileAttachedToCase │
         └───────────────────┘
```

### MongoDB Attachment Document

```go
type Attachment struct {
    ID               string    `bson:"id"`
    Filename         string    `bson:"filename"`
    OriginalFilename string    `bson:"originalFilename"`
    ContentType      string    `bson:"contentType"`
    Size             int64     `bson:"size"`
    S3Key            string    `bson:"s3Key"`
    S3Bucket         string    `bson:"s3Bucket"`
    Status           string    `bson:"status"`
    UploadedBy       string    `bson:"uploadedBy"`
    UploadedAt       time.Time `bson:"uploadedAt"`
    AttachedAt       time.Time `bson:"attachedAt,omitempty"`
    ErrorMessage     string    `bson:"errorMessage,omitempty"`
}
```

---

## Attachment Integration with Case Creation

### Deferred Email Flow (GLCP-323886)

When a manual case includes attachments and uses the email domain (SendGrid):

1. **Case created without attachments** → Initial email sent to case team
2. **Files uploaded in parallel** → Status tracked per attachment
3. **All files uploaded** → caseRelay sends follow-up email with attachments
4. **Retry logic**: If an upload fails, retry up to 3 times before marking fileFailed

```
POST /v3/wellness (with attachments)
    │
    ├──▶ Create wellness object (status: caseCreating)
    │
    ├──▶ Send initial case email (no attachments)
    │
    ├──▶ Upload files to S3 (parallel)
    │     ├── file1: fileUploading → fileUploaded
    │     ├── file2: fileUploading → fileUploaded
    │     └── file3: fileUploading → fileUploaded
    │
    └──▶ All uploaded? Send follow-up email with attachments
          └── Mark all: fileAttachedToCase
```

**Why deferred**: SendGrid has attachment size limits and the upload is asynchronous. Sending the initial email immediately means the support team knows about the case right away. Attachments follow once uploaded. This is better UX than blocking the entire case creation on file uploads.

### CRM Attachment Flow (panhpe domain)

For CRM cases, attachments are added to the CRM case via the DCM API after upload:

1. **Create CRM case** → Get case ID from DCM
2. **Upload files to S3** → Status tracking
3. **Attach to CRM case** → DCM API call with S3 pre-signed URL
4. **Status update** → `fileAttachedToCase`

---

## Security Considerations

### Why These Specific Validations

| Validation | Threat | Severity |
|------------|--------|----------|
| Size limit | DoS via large uploads | High |
| Path traversal | File system escape | Critical |
| Null byte | String truncation attacks | High |
| Extension allowlist | Executable upload | Critical |
| Magic byte verification | Extension spoofing | High |
| S3 key isolation | Cross-tenant file access | Critical |

### What's NOT in This System (By Design)

- **Antivirus scanning**: Handled by separate infrastructure (not in upload pipeline)
- **Deep content analysis**: Beyond magic bytes, we don't parse file internals
- **Compression bomb detection**: Zip bomb detection is done at a different layer

---

## Related Jira Issues

| Issue | Description | My Contribution |
|-------|-------------|-----------------|
| GLCP-333057 | Attachment filename validation | Built the sanitization pipeline |
| GLCP-323886 | Attachment retry logic | Implemented deferred email flow with retry |
| GLCP-325722 | Input validation gaps | Added content-type verification |
| GLCP-325650 | Email format validation | Validated email contacts with attachments |

---

### Office XML Validation Fix (GLCP-340302, GLCP-339197)

**Bug**: .xlsx, .docx, .pptx files were rejected despite being supported formats.
- Error: `file content detected as 'application/zip' (archive) is not compatible with extension '.xlsx'`
- Root cause: Office Open XML formats are internally ZIP archives
- `http.DetectContentType` correctly identifies them as `application/zip`
- `extensionCategoryCompat` had no entry for `.xlsx`, `.docx`, `.pptx`
- The reverse check's else-branch rejected any extension not explicitly mapped

**Fix**: 
- Added explicit Office XML magic-byte verification in `ValidateFileContent()`
- Added extensionCategoryCompat mappings for all Office Open XML extensions
- Also fixed empty file upload validation (GLCP-339197)

**Interview insight**: This is a great example of understanding file format internals. Office Open XML is a ZIP container with XML manifests inside — knowing that ZIP is the container format is crucial for correct content-type validation.

---

## [KEY_POINTS]

- 6-layer defense-in-depth: size → filename → extension → content-type → S3 upload → lifecycle tracking
- Extension allowlist (not denylist) with 60+ types covering HPE support use cases
- Magic byte verification does both forward (bytes→type) and reverse (ext→expected bytes) checks
- Deferred email flow: case email first, attachment email after all files upload
- S3 key includes workspaceId for tenant isolation at the storage level

## [COMMON_MISTAKES]

- Thinking allowlist is restrictive — 60+ types covers all reasonable support file types
- Assuming magic bytes work for text files — they don't, text files are allowed by extension only
- Forgetting the deferred email pattern — attachments don't block initial case creation
- Overlooking null byte injection — it's a real attack on C-based systems that process filenames

## [FOLLOW_UP]

- "What if someone uploads a zip bomb?" → Handled at infrastructure level, not in this pipeline
- "Why not scan with antivirus in the upload path?" → Latency; AV scanning happens asynchronously
- "How do you handle upload failures?" → Retry 3x, mark fileFailed, case still created without attachment
- "What about very large support bundles?" → Separate upload channel for files > 10 MiB
