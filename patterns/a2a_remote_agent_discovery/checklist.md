# Verification checklist

### Registry is the only callable list
- [ ] A fetched Agent Card cannot be called unless an approved registry entry exists
- [ ] A registry entry marked `revoked` blocks the next delegation
- [ ] The endpoint used by the broker comes from the registry, not the card

### Identity is bound to a key
- [ ] Changing the remote signing key blocks delegation until re-approval
- [ ] A new agent whose name matches an existing entry triggers a review prompt

### Payload is bounded
- [ ] A payload with extra fields is rejected at the broker
- [ ] The origin's user token is not present in the remote request
- [ ] The origin's tool credentials are not present in the remote request

### Response is treated as remote input
- [ ] Remote output is labeled with the source `agent_id`
- [ ] Remote output enters the origin through the untrusted-input handler, not as system or developer text

### Audit and fail-closed
- [ ] Every delegation attempt is logged with an allow/deny outcome
- [ ] If the registry is unavailable, delegation is denied
- [ ] If a broker cache contains an agent later marked `revoked`, the deny-list or revocation event overrides the cached entry
