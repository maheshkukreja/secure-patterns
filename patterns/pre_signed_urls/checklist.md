# Verification checklist

### Identity and mapping
- [ ] Client cannot provide bucket / key (only upload_id / object_id)
- [ ] Tenant / user derived from auth context and enforced on finalize + download-url issuance
- [ ] Cross-tenant access attempts return 404 (Not Found), not 403 (Forbidden)

### Pre-sign correctness
- [ ] PUT TTL default 120s, GET TTL 60–300s, and max TTL enforced
- [ ] Exact-key pre-sign only (no prefix / wildcards)
- [ ] Pre-signed URLs are not logged anywhere; signed query params are redacted

### Quarantine → finalize → publish
- [ ] Objects are unusable until finalize succeeds
- [ ] Finalize verifies existence and size bounds; checksum / hash stored when applicable
- [ ] Magic bytes / MIME-type match verified (if inspecting)
- [ ] Abandoned sessions expire; quarantine cleanup runs

### Serving safety
- [ ] Downloads default to attachment + nosniff
- [ ] Inline rendering only happens through a safe transform pipeline

### Detection
- [ ] Object-level access logs enabled and centralized
- [ ] Alerts exist for unusual download patterns and destructive actions (delete spikes)
