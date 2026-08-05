# Verification Checklist

## Token contents

- [ ] Every internal token carries `sub`, `tenant_id`, `act`, `aud`, `scope`, `exp`, and `txn`
- [ ] The external access token appears nowhere inside an internal token or a job payload
- [ ] An exchange never issues a token that expires after the one presented to mint it

## Exchange

- [ ] An exchange request without a valid workload credential is refused
- [ ] A workload not named in the presented token's `aud` cannot exchange it
- [ ] Requesting a scope outside the exchange policy's mapping for the presented token returns a narrowed grant or a refusal, and the response states the issued scope
- [ ] A token minted for service B is rejected by service C

## Per-hop enforcement

- [ ] A call with a valid workload identity and no internal token cannot perform user-scoped operations
- [ ] A token presented by a workload other than its current `act` is rejected
- [ ] Plain identity headers on inbound requests are overwritten at the edge and ignored by services
- [ ] A service that hands token checks to a sidecar cannot be reached around it; a request straight to the app port fails
- [ ] A token for tenant A's user addressing tenant B's resource gets a 404
- [ ] With the token service unavailable, user-scoped calls fail rather than proceed on workload identity alone

## Deferred work

- [ ] Revoking a user's access between enqueue and execution makes the job fail authorization when it runs
- [ ] Job records store user and tenant identifiers; queue storage contains no bearer tokens
- [ ] A worker cannot mint a token by supplying user and tenant values that no job record contains

## Audit

- [ ] Every action's log line carries the token's subject and the current actor
- [ ] One `txn` id query returns the edge request and every internal action behind it
- [ ] The log distinguishes a service acting for a user from the same service acting for itself
