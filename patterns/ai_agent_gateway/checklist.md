# Verification checklist

### Authentication
- [ ] Unauthenticated requests to the gateway return 401 before reaching any agent
- [ ] Slack/chat integration requests are rejected if platform signature validation fails
- [ ] mTLS clients with expired or unknown certificates are rejected at the TLS handshake

### Authorization
- [ ] A user with no explicit policy binding receives 403 when addressing any agent
- [ ] Removing a user's role binding immediately prevents access on the next request (no stale cache)
- [ ] Agent logical names are resolved by the gateway; direct agent endpoint addresses in client requests are rejected

### Data inspection (inbound)
- [ ] Oversized payloads return 413 before forwarding
- [ ] Known prompt-injection patterns in request payloads trigger a block or flag (verified with test payloads)

### Data inspection (outbound)
- [ ] Agent responses containing test secret patterns (e.g., `AKIA...` AWS key format) are redacted before delivery to the client
- [ ] DLP rule triggers are logged with the `request_id`, `agent_id`, and matched rule

### Agent isolation
- [ ] Agent A cannot send a request to Agent B without that request passing through the gateway's authZ and DLP pipeline
- [ ] Only agents registered in the gateway's agent inventory receive traffic; unregistered agents are rejected
- [ ] Agents cannot reach the gateway's admin API or policy store

### Credential management
- [ ] Agent credentials (API keys, service account tokens) never appear in gateway logs or client-visible error messages
- [ ] Credential rotation requires zero gateway downtime

### Audit and detection
- [ ] Every authZ decision (allow and deny) is logged with `user_id`, `agent_id`, `action`, and policy rule
- [ ] DLP scan results (pass, redact, block) are logged for every response
- [ ] Alerts fire when a single user triggers more than N DLP blocks within a time window
