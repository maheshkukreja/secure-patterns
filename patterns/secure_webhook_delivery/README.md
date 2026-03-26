# Secure Webhook Delivery: Signing, Verification, and SSRF Prevention

An architectural pattern for webhook delivery: HMAC signing, receiver-side verification, and SSRF prevention through egress proxies and DNS rebinding defenses. The sender signs every payload with a per-endpoint secret and delivers through a network-level egress proxy; the receiver verifies the signature and timestamp before processing.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/secure-webhook-delivery-signing-verification-and-ssrf-prevention)

## System Description

An event system detects state changes, signs the payload with a per-endpoint secret, and delivers an HTTP POST through an egress proxy that blocks internal network ranges. The customer's server verifies the signature, checks the timestamp, and processes the event idempotently.

```mermaid
flowchart TD
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    subgraph Your_Platform [Your Platform]
        App[Application Layer]:::trusted
        Events>Event Bus / Queue]:::storage
        Dispatcher[Webhook Dispatcher]:::trusted
        Registry[(Endpoint Registry)]:::trusted
        Signer[HMAC Signer]:::trusted
        Proxy[Egress Proxy]:::trusted
    end

    subgraph Customer [Customer Infrastructure]
        Endpoint[Webhook Endpoint]:::untrusted
        Verifier[Signature Verifier]:::untrusted
        Handler[Event Handler]:::untrusted
    end

    App -- "1. Emit event" --> Events
    Events -- "2. Dequeue" --> Dispatcher
    Dispatcher -.-> Registry
    Dispatcher -- "3. Sign payload" --> Signer
    Signer -- "4. POST via proxy" --> Proxy
    Proxy -- "5. Deliver" --> Endpoint

    Endpoint -- "6. Verify signature" --> Verifier
    Verifier -- "7. Process" --> Handler
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across webhook registration, delivery, and receiver-side verification phases
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
