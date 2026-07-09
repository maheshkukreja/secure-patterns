# Verification Checklist

## Indexing and metadata

- [ ] Indexing a document without `tenant_id` or a required access-scope field is rejected; no searchable document is written
- [ ] Every write (index, update, delete) carries the source version, and a replayed older event does not overwrite a newer write
- [ ] A reindex that drops or renames the `tenant_id` field fails validation before it runs

## Query surfaces

- [ ] Every read surface (search, autocomplete, export, aggregations, multi-search, pagination cursors) applies the tenant filter; the set that bypasses the shared query builder is known and empty
- [ ] Autocomplete for tenant A never returns tenant B's terms
- [ ] Facet and aggregation counts for tenant A exclude tenant B's documents
- [ ] The tenant predicate is a hard filter on every query, never an optional or score-only clause, and it constrains facet counts as well as hits
- [ ] Cross-tenant `total`, pagination, and `_id` fetches return an identical empty or 404 shape regardless of what exists in other tenants
- [ ] A user-supplied search string cannot reach a raw query-language parser; leading wildcards and regex are rejected or rewritten

## Boundary placement

- [ ] The tenant filter is enforced in the search engine, not by application code alone
- [ ] No unrestricted search key reaches the browser; a hosted client key is a short-TTL secured key whose tenant filter is derived from the session

## Propagation and freshness

- [ ] Revoking access in the database removes the document from the tenant's results within the documented freshness SLA
- [ ] A cross-tenant canary query against a freshly reindexed index returns zero before traffic is swapped to it
- [ ] Per-tenant document counts match between source and index after a reindex
- [ ] A record that fails to index lands in a dead-letter queue without stalling the permission-change and delete events behind it
