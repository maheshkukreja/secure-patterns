# Threat Model

## Baseline assumptions

- Clients are untrusted: they can retry, replay, and send concurrent duplicates
- The API authenticates callers and derives tenant context from the auth token, not the request body
- Standard infrastructure controls (TLS, WAF, database AuthN) are in place. This model focuses on the idempotency mechanism itself
- The key store is durable (database-backed for the reference design)
- External side effects are delivered via an outbox worker, not inline during request handling

#### A note on risk: you won't fix everything

This table is not a checklist where every row must be fully eliminated. Focus on preventing the worst failures and limiting blast radius. In practice: ship prevention for the High rows first, then add monitoring and response for what you can't realistically prevent.

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|----------------------------|-----------------|
| Request processing | Concurrent first-writer race: two requests with the same key arrive on different servers, both pass the existence check, both process | Shared durable key store | 1. Atomic reservation: single `INSERT ON CONFLICT` with no check-then-act gap<br>2. `locked_at` timestamp with `409` + `Retry-After` for concurrent arrivals<br>3. Database UNIQUE constraint as final safety net | High |
| Cached responses | Cross-tenant key collision: two tenants generate the same key value, one receives the other's cached response | Auth required on all endpoints | 1. Composite unique constraint: `(tenant_id, idempotency_key)`<br>2. Key lookup always includes tenant context from auth token<br>3. Return 404 if key belongs to a different tenant | High |
| Data integrity | Parameter mismatch: same key sent with different request body, server returns wrong cached result | Key existence check | 1. Store request fingerprint (SHA-256, canonicalized) with the key<br>2. Compare fingerprint on retry; return 422 if mismatched | High |
| Financial integrity | Partial failure: external service processes the request but the local write fails, next retry double-processes | Durable key store | 1. Write-first: save intent + key in one DB transaction before calling external service<br>2. Outbox pattern: publish external calls from durable queue<br>3. Reconciliation: compare external records to internal ledger daily | High |
| Payment flow integrity | Incomplete coverage: some endpoints in a multi-step flow enforce idempotency, others don't | Per-endpoint implementation | 1. Require `Idempotency-Key` on all state-changing endpoints<br>2. Return 400 if a POST is missing the header<br>3. Audit: enumerate all state-changing endpoints and verify each enforces the header | High |
| Request processing path | Fail-open on degraded store: key store unavailable, handler bypasses reservation and processes the request anyway | Required header, durable key store | 1. Fail closed: reject state-changing requests if reservation or lookup cannot complete<br>2. Alert on reservation failure rate and store health<br>3. Circuit-break state-changing endpoints when reservation path is unhealthy | High |
| Outbox worker / downstream execution | Worker replay: worker retries the same durable intent multiple times due to retries, poison-pill loops, or missing downstream correlation | Durable outbox, retryable worker | 1. Carry the idempotency key or correlation ID into downstream provider calls<br>2. Record downstream object/provider ID before marking outbox item complete<br>3. Dead-letter repeatedly failing items after bounded retries<br>4. Alert on dead-letter queue depth | High |
| Cached responses | Key enumeration: attacker guesses key values to probe for cached responses | UUID v4 (128-bit entropy) | 1. Composite lookup requires matching tenant_id<br>2. Rate-limit key lookups per caller | Low |
| Request integrity | Replay after TTL: attacker intercepts a request, replays it after the key expires to re-trigger processing | Key TTL | 1. Request-level timestamps: reject requests older than a threshold<br>2. Bind key to session or auth token, not just tenant<br>3. Shorter TTL for high-value operations | Medium |
| Availability | Error caching: 500 response cached, retries return stale error after the bug is fixed | Response storage | 1. Only cache 2xx and 4xx responses; let 5xx be retried fresh<br>2. If you cache all responses, provide a manual cache-bust for operations teams | Medium |
| Retry safety | TTL race: key expires while the client is still retrying, next retry treated as a new request | TTL policy | 1. Set TTL longer than your longest realistic retry window<br>2. Return a header indicating key expiry time<br>3. For multi-day flows, extend TTL or use a separate tracking mechanism | Medium |
| Key store capacity | Key spraying: attacker creates thousands of keys per second, exhausting storage | Auth required | 1. Rate-limit key creation per tenant<br>2. Reaper job deletes expired keys on schedule<br>3. Cap active keys per tenant | Low |
| Response freshness | Stale cache: cached "success" for a resource later cancelled or reversed | Immutable response cache | 1. Cache the response as-is; don't sync with downstream state changes<br>2. Idempotency cache answers "did this request already run?" not "what is the current state?"<br>3. Clients needing current state call a separate GET endpoint | Low |
