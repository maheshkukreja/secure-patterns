# Threat Model

## Baseline assumptions

- The invitee's email account is semi-trusted. Inbox compromise and forwarding rules are in scope; full end-to-end email confidentiality is not assumed
- The invitee is untrusted: the link may be replayed or forwarded, and may be submitted from any account the recipient controls
- The control plane derives tenant context from the auth token, not the request body
- Invitation emails are sent through a verified sender with SPF, DKIM, and DMARC alignment
- Standard infra controls such as TLS, WAF, database AuthN, and logging hygiene are assumed to be in place. This model focuses on the invitation lifecycle itself

#### A note on risk

This table is not a checklist. Focus on preventing the highest-impact failures first. Detection and response are acceptable where prevention is impractical.

### Phase 1: Issuance

Focus: Restricting issuance authority and locking the invitation record at creation time

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Tenant role assignment | Privilege creep: An inviter with a tenant-admin role assigns the invitee a higher role, such as owner, even though they are not allowed to grant it | Inviter must be a tenant admin | 1. Server-side rule based on an explicit assignable-role policy. If roles do not have a clear order, define exactly which roles each role can grant<br>2. Require a second admin to confirm role-elevating invites<br>3. Treat owner-role invites as a separate, audited path with longer review | High |
| Invitation record | Mass assignment: Request body overwrites server-controlled fields (such as `tenant_id`, `inviter_id`, or `consumed_at`) | None | 1. Allowlist invite-creation fields explicitly; derive `tenant_id` and `inviter_id` from auth context<br>2. Make ownership fields immutable in the schema and at the ORM layer | Medium |
| Tenant boundary | Wrong-tenant binding: Invite created against a tenant the caller does not administer | Auth context is available per request | 1. Resolve `tenant_id` from the URL path and verify the caller is an admin of that tenant<br>2. Reject any tenant identifier supplied in the body | Medium |

### Phase 2: Token and email

Focus: Protecting the token between issuance and acceptance

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Claim token | Guessing: Attacker brute-forces tokens against the accept endpoint | None | 1. CSPRNG-sourced token with at least 256 bits of entropy<br>2. Throttle and alert on repeated failed binding checks (per token, per IP, and globally); do not invalidate the token on failed attempts, since invalidation lets a leaked-link holder burn the legitimate user's invitation<br>3. Per-IP and global rate limits on the accept endpoint | Low |
| Token store | Storage exposure: A DB backup or table dump leaks pending invitations | None | 1. Store only `SHA-256(token)`, never the raw value<br>2. Restrict direct access to the invitations table to the auth service role<br>3. Alert on bulk reads of the table | Medium |
| Invitation link | Email transit exposure: Link captured from any place the email reaches (inbox compromise, forwarded message, shared device, etc.) | None | 1. Short TTL: 7 days for member invites, 24 to 48 hours for admin invites<br>2. `Referrer-Policy: no-referrer` on the landing page to stop Referer leaks to analytics and embedded assets<br>3. Recipient binding at accept (Phase 3) means a captured link is not directly usable | Medium |
| Invitation link host | Host header poisoning: Request submitted with a spoofed `Host`; server embeds that host in the email link, so the victim clicks a trusted email pointing at attacker-controlled infrastructure | None | 1. Build invite URLs from a config value, never `req.headers.host`<br>2. Allowlist-validate the host if multi-brand domains exist | Medium |

### Phase 3: Acceptance

Focus: Ensuring only the intended recipient can turn the invitation into membership

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Tenant membership | Wrong-account acceptance: Forwarded or leaked link accepted by a different account than the invited address | None | 1. Require an authenticated session at accept; reject anonymous calls<br>2. Match the session's verified email to `invited_email` exactly after normalization<br>3. For SSO-enforced tenants, require that the session was authenticated by the tenant's configured IdP connection | High |
| Identity binding | Click-to-provision: A combined invite-and-signup flow auto-creates an account and treats the invitation click as proof of email ownership, so any unintended actor that received the link (enterprise link scanner, forwarded recipient, mailbox malware) becomes the account owner | None | 1. Treat the invitation token as proof of link possession only, never as email verification<br>2. Require independent identity establishment (SSO sign-in or signup with a separate email-verification step) before the accept handler runs<br>3. If the UX combines signup and accept, hold the invitation pending until the new account completes its own email verification | High |
| Invitation record | Token reuse: Same token consumed twice by a link preview or a rapid double-click, creating duplicate memberships or partial state | None | 1. Atomic conditional update sets `consumed_at` and inserts membership in the same transaction<br>2. Membership table has a unique constraint on `(tenant_id, principal_id)`<br>3. Second attempt returns the same generic error as an invalid token | Medium |
| Pending grant | State drift: Role or tenant status on the pending grant is no longer accurate at accept time (admin demoted, tenant suspended) | None | 1. Re-check tenant status (`active`) at accept and reject acceptance into a suspended tenant<br>2. Surface the role from the pending grant on the preview page so the invitee sees what they are accepting before clicking accept | Medium |

### Phase 4: Post-acceptance

Focus: Visibility and revocation after the grant is applied

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Pending invitations | No revocation path: Admin notices a misdirected invite (typo on the email, wrong role) but cannot rescind it before acceptance | None | 1. `DELETE /tenants/{id}/invitations/{iid}` flips `revoked_at`; the accept handler rejects revoked rows with the same generic error as expired ones<br>2. Cascade revoke when the inviter is offboarded or the tenant is suspended | Medium |
| Tenant audit | Silent membership change: Membership appears with no record of who invited or who accepted | Membership row only | 1. Log each lifecycle event (issuance, acceptance, revocation) as a distinct audit event with correlation IDs<br>2. Notify the inviter on acceptance | Low |
| Operational logs | Token leakage in logs: Sensitive payload (raw claim tokens, full invite-link query strings, or recipient PII) appears in operational logs (mailer, debug traces, or error reporters) | None | 1. Redact `claim_token` and full link query strings from request and response logs, and scrub them from mailer and error reporters<br>2. Mailer logs include `invitation_id` and a recipient hint only, never the link<br>3. Error reporters scrub query strings and bodies on 4xx responses from the accept endpoint | Medium |

### If you use stateless tokens

If you use signed tokens instead of storing invitations server-side, the trade-offs change:

- To revoke an invite before it expires, you have to rotate the signing key, which invalidates every pending invite
- You cannot reliably enforce one active invite per `(tenant, email, role)` without shared state
- Preventing token reuse also needs shared state, so much of the benefit of going stateless disappears
- Audit data is limited to what was put into the token when it was created

Use stateless tokens only when a server-side store is not an option.
