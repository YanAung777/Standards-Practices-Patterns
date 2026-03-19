**Enterprise Security Standard – Secure Document Upload API (REST)**  
**Version**: 1.0 | **Classification**: Internal Use | **Effective Date**: \[Insert Date\]

**Applies to**: All REST APIs that accept file/document uploads from web, mobile, or partner applications

### **1\. Purpose**

File upload functionality represents one of the highest-risk features in modern applications due to the potential for remote code execution, web shells, denial-of-service, data exfiltration, and malware distribution. This standard defines mandatory security controls for any new or existing API endpoint that accepts document/file uploads, ensuring alignment with **OWASP**, **NIST**, and enterprise risk appetite.

### **2\. Scope**

* All RESTful APIs using multipart/form-data or similar mechanisms for file upload.  
* Excludes: pure metadata-only uploads without binary content.

### **3\. Normative References**

* OWASP File Upload Cheat Sheet (latest version)  
* OWASP REST Security Cheat Sheet  
* OWASP API Security Top 10  
* NIST SP 800-228 Guidelines for API Protection for Cloud-Native Systems  
* Enterprise Authentication & Authorization Standard  
* Enterprise Cryptography & Key Management Standard

### **4\. Mandatory Security Requirements**

#### **4.1 Authentication & Authorization (MUST)**

* Every upload request **MUST** be authenticated using a strong mechanism (OAuth 2.0 Bearer token, JWT with proper signature & expiration, or mutual TLS client certificate).  
* The authenticated principal **MUST** be explicitly authorized to perform uploads via role-based access control (RBAC) or attribute-based access control (ABAC).  
* Unauthenticated / unauthorized requests **MUST** return HTTP 401 or 403 (no detailed error).  
* Upload permissions **MUST** be scoped (e.g., per document type, per project/folder, per user quota).

#### **4.2 Transport Layer Security (MUST)**

* **HTTPS/TLS 1.3 only** (TLS 1.2 MAY be temporarily tolerated during migration).  
* HSTS header **MUST** be sent (Strict-Transport-Security: max-age=31536000; includeSubDomains).  
* HTTP requests to the upload endpoint **MUST** be redirected to HTTPS or rejected.

#### **4.3 API & Request Protections (MUST)**

* Endpoint **MUST** allow only POST method → return 405 Method Not Allowed otherwise.  
* Strict **CORS** policy: allow only trusted origins (enterprise webapp domains), POST method, and necessary headers (Authorization, Content-Type).  
* **CSRF** protection **MUST** be enforced (double-submit cookie, same-site cookies \+ token, or custom header validation for APIs).  
* Global and per-user **rate limitingMUST** be applied (e.g., 10–30 uploads per minute per authenticated user; configurable via API gateway/policy).  
* Total request body size **MUST** be limited (recommended: 10–50 MB depending on approved document types) → return 413 Payload Too Large.

#### **4.4 File Validation Pipeline (MUST – applied in this strict order)**

1. **Extension allow-list** — only business-approved extensions (e.g., .pdf, .docx, .xlsx, .pptx, .txt). **Deny-by-default**; never use a block-list.  
2. **File type / magic-byte validation** — verify actual content matches declared type using server-side libraries (Apache Tika, python-magic, file-type npm, etc.). **Reject** mismatches.  
3. **Filename sanitization & regeneration**:  
   * **NEVER** use client-supplied filename.  
   * Generate new filename: UUID v4 (or better cryptographically random identifier) \+ approved extension.  
   * Enforce character whitelist (alphanumeric, hyphen, underscore, dot).  
   * Maximum filename length: 255 bytes (filesystem safe).  
   * Strip or reject path traversal sequences (../, \\, %2e%2e%2f, null bytes, double extensions).  
4. **Per-file size limit** — enforced independently of request limit (e.g., max 25 MB per file).  
5. **Malware & content scanning**:  
   * All files **MUST** be scanned with enterprise-grade, multi-engine anti-malware (ClamAV \+ commercial engine, or CDR – Content Disarm & Reconstruction service).  
   * PDF / Office documents **MUST** undergo CDR to remove macros, embedded objects, JavaScript, tracking pixels, etc.  
   * ZIP / archive formats **MUST NOT** be accepted unless explicitly approved (high-risk); if allowed, recursively scan and limit nesting depth.  
6. **VirusTotal / equivalent external reputation check** MAY be applied for high-sensitivity environments (rate-limited & privacy reviewed).

#### **4.5 Storage & Processing (MUST)**

* Files **MUST NOT** be stored in the web application root or any executable directory.  
* Recommended: cloud object storage (S3, Azure Blob, GCS) with **private ACLs** (no public read).  
* Files **MUST** be written first to a **quarantine / staging** area, scanned, then moved to final location only if clean.  
* Filesystem permissions **MUST** follow least privilege (application process cannot execute files).  
* Metadata (original sanitized name, hash SHA-256, upload timestamp, uploader identity, scan result) **MUST** be stored in database / audit log — never in filename.  
* Files **MUST** be encrypted at rest (server-side encryption with KMS-managed keys) if containing regulated data (PII, PHI, financial).

#### **4.6 Serving / Download Controls (when applicable)**

* Downloads **MUST** go through an authorized proxy/handler endpoint (e.g., /files/{uuid}).  
* Handler **MUST** re-validate authorization on every download.  
* Response headers **MUST** include:  
  * Content-Disposition: attachment; filename="sanitized-name.ext"  
  * X-Content-Type-Options: nosniff  
  * Content-Security-Policy appropriate to prevent inline execution

#### **4.7 Logging, Monitoring & Incident Response (MUST)**

* Log every upload attempt (authenticated user/subject, IP, request size, filename hash, extension, scan result, success/failure).  
* Logs **MUST** be sent to centralized SIEM; retain ≥ 12 months for regulated environments.  
* Alert on anomalies: high failure rate, unusual file types, large/frequent uploads from single user.  
* Provide mechanism for users/admins to report suspicious uploaded content.

#### **4.8 Error Handling & Information Disclosure (MUST)**

* Return generic messages only (e.g., "Invalid file format", "Upload failed – contact support").  
* **NEVER** expose:  
  * server paths  
  * stack traces  
  * original filenames  
  * antivirus scan details  
* Use appropriate HTTP status codes (400, 413, 415, 422, 429, etc.).

#### **4.9 Additional Enterprise Controls (SHOULD / conditional MUST)**

* **Zero Trust** verification of every request (beyond authN).  
* Cryptographic hash (SHA-256) stored for integrity & duplicate detection.  
* Retention & deletion policy enforced (auto-purge after X days unless tagged).  
* Annual penetration testing focused on file-upload bypass techniques (polyglot files, magic-byte evasion, ZIP bombs, etc.).  
* Compliance mapping to GDPR, CCPA, HIPAA, PCI-DSS, ISO 27001 (as applicable).

### **5\. Compliance & Enforcement**

* All new upload APIs **MUST** undergo architecture & security review demonstrating compliance.  
* Existing APIs **MUST** be remediated within \[insert SLA\] or wrapped in API gateway enforcing these controls.  
* Exceptions **MUST** be documented, risk-accepted by CISO / Security Governance Board, and time-boxed.

Adopting these controls significantly reduces the risk of unrestricted file upload vulnerabilities — one of the most dangerous classes of issues in enterprise applications.

