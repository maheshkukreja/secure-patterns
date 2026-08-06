# Threat Model

## Baseline assumptions

- Internal callers are semi-trusted: any workload that can reach a service can send it requests with any headers and any tokens it holds. Reaching a service over the internal network is not authentication
- Services can verify each other's workload identity through mTLS or platform credentials; forging workload identity itself is out of scope here
- The edge derives user and tenant from a verified external credential, not from request fields
- Services hold the token service's public keys and can verify internal tokens locally
- Standard infra controls such as TLS, secret management, and network segmentation are assumed to be in place. This model focuses on carrying user authorization across internal hops

#### A note on risk

This table is not a checklist. Focus on preventing the highest-impact failures first. Detection and response are acceptable where prevention is impractical.

## Phase 1: Minting and exchange

Focus: Controlling who can turn one credential into another

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| Token service | Anonymous exchange: The exchange endpoint validates the presented token but not the caller, so any holder of a stolen token can trade it for fresh tokens aimed at other services | Subject token validation | 1. Client authentication: every exchange requires the calling workload's own credential<br>2. Audience gate: accept a presented token only from a caller named in its `aud`<br>3. Alerting: flag exchange requests from workloads outside the token's expected call path | Medium |
| Exchange policy | Scope widening: The token service issues whatever scope and audience the caller requests, so a hop can ask for more authority than the token it presented | Exchange requires client authentication | 1. Scope mapping: exchange policy caps what each incoming scope may become at the target service, and requests outside the map are refused<br>2. Per-caller policy: allowlist which workloads may mint for which audiences<br>3. Explicit response: return the issued scope so callers cannot assume they got what they asked for | Medium |
| External token | Interior exposure: The external access token rides the whole chain and lands in logs and traces deep in the stack | TLS on internal calls | 1. Terminate at the edge: the exchange consumes the external token; internal calls carry only the internal one<br>2. Never embed: the internal token must not contain the external one<br>3. Log scrubbing: redact bearer credentials from internal logs and traces | Medium |
| Availability | Fail-open fallback: The token service is down, and callers fall back to plain service-to-service calls to keep traffic moving, so user-scoped requests run without user context | None | 1. Fail closed: user-scoped operations return errors while no internal token can be minted<br>2. Outage drills: take the token service down in staging and verify the chain degrades to errors<br>3. Detection: alert on tokenless calls reaching user-scoped endpoints | High |

## Phase 2: The call chain

Focus: Keeping both identities checked at every hop

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| User identity | Header trust: A service reads the acting user from a plain request header gated only by network position, so any caller that can reach the service sets the header and impersonates any user | None | 1. Signed context only: user identity travels in the signed internal token, and plain identity headers are ignored<br>2. Strip at boundaries: the edge and every proxy overwrite reserved identity headers on inbound requests<br>3. Authenticated asserter: where a proxy or sidecar must assert identity in a header, accept it only over that proxy's authenticated connection, and keep the app port unreachable any other way | Medium |
| Downstream data | Confused deputy: The receiving service authorizes the calling service and never re-evaluates the user, so a crafted request from any authenticated user rides the trusted service's standing privilege to data that user was never allowed to see | Workload authentication | 1. Require user context: user-scoped operations demand a valid internal token, with the resource checked against its user and tenant<br>2. Deprivileged service accounts: strip standing data access from service identities so authority arrives with the token<br>3. Per-request re-check: services that proxy or fetch on demand authorize each induced request instead of reusing an approved channel | High |
| Internal credential | Cross-service replay: One internal token is accepted by many services, so a compromised hop holds a credential that works at every other | Short TTL | 1. Audience per hop: each token names one recipient, and every service rejects tokens not addressed to it<br>2. Exchange per hop: calls to the next service use a new token minted for it<br>3. Possession binding: bind the token to the caller's mTLS identity so a copied token fails without the caller's key | Medium |

## Phase 3: Deferred work and audit

Focus: Acting for the user after the request ends, and proving who acted

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| Deferred job | Stale authority: The job stores a bearer token at enqueue time and presents it at execution, so a revocation in the gap is invisible to a verifier that checks signature and expiry | Token expiry | 1. Re-mint at execution: store user and tenant references on the job and exchange for a fresh token when it runs<br>2. Issue-time cutoff: reject tokens minted before the user's most recent revocation event | Medium |
| Audit trail | Actor-only logs: Each hop logs the service credential that called it, so an investigation sees the deputy on every line and cannot reconstruct which user drove the action | Per-service request logs | 1. Both identities: every action records the token's subject and the current actor<br>2. Chain correlation: one `txn` id from the edge joins every downstream log line<br>3. Edge anchor: the edge record maps `txn` to the external request and user session | Low |

## If you forward the external token instead

If every internal service accepts the token the client presented at the edge, the trade-offs shift:

- The token's `aud` names the public API. Internal services that enforce the audience check reject it, so in practice the check gets dropped
- A compromise of any one service yields a credential valid at all the others and at the public API itself
- Every service parses the external identity provider's token format and trusts its keys; swapping providers later means changing every internal service
- Revocation can be centralized: one token, revoked at the issuer, and hops that introspect it see the revocation. The cost is an introspection dependency on the external issuer at every hop
