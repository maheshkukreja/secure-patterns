# Designing API Idempotency Keys to Prevent Duplicate Writes

An architectural pattern for API idempotency: atomic key reservation, request fingerprinting, the write-first pattern for safe external calls, and outbox-driven side-effect delivery. The server reserves a client-provided key in a durable store before doing any work, writes business state and a pending job in a single transaction, then hands off external calls to a background worker.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/designing-api-idempotency-keys-to-prevent-duplicate-writes)

## System Description

An API receives a state-changing request with an `Idempotency-Key` header, reserves the key in a durable store before doing any work, writes the business state and a pending job in a single transaction, then hands off external side effects to a background worker. On a valid retry for the same request, the server returns the cached result instead of reprocessing.

```mermaid
flowchart TD
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    Client([Client]):::untrusted

    subgraph Trusted_Zone [Trusted Zone / VPC]
        Handler[Request Handler]:::trusted
        KeyCheck{Reserve or replay?}:::trusted
    end

    subgraph DB_Transaction [Database Transaction Boundary]
        Reserve[Reserve Key]:::storage
        AppState[Write Business State]:::storage
        Outbox[Write Pending Job]:::storage
    end

    subgraph Async [Async Delivery]
        Worker[Outbox Worker]:::trusted
    end

    External([External Service]):::untrusted

    Client -- "1. POST + Idempotency-Key" --> Handler
    Handler -- "2. Reserve key" --> KeyCheck
    KeyCheck -- "reserved" --> Reserve
    KeyCheck -- "exists: return cached result" --> Client
    Reserve -- "3. Same transaction" --> AppState
    AppState -- "3. Same transaction" --> Outbox
    Outbox -- "4. Commit" --> Worker
    Worker -- "5. Call external service" --> External
    Worker -- "6. Record completion" --> Reserve
    Handler -- "7. Return result" --> Client
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across key reservation, processing, delivery, and lifecycle phases
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
