# Designing a Safe Approval Flow for MCP Tool Descriptions

An MCP server provides the tool description and input schema. The LLM uses both as part of its reasoning context when deciding how and when to call the tool. A server that can change a description after approval can change agent behavior without any application code change.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/mcp-tool-poisoning-a-safe-approval-flow-for-tool-descriptions)

## System Description

The MCP client connects to a server and presents each returned tool's full description and input schema to the user for per-tool approval. The client hashes the approved description and schema against a stable server identity. On every subsequent connection it recomputes the hashes; any change blocks the tool until the user re-approves.

```mermaid
flowchart TD
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    User([User]):::trusted

    subgraph Server ["MCP Server (Untrusted)"]
        Tools[Tool List + Descriptions + Schemas]:::untrusted
    end

    subgraph Client ["MCP Client / Host"]
        Connect[Connection Manager]:::trusted
        Gate[Approval Gate]:::trusted
        Hasher[Description Hasher]:::trusted
        Dispatch[Tool Dispatcher]:::trusted
    end

    Approvals[(Approval Store: server_id, tool, hash)]:::storage
    Audit[(Tool Call Log)]:::storage

    Server -- "1. tools/list response" --> Connect
    Connect -- "2. Forward full description + schema" --> Gate
    Gate -- "3. Show user; await per-tool approval" --> User
    User -- "4. Approve" --> Gate
    Gate -- "5. Hash description + schema" --> Hasher
    Hasher -- "6. Persist hash by server_id" --> Approvals

    Connect -- "7. On reconnect: re-hash + compare" --> Approvals
    Approvals -- "8. Match: enable; mismatch: re-prompt" --> Gate

    Gate -- "9. Approved tool" --> Dispatch
    Dispatch -- "10. Log call with server_id" --> Audit
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks at first-time approval and across the connection lifecycle, plus how the profile shifts under gateway-mediated approval
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
