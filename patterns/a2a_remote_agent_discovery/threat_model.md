# Threat Model

## Baseline assumptions

- Agent Cards are untrusted text from the remote operator
- Remote agents are semi-trusted: they may pass an initial review and change behavior or hosting later
- The origin's authority and the user's data are higher-value than the discovery metadata
- The LLM inside the origin will follow any text it sees, including text in a card or remote response
- Standard infra controls (TLS, service authentication, secret management) are in place. This model focuses on the discovery-to-delegation boundary

#### A note on risk

This table is not a checklist. Focus on preventing the highest-impact failures first. Detection and response are acceptable where prevention is impractical.

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|-------------------|------|
| Selection decision | Card poisoning: A card contains hidden instructions that steer the origin's LLM toward picking a specific remote agent or leaking the user's task | None | 1. Route selection through the registry; card text never reaches the planning prompt<br>2. Pass only registry-derived labels and `agent_id` values to the model<br>3. Render the full card to a human reviewer at registration | High |
| Remote identity | Impostor agent: A card mimics the name or endpoint of an approved agent | Stable `agent_id` per entry | 1. Bind registry entries to a signed public key, not a name or endpoint<br>2. Verify the remote signature on every invocation against the registered key<br>3. Trigger name-collision review before a similarly-named agent enters the registry | High |
| Discovery surface | Unapproved listing: An attacker publishes a card in a shared discovery channel and waits for an origin to delegate without registry review | None | 1. Block delegation to any `agent_id` absent from the registry<br>2. Require operator review between discovery and registry inclusion<br>3. Disable any auto-ingest path from discovery into the registry | High |
| Delegation payload | Overbroad payload: Origin forwards full user context or tool credentials alongside the task | Per-invocation payload | 1. Build the payload from the task and capability label, not the origin's conversation history<br>2. Reject extra fields at the broker; block known credential fields and configured restricted data classes<br>3. Bind the request to the calling tenant | High |
| Origin authority | Confused deputy: Remote agent reuses the origin's identity or a forwarded user token to reach resources the calling user is not entitled to | Bearer auth between agents | 1. Mint a scoped delegation that names the target agent and capability; do not forward the origin's user token<br>2. Make the delegation short-lived and single-use<br>3. Require remote-agent calls to use the delegation token; reject forwarded user tokens at downstream services | High |
| Delegation credential | Token replay: A delegation credential leaks and is replayed against the same or a different capability | Bearer token in transit | 1. Single-use, consumed at first invocation<br>2. Audience binding to one `agent_id` and one capability<br>3. Mutual TLS or signed request between broker and remote agent | Medium |
| Origin reasoning | Remote output injection: A remote agent returns text that steers the origin's next tool call or user-facing response | Labeled remote response | 1. Pass remote output through the origin's untrusted-input handler<br>2. Restrict the tools available to the origin in the turn after a remote response | High |
| Revocation | Revocation lag: A revoked agent keeps receiving calls because the broker cache has not refreshed | Periodic registry sync | 1. Push revocation events to the broker and refuse delegation when the registry connection is unhealthy<br>2. Cap broker cache TTLs in minutes, not hours<br>3. Maintain a deny-list that overrides cached entries | High |
| Audit chain | Audit gap: Delegations are logged inconsistently, so a behavior change is hard to investigate later | Per-hop request logs | 1. Log every delegation with the caller, registry entry id, allow/deny decision, and timestamp<br>2. Retain records long enough to span typical disclosure windows<br>3. Alert on deny rates that spike or drop unexpectedly | Low |
