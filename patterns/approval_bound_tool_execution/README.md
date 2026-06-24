# Approval-Bound Tool Execution for AI Agents

Bind approval to the exact tool call, not the agent's description of it.

[Read the full post on securepatterns.dev](https://newsletter.securepatterns.dev/p/approval-bound-tool-execution-for-ai-agents)

## System Description

The approval system binds every approved action to an `action_hash` computed over the full normalized envelope. The executor receives only an envelope ID and fetches every parameter from the trusted store, not from the model. No execution proceeds without a matching approval in the store, and each envelope is accepted for execution exactly once.

```mermaid
flowchart TD
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    User([User]):::untrusted
    Model([LLM Agent]):::untrusted
    Tool([Target Tool]):::untrusted

    Normalizer[Action Normalizer]:::trusted
    Policy[Policy Engine]:::trusted
    ApprovalUI[Approval UI]:::trusted
    Executor[Executor]:::trusted
    Authority[Execution Authority]:::trusted

    Envelopes[(Envelope Store)]:::storage
    Audit[(Evidence Log)]:::storage

    User -- "1. Task" --> Model
    Model -- "2. Proposed tool call" --> Normalizer
    Normalizer -- "3. Canonical envelope + hash" --> Policy
    Policy -- "4. Allow / deny / require approval" --> ApprovalUI
    ApprovalUI -- "5. Approver approves action_hash" --> Envelopes
    Envelopes -- "6. envelope_id to executor" --> Executor
    Authority -- "7. Scoped authority" --> Executor
    Executor -- "8. Execute from stored envelope" --> Tool
    Executor -- "9. Outcome + evidence" --> Audit
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across proposal, normalization, approval decision, and execution phases
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
