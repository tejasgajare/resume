# Multi-Layer File Upload Security for RAVE Platform

## STAR Narrative

### Situation
The RAVE Cloud Platform's case management system allows customers and support agents to attach files to support cases — firmware logs, screenshots, configuration exports, and more. The existing file upload endpoint had minimal validation: it checked file size and little else. This left the system vulnerable to a wide range of file upload attacks: path traversal (writing files outside the intended directory), MIME type spoofing (uploading executables disguised as images), zip bombs (compressed files that expand to enormous sizes), and malicious filename injection. As part of a broader security hardening initiative across 13 security/validation Jira issues, I was tasked with implementing comprehensive file upload security.

### Task
I was responsible for implementing multi-layer file upload security covering:
1. Filename sanitization to prevent injection attacks and path traversal.
2. File extension allowlisting with 60+ approved file types.
3. Content-type verification using magic bytes (file signatures) with both forward and reverse checks.
4. Path traversal prevention to ensure files can only be written to intended directories.
5. S3 upload workflow with status tracking for the complete attachment lifecycle.
6. Comprehensive error handling and security logging.

### Action
**Step 1 — Filename Sanitization:**
I implemented a sanitization pipeline for uploaded filenames:
1. Strip any path components — reject filenames containing `/`, `\`, `..`, or null bytes.
2. Normalize Unicode characters to prevent homoglyph attacks (e.g., Cyrillic `а` looking like Latin `a`).
3. Remove or replace characters that are problematic for filesystems or URLs: `<`, `>`, `:`, `"`, `|`, `?`, `*`.
4. Enforce maximum filename length (255 characters).
5. Generate a UUID-prefixed filename for storage (e.g., `{uuid}_{sanitized_name}`) to prevent collisions and make original filenames irrelevant for storage.

**Step 2 — Extension Allowlisting:**
Rather than trying to blocklist dangerous extensions (which is fragile and incomplete), I implemented a strict allowlist of 60+ file types organized by category:
- **Documents**: `.pdf`, `.doc`, `.docx`, `.xls`, `.xlsx`, `.ppt`, `.pptx`, `.txt`, `.csv`, `.rtf`
- **Images**: `.jpg`, `.jpeg`, `.png`, `.gif`, `.bmp`, `.svg`, `.tiff`, `.webp`
- **Archives**: `.zip`, `.tar`, `.gz`, `.7z`, `.rar`
- **Logs/Config**: `.log`, `.xml`, `.json`, `.yaml`, `.yml`, `.ini`, `.conf`, `.cfg`
- **Firmware/Technical**: `.bin`, `.hex`, `.elf`, `.iso`, `.img`

The allowlist is case-insensitive and checked after extracting the extension from the sanitized filename. Files with no extension or disallowed extensions are rejected with a descriptive error.

**Step 3 — Content-Type Verification via Magic Bytes:**
This is the most critical layer. Attackers can rename `malware.exe` to `malware.pdf`, so extension alone is insufficient. I implemented magic byte verification in two directions:

**Forward check** (file → expected type): Read the first N bytes of the file and identify the actual file type by matching against known magic byte signatures. For example:
- PDF: starts with `%PDF-` (hex: `25 50 44 46 2D`)
- PNG: starts with `89 50 4E 47 0D 0A 1A 0A`
- JPEG: starts with `FF D8 FF`
- ZIP: starts with `50 4B 03 04`

**Reverse check** (claimed type → file): Verify that the file's magic bytes are consistent with the claimed Content-Type header AND the file extension. If a file claims to be `image/png` but has JPEG magic bytes, it's rejected.

