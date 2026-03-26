# Threat Model

### Phase 1: Webhook registration

Focus: Preventing registration of malicious endpoints

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Internal network | SSRF via malicious URL: Customer registers an internal IP or cloud metadata endpoint as their webhook URL | URL validation at registration | 1. Block private/loopback/link-local IPs<br>2. Block internal domains<br>3. Require HTTPS<br>4. Egress proxy blocks internal ranges at network level | High |
| Platform availability | DDoS amplification: Attacker registers many endpoints pointing at a victim, then triggers mass events to flood the target from your infrastructure | Per-account endpoint cap (5-10) | 1. Destination dedup: Limit subscriptions per destination URL across accounts<br>2. Rate limit test/ping endpoints<br>3. Per-destination delivery throttling | Medium |
| Signing secret | Secret extraction: Attacker uses API to repeatedly retrieve or enumerate signing secrets | Auth-gated access | 1. Role-based secret visibility<br>2. Audit log every secret access<br>3. Rate limit secret retrieval API | Medium |
| Endpoint configuration | Unauthorized registration: Low-privilege user registers or modifies webhook URL to redirect events or extract signing secrets | API authentication | 1. Role-based access: restrict endpoint CRUD to admin roles<br>2. Re-validate URL on every update<br>3. Rotate secret on URL change<br>4. Audit log all endpoint mutations | Medium |

### Phase 2: Webhook delivery

Focus: Securing the outbound request path

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Internal network | DNS rebinding: URL resolves to a public IP at validation but a private IP at delivery time | DNS check at registration | 1. Re-resolve DNS at delivery time<br>2. Egress proxy blocks private ranges<br>3. Pin resolved IP for the delivery attempt<br>4. Block HTTP redirects | High |
| Dispatcher workers | Resource exhaustion: Malicious endpoint responds slowly, tying up worker threads | HTTP timeout (5-15s connect + read) | 1. Concurrency cap: Limit concurrent deliveries per destination<br>2. Circuit breaker on consistently slow endpoints<br>3. Isolated worker pools per priority tier | Medium |
| Event data | Interception in transit: Webhook payload captured on the network | HTTPS-only registration (HTTP rejected) | 1. Certificate pinning: Verify TLS certificates, never disable cert validation<br>2. Minimum TLS 1.2<br>3. Payload encryption: Encrypt sensitive fields inside the signed body for defense-in-depth | Low |
| Audit trail | Silent failure: Webhooks fail without visibility, causing data loss the customer cannot diagnose | Async retry with backoff | 1. Log every delivery attempt with status code, latency, and event ID<br>2. Surface delivery status to customers via API or dashboard<br>3. Alert on sustained failures<br>4. Dead letter after max attempts for controlled replay | Medium |
| Dispatcher integrity | Retry storm: Persistent failure or malformed event causes unbounded retries, consuming dispatcher capacity | Retry cap (5-8 attempts) with exponential backoff and jitter | 1. Dead-letter: Route failed events after max attempts for manual replay<br>2. Deduplicate events by ID before dispatch<br>3. Circuit breaker on persistently failing endpoints | Medium |

### Phase 3: Webhook verification (receiver side)

Focus: Preventing the customer from processing forged or replayed webhooks

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Business logic | Forged webhook: Attacker sends a fake POST to the customer's endpoint to trigger unauthorized actions (e.g., fake "payment completed" event) | HMAC signature verified on every request; unsigned requests rejected | 1. Constant-time comparison: Prevent timing side-channels in signature check<br>2. Raw bytes: Verify against raw body bytes, not parsed/re-serialized JSON<br>3. Allow-list source IPs: Restrict inbound to your platform's published egress range | High |
| Business logic | Replay attack: Attacker captures a legitimate signed webhook and re-sends it later to re-trigger processing | Timestamp in signature | 1. Reject timestamps older than 5 minutes<br>2. Deduplicate on `webhook-id`<br>3. Store processed IDs with a TTL covering the retry window | Medium |
| Signing secret | Secret compromise: Attacker obtains the signing secret and can forge arbitrary webhooks | Encrypted secret per endpoint | 1. Rotate immediately on suspected compromise<br>2. Support dual-secret rotation window<br>3. Role-gated secret visibility | High |
| Endpoint availability | Volumetric flooding: Attacker sends high volume of requests to the webhook endpoint, saturating network or application capacity | Signature verified before any processing | 1. Rate limit by source IP at the edge<br>2. Async processing: Return 200 quickly, process in a background queue<br>3. L7 protection: Place endpoint behind a CDN or load balancer with rate limiting<br>4. Allow-list source IPs: Restrict inbound to your platform's published egress range | Medium |

### If you use inline dispatch

If you skip the queue and send webhooks synchronously in the request path, the threat profile shifts:

- Resource exhaustion moves from Medium to High: a slow endpoint blocks your API, not just a background worker
- Retry becomes harder. Retrying in the request path multiplies latency and ties up threads for the duration of every attempt
- Isolation disappears. A webhook delivery surge affects your core API latency directly, not an isolated dispatcher pool
- Dead-letter recovery is harder to implement without a queue to catch failures

For low-volume systems inline dispatch works, but accept that every delivery failure is a direct hit to your API, not a background worker.
