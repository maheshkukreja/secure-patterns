# Threat Modeling RAG Access Control

An architectural pattern for preserving source document permissions through RAG retrieval: server-side scope derivation, ACL metadata propagation during ingestion, hard pre-filters at query time, and session isolation in prompt assembly. The source document system is the authority for permissions; the retrieval index is a derived projection.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/threat-modeling-rag-access-control)

## System Description

A RAG access control system resolves caller identity and derives an authorization scope from verified credentials and policy. It retrieves only chunks bound to that scope, and only approved chunks reach the model. The source document system is the authority for permissions and lifecycle state. The retrieval index is a projection, not the source of truth.

```mermaid
flowchart TD
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    User([User / Calling Service]):::untrusted

    subgraph Control_Plane [Control Plane]
        direction TB
        API[RAG Gateway]:::trusted
        Policy[Policy Resolver]:::trusted
        IdP[Identity Provider]:::trusted
    end

    subgraph Ingestion [Ingestion Path]
        direction TB
        Sources([Source Systems]):::storage
        Extractor[ACL Extractor]:::trusted
        Chunker[Chunker + Embedder]:::trusted
    end

    subgraph Data_Plane [Retrieval Data Plane]
        Index[(Vector / Hybrid Index)]:::storage
        LLM[LLM Runtime]:::trusted
    end

    Sources -- "1. Docs + permissions" --> Extractor
    Extractor -- "2. ACL metadata" --> Chunker
    Chunker -- "3. Tagged chunks" --> Index

    User -- "4. Query + token" --> API
    API -- "5. Resolve principal" --> IdP
    IdP -.-> Policy
    API -- "6. Scoped query" --> Index
    Index -- "7. Authorized chunks" --> API
    API -- "8. Approved chunks" --> LLM
    LLM -- "9. Answer + citations" --> User
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across ingestion, query-time retrieval, and prompt assembly phases
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