This dual check catches:
- Extension spoofing (`.pdf` file that's actually an executable)
- Content-Type header spoofing (server says `image/png`, file is actually a zip)
- Polyglot files (files valid as multiple types) — the magic bytes must match the claimed type

**Step 4 — Path Traversal Prevention:**
Beyond filename sanitization, I added defense-in-depth for path traversal:
1. Resolve the target path with `filepath.Clean()` and verify it's within the intended upload directory using `strings.HasPrefix()` after resolution.
2. Reject any filename that, after sanitization, would resolve outside the upload directory.
3. Use UUID-based storage keys for S3 — the original filename is stored as metadata, not as the object key.
4. S3 object keys are constructed server-side using `{case_id}/{uuid}` — no client input in the storage path.

**Step 5 — S3 Upload with Status Tracking:**
The attachment workflow includes lifecycle tracking:
1. **PENDING** — Upload initiated, file validated (steps 1-4 passed).
2. **UPLOADING** — File being streamed to S3.
3. **UPLOADED** — S3 confirms receipt (ETag returned).
4. **FAILED** — Upload failed (network error, S3 error) — client can retry.
5. **ATTACHED** — File linked to the case in the database.

This status tracking enables the deferred email send pattern for manual cases — email is only sent after all attachments reach UPLOADED status.

**Step 6 — Security Logging:**
Every upload attempt is logged with: filename (original and sanitized), claimed Content-Type, detected Content-Type (via magic bytes), file size, validation result, and the user/service that initiated the upload. Failed validations include the specific reason for rejection. This supports security auditing and incident investigation.

### Result
- **Multi-layer defense** catches attacks that any single layer would miss — defense-in-depth applied to file uploads.
- **60+ file types** supported while blocking all others — allowlist approach eliminates the "forgot to block" failure mode.
- **Zero file upload vulnerabilities** reported in subsequent security assessments.
- **Magic byte verification** caught 3 instances during testing where legitimate but misconfigured clients sent files with wrong Content-Type headers — the system correctly rejected them, and clients were fixed.
- **Path traversal eliminated** — UUID-based storage keys make the attack vector impossible.
- **Status tracking** enabled the deferred email send workflow for manual cases with attachments.
- Part of the 13 security/validation fixes tracked across GLCP-325645, GLCP-325722, GLCP-333057, and related issues.

## Interview Delivery Tips

### How to Open This Story
"I implemented multi-layer file upload security for our case management system — filename sanitization, a 60+ type extension allowlist, magic byte content verification with forward and reverse checks, path traversal prevention, and S3 upload with lifecycle tracking. This was part of 13 security fixes I delivered for the platform."

### Time Budget (5-minute answer)
- Situation: 30 seconds (minimal validation, attack surface)
- Task: 30 seconds (6 security layers)
- Action: 3 minutes (magic bytes and path traversal are the technical highlights)
- Result: 1 minute (zero vulnerabilities in subsequent assessments)

### Pivot Points
- **If interviewer asks about web security**: OWASP file upload guidelines, polyglot files, zip bombs
- **If interviewer asks about cloud**: S3 pre-signed URLs, UUID storage keys, lifecycle tracking
- **If interviewer asks about Go**: filepath.Clean, io.LimitReader, bytes comparison for magic bytes
- **If interviewer asks about testing**: Unit tests for each validation layer, edge cases (null bytes, Unicode)

---

## Rubric Annotations

### Key Phrases/Concepts That MUST Appear
- Magic bytes (file signatures) for content-type verification
- Forward AND reverse check (dual direction)
- Extension allowlisting (not blocklisting)
- Filename sanitization (path components, null bytes, special characters)
- Path traversal prevention (filepath.Clean + HasPrefix)
- UUID-based storage keys (removes filename from storage path)
- S3 upload with lifecycle status tracking

### Common Mistakes
- Relying on Content-Type header alone (trivially spoofed)
- Using extension blocklist instead of allowlist
- Forgetting about polyglot files
- Not explaining WHY magic bytes are needed in addition to extension checks
- Ignoring the storage path — path traversal via S3 key construction

### Scoring Rubric [1-5]
- 1 — Poor: Only mentions file size limits; doesn't understand file upload vulnerabilities
- 2 — Below Average: Mentions extension checking and Content-Type but not magic bytes or path traversal
- 3 — Acceptable: Covers magic bytes and extension allowlisting but misses the forward/reverse dual check and storage path security
- 4 — Strong: Full narrative with all 6 layers, explains why each is necessary
- 5 — Exceptional: All of the above plus discusses polyglot files, zip bombs, antivirus integration, and how the layers interact as defense-in-depth

---

## Follow-Up Questions (Progressively Harder)

### Follow-Up 1: Why use an allowlist instead of a blocklist for file extensions?
**Expected Answer**: A blocklist requires you to enumerate every dangerous extension (`.exe`, `.bat`, `.cmd`, `.ps1`, `.sh`, `.scr`, `.com`, `.msi`, `.dll`, `.vbs`, `.js`, `.hta`, etc.) and any new dangerous format you miss is a vulnerability. Attackers constantly find new executable formats. An allowlist is a closed set — only known-safe types are permitted. If a new file format emerges, it's blocked by default until explicitly added. The blocklist approach has the "default-allow" problem; the allowlist approach has "default-deny."
**Red Flags**: Says "blocklist is fine, just add extensions as you find them."
**Green Flags**: Explains default-deny principle, mentions the fragility of trying to enumerate all dangerous extensions.

### Follow-Up 2: What is a polyglot file, and how does your system handle it?
**Expected Answer**: A polyglot file is valid when interpreted as multiple file types. For example, a file can be simultaneously valid as a JPEG and a ZIP (JPEG ignores trailing data, ZIP reads from the end). An attacker could upload a file that passes magic byte checks as a JPEG but is also a valid ZIP containing malicious content. Our dual check mitigates this: the forward check identifies the file type from magic bytes, and the reverse check verifies consistency with the claimed type. Additionally, the extension must match the detected type. A JPEG/ZIP polyglot claiming to be `.jpg` would pass the JPEG magic byte check but could still be extracted as a ZIP by a vulnerable downstream processor. Full mitigation requires re-encoding the file (e.g., re-save the image) to strip non-image data.
**Red Flags**: Doesn't know what a polyglot file is.
**Green Flags**: Explains the JPEG/ZIP example, mentions re-encoding as a mitigation.

### Follow-Up 3: How would you protect against zip bombs?
**Expected Answer**: A zip bomb is a compressed file that expands to an enormous size (e.g., 42.zip is 42KB compressed, 4.5PB uncompressed). Protections: (1) Limit compressed file size at upload. (2) If decompression is needed, use a streaming decompressor that tracks decompressed size and aborts when a threshold is exceeded (e.g., 100:1 compression ratio or absolute size limit). (3) Limit recursion depth for nested archives (zip within zip within zip). (4) Use a sandbox for decompression — don't decompress on the application server. (5) For our system, we don't decompress uploads — archives are stored as-is and passed to downstream processing in a controlled environment.
**Red Flags**: "We just check the file size before upload."
**Green Flags**: Mentions compression ratio limits, streaming decompression, recursion depth limits, and sandbox processing.

---

## Grilling Chain (7 questions drilling deeper)

### Q1: Walk me through all the file upload vulnerabilities you're protecting against. Why is each dangerous?
**Why asked**: Tests comprehensive security knowledge.
**Expected answer**: (1) **Path traversal** — attacker uploads `../../etc/passwd` to overwrite system files. (2) **Extension spoofing** — `malware.pdf.exe` or renaming executables. (3) **Content-Type spoofing** — setting `Content-Type: image/png` for an executable so server-side processing treats it as an image. (4) **XXE in XML uploads** — XML files with external entity declarations that exfiltrate data. (5) **SVG with embedded JavaScript** — SVG files can contain `<script>` tags, leading to stored XSS if served to browsers. (6) **Zip bombs** — denial of service via decompression. (7) **Filename injection** — filenames containing shell metacharacters or SQL that get interpolated unsafely. (8) **Null byte injection** — `malware.php%00.jpg` bypasses extension checks in some languages.
**Red flags**: Can only name 1-2 vulnerabilities.
**Green flags**: Names 5+ with explanations of impact; mentions less obvious ones like XXE and SVG XSS.

### Q2: How do magic bytes work? Why can't an attacker just prepend the right magic bytes?
**Why asked**: Tests understanding of file format internals.
**Expected answer**: Magic bytes are fixed byte sequences at specific offsets that identify file formats. An attacker CAN prepend magic bytes — this creates a polyglot file that satisfies the magic byte check. However: (1) The file must also be PARSEABLE as the claimed type by downstream processors. Prepending PNG magic bytes to an executable doesn't make it a valid PNG — image processors will fail to parse it. (2) Our reverse check verifies the ENTIRE file is consistent with the claimed type, not just the header. (3) For maximum security, you can re-process the file through a type-specific parser (e.g., re-encode images with an image library) — this strips any non-standard data. Magic bytes aren't a silver bullet; they're one layer in the defense.
**Red flags**: Says magic bytes are foolproof; doesn't understand polyglot attacks.
**Green flags**: Acknowledges the limitation, explains re-encoding as a stronger mitigation.

### Q3: Your extension allowlist includes `.svg`. How do you handle SVG XSS?
**Why asked**: Tests awareness of a specific, subtle vulnerability.
**Expected answer**: SVG files are XML-based and can contain `<script>` tags, `onload` attributes, and `<foreignObject>` elements with embedded HTML. If an SVG is served with `Content-Type: image/svg+xml`, browsers will execute the JavaScript — stored XSS. Mitigations: (1) Serve uploaded SVGs with `Content-Disposition: attachment` to prevent inline rendering. (2) Strip SVG files through a sanitizer that removes all script elements and event handlers. (3) Set `Content-Security-Policy` headers that block inline scripts. (4) Consider converting SVGs to rasterized formats (PNG) for display. (5) In our case, attachments are stored in S3 and downloaded, not rendered inline — but if any future feature renders them, the sanitization must be in place.
**Red flags**: "SVGs are just images, no risk."
**Green flags**: Knows about script injection in SVGs, mentions Content-Disposition and CSP.

### Q4: Why did you use UUID-based storage keys in S3 instead of the original filename?
**Why asked**: Tests understanding of storage-level security.
**Expected answer**: (1) **Path traversal prevention** — the original filename is never used in the storage path, eliminating this entire attack class. (2) **Collision avoidance** — multiple uploads with the same filename don't overwrite each other. (3) **Information hiding** — the storage key reveals nothing about the file content, preventing enumeration attacks. (4) **Consistent key format** — `{case_id}/{uuid}` is always valid for S3, regardless of the original filename's characters. The original filename is stored as metadata on the S3 object for retrieval purposes.
**Red flags**: Says "UUIDs are just for uniqueness."
**Green flags**: Mentions path traversal elimination, enumeration prevention, and metadata preservation.

### Q5: How does the S3 upload status tracking interact with the manual case email workflow?
**Why asked**: Tests system integration understanding.
**Expected answer**: Manual cases support file attachments that must be included in the notification email. The email can't be sent until all attachments are uploaded to S3. The workflow: (1) User creates case with attachment metadata. (2) Attachments are uploaded asynchronously → status transitions: PENDING → UPLOADING → UPLOADED. (3) The case creation endpoint uses deferred email send — it doesn't trigger the email immediately. (4) After all attachments reach UPLOADED status, the system triggers the email send with S3 pre-signed URLs for the attachments. (5) If any attachment FAILS, the email is sent without it, and the user is notified of the failed attachment. This prevents sending emails with broken attachment links.
**Red flags**: Doesn't understand why deferred send is needed.
**Green flags**: Explains the async upload → deferred email pattern, mentions pre-signed URLs, handles the failure case.

### Q6: How would you add antivirus scanning to this pipeline?
**Why asked**: Tests ability to extend the security architecture.
**Expected answer**: Insert AV scanning as a step between UPLOADING and UPLOADED/ATTACHED: (1) Upload the file to a quarantine S3 bucket (not the production bucket). (2) Trigger a Lambda function (or container) that runs ClamAV or a commercial scanner on the file. (3) If clean: move to the production bucket, update status to UPLOADED. If infected: delete from quarantine, update status to REJECTED, log the detection. (4) The deferred email send pattern already handles this — the email waits for all attachments to reach final status. (5) Add a timeout — if scanning takes too long, mark as FAILED and alert. (6) Keep the quarantine bucket separate with restrictive access policies.
**Red flags**: "Just scan on the application server" (resource and security risk).
**Green flags**: Mentions quarantine bucket, async scanning, integration with the status tracking workflow.

### Q7: What would you do differently if building this from scratch?
**Why asked**: Tests reflection and awareness of best practices.
**Expected answer**: (1) Use a dedicated file processing service (not the API server) — isolate file handling in a sandboxed environment. (2) Implement content disarm and reconstruction (CDR) — re-create files from scratch rather than scanning them, stripping any embedded threats. (3) Add file size quotas per case and per user to prevent storage abuse. (4) Implement resumable uploads (tus protocol) for large files. (5) Use S3 Object Lock for compliance — prevent attachment deletion for audit retention. (6) Add file type-specific validation (e.g., validate that a PDF is well-formed, an image has valid dimensions). (7) Consider pre-signed upload URLs so files go directly to S3 without passing through the application server.
**Red flags**: "Nothing, the implementation is complete."
**Green flags**: Mentions CDR, pre-signed upload URLs, sandbox processing, resumable uploads.

---

## Tags & Cross-References

### Related STAR Stories
- [Manual Case Workflow](manual-case-workflow.md) — attachments use this upload security pipeline
- [Cross-Workspace Security](cross-workspace-security.md) — part of the same security hardening initiative (13 issues)
- [API Versioning Evolution](api-versioning-evolution.md) — upload endpoints versioned alongside case APIs

### Interview Question Categories This Covers
- Security: OWASP file upload, MIME sniffing, path traversal, defense-in-depth
- System Design: Multi-layer validation pipeline, S3 workflow, status tracking
- Cloud: S3 pre-signed URLs, UUID storage keys, lifecycle management
- Go Programming: filepath.Clean, bytes comparison, io.LimitReader patterns

### Behavioral Dimensions Demonstrated
- **Security expertise**: Multiple validation layers, magic byte implementation
- **Thoroughness**: 60+ file types researched and categorized
- **Defense-in-depth philosophy**: Each layer catches what others miss
- **Part of broader initiative**: 13 security fixes delivered as a cohesive effort
