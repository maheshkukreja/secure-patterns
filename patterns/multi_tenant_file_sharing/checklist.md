# Verification checklist

### Tenant isolation
- [ ] Cross-tenant access using a valid object_id returns 404 Not Found
- [ ] List endpoints only return objects where object.tenant_id == caller.tenant_id

### Authorization
- [ ] Every download path (download, preview, thumbnail) calls the centralized auth function
- [ ] Read-only users cannot generate upload grants or delete objects

### Lifecycle
- [ ] Quarantined / deleted objects never receive delivery grants
- [ ] Soft-deleting an object immediately blocks new downloads (even if bytes exist)

### Share links
- [ ] Tokens are hashed in the database (SHA-256); plain text is never stored
- [ ] Revoked links stop working immediately (return 404 or 410)
- [ ] Share links resolve to a single object only (no directory walking)
- [ ] Link tokens are random strings (not JWTs)
