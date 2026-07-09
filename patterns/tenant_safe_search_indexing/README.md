# Tenant-Safe Search Indexing

Your search index is a second copy of your data, so it needs the same access rules as the database it came from.

[Read the full post on securepatterns.dev](https://newsletter.securepatterns.dev/p/tenant-safe-search-indexing)

## System Description

A search subsystem indexes tenant records into an engine and tags each document with its tenant on write. Every query enforces that boundary. The primary database stays the authority for permissions and lifecycle; the index is a derived copy it keeps in sync.

```mermaid
flowchart TD
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    Client([Client]):::untrusted

    subgraph Control_Plane [Control Plane]
        Indexer[Indexer / Worker]:::trusted
        QueryBuilder[Query Builder]:::trusted
    end

    subgraph Data [Data Stores]
        DB[(Primary DB)]:::storage
        Queue[(Outbox / Queue)]:::storage
        Index[(Search Index)]:::storage
    end

    DB -- "1. Change event (create / update / permission / delete)" --> Queue
    Queue -- "2. Ordered event" --> Indexer
    Indexer -- "3. tenant_id + ACL + version" --> Index

    Client -- "4. Query + session token" --> QueryBuilder
    QueryBuilder -- "5. Server-enforced filtered query" --> Index
    Index -- "6. Tenant-scoped hits / facets / suggestions" --> QueryBuilder
    QueryBuilder -- "7. Results" --> Client
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across indexing, query-time, and propagation/reindex phases
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
