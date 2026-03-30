**Enterprise Security Standard**

**Secure Input Validation and Sanitization**

**Document Version:** 1.0  
**Effective Date:** March 2026  
**Owner:** Application Security Team / Security Architecture  
**Audience:** All Development Teams, Backend Engineers, API Developers, and DevSecOps Engineers

**Classification:** Internal – Mandatory Compliance

---

### **1\. Purpose**

This standard establishes mandatory requirements for **secure input validation and sanitization** across all applications, APIs, microservices, and backend systems developed or maintained by the organization.

The goal is to ensure that only properly formed, syntactically correct, and semantically valid data enters the system, thereby reducing the risk of injection attacks (SQLi, XSS, command injection, etc.), data corruption, business logic bypasses, and other security incidents.

This standard is based on the **OWASP Input Validation Cheat Sheet** (latest version available at [https://cheatsheetseries.owasp.org/cheatsheets/Input\_Validation\_Cheat\_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)) and aligns with OWASP Application Security Verification Standard (ASVS) Level 2+ requirements, OWASP Top 10 Proactive Controls (C3: Validate All Input), and enterprise security policies.

**Important Note:** Input validation is a **defense-in-depth** control. It is **not** a replacement for:

* Parameterized queries / prepared statements (SQL Injection prevention)  
* Contextual output encoding/escaping (XSS prevention)  
* Proper authentication and authorization  
* Secure file handling and other domain-specific controls

### **2\. Scope**

This standard applies to **all** externally supplied or untrusted data, including but not limited to:

* HTTP request parameters, query strings, path variables, headers, cookies  
* API payloads (JSON, XML, form data, multipart)  
* File uploads  
* Data received from third-party integrations, message queues, databases (if re-processed), or internal feeds that could be tainted  
* Any input processed by web, mobile backend, batch jobs, or command-line tools

It covers new development, enhancements, and legacy code remediation where feasible.

### **3\. Key Principles**

* **Server-Side Enforcement Only** — Client-side validation is for user experience only and can be bypassed.  
* **Positive Validation (Allowlisting) Preferred** — Explicitly define what is allowed. Reject everything else by default.  
* **Validate Early and Often** — Perform validation as close as possible to the point of data entry, before any processing, storage, or business logic.  
* **Defense in Depth** — Combine validation with sanitization (where appropriate), canonicalization, and output encoding.  
* **Fail Securely** — Reject invalid input gracefully with a generic error message. Do not leak system details.  
* **Canonicalization First** — Normalize input (decode, Unicode normalization, trim whitespace where safe) before validation to prevent evasion techniques.

### **4\. Mandatory Requirements**

#### **4.1 General Validation Rules**

1. Validate **all** input from untrusted sources using **positive (allowlist)** techniques wherever possible.  
2. Perform both **syntactic** (format, type, length, range) and **semantic** (business rules, logical consistency) validation.  
3. Use strong typing and framework-native validators when available (e.g., Bean Validation in Java, Pydantic in Python, Joi/Zod in Node.js).  
4. For structured data (JSON, XML), enforce schema validation (JSON Schema, XSD).  
5. Enforce strict length, range, and format limits on every field.  
6. For enumerated values (status codes, roles, dropdown options), validate against a server-side allowlist.  
7. Reject requests exceeding defined size limits (e.g., return HTTP 413 Payload Too Large).

#### **4.2 Validation Techniques (in order of preference)**

* Strong data types and framework validators  
* Schema-based validation  
* Allowlist-based checks (exact match for enums, regex for patterns)  
* Range/length checks  
* Character allowlisting for free-text fields  
* Custom business rule validation

**Denylisting** (blacklisting known bad patterns) may only be used as a **secondary** defense layer, never as the primary control.

#### **4.3 File Uploads (Additional Requirements)**

* Validate filename, extension, MIME type, and size against an explicit allowlist.  
* Reject dangerous extensions (e.g., .exe, .php, .jsp, .asp, .sh, .bat, .htaccess).  
* Rename uploaded files to a secure, random name on the server.  
* Store files outside the web root.  
* Perform malware scanning where possible.  
* Limit file size and number of files per request.

#### **4.4 Email Address Validation**

Follow RFC standards with normalization (lowercase domain part). Use a reputable library or regex that balances strictness and usability. Do not rely solely on regex for deliverability.

#### **4.5 Free-Form Text Fields**

* Normalize Unicode first (NFKC or NFKD as appropriate).  
* Allowlist safe character sets (letters, numbers, punctuation commonly used in names/comments).  
* Avoid overly restrictive rules that break legitimate international names or content.

#### **4.6 Error Handling and Logging**

* Return generic error messages to the user/client (e.g., “Invalid input provided”).  
* Log validation failures with sufficient context (source IP, user ID if authenticated, field name – without logging the raw malicious payload if it contains sensitive data).  
* Monitor for patterns of repeated validation failures that may indicate scanning or attacks.

### **5\. Implementation Guidelines for Developers**

**Do:**

* Validate at the earliest possible layer (API gateway, controller, service layer).  
* Use centralized validation utilities or libraries where practical.  
* Include validation in unit and integration tests.  
* Document expected input formats in OpenAPI/Swagger specifications (with schemas).  
* Combine with request size limiting and rate limiting.

**Do Not:**

* Trust “internal” or “trusted” sources without validation (they can become compromised).  
* Store or process data before validation.  
* Reveal validation logic or detailed error messages to clients.  
* Rely exclusively on client-side or framework-default “soft” validation.

**API-Specific Recommendations:**

* Define strict schemas in OpenAPI 3.x.  
* Use strong types for parameters (integer, boolean, enum, date-time).  
* Validate all headers, especially custom or security-related ones.  
* Enforce content-type and reject unexpected formats.

### **6\. Recommended Tools and Libraries**

* **Java/Spring**: Jakarta Bean Validation (Hibernate Validator), Apache Commons Validator  
* **.NET**: DataAnnotations, FluentValidation  
* **Python**: Pydantic, WTForms, Cerberus  
* **Node.js/TypeScript**: Joi, Zod, class-validator  
* **JSON Schema validators** for all languages  
* OWASP-recommended regex patterns (avoid ReDoS-vulnerable patterns)

### **7\. Integration with SecDevOps**

* **Shift-Left**: Include input validation rules in threat modeling, API design reviews, and code reviews.  
* **Automation**: Enforce schema validation and unit tests in CI/CD pipelines. Fail builds on critical validation gaps where possible.  
* **Testing**: Include negative test cases (malformed, oversized, malicious payloads) in automated and manual testing.  
* **Monitoring**: Feed validation failure logs into SIEM for anomaly detection.

### **8\. Exceptions and Risk Acceptance**

Any exception to this standard must be:

* Documented with a formal risk assessment  
* Approved by the Application Security Team  
* Compensated with alternative controls (e.g., stricter runtime protections, WAF rules)  
* Reviewed periodically

### **9\. References**

* OWASP Input Validation Cheat Sheet: [https://cheatsheetseries.owasp.org/cheatsheets/Input\_Validation\_Cheat\_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)  
* OWASP Top 10 Proactive Controls – C3: Validate All Input & Handle Exceptions  
* OWASP Application Security Verification Standard (ASVS) – Validation, Sanitization and Encoding (V5)  
* OWASP REST Security Cheat Sheet  
* OWASP XSS Prevention Cheat Sheet  
* OWASP SQL Injection Prevention Cheat Sheet  
* OWASP File Upload Cheat Sheet

### **10\. Compliance and Enforcement**

* All new code and significant changes must comply with this standard.  
* Security architecture and code reviews will verify adherence.  
* Penetration testing and automated scans will test validation effectiveness.  
* Non-compliance may result in deployment gates being blocked until remediated.

