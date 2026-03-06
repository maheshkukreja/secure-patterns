# Threat Model

### Phase 1: Ceremony integrity

Focus: Preventing replay, phishing, and verification bypasses

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Ceremony | Challenge replay: Attacker captures a signed WebAuthn response and replays it against the RP. If challenges aren't session-bound and single-use, the replayed response passes verification | Server-generated challenge | 1. Session binding: Store challenges server-side, tied to the authenticated or anonymous session that initiated the ceremony<br>2. Single-use: Delete the challenge after successful verification or TTL expiry<br>3. Short TTL: Expire unused challenges within 60-120 seconds | High |
| Origin trust | Sibling-domain phishing: RP sets `rp.id = "example.com"`. Attacker compromises `blog.example.com` (XSS, subdomain takeover) and initiates a ceremony with the same RP ID. The authenticator produces a valid assertion because `rpIdHash` matches. If the RP doesn't verify `origin` in `clientDataJSON`, it accepts the assertion | RP ID binding | 1. Strict origin check: Verify `origin` in `clientDataJSON` against an explicit allowlist using exact string equality (`===`). Do not use regex or suffix matching (`endsWith`), which allows bypasses like `https://malicious-example.com`<br>2. Subdomain hygiene: Audit all subdomains under your RP ID domain. Unused subdomains are takeover targets<br>3. Narrow RP ID: If your app runs on a single subdomain, set `rp.id` to that subdomain rather than the registrable domain | Medium |
| User verification | UV bypass: RP requests `userVerification: "required"` but doesn't check `UV=1` in the response flags. A stolen hardware key without a PIN set would authenticate with UP only (someone touched it) without verifying the user's identity | Flag in authenticatorData | 1. Flag enforcement: If you set `userVerification: "required"`, verify `UV=1` in `authenticatorData` flags. Reject if `UV=0`<br>2. Registration policy: Require UV at registration so the authenticator sets up a local verification method (PIN, biometric) | Medium |

### Phase 2: Credential management

Focus: Preventing enumeration, confusion, and stale credential risks

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| User privacy | Credential enumeration: RP returns `allowCredentials` (list of credential IDs) for a given username during authentication. Attacker probes usernames to determine which accounts exist and how many credentials they have | Auth required for lookup | 1. Discoverable credentials: Use the passkey flow where `allowCredentials` is empty. The authenticator finds matching credentials internally by `rpIdHash`<br>2. Conditional UI: Use `mediation: "conditional"` so the browser populates the credential picker from its own store<br>3. Dummy responses: For non-discoverable fallback flows, return a plausible-looking dummy challenge for nonexistent users | Low |
| Credential integrity | Credential confusion: Attacker registers a credential ID that already belongs to another user. If the RP doesn't enforce uniqueness, authentication becomes ambiguous | Credential lookup by ID | 1. Uniqueness check: Reject registration if the `credentialId` already exists for any user<br>2. User binding: Always verify that the credential's associated `userHandle` matches the authenticating user | Medium |
| Cloned authenticator | Sign count regression: For device-bound credentials (`BE=0`), an attacker who clones the authenticator's private key produces assertions with a stale or divergent sign count. Without verification, the RP cannot detect the clone | signCount stored per credential | 1. Conditional enforcement: For `BE=0` credentials, flag or reject if the new `signCount` is less than or equal to the stored value<br>2. Skip for synced: For `BE=1` credentials, accept `signCount=0` (synced passkeys don't reliably increment counters)<br>3. Alert: Log sign count anomalies for investigation | Low |
| Credential metadata | Missing transport hints: RP doesn't store `transports` from `getTransports()` at registration. Future authentication ceremonies can't hint the browser about which authenticator to prompt (USB, BLE, NFC, internal), degrading UX and causing fallback delays | None | 1. Store transports: Call `response.getTransports()` during registration and persist the result alongside the credential<br>2. Pass hints: Include stored transports in `allowCredentials` descriptors during authentication | Low |

### Phase 3: Post-ceremony session

Focus: Preventing session hijacking after a successful passkey ceremony

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Authenticated session | Session theft: The passkey ceremony proves identity at one moment. The resulting session token is a bearer token. If stolen (XSS, malware, network interception on a misconfigured setup), the attacker inherits the session without ever touching the authenticator | Standard session management | 1. Short-lived sessions: Reduce the window during which a stolen token is useful<br>2. Step-up auth: Require a fresh passkey ceremony for sensitive operations (password change, payment, admin actions)<br>3. Binding signals: Tie the session to client properties (IP range, TLS fingerprint) where feasible, accepting the false-positive cost for mobile users | Medium |
| Account | Credential stuffing the ceremony endpoint: Attacker floods the authentication endpoint with ceremony initiation requests to exhaust server-side challenge storage or cause resource exhaustion | Rate limiting | 1. Per-IP rate limits: Throttle ceremony initiation requests<br>2. Challenge storage limits: Cap the number of pending challenges per session or per IP<br>3. Proof of work: For anonymous ceremony initiation, consider lightweight client puzzles | Low |
