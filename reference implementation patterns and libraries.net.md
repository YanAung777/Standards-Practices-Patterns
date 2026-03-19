Here are recommended reference implementation patterns and libraries for enforcing secure document/file upload in ASP.NET Core (including .NET 6+, .NET 8, and .NET 10 as of 2026). These align directly with the OWASP File Upload Cheat Sheet and the enterprise security standard previously outlined (e.g., authentication, size limits, allow-list validation, magic-byte/content verification, filename regeneration, quarantine scanning, secure storage).

Microsoft's official guidance (from learn.microsoft.com) is the best starting point — it includes built-in features and security warnings referencing OWASP. Build on top of that with targeted libraries for advanced validation and scanning.

### 1\. Core Built-in ASP.NET Core Features (No Extra Libraries Needed for Basics)

Use these for most requirements — they are secure by default when configured properly.

* File Upload Handling Official docs: [Upload files in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/mvc/models/file-uploads?view=aspnetcore-10.0)  
  * Use IFormFile (buffered) for \< \~50–100 MB files or streaming for large files.

Enforce request size limits via middleware or attributes:  
```C#  
**\[RequestSizeLimit(50 \* 1024 \* 1024)\]  *// 50 MB***  
**\[RequestFormLimits(MultipartBodyLengthLimit \= 50 \* 1024 \* 1024)\]**  
**\[HttpPost("upload")\]**
```
* **public async Task\<IActionResult\> Upload(IFormFile file) { ... }**

Request Validation & Limits Built-in RequestSizeLimitAttribute, DisableRequestSizeLimitAttribute, or global middleware. For Minimal APIs (recommended for new APIs):  
```C#   
**app.MapPost("/upload", async (IFormFile file) \=\> { ... })**  
   **.DisableAntiforgery()  *// If using API keys/JWT instead of cookies***

*    **.RequireAuthorization();**  
* **Filename Regeneration & Sanitization Always generate UUID-based names:**  
  **C\#**  
  **var safeFileName \= $"{Guid.NewGuid():N}{Path.GetExtension(file.FileName).ToLowerInvariant()}";**
```
Extension Allow-list (simple but insufficient alone)  
```C#  
**var allowedExtensions \= new\[\] { ".pdf", ".docx", ".xlsx" };**  
**if (\!allowedExtensions.Contains(Path.GetExtension(file.FileName).ToLowerInvariant()))**

*     **return BadRequest("Invalid file type");**  
* **Official Microsoft Sample GitHub: [dotnet/AspNetCore.Docs – File Uploads Samples](https://github.com/dotnet/AspNetCore.Docs/tree/main/aspnetcore/mvc/models/file-uploads/samples) Includes streaming to disk, validation examples, and security notes.**
```
### 2\. Recommended Libraries for Advanced OWASP Compliance

