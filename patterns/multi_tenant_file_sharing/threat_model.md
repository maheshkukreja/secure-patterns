# Threat Model

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Tenant isolation | IDOR: Attacker guesses object_id to access other tenants' files | Opaque IDs | 1. Binding: Enforce object.tenant_id == user.tenant_id on every lookup<br>2. No paths: Never accept bucket or key parameters from clients<br>3. 404s: Cross-tenant requests return 404 (hide existence) | Medium |
| AuthZ logic | Confused deputy: One endpoint skips or weakens checks (e.g., "Download" vs "Preview") | Central authentication | 1. Centralize: Single can_access() authorization function used by all endpoints<br>2. Fail closed: Reject access if policy logic is ambiguous or unhandled | Medium |
| Object lifecycle | Leakage: Quarantined or "Soft Deleted" objects remain downloadable | Status field in the object registry | 1. Status check: Reject grants unless status == published (default deny)<br>2. Exception: Allow quarantine access only for the uploader | Medium |
| Share links | Forever access: Token behaves like a permanent public link | Server-side state | 1. Revoke tokens: DB-backed tokens allow instant revocation<br>2. Re-auth: Re-verify object status before issuing the grant | High |
| Registry integrity | Mass assignment: Attacker modifies tenant_id or policy fields to steal ownership | DB permissions | 1. Field locking: Restrict UPDATE access on ownership fields (immutable after creation)<br>2. Audit: Log all state changes to an immutable audit trail | Medium |
| Offboarding | Zombie access: Deleted tenant’s files remain accessible via old links | Soft delete | 1. Cascade: Revoke all share links when a tenant is disabled<br>2. Block: Global check for tenant.isActive in the auth flow | Low |
| Token leakage | Log leaks: Share tokens appear in logs / analytics | Short-lived delivery grants | 1. Redaction: Redact query params in server logs<br>2. Monitoring: Monitor for unusual access patterns on token endpoints | Low |
