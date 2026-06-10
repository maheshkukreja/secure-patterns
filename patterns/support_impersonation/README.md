# Designing a Safe Support Impersonation Flow

Support impersonation lets an engineer act inside a customer's account to reproduce a bug or unstick a broken state. If the implementation simply swaps the engineer into the customer's session, actions land under the customer's name with no record of who really took them, and the support tool hands over the same authority as the customer with none of the constraints.

The safer pattern is a separate, short-lived act-as credential that names both the agent and the customer on every action, keeps high-risk operations blocked by default, makes the session visible to the customer, and ends on a timeout or on demand.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/designing-a-safe-support-impersonation-flow)

## System Description

An impersonation session is a separate, short-lived credential that lets a named support agent act as a specific customer, with both identities recorded on every action. The agent never receives the customer's real session, and high-risk actions stay blocked for the duration.

```mermaid
flowchart TD
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    subgraph Internal [Semi-Trusted Internal]
        Agent([Support Agent]):::untrusted
    end

    subgraph Tenant [Customer Tenant]
        CustomerAdmin([Customer Admin]):::untrusted
    end

    subgraph Control_Plane [Control Plane]
        Approval[Approval + Reason Check]:::trusted
        ImpSvc[Impersonation Service]:::trusted
        Gate[Action Policy Gate]:::trusted
        App[Product App]:::trusted
    end

    subgraph Data_Plane [Data Plane]
        Sessions[(Impersonation Sessions)]:::storage
        Audit[(Audit Log)]:::storage
    end

    Agent -- "1. Request: target + reason + ticket" --> Approval
    Approval -- "2. Authorize (approve / deny)" --> ImpSvc
    ImpSvc -- "3. Mint short-lived act-as session" --> Sessions
    Agent -- "4. Act in product as customer" --> App
    App -- "5. Check action against policy" --> Gate
    App -- "6. Log with both identities" --> Audit
    App -- "7. Show active and recent sessions" --> CustomerAdmin
    Audit -- "8. Notify on sensitive access" --> CustomerAdmin
    ImpSvc -- "9. Expire or end session" --> Sessions
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across the authorization, active-session, and teardown phases, with mitigation options keyed to the act-as session, the action policy, and the audit trail
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
