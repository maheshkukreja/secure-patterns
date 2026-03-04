# Verification checklist

### Ceremony verification
- [ ] `origin` in `clientDataJSON` is checked against an explicit allowlist of expected origins using exact string equality (not regex or suffix matching)
- [ ] `challenge` in `clientDataJSON` matches the server-side challenge for this specific session
- [ ] Challenges are single-use and expire within 120 seconds
- [ ] `rpIdHash` in `authenticatorData` equals SHA-256 of the configured RP ID
- [ ] `type` in `clientDataJSON` is `"webauthn.create"` for registration and `"webauthn.get"` for authentication
- [ ] For discoverable credential flows, `userHandle` from the assertion identifies the user, and `credentialId` is confirmed to be bound to that user
- [ ] If the challenge store is unreachable, ceremony initiation fails closed (returns error), not open (skips challenge binding)

### Flag enforcement
- [ ] `UP=1` is verified on every ceremony
- [ ] `UV=1` is verified when `userVerification: "required"` was requested
- [ ] `BE` and `BS` flags are stored per credential and available for policy decisions

### Credential storage
- [ ] Each credential record stores: `credentialId`, public key, `signCount`, `transports`, `aaguid`, `BE`, `BS`
- [ ] `credentialId` is unique across all users (registration rejects duplicates)
- [ ] Sign count is verified for `BE=0` credentials; `signCount=0` is accepted for `BE=1` credentials

### Enumeration resistance
- [ ] Authentication endpoints return identical response timing and structure for existing and non-existing users
- [ ] The primary authentication flow uses discoverable credentials (empty `allowCredentials`)

### Session management
- [ ] Sensitive operations require step-up authentication (fresh passkey ceremony), not just an existing session
- [ ] Ceremony initiation endpoints are rate-limited per IP

### Environment isolation
- [ ] RP ID is environment-specific; credentials registered in dev/staging cannot authenticate in production

### Attestation (if verifying)
- [ ] Attestation root certificates are sourced from FIDO Metadata Service and updated regularly
- [ ] Only AAGUIDs on the allowlist are accepted during registration
- [ ] Credentials with unknown or revoked attestation are rejected
