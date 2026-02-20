# Kubernetes Workload Identity: Eliminating Static Cloud Credentials

An architectural pattern for authenticating Kubernetes pods to cloud services (AWS, GCP, Azure) using short-lived, OIDC-federated ServiceAccount tokens instead of static IAM keys.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/kubernetes-workload-identity-eliminating-static-cloud-credentials)

## System Description

In this pattern, a pod proves its identity to a cloud provider without static keys: Kubernetes issues a short-lived, signed token, the cloud verifies that token against the cluster's public keys, and, if the token matches a set of trust rules, exchanges it for temporary cloud credentials.

```mermaid
flowchart TD
    %% Styling
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    Pod([Pod / Workload]):::untrusted

    subgraph K8s_Cluster [Kubernetes Cluster]
        direction TB
        SA[ServiceAccount]:::trusted
        TokenVolume[Projected Token Volume]:::trusted
        OIDC[OIDC Issuer / JWKS]:::trusted
    end

    subgraph Cloud_IAM [Cloud IAM]
        direction TB
        STS[STS / Token Exchange]:::storage
        Federation[Trust Policy]:::storage
        Role[IAM Role]:::storage
    end

    CloudService([Cloud Service eg. S3]):::storage

    %% The Flow
    SA -- "Issues & Rotates JWT" --> TokenVolume
    Pod -- "1. Mounts projected token" --> TokenVolume
    Pod -- "2. Presents token to STS" --> STS

    %% The Cryptographic Handshake
    STS -- "3. Verifies signature via JWKS" --> OIDC
    STS -- "4. Evaluates claims (NS / SA / AUD)" --> Federation
    Federation -. "Binds to" .-> Role

    %% The Credential Grant
    STS -- "5. Issues temp credentials" --> Pod
    Pod -- "6. Accesses cloud resource" --> CloudService

    %% Layout hints
    K8s_Cluster ~~~ Cloud_IAM
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across identity binding, token theft, audience misconfiguration, issuer spoofing, and OIDC drift
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
