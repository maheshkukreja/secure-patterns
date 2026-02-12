# Threat Model

## Phase 1: Intake (Pre-signing & Uploads)

Focus: Preventing unauthorized writes and ensuring tenant isolation

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Upload URL | PUT reuse: Attacker replays URL to overwrite content or drive up costs | Exact-key pre-sign | 1. One-shot keys: Generate unique key per session<br>2. Rotation: Detect multiple PUTs and invalidate / alert<br>3. TTL: Keep upload window short (e.g., < 15 mins) | Low |
| Capacity | Oversized Uploads: 5TB file in 10MB slot (Cost / DoS) | Auth required | 1. Embed policy: Add content-length-range to S3 pre-sign policy<br>2. Quotas: Rate limit calls to POST /uploads endpoint | Low |
| Tenant data | Weak binding (IDOR): Attacker uploads file to another user's ID | Auth context | 1. Trusted context: Derive tenant only from auth token<br>2. Session binding: Bind upload_id + tenant_id at creation; enforce on finalize | High |
| Object namespace | Client-side key gen: Client supplies bucket / key path, enabling overwrites of other objects | Server-generated keys | 1. Opaque IDs: Client only uses upload_id. Server generates the physical storage key<br>2. Validation: Reject any client-supplied path parameters | High |
| IAM role | Blast radius: Signing principal has s3:* or broad prefix access; bug becomes compromise | Scoped role | 1. Least privilege: Restrict signer role to specific putObject actions and prefixes<br>2. Env separation: Separate roles for dev / prod | Medium |
| Pre-sign scope | Scope creep: Wildcards or broad prefixes allow overwriting unintended objects | Exact-key signing | 1. Block wildcards: Sign only specific full path uuid / file<br>2. Allowlist: Assert keys match specific patterns in code | Medium |

## Phase 2: Processing (Storage & Workers)

Focus: Preventing malicious content from crashing the pipeline

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Stored content | The swap (TOCTOU): Client claims benign file type but uploads malware / executable | Quarantine prefix | 1. Finalize gate: Verify size & headers match metadata before marking valid<br>2. Checksums: Store / verify content-MD5 or SHA256 if supported<br>3. Scan: Scan / transform in quarantine before move to publish | High |
| App state | Bypassed finalize: App assumes upload is valid just because URL was issued | Async architecture | 1. State gate: Require explicit /finalize call before creating user-visible record<br>2. Isolation: Ensure quarantine objects are never served to users | High |
| Data at rest | Unencrypted storage: Buckets lack encryption or KMS policy is too broad | Provider defaults | 1. Enforce encryption: Mandate SSE-S3 or SSE-KMS<br>2. Key policy: Tighten KMS decrypt permissions to smallest principal set<br>3. Drift detection: Alert on policy changes | Low |
| Processing pipeline | Poison pill: Malformed file crashes parser / worker (RCE / DoS) | Standard containers | 1. Sandbox: Run workers in ephemeral sandboxes (Firecracker / gVisor)<br>2. Resource limits: Set strict timeouts and memory caps<br>3. Network: Restrict worker egress | Low |
| Storage capacity | Zombie data: Abandoned uploads (never finalized) accumulate indefinitely, increasing costs | Manual cleanup | 1. Auto-expiry: Configure S3 lifecycle rule to delete objects in quarantine/ older than 24h<br>2. Cron: Prune DB sessions where status = 'pending' > 24h | Low |

## Phase 3: Serving (Downloads)

Focus: Preventing the file from attacking the user

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| User browser | Stored XSS: Inline rendering of HTML / SVG payload executes script | Published-only served | 1. Headers: Force Content-Disposition: attachment & X-Content-Type-Options: nosniff<br>2. Transcoding: Only allow inline rendering for safe / transformed formats (e.g., re-encoded JPEGs)<br>3. Sandbox domain: Serve inline content from a separate domain (e.g., assets-myapp.com) | High |
| Download URL | URL Leakage: Logs / Referrers expose signed URL | AuthZ required | 1. No logging: Redact signed query params from all server logs<br>2. Short TTL: Enforce max TTL (e.g., 5 mins)<br>3. Alerting: Alert on download spikes | Low |
| Auditability | Silent exfil: Access bypasses API logs (direct S3 access) | API logging | 1. Storage logs: Enable S3 / GCS access logs and centralize them<br>2. Traceability: Tag objects with uploader_id | Low |
