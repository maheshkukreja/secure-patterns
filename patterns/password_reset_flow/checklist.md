# Verification checklist

### Response parity
- [ ] `POST /auth/password-reset` returns the identical status, body, and headers for existing and non-existent emails
- [ ] Response time for existing and non-existent emails is indistinguishable within normal jitter
- [ ] Rate-limit responses do not reveal account existence (either identical to the baseline response, or the leak is documented and bounded)

### Token entropy and storage
- [ ] Tokens are generated from a CSPRNG with at least 256 bits of entropy
- [ ] Only the SHA-256 hash of the token is stored; the raw token never appears in the database or logs
- [ ] The `token_hash` column is unique and indexed
- [ ] Only the auth service role has INSERT/UPDATE grants on the reset-token table (verifiable from an IAM/DB grants audit)

### Token lifecycle
- [ ] Tokens expire 15 minutes after issuance
- [ ] Issuing a new token invalidates any prior pending token for the same account
- [ ] Consumption is a single atomic conditional update; concurrent consumes of the same token result in at most one success
- [ ] Expired and consumed tokens return the same generic error

### Rate limiting
- [ ] Per-email cap applies before any user lookup, keyed on the normalized submitted email string
- [ ] Per-IP and global caps apply in front of per-email
- [ ] Alerts fire on elevated reset request rates

### Consumption and password update
- [ ] New password is validated against length, complexity, history, and breach-corpus rules
- [ ] Password hash uses argon2id or bcrypt with cost >= 12
- [ ] Password update and session revocation run in the same transaction
- [ ] A failed password-policy check leaves the token usable for the user's retry (policy validation runs before atomic consume)

### Session hygiene
- [ ] Every session for the account is deleted on successful reset (web cookies, mobile refresh tokens, API tokens, "remember this device" cookies)
- [ ] Post-reset login issues a new session ID; no existing cookie is reused
- [ ] CSRF tokens rotate on the new session
- [ ] MFA enrollment is not cleared; MFA challenge is required on post-reset login

### Notification and audit
- [ ] A "password changed" email goes to the account owner on every successful reset
- [ ] The notification includes time, source IP coarse geo, and a "this wasn't me" link that can lock the account
- [ ] Reset issuance, consumption, and session revocation are logged as distinct events with correlation IDs
- [ ] Recent reset attempts appear in the user's security dashboard

### Email transport hygiene
- [ ] Reset links use HTTPS only
- [ ] The reset landing page sets `Referrer-Policy: no-referrer`
- [ ] Reset emails are sent from a sender aligned with SPF, DKIM, and DMARC
- [ ] Email templates do not include the raw token anywhere except the signed link
- [ ] Reset link host comes from server config, not the incoming `Host` header; a request with a spoofed `Host` produces an email whose link still points at the configured domain

### Attacker tests
- [ ] Replay: submitting the same valid token twice succeeds at most once; the second submission returns the generic token error
- [ ] Response parity: reset responses for a known-registered email and a random unregistered email are byte-identical in body and headers, and indistinguishable within timing jitter
- [ ] Flood cap: N+1 reset requests for one email inside the per-email window trigger rate-limiting; the target inbox still receives at most one reset email per window
- [ ] Typo preserves token: a valid token submitted with a policy-violating password returns a policy error and the token remains usable for a retry
- [ ] Host-header spoofing: a reset request with a spoofed `Host` produces an email whose link still points at the configured frontend domain
- [ ] Per-token cap: after N invalid consume attempts with the same token, further attempts on that token are rejected even if the token hasn't expired