| Requirement (OWASP/Enterprise Standard) | Recommended Library / Approach | NuGet Package | Why / Notes |
| ----- | ----- | ----- | ----- |
| File signature / magic-byte validation (critical — don't trust extension or Content-Type) | Custom implementation (common) or ByteGuard.FileValidator | ByteGuard.FileValidator | Open-source, AppSec-focused; layered validation (extension \+ signature \+ more). Actively maintained for real-world bypasses. Alternative: manual byte checks (see Microsoft docs section on file signature validation). |
| Deep content / MIME detection (Apache Tika equivalent) | Custom byte arrays or limited third-party | No direct .NET port of Tika widely used | Implement simple magic number checks (e.g., PDF starts with %PDF-). For full Tika-like parsing, call external service or use limited libs like Mime-Detective. |
| Malware / Antivirus Scanning | nClam (ClamAV client) | nClam | Pure .NET client for ClamAV daemon (run ClamAV in Docker/container). Easy async scanning: await clam.SendAndScanFileAsync(stream). Widely used in .NET for production. Alternative: integrate commercial CDR (e.g., OPSWAT MetaDefender via API). |
| Content Disarm & Reconstruction (CDR) for PDFs/Office | External service (recommended) | — | Use API-based CDR (e.g., OPSWAT, Votiro, or ReSafe) — no strong native .NET open-source CDR. Remove macros/JS via libraries like Aspose.Words / Syncfusion (paid) or custom. |
| Secure Storage (outside web root, encryption) | Azure Blob / AWS S3 SDK \+ Azure.Extensions.AspNetCore.DataProtection.Blobs | Azure.Storage.Blobs Azure.Extensions.AspNetCore.DataProtection.Blobs | Private containers \+ server-side encryption. Use SAS tokens or managed identity for access. |
| Rate Limiting | Built-in or AspNetCoreRateLimit | AspNetCoreRateLimit | Per-user/IP limiting out-of-the-box in .NET 7+. |
| Generic Validation Pipeline | Custom middleware or FluentValidation \+ attributes | — | Create a reusable FileUploadValidator service that chains: size → extension → signature → scan → save. |

### 3\. Recommended Implementation Pattern (Layered Pipeline)

Follow this strict sequence in a service/controller (defense-in-depth):
```C# 
**public class SecureFileUploadService**  
**{**  
    **private readonly IWebHostEnvironment \_env;**  
    **private readonly ClamClient \_clamClient;  *// from nClam***  
    **private readonly string\[\] \_allowedExtensions \= { ".pdf", ".docx", ".xlsx" };**  
    **private readonly long \_maxFileSize \= 25 \* 1024 \* 1024; *// 25 MB***
    **public async Task\<string\> UploadAsync(IFormFile file, ClaimsPrincipal user)**  
    **{**  
        ***// 1\. Basic checks***  
        **if (file \== null || file.Length \== 0) throw new ArgumentException("No file");**  
        **if (file.Length \> \_maxFileSize) throw new ArgumentException("File too large");**

        **var ext \= Path.GetExtension(file.FileName).ToLowerInvariant();**  
        **if (\!\_allowedExtensions.Contains(ext)) throw new ArgumentException("Invalid extension");**

        ***// 2\. Magic-byte / signature validation (example for PDF)***  
        **using var stream \= file.OpenReadStream();**  
        **if (\!await IsValidPdfAsync(stream)) throw new ArgumentException("Invalid file content");**

        ***// 3\. Malware scan (quarantine first)***  
        **stream.Position \= 0;**  
        **var scanResult \= await \_clamClient.SendAndScanFileAsync(stream);**  
        **if (scanResult.Result \!= ClamScanResults.Clean)**  
            **throw new InvalidOperationException("Malware detected");**

        ***// 4\. Generate safe name & store (e.g., to private Azure Blob or disk outside wwwroot)***  
        **var safeName \= $"{Guid.NewGuid():N}{ext}";**  
        **var path \= Path.Combine(\_env.ContentRootPath, "uploads-quarantine", safeName); *// temp***  
        ***// ... save to temp, then move after all checks***

        ***// Return metadata (never original name)***  
        **return safeName;**  
    **}**

    **private async Task\<bool\> IsValidPdfAsync(Stream stream)**  
    **{**  
        **stream.Position \= 0;**  
        **var buffer \= new byte\[5\];**  
        **await stream.ReadAsync(buffer, 0, 5);**  
        **stream.Position \= 0;**  
        **return buffer.SequenceEqual(new byte\[\] { 0x25, 0x50, 0x44, 0x46, 0x2D }); *// %PDF-***  
    **}**  
**}**
```
### 4\. Additional Resources & Examples

* Microsoft Official Sample → GitHub dotnet/AspNetCore.Docs file-uploads samples (includes validation patterns).  
* ByteGuard.FileValidator → GitHub: ByteGuard-HQ (modern, OWASP-aligned open-source validator).  
* nClam \+ ClamAV Docker → Common production pattern (run ClamAV as sidecar/container).  
* Medium / Dev.to Articles (2024–2025) → Search for "Secure file upload ASP.NET Core signature validation" — many include full controllers with layered checks.

Start with Microsoft's built-in features \+ nClam for scanning \+ custom signature checks. For enterprise/high-sensitivity, add a CDR API and object storage.

