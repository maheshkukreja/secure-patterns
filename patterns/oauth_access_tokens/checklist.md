# Verification checklist

### Token encryption
- [ ] Each tenant's tokens are encrypted with a distinct DEK
- [ ] KMS decrypt permissions are restricted to the token service's IAM role
- [ ] Decrypting a token record with a different tenant's DEK fails

### Tenant isolation
- [ ] Querying for a token with a valid `integration_id` but wrong `tenant_id` returns "not found"
- [ ] Every query path includes `WHERE tenant_id = ?` with `tenant_id` from verified auth context
- [ ] Cross-tenant lookups return identical responses whether the integration exists or not

### Token lifecycle
- [ ] Two concurrent requests for the same expired token result in exactly one provider refresh call
- [ ] If the lock backend is unavailable, refresh fails closed and the outbound request is rejected
- [ ] The new refresh token is persisted atomically before the lock is released
- [ ] A failed refresh marks the integration as degraded and alerts the tenant
- [ ] Stored `scopes` are compared on each refresh; scope changes trigger an alert
- [ ] Disconnecting an integration revokes at the provider and deletes the local record
- [ ] A periodic sweep catches zombie tokens for disabled or churned tenants

### Outbound safety
- [ ] The `execute_request` response contains the third-party API's response, not the credential used
- [ ] Token values do not appear in application logs, error messages, or monitoring dashboards
- [ ] Every outbound call is logged with `tenant_id`, `integration_id`, target host, and HTTP status

### Detection
- [ ] KMS throttling or timeout causes a controlled failure and alert, not unbounded retries
- [ ] Alerts fire on: refresh failure rate exceeding threshold, KMS decrypt calls from unexpected principals, lock backend health degradation
- [ ] Token usage logs support investigating "which tenant's credentials were used to call which API at what time"
