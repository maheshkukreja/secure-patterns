# Verification checklist

### Approval gate
- [ ] The approval prompt renders the full description and schema without truncation or scroll-to-reveal
- [ ] Approving one tool from a multi-tool server leaves every other tool from that server uncallable
- [ ] A tool that arrives mid-session via `notifications/tools/list_changed` stays uncallable until it clears the gate

### Hashing and storage
- [ ] The hash is SHA-256 over an RFC 8785 (JCS) serialization of `{server_id, tool_name, description, input_schema}` with `description` preserved byte-for-byte, so any whitespace change produces a new hash
- [ ] Re-versioning a previously-approved server invalidates prior approvals; nothing auto-applies under the new `server_id`
- [ ] The approval store survives client restart and client upgrade

### Reconnection check
- [ ] After disconnect and reconnect, no tool is callable until per-tool hash verification completes
- [ ] On a hash mismatch, the affected tool stays disabled and the user sees a diff rather than a yes/no prompt
- [ ] Connecting a server with a renamed identity inherits no prior approval; the gate fires as first-time
- [ ] Simulated approval-store or hash failure makes tool execution fail closed with no tool callable

### Multi-server review
- [ ] When more than one server is connected, the approval UI shows descriptions from both in a single review step before enabling tools
- [ ] The number of simultaneously connected servers is bounded and reviewed; untrusted servers run in a separate agent instance

### Audit trail
- [ ] Every tool call logs `server_id`, `tool_name`, the `approval_hash` active at call time, arguments, and a result summary
- [ ] An approval-hash change between connections is logged and alerted even when the user accepts the re-approval prompt
- [ ] Logs are retained long enough to investigate a description rotation discovered weeks later

### Attacker tests
- [ ] Description swap: changing the description text on a server and reconnecting leaves the tool disabled until re-approval
- [ ] Schema swap: widening an input schema (adding a free-form `context` field) and reconnecting fires the same gate
- [ ] Identity swap: renaming or re-versioning the server does not auto-trust previously approved tools under the new `server_id`
- [ ] Mid-session tool addition: a server returning an extra tool on a refreshed `tools/list` is gated, not silently added
- [ ] Shadowing rehearsal: connecting a second server whose description references another server's tool by name surfaces the reference in the combined-review prompt
