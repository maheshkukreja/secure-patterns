# Passkey Authentication: Architecting a Secure Relying Party

An architectural pattern for implementing passkey authentication at the relying party: ceremony verification, credential storage, and the trust boundaries between browser, authenticator, and your server. The RP never stores a shared secret; it stores a public key and verifies signatures produced by an authenticator the user controls.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/passkey-authentication-architecting-a-secure-relying-party?draft=true)

## System Description

A relying party issues random challenges, verifies signed responses from authenticators, and stores credential public keys. Two ceremonies define the protocol: registration (create a credential) and authentication (prove you hold it). The browser mediates both ceremonies, enforcing origin binding before the authenticator ever sees the request.

```mermaid
flowchart LR
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    subgraph Client_Zone ["Client (Untrusted)"]
        User([User / Browser]):::untrusted
        Authenticator([Authenticator]):::untrusted
        User -- "2. CTAP2 / Local API" --> Authenticator
    end

    subgraph RP_Server ["Relying Party Server"]
        direction TB
        ChallengeGen[1. Challenge Generator]:::trusted
        ChallengeStore[(Challenge Store)]:::trusted
        Verifier[3. Ceremony Verifier]:::trusted
        CredStore[(Credential Store)]:::trusted
        SessionMgr[4. Session Manager]:::trusted
    end

    User -- "1. Initiate ceremony" --> ChallengeGen
    ChallengeGen -- "Store challenge" --> ChallengeStore
    ChallengeGen -- "Challenge + options" --> User
    Authenticator -- "Signed response" --> User
    User -- "3. Submit response" --> Verifier
    Verifier -- "Verify challenge" --> ChallengeStore
    Verifier -- "Lookup / store" --> CredStore
    Verifier -- "4. Issue session" --> SessionMgr
    SessionMgr -- "Session token" --> User
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across ceremony integrity, credential management, and post-ceremony session phases
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
