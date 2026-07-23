# Designing a Safe Account Linking Flow

A login method is a credential binding, not an email match.

[Read the full post on securepatterns.dev](https://newsletter.securepatterns.dev/p/designing-a-safe-account-linking-flow)

## System Description

Each account keeps a list of login methods. A method is a record that binds a provider's issuer and stable subject to the account; sign-in resolves accounts through that binding. Adding a method requires a fresh authentication of the account it joins.

```mermaid
flowchart TD
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    User([User / Browser]):::untrusted
    Provider([Identity Provider]):::untrusted

    subgraph Control_Plane [Control Plane]
        API[Auth API]:::trusted
        Verifier[Assertion Verifier]:::trusted
    end

    subgraph Data_Plane [Data Plane]
        Methods[(Login Methods)]:::storage
        Audit[(Audit Log)]:::storage
    end

    User -- "1. Add method (fresh re-auth)" --> API
    API -- "2. Redirect with state + nonce" --> User
    User -- "3. Authenticate" --> Provider
    Provider -- "4. Assertion" --> User
    User -- "5. Callback" --> API
    API -- "6. Verify against pinned issuer" --> Verifier
    Verifier -- "7. issuer + subject" --> API
    API -- "8. Conflict check + write binding" --> Methods
    API -- "9. Link event + notify" --> Audit
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across adding a method, sign-in resolution, and lifecycle/policy
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
