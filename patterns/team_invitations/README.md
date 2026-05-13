# Designing a Safe Team Invitation Flow

An architectural pattern for team invitations: a server-side pending grant, a short-lived claim token in the email, and an atomic accept handler that binds the invitation to an independently verified identity before writing membership. An admin requests an invite; the API persists a pending grant and sends an email link; the invitee signs in and accepts; the server matches the session's verified email to the invitation and writes membership in a single transaction.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/designing-a-safe-team-invitation-flow)

## System Description

Every invitation lives server-side as a pending grant. The email carries only a short-lived claim token. To accept, the recipient signs in; the server matches the session's verified email to the invitation and writes the membership in a single transaction.

```mermaid
flowchart TD
    classDef untrusted fill:#ffebee,stroke:#c62828,stroke-width:2px,color:black;
    classDef trusted fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:black;
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:black;

    Inviter([Inviter]):::untrusted
    Invitee([Invitee]):::untrusted

    subgraph Control_Plane [Control Plane]
        API[Invitations API]:::trusted
        IdP[Identity Provider]:::trusted
    end

    subgraph Data_Plane [Data Plane]
        Invites[(Invitations)]:::storage
        Members[(Memberships)]:::storage
    end

    Inviter -- "1. Create invite" --> API
    API -- "2. Persist + email link" --> Invites
    Invitee -- "3. Authenticate" --> IdP
    Invitee -- "4. POST accept" --> API
    API -- "5. Verify session + recipient binding" --> IdP
    API -- "6. Atomic consume + membership" --> Members
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across issuance, token and email, acceptance, and post-acceptance phases
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
