# Threat Model

## Phase 1: Client to gateway

Focus: Preventing unauthorized access to agents and injection of malicious payloads

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Agent access | AuthZ bypass: Attacker crafts request to reach an agent they're not permitted to use (e.g., manipulating routing headers or agent IDs) | AuthZ policy evaluation on every request | 1. Deny default: Reject if no explicit policy match<br>2. Routing lockdown: Clients address agents by logical name only; the gateway resolves to internal endpoints and rejects requests containing direct agent addresses | High |
| User identity | Session hijack: Attacker steals OAuth token or session cookie to impersonate a legitimate user | Token validation at gateway | 1. Short-lived tokens: Access tokens expire in minutes, not hours<br>2. Binding: Bind sessions to client fingerprint (IP, TLS session) where feasible<br>3. Revocation: Support immediate token revocation via introspection or short cache TTL | Medium |
| Agent runtime | Prompt injection via client: User crafts input designed to override the agent's system prompt, causing it to execute unintended tool calls | Input validation at gateway | 1. Input scanning: Pattern-match known injection templates (e.g., "ignore previous instructions")<br>2. Tool governance: See tool execution row in Phase 2<br>3. Agent hardening: Agents use structured tool-call interfaces, not free-form command parsing | High |
| Gateway availability | Request flooding: Attacker sends high volume of requests to exhaust gateway resources | Rate limiting | 1. Per-user rate limits: Throttle by authenticated identity<br>2. Per-agent limits: Protect individual agents from traffic spikes<br>3. Backpressure: Gateway returns 429 and sheds load before agents are affected | Medium |

## Phase 2: Gateway to agent

Focus: Securing the communication channel and preventing credential leakage

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Agent credentials | Credential leak: Agent API keys or service account tokens are exposed in logs, error messages, or configuration | Credentials stored in secrets manager | 1. Injection: Gateway injects agent credentials at request time; clients never see them<br>2. Rotation: Automate credential rotation; detect stale credentials<br>3. Redaction: Scrub credentials from all log outputs | High |
| Tool execution | Unauthorized tool call: User or injected prompt triggers a tool invocation (shell, SQL, HTTP) beyond the user's permitted scope, causing data modification or lateral movement | AuthZ policy evaluation | 1. Tool allow-list: Gateway maintains per-role allow-lists of permitted tool types and blocks unlisted calls before they reach the agent<br>2. Structured intents: Agents emit structured tool-call intents (tool name, arguments as typed fields); gateway parses and validates before execution<br>3. Parameter validation: Gateway enforces argument constraints (e.g., allowed hostnames for HTTP fetch, read-only for SQL) | High |
| Agent identity | Agent spoofing: Rogue process registers as a legitimate agent and receives user requests | Agent authentication required | 1. mTLS: Require client certificates for agent-to-gateway connections<br>2. Registration: Agents must be registered in the gateway's agent inventory before receiving traffic<br>3. Health checks: Gateway periodically verifies agent identity and liveness | Medium |
| Request integrity | Tampering: Man-in-the-middle modifies request payloads between gateway and agent | Private network / VPC | 1. mTLS: Encrypt and authenticate all gateway-to-agent traffic<br>2. Signing: Gateway signs forwarded requests; agent verifies signature before processing | Low |

## Phase 3: Agent response to client

Focus: Preventing data exfiltration and ensuring response integrity

| Asset | Threat | Baseline Controls | Mitigation Options | Risk |
| :--- | :--- | :--- | :--- | :--- |
| Sensitive data | Data exfil via response: Agent returns API keys, database credentials, PII, or internal infrastructure details in its natural-language response | Outbound DLP scan | 1. Pattern matching: Scan for known secret formats (AWS keys, connection strings, SSNs)<br>2. Redaction: Replace matched patterns with `[REDACTED]` before delivering to client<br>3. Alerting: Notify security team when DLP rules trigger repeatedly for the same agent | High |
| Response scope | Over-fetching: Agent retrieves and returns data beyond what the user's role permits (e.g., agent has broad DB access but user should only see their own records) | AuthZ at gateway | 1. Scoped context: Gateway passes the user's permission scope to the agent as part of the request context<br>2. Structured responses: Where agents return structured data (JSON, SQL results), validate response fields against the user's authorized scope. For free-form text responses, this control does not apply; rely on scoped agent credentials instead<br>3. Least privilege agents: Run agents with minimal credentials scoped to the task | Medium |
| Audit trail | Silent exfil: Data leaves through agent tool calls (HTTP requests, file writes) rather than through the response path | Agent in private network | 1. Egress control: Restrict agent outbound network access to an allowlist<br>2. Tool-call logging: Log all tool invocations and their arguments<br>3. Alerting: Alert on unexpected egress destinations or high-volume data tool calls | High |
| Client trust | Response injection: Agent response contains executable content (scripts, links) that the client renders unsafely | Client-side rendering controls | 1. Content typing: Gateway sets response content type to plain text / structured JSON<br>2. Sanitization: Strip HTML/script tags from natural-language responses<br>3. Client hardening: Client apps treat agent responses as untrusted content (no eval, no innerHTML) | Low |
