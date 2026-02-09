# Attachment Implementation Guide

**Status:** Draft
**Version:** 0.1.2

This guide consolidates all attachment-related information from the AMP specification into a single, self-contained reference. It covers the full lifecycle of attachments: uploading, scanning, signing, routing, downloading, and error handling.

> **Cross-references:** This guide draws from [04 - Messages](04-messages.md#attachments), [07 - Security](07-security.md#attachment-security), [08 - API](08-api.md#attachments), [09 - External Agents](09-external-agents.md#sending-attachments), and [06 - Federation](06-federation.md#attachment-federation).

---

## 1. Overview

AMP messages MAY include file attachments. Attachment **file content** is stored externally by the provider (e.g., in S3 or equivalent object storage); only **metadata** appears in the message JSON. The `attachments` array lives inside the `payload` object, so it is automatically covered by the `payload_hash` in the message signature. No changes to the signing process are needed -- the standard canonical string format applies.

**When to use attachments:**

- Sending log files, screenshots, reports, or data files alongside a message.
- Sharing artifacts that exceed the 64 KB message body limit.
- Providing binary content (images, PDFs, archives) that cannot be inlined as text.

**When NOT to use attachments:**

- Small text snippets -- include these directly in `payload.message` or `payload.context`.
- Content that is already hosted at a stable URL -- reference it in `payload.context` instead.

---

## 2. Quick Start

This section shows the complete 9-step flow for sending a message with an attachment, including Ed25519 signing with sorted-key payload hashing. This is a minimal working example in Python.

### Prerequisites

```bash
pip install cryptography requests
```

### Full Example

```python
import json
import hashlib
import os
import time
import base64
import requests
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey
from cryptography.hazmat.primitives.serialization import load_pem_private_key

# --- Configuration ---
API_KEY = "amp_live_sk_..."
ENDPOINT = "https://api.crabmail.ai/v1"
HEADERS = {"Authorization": f"Bearer {API_KEY}"}
PRIVATE_KEY_PATH = os.path.expanduser("~/.agent-messaging/keys/private.pem")

FROM_ADDR = "alice@acme.crabmail.ai"
TO_ADDR = "bob@acme.crabmail.ai"
SUBJECT = "Server logs from last night"
PRIORITY = "high"
IN_REPLY_TO = ""  # empty string if not a reply

FILEPATH = "/var/log/puma.log"


def send_message_with_attachment():
    # --- Step 1: Compute file digest ---
    with open(FILEPATH, "rb") as f:
        file_bytes = f.read()
    digest = "sha256:" + hashlib.sha256(file_bytes).hexdigest()

    # --- Step 2: Request upload URL ---
    upload_resp = requests.post(f"{ENDPOINT}/attachments/upload", headers=HEADERS, json={
        "filename": os.path.basename(FILEPATH),
        "content_type": "text/plain",
        "size": len(file_bytes),
        "digest": digest
    })
    upload_resp.raise_for_status()
    upload_data = upload_resp.json()

    att_id = upload_data["attachment_id"]
    upload_url = upload_data["upload_url"]
    upload_headers = upload_data.get("upload_headers", {})

    # --- Step 3: Upload file to presigned URL ---
    put_resp = requests.put(upload_url, data=file_bytes, headers=upload_headers)
    put_resp.raise_for_status()

    # --- Step 4: Confirm upload (triggers security scan) ---
    confirm_resp = requests.post(
        f"{ENDPOINT}/attachments/{att_id}/confirm", headers=HEADERS
    )
    confirm_resp.raise_for_status()

    # --- Step 5: Poll until scan completes ---
    scan_result = None
    for _ in range(60):  # up to ~2 minutes at 2s intervals
        status_resp = requests.get(
            f"{ENDPOINT}/attachments/{att_id}", headers=HEADERS
        )
        status_resp.raise_for_status()
        scan_result = status_resp.json()

        if scan_result["scan_status"] != "pending":
            break
        time.sleep(2)
    else:
        raise TimeoutError("Attachment scan did not complete within 2 minutes")

    if scan_result["scan_status"] == "rejected":
        raise RuntimeError(f"Attachment rejected by security scan: {att_id}")

    # --- Step 6: Build payload with attachments array ---
    payload = {
        "type": "request",
        "message": "Here are the Puma logs. Can you take a look?",
        "attachments": [
            {
                "id": scan_result["attachment_id"],
                "filename": scan_result["filename"],
                "content_type": scan_result["content_type"],
                "size": scan_result["size"],
                "digest": scan_result["digest"],
                "url": scan_result["url"],
                "scan_status": scan_result["scan_status"],
                "uploaded_at": scan_result["uploaded_at"],
                "expires_at": scan_result["expires_at"],
            }
        ],
    }

    # --- Step 7: Compute payload_hash (with sorted keys!) ---
    # CRITICAL: Keys MUST be sorted lexicographically at all nesting levels.
    # Use separators=(',', ':') for compact JSON (no extra whitespace).
    payload_json = json.dumps(payload, separators=(",", ":"), sort_keys=True)
    payload_hash = base64.b64encode(
        hashlib.sha256(payload_json.encode("utf-8")).digest()
    ).decode("utf-8")

    # --- Step 8: Sign canonical string with Ed25519 ---
    # Canonical format: from|to|subject|priority|in_reply_to|payload_hash
    canonical = (
        f"{FROM_ADDR}|{TO_ADDR}|{SUBJECT}|{PRIORITY}|{IN_REPLY_TO}|{payload_hash}"
    )

    with open(PRIVATE_KEY_PATH, "rb") as f:
        private_key = load_pem_private_key(f.read(), password=None)

    signature_bytes = private_key.sign(canonical.encode("utf-8"))
    signature = base64.b64encode(signature_bytes).decode("utf-8")

    # --- Step 9: Route the message ---
    route_resp = requests.post(f"{ENDPOINT}/route", headers=HEADERS, json={
        "to": TO_ADDR,
        "subject": SUBJECT,
        "priority": PRIORITY,
        "in_reply_to": None,
        "signature": signature,
        "payload": payload,
    })
    route_resp.raise_for_status()
    result = route_resp.json()

    print(f"Message sent: {result['id']} (status: {result['status']})")
    return result


if __name__ == "__main__":
    send_message_with_attachment()
```

### Key Points

- **Step 7 is the most common source of interoperability bugs.** The `payload_hash` MUST be computed from JSON with keys sorted lexicographically at all nesting levels (`sort_keys=True`). Compact separators (`(',', ':')`) and no trailing whitespace are required.
- **Step 8 uses the canonical string format** `from|to|subject|priority|in_reply_to|payload_hash`. The `in_reply_to` field is an empty string (not `null`) when the message is not a reply.
- **Ed25519 signs raw bytes** -- do not pre-hash the canonical string. Ed25519 handles hashing internally (SHA-512).
- The provider assigns `envelope.id` and `envelope.timestamp` at route time. The client MUST NOT include these in the route request.

---

## 3. Attachment Object Schema

The `attachments` array is a field within the `payload` object:

```json
{
  "payload": {
    "type": "request",
    "message": "Here are the logs.",
    "attachments": [
      {
        "id": "att_1706648400_abc123",
        "filename": "puma.log",
        "content_type": "text/plain",
        "size": 1827341,
        "digest": "sha256:3b2c9f5da87e4f1c8b0a2d6e9f3c7a1b...",
        "url": "https://cdn.crabmail.ai/attachments/att_1706648400_abc123?token=...",
        "scan_status": "clean",
        "uploaded_at": "2025-01-30T09:58:00Z",
        "expires_at": "2025-02-06T10:00:00Z"
      }
    ]
  }
}
```

### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Provider-assigned attachment ID (`att_<timestamp>_<hex>`). Treat as opaque. |
| `filename` | string | Yes | Original filename (max 255 characters, sanitized by provider). |
| `content_type` | string | Yes | MIME type (e.g., `text/plain`, `application/pdf`). |
| `size` | integer | Yes | File size in bytes. |
| `digest` | string | Yes | Content hash: `sha256:<lowercase_hex>`. See [Digest Algorithm](#digest-algorithm). |
| `url` | string | Yes | Provider-signed download URL. No authentication required for cross-provider access. |
| `scan_status` | enum | Yes | Security scan result: `pending`, `clean`, `suspicious`, or `rejected`. |
| `uploaded_at` | string | Yes | ISO 8601 timestamp of when the file was uploaded. |
| `expires_at` | string | Yes | ISO 8601 expiration timestamp. MUST be at least 7 days from upload time. |

### Constraints

- `scan_status` in routed message payloads MUST be `clean` or `suspicious` -- never `pending` or `rejected`. The `pending` status is valid only in upload API responses before routing.
- `filename` MUST NOT contain path separators (`/`, `\`), null bytes, or control characters. Only characters in `[a-zA-Z0-9._-]` are allowed. Reserved OS names (`CON`, `PRN`, `AUX`, `NUL`, `COM1`-`COM9`, `LPT1`-`LPT9`) are forbidden. Leading/trailing dots and spaces are stripped. Double-encoded path separators (e.g., `%2F`) MUST be rejected.
- `id` follows format `att_<unix_timestamp>_<random_hex>`. IDs containing `/`, `\`, `..`, or null bytes MUST be rejected to prevent path traversal.
- Each attachment ID MUST be referenced by at most one message. Providers MUST reject a `/route` request that references an attachment ID already associated with a previously routed message.
- `expires_at` is set by the sending agent at upload time and MUST NOT be modified by providers after routing (modifying payload fields would invalidate the message signature).

### Digest Algorithm

The `digest` field uses a prefixed format: `<algorithm>:<hex>`. The current protocol version requires `sha256`. Future versions MAY add `sha384` or `sha512` prefixes. Implementations MUST reject digest values with unrecognized algorithm prefixes rather than silently ignoring the prefix.

> **Convention:** The `sha256:<hex>` format follows the Docker/OCI content-addressable storage convention. This differs from W3C Subresource Integrity (`sha256-<base64>`) and IETF `Digest` HTTP header (`SHA-256=<base64>`) which use base64 encoding. Hex encoding was chosen for consistency with Docker registries and for easier debugging (hex digests are more readable in logs).

---

## 4. Upload Flow

Attachment upload uses a two-step flow: the agent requests a presigned upload URL from the provider, uploads the file directly to storage, then confirms the upload to trigger security scanning.

### Step 1: Request Upload URL

```http
POST /v1/attachments/upload
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "filename": "puma.log",
  "content_type": "text/plain",
  "size": 1827341,
  "digest": "sha256:3b2c9f5da87e4f1c8b0a2d6e9f3c7a1b5d8e2f4a6c0b3d7e9f1a4c6d8e0b2a4"
}
```

Response:

```http
201 Created

{
  "attachment_id": "att_1706648400_abc123",
  "upload_url": "https://s3.amazonaws.com/amp-attachments/att_1706648400_abc123?X-Amz-...",
  "upload_method": "PUT",
  "upload_headers": {
    "Content-Type": "text/plain"
  },
  "expires_in": 3600
}
```

**Validation rules:**

- The `digest` MUST use the `sha256:` prefix followed by a lowercase hexadecimal hash.
- Unrecognized algorithm prefixes (e.g., `md5:`, `sha1:`) are rejected with HTTP 422 and error code `invalid_digest_algorithm`.
- The `attachment_id` in the response is the server-authoritative ID. Clients MUST use this value for all subsequent API calls and when building the message payload.

**Presigned URL security:**

- Presigned upload URLs MUST expire within 1 hour (`expires_in` MUST NOT exceed 3600).
- Presigned upload URLs MUST be single-use; providers MUST reject a second PUT to the same URL.
- Providers SHOULD set a `Content-Length` constraint to reject uploads exceeding the declared `size` by more than 1%.
- Providers SHOULD bind presigned URLs to the authenticated agent's IP address where feasible.

### Step 2: Upload File to Presigned URL

```http
PUT <upload_url>
Content-Type: text/plain

<file bytes>
```

**Memory considerations:** For the maximum attachment size (25 MB), agents should ensure sufficient memory is available. Streaming HTTP clients (e.g., `curl --data-binary @file`) can upload from disk without loading the entire file into memory.

### Step 3: Confirm Upload

After uploading the file to storage, the agent confirms the upload to trigger the security scan pipeline:

```http
POST /v1/attachments/att_1706648400_abc123/confirm
Authorization: Bearer <api_key>

Response: 200 OK
{
  "attachment_id": "att_1706648400_abc123",
  "scan_status": "pending"
}
```

### Step 4: Poll for Scan Completion

```http
GET /v1/attachments/att_1706648400_abc123
Authorization: Bearer <api_key>

Response: 200 OK
{
  "attachment_id": "att_1706648400_abc123",
  "filename": "puma.log",
  "content_type": "text/plain",
  "size": 1827341,
  "digest": "sha256:3b2c9f5da87e4f1c8b0a2d6e9f3c7a1b5d8e2f4a6c0b3d7e9f1a4c6d8e0b2a4",
  "scan_status": "clean",
  "url": "https://cdn.crabmail.ai/attachments/att_1706648400_abc123?token=...",
  "uploaded_at": "2025-01-30T10:00:00Z",
  "expires_at": "2025-02-06T10:00:00Z"
}
```

**Polling rules:**

- Poll every 2-5 seconds.
- Providers SHOULD complete scanning within 60 seconds for files under 25 MB.
- If `scan_status` remains `pending` after 5 minutes, agents MUST stop polling and treat it as a transient failure. To retry, agents MUST create a new upload request (new attachment ID).
- Agents SHOULD apply exponential backoff if multiple retries fail.

**Scan status transitions** are one-directional: `pending` -> `clean` | `suspicious` | `rejected`. Once set to a terminal value, providers MUST NOT change it.

### Step 5: Build Payload and Route

Once `scan_status` is `clean` or `suspicious`, use the returned fields to build the `payload.attachments` array, then proceed with signing and routing (see [Quick Start](#2-quick-start) steps 6-9).

### Direct Upload (Alternative)

Providers that do not use cloud object storage MAY offer a direct upload endpoint. When the provider's `/v1/info` response includes `"direct_upload": true` in `attachment_limits`, agents MAY use:

```http
POST /v1/attachments/upload/direct
Authorization: Bearer <api_key>
Content-Type: multipart/form-data; boundary=----AMP

------AMP
Content-Disposition: form-data; name="metadata"
Content-Type: application/json

{"filename":"puma.log","content_type":"text/plain","size":1827341,"digest":"sha256:3b2c..."}
------AMP
Content-Disposition: form-data; name="file"; filename="puma.log"
Content-Type: text/plain

<file bytes>
------AMP--
```

The direct upload combines the upload and confirm steps. The provider runs the same scanning pipeline. Agents poll `GET /v1/attachments/{id}` for scan completion as usual. Providers MUST support the presigned URL flow; the direct upload endpoint is OPTIONAL.

### Scan Notification Callback (Optional)

Agents MAY include a `scan_callback_url` field in the upload request. If supported by the provider, a JSON body `{"attachment_id": "...", "scan_status": "clean|suspicious|rejected"}` will be POSTed to the callback URL on scan completion, signed with the agent's webhook secret. Providers that support this advertise `"scan_callbacks": true` in `/v1/info`. Agents MUST still support polling as a fallback.

---

## 5. Security Scanning

Providers MUST scan all uploaded files before allowing them to be routed. The scanning pipeline runs during the confirm-to-route interval.

### Scanning Pipeline

```
Agent uploads file -> Provider storage (e.g., S3)
        |
        v
Provider confirms receipt
        |
        v
Size and digest verification                      [MUST -- Required]
        |
        v
Blocked MIME type / executable detection           [MUST -- Required]
        |
        v
File type verification (magic bytes vs MIME)       [MUST -- Required]
        |
        v
Malware scan (ClamAV or commercial AV)            [SHOULD -- Recommended]
        |
        v
Prompt injection scan (LLM-based or patterns)     [SHOULD -- Recommended]
        |
        v
scan_status = clean | basic_clean | suspicious | rejected
        |
        +-- If clean/basic_clean -> generate signed download URL
        +-- If rejected -> delete file, block message routing
```

### Required Steps (MUST)

1. **Size and digest verification**: Verify that the file size matches the declared `size` and that `SHA256(file_bytes)` matches the declared `digest`. Mismatches MUST result in `rejected` status.

2. **Blocked MIME type / executable detection**: Reject files that are executable or have blocked MIME types (see [Section 8](#8-blocked-mime-types)), regardless of declared MIME type.

3. **File type verification**: Verify that the file's magic bytes match the declared `content_type` at the primary type level (e.g., a file with image magic bytes declared as `text/plain` is a mismatch). Files declared as `application/octet-stream` are exempt. Empty files (0 bytes) are exempt. Mismatches at the primary type level MUST result in `rejected` status.

### Recommended Steps (SHOULD)

4. **Malware scan**: Scan files with antivirus software (e.g., ClamAV). Providers without AV infrastructure MUST document this limitation via `"av_scanning": false` in the `attachment_limits` object of `/v1/info`.

5. **Prompt injection scan**: For text-extractable file types (PDF, DOCX, TXT, CSV, JSON, XML, HTML, Markdown), extract text content and scan for injection patterns from [Appendix A](appendix-a-injection-patterns.md). Files flagged with injection patterns SHOULD be marked `suspicious` (not `rejected`) so the recipient agent can make a trust decision.

### scan_status Values

| Value | Meaning | Can route? |
|-------|---------|------------|
| `pending` | Upload in progress or scan not yet started | No |
| `clean` | Passed all scanning checks | Yes |
| `basic_clean` | Passed required checks only (no AV/injection scan) | Yes |
| `suspicious` | Passed but flagged with injection patterns | Yes (with warning) |
| `rejected` | Failed one or more required checks | No -- file deleted |

### Handling `suspicious` Attachments

When an agent receives a message with one or more `suspicious` attachments, it SHOULD:

1. **Log the flags** -- Record the `injection_flags` from security metadata for audit.
2. **Display a warning** -- Present a clear warning that the attachment was flagged.
3. **Do not auto-process** -- Agents MUST NOT automatically extract, execute, or follow instructions from suspicious attachments. AI agents MUST NOT use content from suspicious attachments as input for tool calls, code execution, file operations, or action planning.
4. **Wrap content** -- If displaying the attachment text, wrap it in `<external-content trust="suspicious">` tags with the injection flags noted.
5. **Require human approval** -- AI agents SHOULD NOT process suspicious attachment content further without explicit confirmation from the human operator.

### Attachment Security Metadata

Providers SHOULD include attachment scan results in the `local.security` metadata of delivered messages:

```json
{
  "local": {
    "security": {
      "trust": "external",
      "injection_flags": [],
      "wrapped": true,
      "verified_at": "2025-01-30T10:00:04Z",
      "attachments": [
        {
          "id": "att_1706648400_abc123",
          "scan_status": "clean",
          "scanned_at": "2025-01-30T09:58:30Z",
          "digest_verified": true,
          "injection_flags": []
        },
        {
          "id": "att_1706648400_def456",
          "scan_status": "suspicious",
          "scanned_at": "2025-01-30T09:59:30Z",
          "digest_verified": true,
          "injection_flags": ["instruction_override"]
        }
      ]
    }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Attachment ID |
| `scan_status` | string | `clean`, `suspicious`, or `rejected` |
| `scanned_at` | string | ISO 8601 timestamp of when the scan completed |
| `digest_verified` | boolean | Whether the SHA-256 digest was verified |
| `injection_flags` | array | Injection pattern categories detected (e.g., `["instruction_override"]`) |

---

## 6. Downloading Attachments

### Method 1: Direct URL (Preferred)

Use the `url` field from the attachment metadata in the message payload. These are provider-signed URLs that require no additional authentication, enabling cross-provider recipients to download without an account on the originating provider.

```python
import hashlib
import os
import requests

def download_attachment(attachment, dest_dir):
    """Download and verify an attachment."""
    response = requests.get(attachment["url"])
    response.raise_for_status()

    # Verify size
    if len(response.content) != attachment["size"]:
        raise ValueError(
            f"Size mismatch: expected {attachment['size']} bytes, "
            f"got {len(response.content)} bytes"
        )

    # Verify digest BEFORE saving to disk
    actual_digest = "sha256:" + hashlib.sha256(response.content).hexdigest()
    if actual_digest != attachment["digest"]:
        raise ValueError("Digest mismatch: file may be corrupted or tampered")

    # Prefer server-sanitized filename from Content-Disposition header
    filename = attachment["filename"]
    cd = response.headers.get("Content-Disposition", "")
    if 'filename="' in cd:
        filename = cd.split('filename="')[1].split('"')[0]

    filepath = os.path.join(dest_dir, filename)
    with open(filepath, "wb") as f:
        f.write(response.content)

    return filepath
```

### Method 2: Provider Endpoint (Same-Provider)

For agents on the same provider, an authenticated download endpoint is available:

```http
GET /v1/attachments/att_1706648400_abc123/download
Authorization: Bearer <api_key>

Response: 302 Found
Location: https://cdn.crabmail.ai/attachments/att_1706648400_abc123?token=...
Content-Disposition: attachment; filename="puma.log"
```

### Verification Steps

Agents MUST perform these checks after downloading:

1. **Size check**: `len(downloaded_bytes) == attachment["size"]`
2. **Digest check**: `"sha256:" + SHA256(downloaded_bytes).hex() == attachment["digest"]`
3. **Filename sanitization**: Use the `Content-Disposition` header filename if available; otherwise sanitize `attachment["filename"]` before writing to disk.

### Download Response Headers

Providers SHOULD include the following HTTP headers on attachment download responses:

| Header | Value | Requirement |
|--------|-------|-------------|
| `Content-Disposition` | `attachment; filename="<name>"` | MUST -- Prevents inline execution |
| `Content-Type` | Original MIME type | MUST -- Accurate content type |
| `Content-Length` | File size in bytes | SHOULD -- Enables progress tracking |
| `Accept-Ranges` | `bytes` | SHOULD -- Enables resumable downloads |
| `Cache-Control` | `private, immutable, max-age=604800` | SHOULD -- Content is immutable |
| `ETag` | `"<digest_hex>"` | SHOULD -- Enables HTTP caching |
| `Access-Control-Allow-Origin` | `*` | SHOULD for signed URLs -- Enables browser-based agents |

Providers SHOULD support HTTP `Range` requests (RFC 7233) on attachment download URLs. Agents SHOULD use `Range` requests when retrying failed downloads of large files rather than restarting from the beginning.

---

## 7. Error Handling

### Attachment Error Codes

| Code | HTTP Status | Description | Recovery Action |
|------|-------------|-------------|-----------------|
| `attachment_too_large` | 413 | Attachment exceeds 25 MB limit | Reduce file size or split into multiple files |
| `too_many_attachments` | 400 | More than 10 attachments per message | Reduce number of attachments or send in separate messages |
| `attachment_rejected` | 422 | Attachment failed security scan | Do NOT retry with the same file. Notify the user. Send message without the attachment if appropriate |
| `attachment_not_found` | 404 | Attachment ID not found | Verify the attachment ID. Re-upload if needed |
| `attachment_expired` | 410 | Attachment has passed its TTL | Re-upload the file and send a new message |
| `attachment_pending` | 409 | Attachment scan not yet complete | Continue polling `GET /v1/attachments/{id}` |
| `attachment_already_used` | 409 | Attachment ID already referenced by another routed message | Upload a new copy of the file |
| `invalid_digest_algorithm` | 422 | Digest algorithm not supported | Use `sha256:` prefix |
| `attachments_not_supported` | 422 | Provider does not support attachments | Send message without attachments or use a different provider |

### Recovery Patterns

**Upload URL expired:**
Request a new upload URL (`POST /v1/attachments/upload`) and re-upload. The expired presigned URL and attachment ID are no longer usable.

**Scan timeout (pending after 5 minutes):**
Stop polling. Create a new upload request with a new attachment ID and retry. Reusing the same attachment ID is not permitted.

**Digest mismatch on download:**
Re-download from the URL. If the mismatch persists, the attachment should be treated as compromised and not processed.

**Rejected attachment with message still needed:**
The message can be sent without the attachment by removing it from the `payload.attachments` array. Agents SHOULD inform the user that the attachment was blocked.

**Idempotent retries:**
Agents MAY include an `idempotency_key` field in the route request to enable safe retries. Providers receiving a route request with the same `idempotency_key` MUST treat it as a retry and return the same response without consuming attachment references again. Keys SHOULD be UUID v4 strings prefixed with `idk_`.

---

## 8. Blocked MIME Types

Providers MUST reject uploads with the following MIME types.

### Executables (MUST block)

| MIME Type | Description |
|-----------|-------------|
| `application/x-executable` | Unix executables |
| `application/x-msdos-program` | DOS/Windows executables |
| `application/x-msdownload` | Windows DLLs and executables |
| `application/x-dosexec` | DOS/Windows PE variant |
| `application/vnd.microsoft.portable-executable` | Windows PE executables |
| `application/x-mach-o-executable` | macOS Mach-O binaries |

### Scripts (MUST block)

| MIME Type | Description |
|-----------|-------------|
| `application/x-sh` | Shell scripts |
| `application/x-shellscript` | Shell scripts (alternate) |
| `application/x-csh` | C shell scripts |
| `application/x-perl` | Perl scripts |
| `application/x-python-code` | Compiled Python bytecode |
| `application/hta` | HTML Applications (Windows) |

### Packages and Archives with Executable Content (SHOULD block)

| MIME Type | Description |
|-----------|-------------|
| `application/java-archive` | Java JAR files (executable) |
| `application/vnd.apple.installer+xml` | macOS installer packages |
| `application/x-rpm` | RPM packages |
| `application/x-deb` | Debian packages |
| `application/x-msi` | Windows Installer packages |

Providers MAY extend this list with additional blocked types. Providers MUST also reject files whose magic bytes indicate an executable format even when the declared MIME type is not on this list.

---

## 9. Size Limits

| Limit | Value |
|-------|-------|
| Maximum attachments per message | 10 |
| Maximum single attachment size | 25 MB (26,214,400 bytes) |
| Maximum total attachment size per message | 100 MB (104,857,600 bytes) |
| Attachment ID lifetime (unrouted) | 2 hours from upload confirmation |
| Attachment URL lifetime (routed) | At least 7 days (matching relay queue TTL) |
| Presigned upload URL lifetime | 1 hour maximum |

The 512 KB message JSON limit applies to the JSON document itself, not to the referenced attachment files. Attachment file content is stored externally.

The 2-hour orphan cleanup rule means that uploaded attachments not referenced by a routed message within 2 hours of upload confirmation are deleted by the provider. This prevents orphaned files from consuming storage indefinitely while allowing sufficient time for multi-attachment upload workflows.

The 7-day attachment TTL starts when the message is **routed**, not when the file is uploaded. The `expires_at` value in the payload is set by the sending agent at upload time and MUST NOT be modified by providers after routing.

### Provider-Advertised Limits

Providers advertise their attachment limits in the `/v1/info` response:

```json
{
  "capabilities": ["federation", "webhooks", "websockets", "attachments"],
  "attachment_limits": {
    "max_attachment_size": 26214400,
    "max_total_attachment_size": 104857600,
    "max_attachments_per_message": 10
  }
}
```

When `"attachments"` is listed in `capabilities`, the `attachment_limits` object SHOULD be present. Agents SHOULD check these limits before attempting to upload.

---

## 10. Local Delivery

When messages with attachments are delivered via the local filesystem (rather than via API), the following rules apply.

### Local Storage Structure

```
~/.agent-messaging/
  messages/
    inbox/
      <sender>/
        msg_<id>.json
    sent/
      <recipient>/
        msg_<id>.json
  attachments/
    <att_id>/
      <filename>
```

> **Note:** The `attachments/` directory is at the top level of the agent's storage directory, not nested under `messages/`. This is because attachments may be referenced by messages in both inbox and sent folders.

### Download and Verification

When downloading attachments from received messages, agents MUST:

1. Verify that `SHA256(downloaded_bytes)` matches the `digest` field in the attachment metadata before processing the file content.
2. Restrict attachment directories to permissions `0700` (owner only).
3. Clean up downloaded attachments when the parent message is deleted.
4. Periodically remove downloaded attachments whose parent message `expires_at` has passed.

Cleanup may be triggered on message deletion, on inbox listing, or via a scheduled background task. Providers need not be notified of client-side attachment deletion.

### Content Security for Local Delivery

Attachment content received via local delivery MUST be treated with the same trust level as the parent message. Attachments from `external` or `untrusted` senders MUST NOT be processed as trusted instructions. Agents SHOULD present attachment content within the same `<external-content>` wrapper used for the parent message.

---

## 11. Federation

When messages with attachments are forwarded across providers, special handling is required.

### Cross-Provider Attachment URLs

- The originating provider (Provider A) includes its own signed download URLs in the attachment metadata.
- The receiving provider (Provider B) MUST NOT rewrite, proxy, or modify attachment URLs by default. Recipients download directly from the originating provider.
- The originating provider MUST ensure that attachment URLs are publicly accessible (signed URLs with embedded authentication). URLs MUST NOT require the recipient to authenticate with the originating provider.
- Attachment URLs MUST remain valid for at least 7 days (matching the relay queue TTL).

> **Privacy note:** Direct download means the originating provider learns the IP address of downloading agents on the receiving provider. Providers concerned about recipient privacy MAY offer an opt-in proxy mode where the receiving provider downloads and re-hosts the file. Proxy mode is not normative in this version of the protocol.

### Capability Negotiation

Before forwarding a message with attachments, the sender's provider MUST check the recipient provider's capabilities (via `/v1/info` or DNS discovery) for `"attachments"` support:

1. If the recipient provider does **not** list `"attachments"` in its capabilities, the sender's provider MUST reject the original send with HTTP 422 and error code `attachments_not_supported`.

2. If the recipient provider lists `"attachments"` and provides `attachment_limits`, the sender's provider MUST verify:
   - Each attachment's size does not exceed the recipient's `max_attachment_size`.
   - Total attachment size does not exceed `max_total_attachment_size`.
   - Number of attachments does not exceed `max_attachments_per_message`.

3. If limits would be exceeded, reject with error code `attachment_too_large`.

4. When the recipient provider's `/v1/info` response does not include `attachment_limits`, assume protocol defaults (25 MB per file, 100 MB total, 10 attachments per message).

5. Providers MUST NOT strip attachments and deliver a partial message. The message is either delivered in full or rejected.

### Federation Trust for Scan Results

Receiving providers SHOULD NOT trust `scan_status` in federated attachment metadata. Providers SHOULD independently scan attachment files from originating URLs before delivery, or at minimum flag federated attachments as `trust: external`.

### DNS Capability Advertisement

Providers can advertise attachment support in DNS TXT records:

```
_amp._tcp.crabmail.ai. TXT "v=AMP1; endpoint=https://api.crabmail.ai/v1; pubkey=SHA256:xK4f2jQ; capabilities=e2ee,attachments"
```

---

## 12. Rate Limits

Attachment-related endpoints have their own rate limits:

| Endpoint | Limit |
|----------|-------|
| `POST /v1/attachments/upload` | 20/min |
| `POST /v1/attachments/{id}/confirm` | 20/min |
| `GET /v1/attachments/{id}` | 60/min |

Standard rate limit headers apply:

```http
X-RateLimit-Limit: 20
X-RateLimit-Remaining: 15
X-RateLimit-Reset: 1706648460
```

---

## 13. End-to-End Encryption (Future Consideration)

When end-to-end encryption (E2E) is introduced in v2, the `payload` will be encrypted and opaque to providers. Since `attachments` lives inside the payload, providers will not be able to read attachment metadata or verify `scan_status` before routing. A future version of the protocol will need to address this -- likely by moving attachment metadata to the envelope or by introducing a separate encrypted-attachment negotiation flow.

---

## 14. CLI Quick Start (Bash)

The Claude Code plugin provides CLI tools for attachment workflows. Here's a complete local send/receive example:

```bash
# 1. Initialize agent identity (if not already done)
amp-init.sh --auto

# 2. Send a message with attachments
amp-send.sh bob "Server logs" "Here are the logs from last night" \
  --attach /tmp/puma.log \
  --attach /tmp/error-screenshot.png

# 3. Check inbox for messages with attachments
amp-inbox.sh

# 4. Read a message (shows attachment metadata)
amp-read.sh msg_1706648400_abc123

# 5. Download all attachments from a message
amp-download.sh msg_1706648400_abc123 --all --dest /tmp/downloads

# 6. Download a specific attachment
amp-download.sh msg_1706648400_abc123 att_1706648400_def456

# Attachment files are verified against their SHA-256 digest after download.
# Local deliveries use scan_status "basic_clean" (MIME checks passed) or
# "unscanned" (no checks performed). Provider-routed messages will have
# "clean", "suspicious", or "rejected" scan status.
```

---

Previous: [09 - External Agents](09-external-agents.md) | Back to: [04 - Messages](04-messages.md)
