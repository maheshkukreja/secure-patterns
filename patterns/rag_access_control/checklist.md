# Verification checklist

### Identity and scope
- [ ] A request with client-supplied tenant_id, group_ids, namespace names, or raw ACL filters is ignored or rejected
- [ ] A backend service calling the RAG gateway without delegated end-user context can't perform user-scoped retrieval
- [ ] If the IdP or policy resolver is unavailable, the request fails closed and no unfiltered retrieval occurs
- [ ] A request with the same query but a narrower delegated identity returns a strictly narrower set of approved chunks, even when sent through the same backend service

### Ingest and metadata
- [ ] Ingesting a document without ACL metadata fails; no searchable chunks are written
- [ ] A chunk missing tenant_id, doc_id, chunk_id, ACL metadata, version, or state is rejected by ingest validation
- [ ] A mixed-tenant source payload is rejected and doesn't produce a merged chunk
- [ ] Restricted fields are redacted or split before chunking and embedding
- [ ] A connector whose service account gains read access to an unapproved source space doesn't index that content until it appears on the allow-list

### Revocation and deletion
- [ ] Removing a user from a source group blocks retrieval within the documented freshness SLA
- [ ] Deleting or revoking a document marks affected chunks non-retrievable before background cleanup completes
- [ ] If reindex is pending after an ACL change, affected content is excluded from retrieval until new metadata is active

### Retrieval enforcement
- [ ] A caller querying with Tenant A credentials for a document known to exist in Tenant B receives zero chunks, zero hit counts, and no signal distinguishing "does not exist" from "not authorized"
- [ ] Retrieval executes inside the authorized partition or hard filter before ranking returns results
- [ ] Post-filter-only retrieval paths are disabled or blocked in production

### Prompt assembly and response
- [ ] Switching tenant, workspace, or principal clears prior retrieval context from the session
- [ ] Every citation in the answer maps to a chunk returned by the filtered retrieval step
- [ ] Answers referencing documents outside the approved set are rejected or downgraded to a safe refusal

### Logging and detection
- [ ] Audit logs record request_id, principal, effective scope, retrieved chunk IDs, cited document IDs, and denial events
- [ ] Alerts fire on repeated scope-probing attempts, cross-tenant retrieval attempts, or sudden spikes in denied queries
- [ ] Post-hoc audit can reconstruct which chunks contributed to any historical response
