# Password Reset Flows: The Secure Implementation Guide

An architectural pattern for password reset: a generic request endpoint, stateful single-use tokens, atomic consume with policy validation first, full session revocation, and a notification email as the compensating control. The API issues short-lived tokens by email; submitting a valid token with a new password updates the credential and revokes existing sessions in the same transaction.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/password-reset-flows-the-secure-implementation-guide)

## System Description

An API issues short-lived, single-use reset tokens via email. Submitting a valid token with a new password updates the password and revokes existing sessions. A separate email notifies the owner.

```mermaid
flowchart TD
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    User([User]):::untrusted
    Mail([Email Inbox]):::untrusted

    subgraph Control_Plane [Control Plane]
        API[Reset API]:::trusted
        Worker[Async Issuer]:::trusted
    end

    subgraph Data_Plane [Data Plane]
        Tokens[(Token Store)]:::storage
        Accounts[(Accounts + Sessions)]:::storage
    end

    User -- "1. Request reset" --> API
    API -- "2. Return 202" --> User
    API -- "3. Enqueue (blind)" --> Worker
    Worker -- "4. Lookup user (drop if missing)" --> Accounts
    Worker -- "5. Store hashed token, TTL 15m" --> Tokens
    Worker -- "6. Send reset link" --> Mail
    Mail -- "7. Click link" --> User
    User -- "8. Submit token + new password" --> API
    API -- "9. Validate + atomic consume" --> Tokens
    API -- "10. Update password + revoke sessions" --> Accounts
    API -- "11. Notify owner" --> Mail
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across request intake, token lifecycle, and post-reset state phases
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
