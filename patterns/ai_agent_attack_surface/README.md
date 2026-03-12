# The AI Agent Attack Surface: Tools, Loops, and Memory

Agents don’t just generate text. They execute tools, persist memory, and make decisions across multiple steps. Each of these capabilities introduces a new security boundary. When they combine, they create an attack surface that doesn’t exist in stateless LLM applications.

This post breaks that attack surface into three primitives: tools, loops, and memory; and threat models each one.

[**Read the full context on securepatterns.dev**](https://newsletter.securepatterns.dev/p/ai-agent-attack-surface-tools-loops-and-memory)

## System Description

An agent is a pipeline where an LLM reasons about a task, executes tool calls against external systems, stores results in memory, and loops until the task is complete or a termination condition fires. The three primitives (tools, loop, memory) are the security boundaries that don't exist in a stateless LLM wrapper.

```mermaid
flowchart LR
 %% ========= Styles =========
 classDef untrusted fill:#fdecea,stroke:#b71c1c,stroke-width:2px,color:#000;
 classDef trusted fill:#e3f2fd,stroke:#0d47a1,stroke-width:2px,color:#000;
 classDef storage fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000;

 %% ========= Left: Memory =========
 subgraph MemoryLayer ["State & Memory (Persistence)"]
 direction TB
 WorkMem[(Working Memory)]:::storage
 LongMem[(Long-term Memory)]:::storage
 end

 %% ========= Middle: Agent Core =========
 subgraph AgentCore ["Agent Core (The Loop)"]
 direction TB
 Input([User / System Input]):::untrusted
 LLM[LLM Reasoning]:::trusted
 Decide{Evaluate next step}:::trusted
 Output([Agent Output]):::untrusted

 Input -->|"1. Receive task"| LLM
 LLM -->|"6. Decide"| Decide
 Decide -->|"7a. Loop"| LLM
 Decide -->|"7b. Complete"| Output
 end

 %% ========= Right: Tools =========
 subgraph ToolLayer ["Tool Execution (Agency)"]
 direction TB
 ToolGW[Tool Gateway]:::trusted
 ToolExec([External System]):::untrusted

 ToolGW -->|"3. Execute"| ToolExec
 end

 %% ========= Invisible layout enforcers =========
 %% This forces the engine to place the columns side-by-side
 MemoryLayer ~~~ AgentCore ~~~ ToolLayer

 %% ========= Cross-boundary connections =========
 LLM -->|"5. Read / write"| WorkMem & LongMem

 LLM -->|"2. Request tool call"| ToolGW
 ToolExec -->|"4. Return result"| LLM
```

## Security Artifacts

- [Threat Model](threat_model.md): Risks across tool calls, the loop, state and memory, and cross-primitive chains
- [Verification Checklist](checklist.md): A manual test list to audit your implementation
