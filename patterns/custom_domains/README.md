# Custom Domains: Binding Customer Hostnames to Tenants

A customer only needs to prove they own a domain once, but there is no ongoing check to confirm they still own it later.

[Read the full post on securepatterns.dev](https://newsletter.securepatterns.dev/p/custom-domains-binding-customer-hostnames-to-tenants)

## System Description

A tenant claims a hostname by showing they can update the customer's DNS zone. The platform then creates a binding between the hostname and the tenant, and the edge only serves the hostname while this binding is active. A scheduled job keeps checking the proof until the binding is released.

```mermaid
flowchart LR
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    Admin([Tenant Admin]):::untrusted
    DNS[(Customer DNS)]:::untrusted
    User([End User]):::untrusted

    subgraph Control_Plane [Control Plane]
        API[Domain API]:::trusted
        Verifier[DNS Verifier]:::trusted
        CertMgr[Certificate Manager]:::trusted
    end

    subgraph Data_Plane [Data Plane]
        Edge[Edge Router]:::trusted
        Registry[(Binding Registry)]:::storage
    end

    Admin -- "1. Submit hostname" --> API
    API -- "2. Record claim" --> Registry
    Admin -- "3. Publish challenge record" --> DNS
    Verifier -- "4. Resolve challenge" --> DNS
    Verifier -- "5. Create verified binding" --> Registry
    CertMgr -- "6. Order certificate" --> Registry
    Edge -- "7. Load active bindings" --> Registry
    User -- "8. Request app.customer.com" --> Edge
    Verifier -- "9. Re-check active bindings" --> DNS
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across the claim, serve, and drift-and-release phases of the hostname binding lifecycle
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
