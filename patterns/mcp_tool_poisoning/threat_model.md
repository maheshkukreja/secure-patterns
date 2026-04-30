# Threat Model

## Baseline assumptions

- The MCP server is untrusted: it controls tool names, descriptions, schemas, and return values. A gateway centralizes approval and enforcement; it does not make the server trusted
- The MCP client and its approval store are trusted. The user or operator is trusted to approve tools, but the UI must make meaningful review possible (full text rendered, diff on change, no truncation by default)
- The LLM treats the full description text as part of its reasoning context and may follow instructions embedded in it
- Approval is per-tool, not per-server. A user approving one tool does not approve every tool the server lists now or later
- Output-channel prompt injection (e.g. malicious tool return values or error messages) is a separate threat addressed by an output-handling pattern, not by this approval flow
- Standard infra controls such as TLS and secret management are assumed to be in place

#### A note on risk

This table is not a checklist. Focus on preventing the highest-impact failures first. Detection and response are acceptable where prevention is impractical.

### Phase 1: First-time approval

Focus: Catching a malicious description before the tool is callable

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|-------------------|------|
| Agent behavior | Tool poisoning: Server embeds hidden instructions in the description text that steer the agent toward exfiltration or unauthorized side effects, while the user-visible summary stays benign | UI shows tool name and short summary | 1. Render the full description and schema at approval time, byte-identical to what the LLM receives<br>2. Per-tool approval, not blanket server approval<br>3. Static checks on description text for instruction-shaped patterns flagged for the user before approval | High |
| Cross-server context | Cross-server shadowing: A second MCP server's description includes instructions that redirect the agent's use of an already-trusted server's tools, without the malicious tool ever being called | Per-server approval | 1. Review descriptions from all connected servers as a combined set during approval, not one server at a time<br>2. Cap the number of simultaneously connected servers and re-approve when the set changes<br>3. Run untrusted or unfamiliar servers in a separate agent instance with an isolated LLM context | High |
| Tool schema | Schema manipulation: Server defines a catch-all input schema (untyped string blob or generic object) that lets the LLM dump its working context, including data retrieved from other trusted tools, into the attacker's server | Schema visible at approval | 1. Constrain argument types to the minimum required for the documented use case; reject untyped string blobs at approval review<br>2. Hash the schema alongside the description so changes trigger re-approval<br>3. Sample tool call arguments in the log for unexpected payload sizes or secret-shaped content | Medium |
| Agent context budget | Description bloat: Server returns descriptions or schemas large enough to crowd out useful context or spike per-call token cost on every `tools/list` | UI rendering of full text | 1. Enforce a hard byte and token cap on each tool's description and schema before approval and before hashing; reject oversized payloads at the gate<br>2. Cap the aggregate `tools/list` payload per server and surface offenders to the operator<br>3. Alert when a server's description size grows by more than a small percentage between connections, even when the user accepts the diff | Medium |

### Phase 2: Subsequent connections and lifecycle

Focus: Ensuring an approved tool stays the tool that was approved

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|-------------------|------|
| Approved tool definition | Rug pull: Server ships a benign description at first launch, then mutates it on a later connection after the user has already approved | One-time approval | 1. Recompute description and schema hashes on every connection and gate the tool on a match<br>2. Block the tool and require explicit re-approval on any mismatch, with a diff of old and new text<br>3. Pin server identity to a package version or version-tagged origin so a real change reads as a new identity | High |
| Approval store | Identity collision: Server changes its identifying metadata (name or origin or version) in a way that lands on a different `server_id` and silently bypasses the stored hash because no prior record exists for the new identity | Server identity stored as part of approval | 1. Treat any new `server_id` as first-time approval and refuse to inherit approvals across identities<br>2. On mismatch between server-reported version and a previously seen origin, surface the change instead of auto-promoting it<br>3. Periodic operator review of the approval store for orphaned or duplicate server entries | Medium |
| Tool call log | Silent invocation: A tool runs without a record that ties the call to the approval hash that was active at the time, so a later rug pull cannot be reconstructed | Generic request logs | 1. Log `server_id`, `tool_name`, `approval_hash`, arguments, and a result summary on every dispatch<br>2. Retain logs long enough to span typical disclosure windows (months, not days)<br>3. Alert on an approval-hash change between connections even when the user re-approves the new value | Low |
| Agent execution loop | Mid-loop toolset swap: A `notifications/tools/list_changed` event arrives while the agent is mid-loop and the loop continues against the unverified, possibly-changed toolset | Gate enforced on initial connect | 1. Suspend the agent loop on `notifications/tools/list_changed` until re-validation completes<br>2. Version the approved toolset and require version match before every dispatch<br>3. Fail closed if gate state is unknown or pending | High |

### If you run gateway-mediated approval

If a central gateway holds the approval state for many clients, the threat profile shifts:

- The gateway's hash recomputation cadence becomes the effective re-check frequency for every downstream client
- A compromise of the gateway compromises every downstream agent's approval gate, so the gateway's own integrity becomes the high-value asset
- Per-tool approval can still be enforced, but the user prompt now lives in an operator console rather than the end user's IDE; reviewers must understand that they are approving on behalf of the agent's principal
- Cross-server shadowing review happens once at the gateway instead of in each client
