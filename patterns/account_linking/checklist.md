# Verification Checklist

## Binding keys

- [ ] The same email asserted by a different provider subject does not resolve to or link with the existing account
- [ ] Sign-in resolution queries `(issuer, subject)` only; deleting `email_hint` from every record breaks nothing
- [ ] A second link attempt for an already-bound `(issuer, subject)` pair fails against the unique constraint, with a generic error, and the existing account keeps its binding
- [ ] Changing the email at the provider changes `email_hint` at most; account resolution is unaffected

## Link flow

- [ ] A link attempt on a session older than the re-authentication window is rejected
- [ ] An assertion with `email_verified` absent, false, or the string `"false"` is treated as unverified
- [ ] The callback ignores any account or email identifier in the request; the target account comes from the session
- [ ] State and nonce are single-use; a replayed callback fails
- [ ] A callback opened in a session other than the one that started the link request is rejected

## Issuer discipline

- [ ] A validly signed token from an issuer outside the configured set is rejected before any account lookup
- [ ] For multi-tenant issuers, a token with an unlisted tenant id is rejected even though the signature and issuer template match
- [ ] An audit query listing distinct stored `issuer` values returns only configured providers

## Tenant policy

- [ ] A `directory` account cannot add or remove a login method through self-service endpoints
- [ ] Enforcing SSO on a tenant disables existing password and social methods, and the disabled methods no longer authenticate
- [ ] API tokens that survive SSO enforcement are enumerable by the tenant admin

## Lifecycle and audit

- [ ] Removing the last recovery-capable method is refused
- [ ] Every link and unlink event carries issuer, subject reference, acting session, tenant, and timestamp
- [ ] A link on a dormant account raises an alert
