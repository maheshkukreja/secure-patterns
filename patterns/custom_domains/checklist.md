# Verification Checklist

## Claim intake

- [ ] A name claimed with mixed case or Unicode labels resolves to the same binding row the serve path looks up
- [ ] Two tenants cannot hold bindings for the same canonical hostname
- [ ] A pending claim by one tenant does not stop another tenant from claiming and proving the same hostname
- [ ] A claim for `co.uk` or another public suffix is rejected at intake
- [ ] Publishing the correct token after the claim expired does not create the binding

## Proof of control

- [ ] A challenge token issued to one tenant does not verify a claim by another tenant, even for the same hostname
- [ ] The verifier resolves the exact challenge name; a token published at the parent zone does not pass a child claim
- [ ] A wildcard claim fails while another tenant holds a verified hostname one label below the wildcard's parent
- [ ] Concurrent proofs for `*.customer.com` and `app.customer.com` from different tenants create at most one binding

## Serving

- [ ] A TLS handshake for an unclaimed hostname fails with no certificate presented
- [ ] Reusing a connection opened for one hostname to request an unbound hostname returns 421
- [ ] Requests for two bound hostnames on one reused connection each reach their own tenant
- [ ] The application receives `tenant_id` from the routing layer; removing every Host-header read below the router breaks nothing
- [ ] Password-reset and invite links carry the binding record's hostname even when the request arrived with a different Host

## Certificates

- [ ] Pointing DNS at the platform without completing verification produces no certificate order
- [ ] Suspending or releasing a binding stops its renewal orders
- [ ] After a release, the certificate presented for surviving hostnames on a shared multi-name certificate no longer lists the released name

## Lifecycle

- [ ] In the release sequence, the edge stops answering for the hostname before the name becomes claimable
- [ ] Re-claiming a hostname another tenant released fails until a new challenge token is published and verified
- [ ] The old tenant's standing token cannot move an active hostname to a different tenant; a transfer verifies only against the claimant's fresh challenge
- [ ] After a transfer verifies, the new tenant receives no traffic until the old edge mapping is gone
- [ ] Deleting the challenge record for an active binding produces a warning to the tenant before routing stops
- [ ] When every configured resolver times out, active bindings stay active and an alert fires for the verifier itself
- [ ] Every state transition records the actor and the old and new states
