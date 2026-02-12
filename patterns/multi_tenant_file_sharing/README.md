# Multi-Tenant File Sharing: Secure Control Plane Architecture

An architectural pattern for decoupling storage from authorization to ensure strict tenant isolation and secure object sharing.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/multi-tenant-file-sharing-secure-control-plane-architecture)

## System Description

Object storage services (S3, GCS) are effectively infinite, flat key–value stores. They do not understand your application’s concepts of users, tenants, or sharing.

Secure multi-tenant file sharing requires a control plane: an application-owned registry that decouples where bytes live from who is allowed to access them. Storage is the data plane. Authorization lives entirely in your app.

```mermaid
flowchart TD
    %% Styling for clarity
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    %% --- External World ---
    Client([Client]):::untrusted
    PublicUser([Public User]):::untrusted

    %% --- Trusted Infrastructure ---
    subgraph Trusted_Zone [Trusted Zone / VPC]
        direction TB
        API[API / AuthZ Layer]:::trusted
        Registry[(Object Registry)]:::trusted
        Links[(Share Link Store)]:::trusted
    end

    %% --- Data Plane ---
    subgraph Data_Plane [Cloud Storage]
        Private_Store[(Private Tenant Storage)]:::storage
    end

    %% --- The Flows ---
    
    %% Standard Access
    Client -- "1. GET /file/{object_id}" --> API
    API -- "2. Check AuthZ & Tenant" --> Registry
    API -- "3. Issue Delivery Grant" --> Private_Store
    Private_Store -. "4. Bytes" .-> Client

    %% Shared Link Access
    PublicUser -- "5. GET /share/{token}" --> API
    API -- "6. Resolve & Verify Token" --> Links
    Links -.-> Registry
    API -- "7. Issue Short-lived Grant" --> Private_Store
    Private_Store -. "8. Bytes" .-> PublicUser
```

## Security Artifacts

- [Threat Model](threat_model.md): A detailed breakdown of risks (IDOR, Confused Deputy, Token Leakage) and their specific mitigations
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
