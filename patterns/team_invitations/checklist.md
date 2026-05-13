# Verification checklist

### Issuance and grant integrity
- [ ] Inviter cannot assign a role higher than their own
- [ ] `tenant_id` and `inviter_id` come from auth context, never from the request body
- [ ] Owner-role invites go through a separate, audited path
- [ ] Ownership fields (`tenant_id`, `invited_email`, `role`, `inviter_id`) are immutable after creation
- [ ] Issuing a new invitation for the same target key supersedes any prior pending row in the same transaction (`revoked_at` set, token hash rotated)
- [ ] A member-role admin cannot create an admin-role invitation

### Claim token storage
- [ ] Tokens generated from a CSPRNG with at least 256 bits of entropy
- [ ] Only `SHA-256(token)` is stored; the raw token never appears in the database or logs
- [ ] The `claim_token_hash` column is unique and indexed
- [ ] Only the auth service role has `INSERT` / `UPDATE` on the invitations table (verifiable from an IAM/DB grants audit)

### Recipient binding and identity
- [ ] Invitation acceptance requires an independently verified identity (SSO sign-in or completed signup with a separate email-verification step); the click itself is never treated as proof of email ownership
- [ ] An unauthenticated POST to the accept endpoint returns 401, not 400 or 404
- [ ] The session's verified email is compared to `invited_email` after normalization (lowercased, IDNA, trimmed)
- [ ] For SSO-enforced tenants, the accept handler verifies the session's authentication method matches the tenant's configured IdP connection (a local-password session at a matching address is rejected)
- [ ] The session's current verified email is used for the comparison; an invitee whose IdP email has changed since issuance must be reissued an invitation by an admin

### Atomic acceptance
- [ ] `GET /invitations/{token}` returns preview data with no state change; only `POST /invitations/{token}/accept` mutates state
- [ ] Acceptance is a single transaction: atomic conditional consume (`consumed_at` set), then membership insert, then audit event
- [ ] Concurrent accepts of the same token result in at most one success
- [ ] The membership table has a unique constraint on `(tenant_id, principal_id)`
- [ ] The durable audit row is written inside the acceptance transaction; the notification email is enqueued only after commit
- [ ] All failed accepts use the same status, body, and timing within normal jitter
- [ ] Submitting the same valid token twice succeeds at most once

### Lifecycle and revocation
- [ ] Revoked invitations are rejected at accept with the same generic error as expired ones
- [ ] Inviter offboarding cascades to revocation of pending invitations they created
- [ ] Tenant suspension cascades to revocation of all pending invitations for that tenant
- [ ] The old link returns the generic invitation error immediately after a replacement invitation is issued (same status, body, timing as any revoked row)

### Audit and logging
- [ ] Each lifecycle event (issuance, acceptance, revocation) is logged as a distinct audit event with correlation IDs
- [ ] The inviter is notified on acceptance
- [ ] The pending invitations list is visible to tenant admins on a settings screen
- [ ] Raw claim tokens and full invite-link query strings never appear in any operational log; recipient PII follows the same redaction policy (verifiable with a log-grep test on a known-issued token)
- [ ] 4xx responses from the accept endpoint do not include the submitted token in error bodies or telemetry

### Email transport hygiene
- [ ] Invite links use HTTPS only
- [ ] The invite landing page sets `Referrer-Policy: no-referrer`
- [ ] Invitation emails are sent from a sender aligned with SPF, DKIM, and DMARC
- [ ] Invite link host comes from server config, not the incoming `Host` header; a request with a spoofed `Host` produces an email whose link still points at the configured domain
