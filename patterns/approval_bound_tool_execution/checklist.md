# Verification Checklist

## Envelope and normalization

- [ ] Every sensitive action converts to a canonical envelope before approval is requested
- [ ] The approval UI renders `tool_id`, `operation`, `target`, `tenant_id`, and every parameter from the stored envelope, not from a model-generated summary
- [ ] The same proposed action produces the same `parameters_hash` and `action_hash` across app nodes and runtimes
- [ ] Changing `tool_id`, `operation`, `target`, `tenant_id`, any parameter, `normalizer_version`, `tool_schema_version`, or `expires_at` produces a different `action_hash`
- [ ] If `normalizer_version` or `tool_schema_version` is no longer active at execution time, execution is rejected and re-approval is required
- [ ] Any tool call matching no policy rule is denied, not forwarded to a human for improvised approval

## Policy and approver selection

- [ ] `approver_id != actor_id` is enforced at the store layer, not only in UI routing
- [ ] High-risk approvals require a fresh WebAuthn or step-up MFA context
- [ ] Policy engine excludes the requester from the candidate approver pool before dispatching
- [ ] Approval routing operates on the normalized envelope, not on raw model output

## Approval lifecycle

- [ ] Approvals expire on a short TTL by action class; the executor rejects expired envelopes
- [ ] Pending and approved envelopes can be revoked before execution; revocation takes effect immediately
- [ ] A revoked envelope fails the status check regardless of what the caller asserts
- [ ] Approval store uses append-only state transitions: revocations are new records, not mutations to existing ones

## Execution

- [ ] Executor fetches the envelope from the server-side store by `envelope_id`; it does not accept tool arguments from the model at execution time
- [ ] Executor recomputes `parameters_hash` from `envelope.parameters` and `action_hash` from the stored envelope fields; any mismatch is a security event, not a normal error
- [ ] Executor marks the envelope `consumed` before the first non-idempotent side effect
- [ ] A second execution attempt on a `consumed` envelope is rejected
- [ ] Concurrent execution requests for the same `envelope_id` result in exactly one accepted attempt; the others are rejected by the compare-and-swap

## Evidence

- [ ] Evidence log joins `envelope_id`, `actor_id`, `approved_by`, and the outcome event
- [ ] Reconciliation alerts on `execution.claimed` or `execution.started` events with no terminal outcome after twice the envelope TTL
