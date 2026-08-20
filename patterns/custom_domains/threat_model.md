# Threat Model

## Baseline assumptions

- Tenant admins are authenticated; the hostnames they submit are untrusted input
- The customer's DNS zone is outside your control and can change at any time after verification
- The control plane derives tenant context from the session, not from the submitted hostname or the request's Host header
- The edge terminates TLS and can rebuild its serving set from the binding registry
- Standard infra controls such as TLS configuration, WAF, and database AuthN are assumed to be in place. This model focuses on the hostname-to-tenant binding lifecycle

#### A note on risk

This table is not a checklist. Focus on preventing the highest-impact failures first. Detection and response are acceptable where prevention is impractical.

## Phase 1: Claim

Focus: Binding a hostname to the tenant that controls it

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| Tenant binding | Pre-claim squat: A hostname points at the platform before its owner registers it (DNS staged early, setup docs that say point first), and whichever tenant submits the name first receives the binding and its traffic | None | 1. Fresh challenge: verify a platform-generated token published in the customer zone before any claim activates<br>2. Tenant-scoped record: put the claim identity in the record name or value, with a random token per claim | High |
| Onboarding availability | Claim parking: Any tenant files a claim for a hostname it cannot prove, and a claim that reserves the name blocks the real owner from onboarding for as long as the claim lives | None | 1. Non-exclusive claims: let several tenants hold claims on one name and hand out the unique binding only on proof<br>2. Claim expiry: drop unproven claims after 72 hours<br>3. Rate limits: cap open claims per tenant | Medium |
| Certificate issuance | Cert in the claim window: Managed certificate automation answers the CA's challenge from the platform's own edge once traffic routes there, so a claim that never verified still yields a valid certificate for the name | CA domain-control validation | 1. State gate: create certificate orders only for bindings in `verified` or `active` state<br>2. Registry-sourced orders: the issuance job reads the hostname from the binding row, never from the claim request<br>3. Issuance alerting: alert on any certificate order for a hostname without a verified binding | High |
| Binding uniqueness | Double binding: The claim path stores the hostname as typed while the serve path indexes what clients send (lowercase A-labels without a trailing dot), so the binding that verified is not the binding the serve path finds, and a second encoding of the same name is free for another claim | Exact-string uniqueness | 1. One canonical form: run the same domain-to-ASCII conversion on both paths and key the uniqueness constraint on its output<br>2. Display split: keep the typed form as display metadata only | Medium |
| Overlapping claims | Wildcard shadow: A tenant claims `*.customer.com` while another tenant holds a verified `app.customer.com`, and most-specific-first routing sends that name to one tenant and its sibling names to the other | Exact-hostname uniqueness | 1. Parent proof: verify wildcard claims at the parent name<br>2. Same-owner parent: reject wildcard and exact bindings under one parent that belong to different tenants<br>3. Claim floor: refuse claims at or above the public-suffix boundary | Medium |
| Verification integrity | Forged DNS answer: An attacker who can poison or intercept the verifier's DNS resolution makes the expected token appear for a name they do not control, and the binding verifies | Random per-claim token | 1. DNSSEC validation: for signed zones, an answer counts as proof only if validation succeeds<br>2. Independent paths: corroborate first-time claims and transfers from a second network vantage point<br>3. Indeterminate on error: a resolver failure is neither a pass nor a fail | Medium |

## Phase 2: Serve

Focus: Routing every request through the verified binding

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| Tenant routing | Host-picked tenant: The application resolves the tenant from the request authority before checking that the hostname is a registered binding, so any client-controlled value selects the tenant context | Edge terminates TLS per hostname | 1. Binding-first dispatch: reject a request whose Host has no active binding before any tenant code runs<br>2. Header hygiene: strip or overwrite forwarded host headers between edge and application so internal services see only the binding's hostname<br>3. Registry-sourced links: build password-reset and invite links from the binding record's hostname | High |
| Unbound hostnames | Catch-all serving: A default virtual host or a wildcard platform certificate answers for names with no binding, so any hostname pointed at the platform serves content and completes TLS without a claim | None | 1. Refuse the handshake: abort the TLS handshake for SNI values without an active binding<br>2. 421 on established connections: return `421 Misdirected Request` for an unknown authority, with no tenant content | Medium |
| Connection reuse | Per-connection tenant: The edge resolves the tenant once from the SNI and caches it on the connection, so a reused connection carries requests for a different bound hostname to the first tenant's application | None | 1. Per-request lookup: resolve the binding from each request's authority, whatever SNI opened the connection<br>2. Coverage check: return `421` when the connection's certificate does not cover the requested Host | Medium |

## Phase 3: Drift and release

Focus: Catching ownership changes and cleaning up on release

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| Released hostname | Leftover-record re-claim: The customer's routing record stays up after the tenant releases the hostname, and a claim flow that accepts the routing record as proof activates the name for a different tenant | None | 1. Cooldown: hold released names in a reserved state for a set period, with a fresh proof as the early exit<br>2. Re-claim alerting: alert when a hostname released by one tenant is claimed by another | High |
| Active binding | Silent transfer: The customer's domain expires or is sold, and the binding keeps routing the name to the old tenant because nothing re-checks control after the first proof | One-time verification at claim | 1. Scheduled re-verification: re-resolve the challenge for active bindings<br>2. Grace window: warn the tenant and keep routing through a short window before suspending<br>3. Watch the pointing record: alert when the hostname stops resolving to your target | Medium |
| Certificate and edge state | Teardown gap: Release removes the registry row but the certificate keeps renewing or an edge node serves from a stale copy of the set, so the platform still answers for a hostname it released | Registry drives the serving set | 1. State-coupled renewal: renew only for `active` bindings<br>2. Reissue shared certificates: cut the released name from any multi-name certificate at release | Medium |
| Lifecycle audit | Attribution gap: A hostname moves between tenants through a support transfer, and the record shows the new owner but not who approved it or which proof was checked | State transitions logged | 1. Full transitions: record the actor, both states, and the proof that satisfied the check on every change<br>2. Transfer reason: require a ticket reference on manual transfers | Low |

## If you use the routing record as proof

If the CNAME that routes traffic is also the ownership proof, the trade-offs shift:

- The pre-claim and post-release squats come back unless every routing target is unique per tenant; with a shared target, any tenant can claim any name that already points at you
- Apex domains usually resolve to shared IPs that do not name a tenant, so they still need the TXT challenge
- The routing record is also the only warning you get that ownership changed; when it stops pointing at your target, treat the binding as unverified
- A transfer between two of your own tenants changes nothing in DNS, so there is nothing to check; require a TXT challenge or a support ticket for transfers
