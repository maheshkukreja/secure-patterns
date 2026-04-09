# Verification checklist

### Key reservation and tenant isolation
- [ ] Send two identical POST requests simultaneously; only one creates the resource, both return the same response
- [ ] Same key with different request body returns 422
- [ ] Two different tenants send the same key value; each gets their own independent result, no cached-response bleed
- [ ] Same key and payload but different auth/session context: verify lookup is scoped correctly
- [ ] Keys have `(tenant_id, idempotency_key)` as the unique constraint

### Concurrent and distributed behavior
- [ ] Concurrent requests with the same key from different app nodes: only one path performs side effects
- [ ] `409 Conflict` response includes `Retry-After` header
- [ ] POST request without `Idempotency-Key` header returns 400

### Failure handling and resilience
- [ ] Crash the worker after the external side effect succeeds but before local completion is recorded; retry and reconciliation do not create a duplicate side effect
- [ ] Bring the key store down; endpoint fails closed before any business mutation or external call
- [ ] First request returns 500; retry re-attempts processing instead of returning cached 500
- [ ] Every state-changing endpoint in a multi-step flow either enforces `Idempotency-Key` or rejects the request

### Lifecycle
- [ ] Retry after TTL expiry creates a new request and does not silently duplicate the original side effect
- [ ] Reaper job deletes expired keys without locking the table for production traffic

### Observability and detection
- [ ] Duplicate/replay metrics and alerts exist
- [ ] Duplicate request rate is tracked per tenant (spikes may indicate client bugs or replay attacks)
- [ ] Stale-processing sweeper resolves stuck keys within the lock timeout window
- [ ] Cached response for tenant A is not accessible by tenant B
