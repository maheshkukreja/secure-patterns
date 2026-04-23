# Threat Model

## Baseline assumptions

- The user's email account is semi-trusted. Inbox compromise and email forwarding rules are in scope; full end-to-end email confidentiality is not assumed
- Attackers can make arbitrary authenticated and unauthenticated HTTP requests. They control their own user agent, network, and any cookies they can steal
- The password store uses a modern memory-hard hash (argon2id or bcrypt with cost >= 12). Credential stuffing controls, MFA, and device posture are out of scope for this pattern; assume they're handled separately
- The reset email is sent through a verified sender with SPF, DKIM, and DMARC alignment
- Standard infra controls such as TLS, WAF, database AuthN, and logging hygiene are assumed to be in place. This model focuses on the state transitions of the reset flow itself

#### A note on risk: you won't fix everything

This table isn't a checklist where every row must be fully eliminated. Focus on preventing the worst failures and limiting blast radius. In practice: ship prevention for the High rows first, then add monitoring and response for what you can't realistically prevent.

### Phase 1: Request intake

Focus: Preventing the request endpoint from leaking account existence or amplifying abuse

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| User account | Enumeration via response parity, timing, or rate-limit behavior: Attacker submits emails and distinguishes accounts from response body, headers, latency, or per-email 429 timing | Generic 202 + async worker |<br>1. Fixed body, status, Content-Length, and headers on every response, regardless of account existence<br>2. Do all user-lookup and rate-limit work on the async path; key limits on the raw submitted email<br>3. Keep the synchronous handler's timing identical across inputs | Low |
| Victim's inbox | Targeted email flood: Attacker triggers many resets to fill a victim's inbox or push legitimate emails out | No rate limits by default |<br>1. Per-email cap (e.g., 5 per 15 min)<br>2. Consolidate repeated requests in a short window into one email ("we already sent you a reset link in the last 5 minutes")<br>3. Per-IP (e.g., 20 per 15 min) and global caps in front of per-email | Medium |

### Phase 2: Token lifecycle

Focus: Protecting the token between issuance and consumption

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Reset token | Guessing: Attacker enumerates tokens by brute force against the consume endpoint | Unbounded guessing against a predictable token |<br>1. CSPRNG-sourced token with at least 256 bits of entropy<br>2. Per-token attempt cap (e.g., 3 attempts before the token is invalidated)<br>3. Per-IP and global rate limits on the consume endpoint | Low |
| Token store | Storage exposure: DB backup or table dump leaks stored tokens | Raw token persisted |<br>1. Store only `SHA-256(token)`, never the raw value<br>2. Restrict direct access to the token table to the auth service<br>3. Alert on bulk reads of the table | Medium |
| Reset link | Email transit exposure: Link is captured from an insecure mail hop, browser history, or shared device | No TTL, long-lived link |<br>1. Short TTL (e.g., 15 minutes)<br>2. Single-use consume: an atomic conditional update invalidates the token on the first success; superseding prior pending tokens on new issue closes the stale-link gap<br>3. `Referrer-Policy: no-referrer` on the landing page to stop Referer leaks to analytics and embedded assets | High |
| Reset link host | Host header poisoning: Attacker submits the reset request with a spoofed `Host` header; server embeds that host in the email link, so the victim clicks a trusted email that points at attacker-controlled infrastructure | Framework-default (link host from request) |<br>1. Build reset URLs from a config value, never from `req.headers.host`<br>2. Allowlist-validate the host if multi-brand domains exist<br>3. Enforce `https` at link-construction time | Medium |
| Reset flow | Inbox takeover: Attacker with mailbox access triggers and consumes a reset without the owner's knowledge | None (silent reset by default) |<br>1. Send a "password changed" email on every successful reset (including admin-initiated), including source IP, coarse geo, time, and a "this wasn't me" link that can lock the account, force email re-verification, and revoke lingering sessions<br>2. Require MFA re-verification on post-reset login for high-risk accounts<br>3. Log reset issuance, consumption, and revocation as distinct audit events | Medium |
| Reset token | Token reuse: Same token consumed twice (for example, two rapid clicks from the email client's link preview) | Non-atomic lookup-then-update |<br>1. Atomic conditional update that sets `consumed_at` and returns success only on the first call<br>2. Second attempt returns the same generic error as an invalid token<br>3. Do not expose "already used" as a distinct error to the client | Medium |

### Phase 3: Consumption and post-reset state

Focus: Ensuring that the reset actually cleans up post-compromise state and cannot be used to escalate

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Account sessions | Surviving session after reset: Attacker's existing session remains valid even though the password changed | Default session lifecycle (sessions unaffected by password change) |<br>1. Invalidate every session for the user on password change<br>2. Revoke refresh tokens, mobile tokens, and "remember this device" markers<br>3. Revoke password-protected API tokens the user created | High |
| Account sessions | Session fixation through reset: Auto-login after reset reuses a session ID the attacker planted | Login flow may reuse any cookie the client sends |<br>1. Issue a brand-new session cookie with a new session ID on post-reset login<br>2. Set `HttpOnly`, `Secure`, `SameSite=Lax` or stricter<br>3. Rotate CSRF tokens | Medium |
| MFA enrollment | MFA bypass via reset: Reset flow clears the user's 2FA enrollment, so an attacker with email access bypasses 2FA entirely | Framework defaults may clear MFA on reset |<br>1. Do not unenroll MFA as part of reset<br>2. Require MFA challenge on the post-reset login even if the user just reset<br>3. If MFA is lost, route to a separate recovery flow with longer delays and out-of-band verification | High |

### If you use stateless tokens

If you ship signed tokens instead of a server-side store, the profile shifts:

- Token revocation before the token expires is not possible without rotating the service signing key
- The "at most one active reset token" invariant becomes unenforceable, so the email-flood and inbox-takeover rows become harder to constrain
- Token reuse prevention requires a shared store anyway (a consumed-nonce set), so there's little left to gain by going stateless
- Forensic context per issued token is limited to whatever is encoded in the token; no later annotations

Use stateless tokens only if the alternative is nothing (no state available at all).
