# File Upload Security

File uploads hand an attacker a way to put their bytes onto your server. Done wrong, they lead to remote code execution (uploading a web shell), path traversal (overwriting system files), stored XSS (malicious SVG/HTML), and denial of service (huge files). Treat every upload as hostile.

## Checklist

- [ ] File types and MIME types are validated
- [ ] Upload size is restricted
- [ ] Filenames are sanitized to prevent path traversal
- [ ] Uploads aren't stored in public directories without validation
- [ ] Uploaded files are scanned for malware where appropriate

## What to look for

### Type and MIME validation
Flag uploads that accept any file, or that trust the client-supplied `Content-Type` header or the file extension alone — both are trivially spoofed. A file named `photo.jpg` can contain PHP.

**Fix:** Validate against an allowlist of permitted types. Check the actual content (magic bytes / a library like `python-magic`, `file-type`), not just the extension or the client's MIME header. Re-encode images through an image library to strip embedded payloads. For documents, validate structure.

### Size limits
Without a cap, a single large (or zip-bomb) upload can exhaust disk or memory and take the service down.

**Fix:** Enforce a maximum size at the proxy/web server (e.g., Nginx `client_max_body_size`) *and* in the app, before buffering the whole file into memory. Limit per-request and per-user totals.

### Filename sanitization / path traversal
Flag code that uses the user-supplied filename to build a storage path. A filename like `../../etc/cron.d/evil` or `..\..\config` can write outside the intended directory.

```python
# VULNERABLE
open(os.path.join(UPLOAD_DIR, request.files["f"].filename), "wb")

# FIXED: generate your own name, ignore the client's path
import uuid, os
ext = validate_and_get_extension(file)         # from an allowlist
name = f"{uuid.uuid4()}{ext}"
path = os.path.join(UPLOAD_DIR, name)
assert os.path.realpath(path).startswith(os.path.realpath(UPLOAD_DIR))
```

**Fix:** Never trust the client filename for the storage path. Generate a random name, strip directory components, validate the extension against an allowlist, and confirm the resolved path stays inside the upload directory.

### Storage location and serving
Flag uploads written into a web-served directory (e.g., under `public/`, the docroot, or anywhere the server will execute scripts). If an attacker uploads `shell.php` there and it's executed, that's RCE. Serving user HTML/SVG inline can cause stored XSS.

**Fix:** Store uploads outside the web root, or in object storage (S3, GCS) with execution disabled. Serve them through a handler that sets `Content-Disposition: attachment` (or a safe content type) and never executes them. Serve user content from a separate domain/origin to contain XSS.

### Malware scanning
For files that other users will download, or in higher-risk contexts, unscanned uploads can distribute malware.

**Fix:** Run uploads through a scanner (e.g., ClamAV) or a cloud scanning service before they're made available, and quarantine until clean.
