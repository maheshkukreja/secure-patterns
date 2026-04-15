# Threat Model

## Baseline assumptions

- Your SaaS authenticates tenants and derives `tenant_id` from verified credentials, not request parameters
- OAuth providers implement RFC 6749 correctly (authorization code flow, token endpoint, revocation endpoint)
- Tokens are bearer credentials: anyone holding a valid access token can use it. Token binding (DPoP, mTLS) is not yet widely supported by third-party providers, so the architecture does not assume it
- Standard OAuth flow hardening (exact-match redirect URIs, PKCE, CSRF-bound `state` parameter) is in place. This model focuses on what happens after tokens are acquired
- Standard infra controls (TLS, WAF, database AuthN, SQLi prevention) are in place

### Phase 1: Token storage and lifecycle

Focus: Preventing token exposure, enforcing tenant isolation, and managing credential lifecycle

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Token store | Bulk exposure: Database breach, backup leak, or snapshot copy exposes all tokens | Database access controls | 1. Envelope encryption: per-tenant DEK wrapped by KEK in KMS/HSM<br>2. Per-tenant keys: a single DEK compromise limits blast radius to one tenant<br>3. No plaintext path: verify no token value appears in logs, error messages, monitoring, or backups | High |
| Tenant isolation | Cross-tenant token access: Application bug or IDOR allows one tenant's code path to use another tenant's credential through the token service | Auth context | 1. Tenant filter: every token service lookup includes `WHERE tenant_id = ?` with `tenant_id` from verified auth context, never from request parameters<br>2. Opaque errors: cross-tenant lookups return "integration not found," never "access denied" | High |
| Stored scopes | Scope overreach: Stored tokens carry broader scopes than the integration uses, so a vault compromise gives attackers more access than the feature requires | OAuth consent screen | 1. Scope inventory: document required scopes per integration and compare against stored grants<br>2. Scope drift detection: compare stored `scopes` against provider response on each refresh, alert on unexpected expansion | Medium |
| Refresh token | Rotation race: Two concurrent requests trigger refresh simultaneously. The provider rotates the refresh token on first use; the second caller sends a stale token, and the provider revokes the entire token family | Centralized refresh | 1. Distributed lock: acquire a per-integration lock before refreshing<br>2. Wait and re-read: if the lock is held, back off and read the (likely already refreshed) token<br>3. Atomic store: persist the new refresh token before releasing the lock<br>4. Fail closed: if the lock backend is unavailable, reject the outbound request rather than refreshing without coordination | Low |
| Encryption keys | KEK compromise: Attacker gains KMS access, making all DEK encryption ineffective | Cloud KMS access policies | 1. Least privilege: restrict KMS decrypt to the token service's IAM role only<br>2. Audit: alert on decrypt calls from unexpected principals or unusual volume 3. Automatic KEK rotation in KMS | Medium |
| Token lifecycle | Zombie tokens: Tenant disconnects an integration or churns, but tokens persist and remain usable at the provider | Manual cleanup | 1. Revoke at provider: call the revocation endpoint (RFC 7009) on disconnect<br>2. Delete local: purge the encrypted record and evict any cached plaintext<br>3. Sweep: periodic job catches tokens the disconnect flow missed | Low |

### Phase 2: Outbound token use

Focus: Preventing credential misrouting and leakage when the token service makes API calls on behalf of tenants

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Outbound request | Confused deputy: Bug in the token service resolves the wrong tenant's credential for an outbound call, making an API request using another customer's token | Tenant context binding | 1. Strict lookup: `execute_request` resolves credentials by `tenant_id` + `integration_id`; both must match the same record<br>2. No fallback: if the lookup returns no match, fail the request, never try a broader search<br>3. Audit: log `tenant_id`, `integration_id`, and target URL for every outbound call | High |
| Token service | Outage: The token service is unavailable, blocking all outbound integrations for all tenants | Redundant deployment | 1. High availability: deploy the token service with replica count and health checks matching your SLA<br>2. Graceful degradation: application code receives a clear "integration unavailable" error, not a timeout<br>3. No bypass: application code must not cache or store tokens as a fallback when the service is down | Medium |
| Access tokens | Credential leakage: Access token appears in request logs, error stack traces, monitoring dashboards, or crash dumps from the token service process | Standard logging, short-lived cache | 1. Header redaction: strip `Authorization` headers from all log outputs<br>2. Structured logging: field-level redaction rules in the logging framework<br>3. Short TTL: cache entries expire in minutes, evict on disconnect<br>4. Process isolation: restricted core dump settings for the token service | Medium |
