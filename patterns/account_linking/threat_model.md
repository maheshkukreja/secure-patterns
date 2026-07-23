# Threat Model

## Baseline assumptions

- Clients are untrusted: they can replay callbacks and claim any address in a request body
- Providers are semi-trusted: signatures and transport are sound, but a provider's claims about addresses on domains it does not host carry no authority
- The control plane derives the acting account from the session, not from any assertion or request field
- Provider signing keys are fetched over verified channels from configured endpoints
- Standard infra controls such as TLS, CSRF protection on callbacks, and secret management are assumed to be in place. This model focuses on the binding decision: which external identity may attach to which account

#### A note on risk

This table is not a checklist. Focus on preventing the highest-impact failures first. Detection and response are acceptable where prevention is impractical.

## Phase 1: Adding a method

Focus: Ensuring only the account owner can attach a new identity

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| Account ownership | Email-match takeover: Attacker authenticates at a provider that asserts the victim's address without verifying it, or that reports verification in a type the parser misreads, and the system attaches the attacker's identity to the victim's account because the addresses match | None | 1. Never link from an email match alone; require an authenticated session on the target account plus fresh re-authentication<br>2. Key the binding on issuer and subject; keep the address as a display hint<br>3. Typed comparison per provider profile; treat any other value as unverified | High |
| Link state | Pre-hijack seeding: Attacker creates the account, or parks a pending link, on an address the victim will later prove ownership of; the planted method survives the victim's signup or reset | Email verification at signup | 1. Void pending links and unverified methods whenever the address is verified by a different session<br>2. Surface the method list in security settings and notify on every addition<br>3. Expire pending link state in minutes | Medium |
| Link request | Forced link: Attacker finishes the provider round trip with their own account, then gets the victim to open the callback URL; the victim's signed-in session completes the link and the attacker's identity is added to the victim's account | State parameter checked | 1. Accept a callback only from the session that started the link request; reject state issued to any other session<br>2. Consume state and nonce on first use<br>3. Show a confirmation naming the provider and asserted address before the write | Medium |

## Phase 2: Sign-in resolution

Focus: Keeping the stored binding the only path from an assertion to an account

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| Issuer binding | Issuer confusion: The verifier fetches signing keys from whatever issuer the token names, so an attacker who runs their own OIDC issuer can mint a token asserting the victim's email, sign it with their own key, and pass validation | Signature verification | 1. Pin issuers in provider configuration and resolve keys only for pinned values<br>2. Exact-match `iss` against the configured value; validate templated tenant ids against the allowlist<br>3. Periodically audit stored `issuer` values against the configured set | High |
| Existing accounts | JIT takeover: JIT provisioning finds an existing account whose email matches the SSO assertion and attaches the SSO identity to it; an admin of any tenant on the platform who can configure an IdP can assert a victim's email and sign in to the victim's account | JIT provisioning creates missing accounts | 1. Let JIT create accounts only; on an email match with an existing account, fail and route to an explicit link flow<br>2. Scope each SSO connection to the domains the tenant has verified<br>3. Offer a per-connection setting that turns JIT off | High |
| Login lookup | Recycled identifier: The provider reassigns an address or username to a different person, and a lookup that falls back to email resolves the new holder to the old owner's account | Subject lookup on the primary path | 1. Remove the email fallback; a subject with no binding is a new identity, whatever the address says<br>2. For providers without stable subjects, use the documented numeric or opaque id rather than the handle<br>3. Flag sign-ins to dormant accounts for step-up verification | Medium |

## Phase 3: Lifecycle and policy

Focus: Keeping the method list consistent with tenant policy and recoverable ownership

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| Tenant policy | Policy bypass: A user under enforced tenant SSO adds a personal login or password, keeping a path open that the tenant cannot see or revoke | SSO required at the login prompt | 1. Block self-service method changes for `directory` accounts<br>2. Evaluate domain policy at link time as well as at the prompt<br>3. On enforcement, disable existing non-SSO methods and revoke their sessions and API tokens rather than hiding the buttons | Medium |
| Method removal | Removal abuse: A hijacked session strips the owner's methods, or a legitimate removal leaves the account with no strong method that recovery accepts | Authenticated session required | 1. Fresh re-authentication for removal; refuse to remove the last method recovery can verify<br>2. Notify every channel on removal and keep a short undo window<br>3. Record removals with the same fields as links | Low |
| Audit trail | Attribution gap: The event says a provider was linked but not which issuer, subject, actor, or tenant, so a malicious link is indistinguishable from a legitimate one later | Event logging exists | 1. Record issuer, subject reference, acting session, and tenant on every change<br>2. Retain across disclosure windows (months)<br>3. Alert on links to dormant or high-privilege accounts | Low |

## If you auto-link on a verified email match

- Your effective verification standard becomes the weakest flow among the providers you enable
- Restrict the shortcut to providers that host the asserted domain; assertions about domains the provider does not control drop back to explicit linking
- A silent link makes the notification the only detection control; send it to channels that existed before the link
- Exclude `directory` and SSO-governed accounts from auto-linking entirely
