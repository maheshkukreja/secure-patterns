# Multi-Tenant File Sharing: Secure Control Plane Architecture

An architectural pattern for decoupling storage from authorization to ensure strict tenant isolation and secure object sharing.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/multi-tenant-file-sharing-secure-control-plane-architecture)

## System Description

Object storage services (S3, GCS) are effectively infinite, flat key–value stores. They do not understand your application’s concepts of users, tenants, or sharing.

Secure multi-tenant file sharing requires a control plane: an application-owned registry that decouples where bytes live from who is allowed to access them. Storage is the data plane. Authorization lives entirely in your app.

```mermaid
flowchart TD
    %% Styling
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef control fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    Client([Client]):::untrusted

    subgraph Control_Plane [Control Plane / API]
        API[API Layer]:::control
        Registry[(Registry DB)]:::control
    end

    subgraph Data_Plane [Data Plane / S3]
        Storage[(Private Bucket)]:::storage
    end

    %% Standard Flow
    Client -- "1. GET /files/{uuid}" --> API
    API -- "2. AuthZ Lookup" --> Registry
    Registry -- "Tenant/Policy" --> API
    API -- "3. Issue Pre-signed URL" --> Client

    %% Share Link Flow
    Client -. "1. GET /share/{token}" .-> API
    API -. "2. Validate Token" .-> Registry
    
    %% Download
    Client -- "4. GET (Signed URL)" --> Storage

    %% Layout hints
    Control_Plane ~~~ Data_Plane
```

## Security Artifacts

- [Threat Model](threat_model.md): A detailed breakdown of risks (IDOR, Confused Deputy, Token Leakage) and their specific mitigations
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
