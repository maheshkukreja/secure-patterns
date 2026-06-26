# Threat Model

## Baseline assumptions

- The LLM agent and its tool invocations are untrusted for authorization purposes
- The normalizer, policy engine, envelope store, executor, and evidence log are trusted components
- The requester may be operating under prompt injection or plan drift
- The approver is authenticated and can provide freshness or step-up context for high-risk actions
- Target systems still enforce their own authorization independently of the approval check
- Standard infra controls (TLS, service authentication, secret management, database access controls) are assumed to be in place. This model focuses on the proposal-to-execution binding

#### A note on risk

This table is not a checklist. Focus on preventing the highest-impact failures first. Detection and response are acceptable where prevention is impractical.

## Phase 1: Proposal and normalization

Focus: Preventing the proposed action from diverging into a different effective action through input manipulation or cross-tenant substitution

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| Envelope record | Plan drift: Agent presents an action for approval at T1, re-reasons between approval and execution due to prompt injection or new retrieval context, and submits mutated arguments at T3 | Envelope created at proposal time | 1. Executor fetches all parameters from the stored envelope; ignores any arguments supplied by the model at execution time<br>2. Short envelope expiry (5–30 min) bounds the drift window<br>3. Pre-execution state check on the target resource for high-impact operations | High |
| Normalization layer | Alias bypass: Agent proposes `deploy(env="prod")`; policy denies `"production"` but not `"prod"`, and the alias passes through | Tool schema validation | 1. Alias registry maps all known equivalents to canonical form before the envelope is created<br>2. Policy engine operates on the post-normalization canonical form, never on raw model output<br>3. Unknown parameter values are rejected at normalization time, before the envelope is created | Medium |
| Approved parameters | Hidden parameter injection: Model includes optional flags (`drain_timeout=0`, `restart_dependents=true`) valid per the tool schema but absent from a truncated approval display | Envelope contains full parameters object | 1. Approval UI renders every field from the envelope's parameters object, not a model-generated summary<br>2. Tool schema marks parameters requiring explicit acknowledgment before they can be included in an envelope<br>3. Policy engine evaluates the full parameters object, including optional fields | Medium |
| Tenant namespace | Cross-tenant substitution: Agent proposes an action targeting a resource in another tenant's namespace; if the executor derives the target from model output, it runs on the wrong tenant | Auth context available per request | 1. Envelope service derives `tenant_id` from the session token, never from the request body or model output<br>2. Policy engine verifies the proposed target is within the resource registry for `envelope.tenant_id` before storing the envelope<br>3. Executor re-verifies tenant match from the stored envelope before issuing scoped credentials | High |

## Phase 2: Approval decision

Focus: Ensuring approval is granted by the right principal with deliberate intent, and that the approval record cannot be silently bypassed

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| Approval decision | Self-approval: Requester and approver are the same principal, defeating independent review | Approval routing exists | 1. Hard constraint: `approval_record.approver_id != envelope.actor_id`, enforced at the store layer, not only in the UI<br>2. Policy engine excludes the requester from the candidate approver pool before dispatching the request<br>3. Alert when the eligible approver pool for a required action has fewer than two candidates | Medium |
| Approver session | Stale session: High-risk approval granted from an hours-old browser session, without deliberate re-authentication | Session authentication required | 1. Require WebAuthn or step-up MFA bound to the specific `envelope_id` for high-risk operations<br>2. Maximum session age for approval: reject if the session is older than the policy window (typically 10–30 min for write actions)<br>3. Approval UI shows elapsed time since last authentication when the freshness limit is approaching | Medium |
| Approval view | Approval phishing: Agent frames a destructive action as benign in conversational context; approver consents based on the framing rather than the envelope fields | Approval UI exists | 1. Approval UI renders exclusively from envelope fields; the agent's conversational context is not displayed on the approval screen<br>2. For destructive operations, require the approver to type the target or quantity before submitting<br>3. "Cannot be undone" label derived from the tool schema, not from the model's description of the action | Medium |

## Phase 3: Execution and evidence

Focus: Ensuring the executor accepts the approved envelope once and emits complete evidence regardless of outcome

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
|-------|--------|-------------------|--------------------|------|
| Envelope state | Approval replay: A previously-consumed or expired envelope is presented again to trigger a second execution | Single-use flag on envelope | 1. Atomic compare-and-swap: `UPDATE SET status='consumed' WHERE status='approved'`; abort if zero rows are updated<br>2. Mark the envelope consumed before the side effect, not after<br>3. Reject execution requests for envelopes in `consumed` or `expired` status regardless of the request source | High |
| Claim transition | Concurrent consume race: Two executor instances receive the same `envelope_id` simultaneously and both attempt to transition the envelope to consumed, risking duplicate side effects | Compare-and-swap on consume | 1. Treat zero rows updated by the compare-and-swap as an immediate rejection<br>2. Reject execution requests for envelopes already in `consumed` status regardless of the caller's identity<br>3. Reconcile `execution.claimed` or `execution.started` events with no terminal outcome after twice the envelope TTL; require manual recovery before retrying non-idempotent actions | High |
| Execution authority | Authority overreach: Executor has reusable broad credentials and can perform actions beyond the approved envelope | Service authentication | 1. Target service fetches `envelope_id` and verifies tenant, operation, target, status, and expiry before executing<br>2. For systems that cannot enforce envelope checks, issue a short-lived scoped token or assumed-role session bounded to the approved operation and target<br>3. Include `envelope_id` in target-side logs for attribution | Medium |
| Evidence log | Missing outcome: Execution fails or partially succeeds but no terminal evidence event is emitted, leaving the audit record ambiguous | Log infrastructure present | 1. Emit `execution.failed` with an error code for every failure, including timeouts; never swallow the error silently<br>2. Alert on `execution.claimed` or `execution.started` events with no terminal outcome event after twice the envelope TTL<br>3. For partial executions, emit `execution.partial` with a description of what completed and what did not | Low |
