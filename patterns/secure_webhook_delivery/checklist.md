# Verification checklist

### Signing (sender)
- [ ] Every outgoing webhook includes `webhook-id`, `webhook-timestamp`, and `webhook-signature` headers
- [ ] Signature is HMAC-SHA256 over `{id}.{timestamp}.{raw_body}`
- [ ] Each endpoint has its own signing secret with at least 256 bits of entropy
- [ ] Rotation supports two concurrent active secrets
- [ ] Secrets never appear in logs or error responses. Dashboard access is auth-gated and audit-logged

### SSRF prevention (sender)
- [ ] Registration requires HTTPS and rejects any URL that resolves to private, loopback, link-local, or internal ranges (including `169.254.169.254`, `127.0.0.1`, `[::1]`, and `fc00::/7`)
- [ ] DNS re-resolved before each delivery; if any resolved address falls in a blocked range or resolution fails, delivery is rejected and logged
- [ ] HTTP redirects blocked, or every redirect target re-validated against the same rules
- [ ] Dispatcher runs behind an egress proxy. Network policy enforces this, not application config
- [ ] Resolved IP pinned per delivery attempt to prevent rebinding between resolution and connection
- [ ] Dispatcher cannot reach RFC 1918 ranges, loopback, or link-local addresses even if application-layer checks are bypassed

### Delivery (sender)
- [ ] Non-2xx responses trigger retry with exponential backoff and jitter
- [ ] HTTP client timeout is 15 seconds or less
- [ ] Retries capped at 5-8 attempts over 24-48 hours
- [ ] Failed deliveries retained for inspection and controlled replay
- [ ] Every delivery attempt logged with status code, latency, and event ID

### Abuse prevention (sender)
- [ ] Endpoints per account capped (e.g., 5-10)
- [ ] Test/ping endpoints rate-limited
- [ ] Per-destination delivery rate capped

### Verification (receiver)
- [ ] Every incoming request is signature-verified before any processing
- [ ] Constant-time comparison used for signature check
- [ ] Verification runs against raw request body bytes, not parsed JSON
- [ ] Signature verification fails on re-serialized (key-reordered) JSON body
- [ ] Requests with timestamps older than 5 minutes rejected
- [ ] Missing `webhook-id`, `webhook-timestamp`, or `webhook-signature` headers return 401

### Idempotency (receiver)
- [ ] `webhook-id` stored and checked on every request; duplicates acknowledged but not re-processed
- [ ] Processed event IDs stored with a TTL covering the retry window

### Endpoint hardening (receiver)
- [ ] Webhook endpoint returns 200 quickly and processes asynchronously
- [ ] Response bodies do not expose internal error details
