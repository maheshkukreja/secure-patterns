# Verification checklist

### Authorization and start
- [ ] Impersonation requires a permission distinct from general admin access; removing it blocks new sessions
- [ ] Starting a session requires a fresh step-up (WebAuthn or MFA), not just an existing login
- [ ] A request without a reason and a ticket reference is rejected
- [ ] Where second-party approval is required, the requester cannot approve their own request
- [ ] An agent cannot impersonate a target whose role outranks the agent's grant authority

### Session scope and sensitive actions
- [ ] A blocked high-risk operation fails inside an impersonation session (for example, a password reset or a data export)
- [ ] A new session is read-only; writes require explicit elevation and a second approval
- [ ] The act-as token cannot be exchanged for the customer's normal session or refresh token

### Visibility
- [ ] The customer's admins can see active and recent sessions
- [ ] The customer is notified when a session starts on a sensitive account
- [ ] The agent sees a persistent banner naming the impersonated customer and the time remaining
- [ ] Ending the session from the banner stops access immediately

### Identity and audit
- [ ] Every impersonated action records both the agent and the customer identity, the `impersonation_id`, and the ticket
- [ ] Session start and end are distinct audit events
- [ ] Impersonation logs live in an append-only store separate from general logs, restricted to security and compliance

### Enforcement and propagation
- [ ] If the action policy cannot be evaluated, the action is denied rather than allowed
- [ ] An internal service called during a session records the support agent, not the customer alone
- [ ] A tenant with strict conditional access can disable impersonation entirely

### Lifecycle and teardown
- [ ] The session expires on a short absolute TTL and on idle timeout
- [ ] A killed or expired session's token stops working on the next request
- [ ] Removing an agent's impersonation permission revokes their active sessions
- [ ] Direct reuse of the act-as token after the session ends fails
