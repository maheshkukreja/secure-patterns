# Pre-signed URLs: The Secure Implementation Guide

An architectural pattern for handling direct-to-storage uploads without exposing your application to malware hosting, cost abuse, or unverified content.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/pre-signed-urls-the-secure-implementation-guide)

## System Description

In this pattern, the API issues short-lived, exact-scope URLs for direct storage uploads. To keep the system safe, the app enforces a strict pipeline: **Quarantine → Finalize → Scan → Publish**.

```mermaid
flowchart TD
    %% Styling for clarity
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    %% --- External World ---
    Client([Client]):::untrusted
    
    %% --- Trusted Infrastructure ---
    subgraph Trusted_Zone [Trusted Zone / VPC]
        direction TB
        API[API Control Plane]:::trusted
        Registry[(Object Registry)]:::trusted
        Queue>Async Queue]:::trusted
        Worker[Scan / Transform Worker]:::trusted
    end

    %% --- Data Plane ---
    subgraph Data_Plane [Cloud Storage]
        Quarantine[(Quarantine Prefix)]:::storage
        Published[(Published Prefix)]:::storage
    end
    
    CDN(CDN - Optional):::storage

    %% --- The Golden Path Flows ---
    
    %% Upload Flow
    Client -- "1. Request Upload URL" --> API
    API -.-> Registry
    
    Client -- "2. PUT Bytes (Direct)" --> Quarantine
    
    Client -- "3. POST /finalize" --> API
    API -- "4. Trigger Scan" --> Queue
    
    Queue -.-> Worker
    Worker -- "5. Read & Scan" --> Quarantine
    Worker -- "6. Move Safe File" --> Published
    Worker -. "Update Status" .-> Registry

    %% Download Flow
    Client -- "7. Request Download URL" --> API
    Client -- "8. GET Content" --> CDN
    CDN -.-> Published

    %% Layout hints
    Quarantine ~~~ Published
```

## Security Artifacts

- [Threat Model](threat_model.md): A detailed breakdown of risks (TOCTOU, DoS, IDOR) and their specific mitigations
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
