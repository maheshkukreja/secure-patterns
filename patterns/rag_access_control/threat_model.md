# Threat Model

### Phase 1: Ingest and indexing

Focus: Preserving the source system's access boundary in retrieval metadata

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Retrieval boundary | ACL drop: Ingest pipeline chunks and embeds a document but omits tenant or ACL metadata, leaving chunks globally searchable inside the index | Source ACL extraction | 1. Required fields: Reject indexing if tenant_id or ACL metadata is missing 2. Schema validation: Validate chunk metadata in CI and at ingest time 3. Dead-letter queue: Hold malformed documents out of the searchable index | High |
| Authorization freshness | Stale grants: User loses access in the source system, but old ACL metadata in the index keeps returning those chunks | Periodic reindex cycle | 1. Event-driven sync: Subscribe to ACL-change events where the source supports them 2. Differential polling: High-frequency comparison with a defined freshness SLA where events are unavailable 3. Fail closed: Mark affected documents pending_reindex and exclude from retrieval | High |
| Tenant isolation | Mixed-tenant chunks: Chunker or ETL job merges data from different tenants into one chunk or document record | Tenant ID in chunk metadata | 1. Partition first: Chunk only within a single tenant or workspace boundary 2. Deterministic IDs: Carry tenant_id through every transform stage 3. Negative tests: Assert no chunk contains multiple tenant identifiers in source provenance | High |
| Source collection scope | Over-collection: Connector service identity can read or ingest document spaces that were never approved for retrieval, because the service account has overly broad source visibility or the sync scope is misconfigured | Source ACL extraction, connector auth | 1. Scope connector identities: Limit connector service accounts to approved corpora only 2. Allow-list indexed sources: Maintain an explicit allow-list of indexed sources, spaces, or folders 3. Audit before first ingest: Review newly discovered sources before they enter the index | Medium |
| Deleted content | Zombie chunks: Source document is deleted or revoked, but old chunks remain queryable until the next batch rebuild | Document lifecycle state | 1. Tombstones: Push delete/revoke events into the index immediately, marking chunks as non-retrievable 2. State filter: Exclude deleted, revoked, and pending_reindex chunks at query time 3. Short rebuild windows: Keep batch lag within your documented freshness SLA | Medium |
| Field-level secrets | Chunk overexposure: A chunk contains both allowed business text and restricted fields because chunking happened before redaction | Classification metadata | 1. Redact before embedding: Remove secret and restricted fields upstream of the chunker 2. Sensitivity-aware chunking: Split restricted fields into separate chunks with appropriate classification 3. Classification filters: Exclude restricted classes for broad roles | Medium |

### Phase 2: Query-time retrieval

Focus: Preventing unauthorized chunks from entering the prompt

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Tenant data | Filter bypass: Service runs semantic search across the whole corpus, then applies ACL checks after top-k retrieval, letting unauthorized chunks influence ranking or the prompt | Verified user identity, tenant partitioning | 1. Pre-filter: Apply tenant and ACL scope inside the search request itself, before ranking 2. Partition query: Query only the caller's namespace, shard, or collection 3. Guardrails: Reject retrieval paths that cannot enforce hard pre-filters | High |
| Scope integrity | Client-supplied scope: Caller sends tenant_id, group_ids, or namespace names in the request body to widen retrieval beyond their grants | Verified caller identity | 1. Server-side derivation: Build scope only from the verified token and policy store 2. Discard client principals: Ignore or reject client-supplied ACL fields 3. Audit: Log requested scope separately from effective scope for drift detection | High |
| Retrieval authorization | Delegated identity collapse: Backend service calls the RAG gateway using its own service token, and retrieval is authorized as the service instead of the end user, bypassing user-level ACLs | Verified caller identity | 1. Require delegated token: User-scoped retrieval requires an end-user token or explicit token exchange 2. Bind to end-user subject: Retrieval decisions reference the delegated user, not the calling service account 3. Reject generic callers: Block user-scoped queries from service identities without delegation context | High |
| Availability | Fail-open fallback: Policy resolver, IdP, or filter path is slow or unavailable, and the service retries with an unfiltered search to keep answers flowing | Policy evaluation required | 1. Fail closed: Return an error when effective scope cannot be resolved 2. Bounded cache: Cache short-lived policy snapshots per principal, only if bounded and auditable 3. Chaos tests: Prove that policy-store outage does not widen retrieval scope | High |
| Permission cache | Cache staleness: Cached group memberships or policy snapshots outlive a revocation, granting access after it was removed | Policy cache in use | 1. Short TTLs: Cache policy snapshots for minutes, not hours 2. Invalidation events: Subscribe to revocation signals to bust cache entries early 3. Freshness header: Require cache entries to carry a max-age and fail closed when expired | Medium |
| Query privacy | Side-channel probing: Caller probes the corpus with repeated queries and infers document existence from score changes, hit counts, or denial wording | Opaque document IDs | 1. Uniform responses: Do not reveal whether inaccessible documents exist 2. Suppress telemetry: No global hit counts outside the authorized scope 3. Rate limits: Detect repeated probing across document names or project terms | Medium |

### Phase 3: Prompt assembly and response

Focus: Keeping unauthorized context out of the answer path

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Session context | Context bleed: Previously retrieved content leaks across users, tenants, or workspace switches because session context is reused without clearing | Authenticated sessions | 1. Scope binding: Bind conversation state to principal and tenant 2. Context reset: Clear retrieved chunks on principal or workspace change 3. Isolation tests: Verify that switching context produces zero carryover from prior scope | High |
| Response integrity | Citation mismatch: Model answers with claims not grounded in approved chunks, or cites documents outside the filtered retrieval set | Citations enabled | 1. Citation binding: Return only citations from retrieved chunk IDs 2. Post-check: Reject or downgrade answers referencing documents outside the approved set 3. Safe refusal: Prefer "I don't have enough information" over speculative synthesis | Medium |
| Audit trail | Silent leakage: Service can't reconstruct which chunks reached the model, so an access incident can't be investigated | Request logs | 1. Retrieval manifest: Record request_id, principal, effective scope, and retrieved chunk IDs per request 2. Prompt manifest: Persist the final approved context list 3. Alerts: Trigger on cross-tenant retrieval attempts or sudden spikes in denied queries | Low |

### If you rely on post-retrieval trimming instead

If retrieval searches the whole corpus and your application removes unauthorized hits after ranking, the threat profile shifts:

- Unauthorized chunks influence ranking and similarity scores before they're dropped
- Blocked results occupy candidate slots, making top-k recall unpredictable
- Caching layers may store broad search results that later callers shouldn't see
- Debugging pressure creates dangerous shortcuts: teams start requesting raw unfiltered hits

Post-retrieval trimming isn't a safe default. Use it only as an additional check after hard retrieval boundaries, not instead of them.
