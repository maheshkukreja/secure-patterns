# Threat Model

## Baseline assumptions

- Clients are untrusted: they can hit any documented query surface and send arbitrary filter parameters
- Control plane authority: the query builder derives tenant scope from the session token, not from request parameters or the query body
- The primary database is the authority for permissions and lifecycle; the index is a derived copy
- The engine can apply a hard filter to a query, and can enforce that filter server-side through a filtered view or a scoped key
- Standard infra controls such as TLS, WAF, cluster network isolation, and database AuthN are assumed to be in place. This model focuses on tenant isolation through the search index

#### A note on risk

This table is not a checklist. Focus on preventing the highest-impact failures first. Detection and response are acceptable where prevention is impractical.

## Phase 1: Indexing

Focus: Carrying tenant and access metadata into the index and keeping writes ordered

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| Index metadata | Access metadata drop: Indexer writes a document with no `tenant_id` or required access-scope field, so it stays globally searchable inside a shared index | Source metadata extraction | 1. Reject any index write missing `tenant_id`; write nothing rather than an unscoped document<br>2. Validate document schema in CI and at ingest<br>3. Dead-letter malformed records instead of indexing them | High |
| Write order | Stale resurrect: In an event-driven indexing pipeline, events arrive out of order, so an old update lands after a newer delete and brings a removed document back into results | Internal last-write-wins versioning | 1. Stamp every write (index, update, delete) with the source's version number and reject an incoming write older than the stored version<br>2. Outbox pattern so the index event commits with the source transaction<br>3. Make writes idempotent so replays do not double-apply | Medium |

## Phase 2: Query time

Focus: Keeping every read surface tenant-scoped and bounded in cost, including the surfaces that don't look like search

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| Tenant isolation | Forgotten filter: A secondary surface such as autocomplete or export issues a query without the tenant clause and returns other tenants' data | Tenant filter on the main search path | 1. Put the filter in the search engine (a filtered view or a scoped key) so a caller cannot omit it<br>2. Where the engine can bind a filter to a credential or role, use it as a floor beneath the app filter<br>3. Audit every read surface against the shared query builder; treat bypasses as the review question | High |
| Tenant data | Count oracle: The UI shows facet counts (`Status: Open (42)`). The tenant filter scopes the returned documents but not the aggregation, so those counts are computed across all tenants and leak other tenants' category values and volumes | Documents correctly scoped in `hits` | 1. Apply the tenant filter to the facet counts as well as the returned hits<br>2. Forbid unscoped or global aggregations on a shared index<br>3. Restrict which fields are facetable | Medium |
| Tenant data | Suggester bypass: Autocomplete is served from a separate suggestion structure that the normal tenant filter does not reach, so even with search filtered, suggestions still span tenants | Main search path filtered | 1. Scope the suggestion structure to the tenant when it is built<br>2. Apply the tenant filter to term or value enumeration endpoints (the ones that populate filter dropdowns)<br>3. Serve autocomplete from a tenant-scoped query rather than the shared suggestion structure | Medium |
| Client-side search key | Key scrape: The browser queries a hosted search service directly, and an unrestricted search key lets any visitor read the whole shared index (Algolia, Typesense Cloud, and similar) | Search-only (no write) key | 1. Server-mint a scoped, short-lived key with the tenant filter embedded and unmodifiable by the client (for example, an Algolia secured API key)<br>2. Derive the tenant id from the session, never from the client<br>3. Rotate the base key to revoke every derived key at once | Medium |
| Cluster availability | Query-cost DoS: A tenant submits expensive query operators (leading wildcards, broad regex, open-ended fuzzy matching) that spike CPU on the shared index and starve every other tenant | None | 1. Build queries from safe, structured query types rather than exposing a raw query-language parser to user input<br>2. Disallow leading wildcards, unbounded regex, and open-ended fuzzy matching on user terms<br>3. Bound query cost with server-side timeouts and per-tenant concurrency limits | Low |

## Phase 3: Propagation and reindex

Focus: Keeping the index consistent with the source as permissions change and indices are rebuilt

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| ACL freshness | Stale ACL: To filter on per-user access, you copied the ACL into the document (`acl_principals`); access is later revoked in the database, but the indexed copy keeps the old ACL until that document is reindexed, so the revoked user still finds it | Reindex-on-write pipeline | 1. Push each permission change as an event that reindexes the affected document<br>2. Prefer filtering on stable `tenant_id` over churny per-object ACLs, and push fine-grained checks to a post-search database authorization step<br>3. Alert when reindex lag exceeds the freshness SLA | High |
| Index freshness | Delete lag: A deleted document stays searchable for the index refresh interval plus the pipeline delay | Near-real-time index plus async pipeline | 1. Filter queries on the freshest `status` in addition to issuing the delete<br>2. Bound and monitor pipeline lag; alert on backlog<br>3. Write a higher-version tombstone before any physical deletion, so a stale active copy loses to the delete state | Medium |
| Propagation liveness | Poison pill: One malformed record keeps crashing the indexer and blocks the permission-change and delete events queued behind it, extending stale access | Queue retry policy | 1. Move a record to a dead-letter queue after bounded retries, keeping its `doc_id`, `tenant_id`, and failure reason<br>2. Process events so one bad record cannot stall the tenant or global stream<br>3. Alert on dead-letter backlog and the age of the oldest unprocessed event | Low |
| Reindex integrity | Tenant merge: A backfill drops or mis-derives `tenant_id`, so the new index silently blends tenants | None | 1. Assert the new index keeps the `tenant_id` field indexed for exact-term filtering; validate per-tenant document counts before and after<br>2. Keep writes pointed at a single index during the rebuild; make the cutover atomic and retain the old index for rollback<br>3. Run a cross-tenant canary query against the new index before swapping traffic | Medium |
