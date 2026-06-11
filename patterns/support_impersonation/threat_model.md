# Threat Model

## Baseline assumptions

- Support agents are semi-trusted: authenticated internal users who may act outside policy, by mistake or intent
- The product derives the acting agent's identity from the agent's own authenticated session, not from the request body
- The customer's data and account-control actions are higher-value than any single support task
- Audit logs are append-only and stored apart from general application logs
- Impersonation runs inside your app, below the customer's identity provider, so the tenant's conditional-access rules (IP allowlists, device posture) do not apply to the support agent
- Standard infra controls (TLS, admin AuthN, secret management, log hygiene) are assumed to be in place. This model focuses on the impersonation lifecycle itself

#### A note on risk

This table is not a checklist. Focus on preventing the highest-impact failures first. Detection and response are acceptable where prevention is impractical.

### Phase 1: Authorization and start

Focus: Restricting who may impersonate whom, and on what grounds

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Customer account | Unauthorized impersonation: Anyone with admin-panel access can act as any account without a separate grant | Admin-panel role required | 1. Gate impersonation behind a dedicated permission, separate from general admin access<br>2. Scope each agent to the accounts they are assigned, not the whole customer base<br>3. Govern access with a role policy, and add a second approver for security-sensitive or high-tier accounts | High |
| Grant integrity | Identity spoofing: The action request sets the actor or subject, so a caller chooses who it acts as or on whose behalf | None | 1. Derive the actor from the agent's authenticated session, never the request body<br>2. Take the subject and mode from the stored grant, not the action request<br>3. Reject actor and subject fields supplied in the body | Medium |
| Accountability | Missing reason: A session starts with no recorded purpose, so a later review cannot separate a legitimate session from abuse | None | 1. Require a non-empty reason and a ticket reference, validated against the ticketing system where possible<br>2. Record the reason and ticket on the session at start<br>3. For sensitive accounts, require and record a second-party approval | Medium |
| Privileged accounts | Privilege jump: An agent impersonates an org owner or admin to take actions the agent could never authorize directly | None | 1. Cap impersonation to target roles at or below the agent's own grant authority<br>2. Route owner-level impersonation through a separate, heightened-approval path<br>3. For consent models, require the customer to opt in before admin-tier accounts are impersonable | Medium |
| Tenant access policy | Conditional-access bypass: Impersonation runs below the customer's IdP, so the session skips the tenant's IP allowlist, device posture, and step-up rules | None | 1. Re-evaluate the tenant's IP and device rules against the support agent's context where the product can read them<br>2. Notify the tenant's security contacts that a session bypassed their access policy<br>3. Let strict zero-trust tenants disable impersonation and fall back to a screen-share | Medium |
| Emergency access path | Break-glass abuse: An emergency bypass of consent or approval becomes the routine way to reach sensitive accounts | Audit log | 1. Put break-glass behind its own permission and a second approver<br>2. Require a short TTL, a mandatory ticket, and a post-session review before the session is marked closed<br>3. Notify security and the customer's admins whenever break-glass starts on a sensitive account | Medium |
| Support agent session | Stolen support session: An attacker with the agent's session or device opens impersonation sessions across many customers, reading PII and keys without a single write | Admin-panel SSO session | 1. Require a fresh step-up (WebAuthn or MFA) at the moment impersonation is requested, not an existing login<br>2. Bind the grant to that step-up so it cannot be replayed from a stale session<br>3. Rate-limit and alert on a burst of impersonation starts by one agent | Low |

### Phase 2: Active session

Focus: Constraining what the session can do and keeping it visible

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Customer data | Sensitive action abuse: During a session the agent changes credentials, exports data, deletes the account, or alters billing | None | 1. Deny a defined high-risk set in the impersonation context (credential, MFA, and email changes, data export, deletion, billing, and new API keys or impersonations), and deny any action whose policy decision is unavailable<br>2. Default the session to read-only; require explicit elevation and a second approval for writes<br>3. Step-up re-authenticate the agent before any allowed write | Medium |
| Account oversight | Silent access: The customer has no way to see that support is acting in their account | None | 1. Surface active and recent impersonation sessions to the customer's own admins in the product<br>2. Notify the customer when a session starts, in real time for sensitive accounts or regulated data | Medium |
| Account integrity | Wrong-account action: The agent forgets a session is live and acts in the customer's account thinking it is their own or a test tenant | None | 1. Show the agent a persistent banner naming the impersonated customer and the time remaining<br>2. Visually distinguish the impersonation view from the agent's normal view<br>3. Confirm the target account before the first write | Low |
| Session credential | Becoming the user: The act-as token behaves like a normal customer session or can be exchanged for the customer's refresh token | None | 1. Mint a distinct token type with an `act` claim, recognizable to downstream services as impersonation rather than a normal session<br>2. Bind the audience to the one subject and tenant, and block exchange for any standard session or refresh token | High |

### Phase 3: Audit and teardown

Focus: Attributing every action and ending access cleanly

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Audit trail | Audit ambiguity: Impersonated actions are attributed to the customer alone, either in the app's own logs or after the gateway downgrades the act-as token for a downstream service, so an investigation cannot tell who acted | Per-user action logs | 1. Write the agent (actor), the customer (subject), the `impersonation_id`, and the ticket on every action<br>2. Propagate the `act` claim to every downstream service and never downgrade it to a plain customer token; reject act-as calls that arrive without it<br>3. Keep impersonation events in an append-only store separate from general logs, and emit distinct start and end events | Medium |
| Session lifecycle | Lingering session: The session outlives the support need, or an ended, expired, or revoked session keeps working | TTL expiry | 1. Set a short absolute TTL and an idle timeout, and end the session when the linked ticket closes<br>2. Back the token with a server-side session so revocation stops it on the next request, with no offline-verifiable token outliving the record<br>3. Give security and the customer's admins a kill switch, and revoke an agent's active sessions when their impersonation permission is removed | Medium |

### If you use customer-granted access

If the customer grants access instead of an internal policy or approver, the trade-offs shift:

- A revoked customer permission must end active sessions at once, not only block new ones
- Consent does not retire the high-risk action policy; a consenting customer still does not expect support to reset their password
- The audit record must capture the consent grant and its scope alongside each session
- Emergencies still need a break-glass path, which brings back the internal-approval threats for that path
