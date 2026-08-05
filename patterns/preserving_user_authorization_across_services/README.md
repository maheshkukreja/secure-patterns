# Preserving User Authorization Across Services

A request three services deep should still name the user who started it.

[Read the full post on securepatterns.dev](https://newsletter.securepatterns.dev/p/preserving-user-authorization-across-services)

## System Description

A user request crosses several internal services before it touches data. The external token stops at the edge service, which exchanges it for a short-lived internal token carrying the user and tenant. Each downstream service verifies the calling workload and authorizes its own action against that context.

```mermaid
flowchart TD
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    Client([Client]):::untrusted

    subgraph Request_Path [Request Path]
        EdgeSvc[Edge Service]:::trusted
        SvcB[Service B]:::trusted
        SvcC[Service C]:::trusted
    end

    subgraph Control_Plane [Control Plane]
        STS[Token Service]:::trusted
    end

    subgraph Data_Plane [Data Plane]
        Data[(Data Store)]:::storage
        Audit[(Audit Log)]:::storage
    end

    Client -- "1. Request + external token" --> EdgeSvc
    EdgeSvc -- "2. Exchange external token" --> STS
    STS -- "3. Internal token for service B" --> EdgeSvc
    EdgeSvc -- "4. Call with internal token" --> SvcB
    SvcB -- "5. Exchange for narrower token" --> STS
    SvcB -- "6. Call service C" --> SvcC
    SvcC -- "7. Act on the resource" --> Data
    SvcB -- "8. Log subject + actor" --> Audit
    SvcC -- "9. Log subject + actor" --> Audit
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across minting and exchange, the call chain, and deferred work and audit
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
