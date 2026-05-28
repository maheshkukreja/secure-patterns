# A2A Remote Agent Discovery: Trust the Registry, Not the Agent Card

An Agent Card is a business card for a remote agent. It tells other agents how to find and call it. In A2A-style systems, the remote agent publishes this card so other agents can discover it. The card should not decide whether your agent is allowed to delegate work.

If an origin agent calls a remote agent just because it found a card, discovery has become authorization. That can send user context and task data to a remote agent before anyone has approved that delegation. The safer pattern is to review Agent Cards before they become callable, register approved agents, and route runtime delegation through a broker that enforces the registry decision.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/a2a-remote-agent-discovery-trust-the-registry-not-the-agent-card)

## System Description

An origin agent may discover remote agents through Agent Cards, but it cannot call a discovered agent directly. Approved agents are added to a capability registry first. At runtime, the origin delegates through a broker, and the broker only calls agents that are already approved in the registry.

```mermaid
flowchart LR
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    subgraph Untrusted ["Untrusted"]
        User([User]):::untrusted
        Card[Agent Card]:::untrusted
        Remote[Remote Agent]:::untrusted
    end

    subgraph Trusted ["Trusted (your infrastructure)"]
        Review[Operator Review]:::trusted
        Origin[Origin Agent]:::trusted
        Broker[Delegation Broker]:::trusted
        Registry[(Capability Registry)]:::storage
        Audit[(Audit Log)]:::storage
    end

    Card -->|A review| Review
    Review -->|B approve| Registry
    User -->|1 task| Origin
    Origin -->|2 delegate| Broker
    Broker -->|3 lookup| Registry
    Broker -->|4 call approved agent| Remote
    Remote -->|5 remote input| Broker
    Broker -->|6 return labeled result| Origin
    Broker -->|7 log| Audit
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks at the discovery-to-delegation boundary, with mitigation options keyed to the registry, broker, and delegation token
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
