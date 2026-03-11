# Verification checklist

### Tool execution
- [ ] Agent rejects tool calls not on the explicit allow-list
- [ ] A tool cannot access resources outside its assigned credential scope
- [ ] User-controlled input injected into tool arguments is rejected by schema validation before reaching the external system
- [ ] Tool results from external sources pass through a separate data channel (distinct message role or structured field), not concatenated into instructions

### Loop controls
- [ ] A task exceeding the iteration cap is terminated, not warned
- [ ] Write operations require approval or are capped per task
- [ ] A task exceeding the wall-clock timeout is killed, not logged
- [ ] Each iteration logs: decision, tool call name, arguments, result summary, memory operations

### Memory integrity
- [ ] Retrieved memory entries include provenance metadata that survives storage and retrieval
- [ ] Memory is partitioned per tenant; cross-partition queries return empty results
- [ ] Working memory is cleared between sessions unless items are explicitly persisted
- [ ] No raw credentials or PII in stored context (encrypt with scoped keys if retention is required)
- [ ] A revoked permission blocks tool execution on the next loop iteration, not at next session start

### Cross-primitive controls
- [ ] Tool results pass through channel separation before being written to memory
- [ ] Memory retrieval marks untrusted-source content when passing it to the LLM
- [ ] Poisoned memory entries still go through the tool gateway before triggering any tool call
- [ ] End-to-end audit trail links tool calls, memory writes, and loop iterations by task ID

### Negative tests (attacker perspective)
- [ ] Attempt to invoke an unregistered tool: request rejected by the orchestrator
- [ ] Inject instructions into a tool result: content remains in the untrusted data channel, not interpreted as directives
- [ ] Insert a poisoned entry into long-term memory: provenance tag preserved on retrieval, entry treated as untrusted
- [ ] Trigger a recursive task: terminates at the iteration cap, not after exhausting resources
- [ ] Query tenant A's memory from tenant B's session: zero records returned
- [ ] Simulate approval service failure: destructive operations fail closed (blocked, not permitted by default)
- [ ] Revoke a user's permission mid-task: next loop iteration re-validates and blocks further execution
