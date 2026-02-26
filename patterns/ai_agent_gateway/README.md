# AI Agent Gateway: The Authorization Chokepoint

An architectural pattern for mediating all communication between client applications and AI agents through a centralized gateway that enforces authorization, inspects payloads in both directions, and logs every interaction.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/ai-agent-gateway-the-authorization-chokepoint)

## System Description

An AI Agent Gateway sits between client applications and AI agents, mediating every request and response. The gateway enforces authorization, inspects payloads in both directions, and logs every interaction. Agents are workloads. The gateway is the control plane. Client apps are untrusted callers.

```mermaid
flowchart LR
    %% ========= Styles =========
    classDef untrusted fill:#fdecea,stroke:#b71c1c,stroke-width:2px,color:#000;
    classDef control fill:#e3f2fd,stroke:#0d47a1,stroke-width:2px,color:#000;
    classDef workload fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000;
    classDef data fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px,color:#000;

    %% ========= Untrusted Zone =========
    subgraph Untrusted["Untrusted: Client Applications"]
        Slack[Slack Bot]:::untrusted
        Web[Web UI]:::untrusted
        CLI[Custom App / mTLS]:::untrusted
    end

    %% ========= Trusted Control Plane =========
    subgraph Gateway["Trusted: Agent Gateway (Control Plane)"]
        direction LR
        AuthN[1. Authentication]:::control
        AuthZ[2. Authorization Policy]:::control
        InDLP[3. Inbound Inspection]:::control
        Router[4. Agent Routing]:::control
        OutDLP[6. Outbound Inspection]:::control
        Egress[7. Deliver to Client]:::control
        Logs[(Audit Log)]:::data
    end

    %% ========= Agent Workloads =========
    subgraph Agents["Trusted: Agent Workloads"]
        direction TB
        A1[Agent A]:::workload
        A2[Agent B]:::workload
        A3[Agent C]:::workload
    end

    %% ========= Request Flow =========
    Slack -->|1| AuthN
    Web -->|1| AuthN
    CLI -->|1| AuthN
    AuthN -->|2| AuthZ
    AuthZ -->|3| InDLP
    InDLP -->|4| Router
    Router -->|5| A1
    Router -->|5| A2
    Router -->|5| A3

    %% ========= Response Flow =========
    A1 -->|6| OutDLP
    A2 -->|6| OutDLP
    A3 -->|6| OutDLP
    OutDLP -->|7| Egress

    %% ========= Logging =========
    AuthZ -.->|decision| Logs
    InDLP -.->|scan result| Logs
    OutDLP -.->|scan result| Logs
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across client-to-gateway, gateway-to-agent, and agent-response-to-client phases
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
