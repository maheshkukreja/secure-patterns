# OAuth Token Storage: Securing Third-Party Credentials in Multi-Tenant SaaS

An architectural pattern for storing and using third-party OAuth tokens in multi-tenant SaaS. A centralized token service encrypts credentials per tenant, refreshes them under lock, and mediates all outbound API calls so that application code never touches raw tokens.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/oauth-token-storage-securing-third-party-credentials-in-multi-tenant-saas)

## System Description

A centralized token service encrypts and stores OAuth credentials per tenant, refreshes them under lock, and makes outbound API calls on behalf of application code. Application code sends requests in authenticated tenant context, specifying `integration_id`; it never receives, caches, or persists credentials itself.

```mermaid
flowchart TD
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    App([Application Code]):::untrusted
    ThirdParty([Third-Party API]):::untrusted

    subgraph Your_Platform [Your Platform]
        TokenSvc[Token Service]:::trusted
    end

    subgraph Data_Layer [Data Layer]
        TokenStore[(Encrypted Token Store)]:::storage
        KMS[KMS / HSM]:::storage
    end

    App -- "1. Request with auth context + integration_id" --> TokenSvc
    TokenSvc -- "2. Resolve credential" --> TokenStore
    TokenSvc -- "3. Unwrap DEK" --> KMS
    TokenSvc -- "4. Call with injected Authorization header" --> ThirdParty
    ThirdParty -- "5. Response" --> TokenSvc
    TokenSvc -- "6. Return response (no credential)" --> App
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across token storage, lifecycle management, and outbound token use phases
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
