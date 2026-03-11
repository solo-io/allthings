# HITL Architecture Deep Dive: How kagent Pauses and Resumes Agent Execution

This document breaks down the internal architecture of Human-in-the-Loop in kagent — how tool calls get intercepted, how execution blocks, and how human decisions flow back to the agent.

---

## The Request Flow

When a user sends a message to a kagent agent, here's the full lifecycle:

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│          │     │          │     │          │     │          │     │          │
│   User   │────▶│    UI    │────▶│ A2A Msg  │────▶│ Executor │────▶│   ADK    │
│          │     │          │     │          │     │          │     │          │
└──────────┘     └──────────┘     └──────────┘     └──────────┘     └────┬─────┘
                                                                        │
                                                                        ▼
                                                                   ┌──────────┐
                                                                   │   LLM    │
                                                                   │ decides  │
                                                                   │ to call  │
                                                                   │  a tool  │
                                                                   └────┬─────┘
                                                                        │
                                                              ┌─────────▼─────────┐
                                                              │                   │
                                                              │ Is tool in        │
                                                              │ requireApproval?  │
                                                              │                   │
                                                              └─────┬───────┬─────┘
                                                                    │       │
                                                                   YES      NO
                                                                    │       │
                                                                    ▼       ▼
                                                              ┌─────────┐ ┌─────────┐
                                                              │  PAUSE  │ │  RUN    │
                                                              │  wait   │ │  tool   │
                                                              │  for    │ │ directly│
                                                              │ human   │ └─────────┘
                                                              └────┬────┘
                                                                   │
                                                    ┌──────────────▼──────────────┐
                                                    │                             │
                                                    │  UI shows Approve / Reject  │
                                                    │                             │
                                                    └──────────┬──────────────────┘
                                                               │
                                                        ┌──────▼──────┐
                                                        │             │
                                                   Approve         Reject
                                                        │             │
                                                        ▼             ▼
                                                   ┌─────────┐  ┌──────────┐
                                                   │  Tool   │  │  Agent   │
                                                   │ executes│  │ receives │
                                                   │ normally│  │ reason & │
                                                   │         │  │ adapts   │
                                                   └─────────┘  └──────────┘
```

---

## Key Components

### 1. The Agent CRD

The Agent Custom Resource is where HITL is configured. The `requireApproval` field sits inside the `mcpServer` block of each tool definition:

```yaml
spec:
  type: Declarative
  declarative:
    tools:
      - type: McpServer
        mcpServer:
          name: kagent-tool-server
          toolNames:
            - k8s_get_resources      # no approval
            - k8s_delete_resource    # approval required
          requireApproval:
            - k8s_delete_resource
```

The CRD includes a validation rule that ensures every tool in `requireApproval` also appears in `toolNames`:

```yaml
x-kubernetes-validations:
  - message: each RequireApproval entry must also appear in ToolNames
    rule: '!has(self.requireApproval) || self.requireApproval.all(x, has(self.toolNames) && x in self.toolNames)'
```

This means you get a clear error at `kubectl create` time if you mistype a tool name.

### 2. The Executor (before_tool_callback)

When the ADK (Agent Development Kit) is about to execute a tool, it calls the `before_tool_callback`. This is where kagent checks the `requireApproval` list.

If the tool requires approval:
- The callback calls `request_confirmation()`
- Execution **blocks** — the agent doesn't proceed
- An `adk_request_confirmation` event is emitted

If the tool doesn't require approval:
- The callback returns immediately
- The tool executes normally

### 3. The A2A Protocol

The confirmation event is transmitted using the A2A (Agent-to-Agent) protocol. The event includes:
- The tool name
- The arguments the agent wants to pass
- A unique confirmation ID

The UI subscribes to these events and renders the appropriate controls.

### 4. The UI Layer

When the UI receives a confirmation event, it renders:
- The tool name and its arguments (so you can see exactly what the agent wants to do)
- **Approve** button
- **Reject** button with an optional reason field

For `ask_user` events, it renders:
- The agent's question
- Input controls (text field, choice chips, or multi-select)

### 5. The Response Path

When the human makes a decision:
1. The UI sends an A2A message back to the executor
2. The executor unblocks the `request_confirmation()` call
3. The result (approved/rejected + reason) is returned to the ADK
4. The ADK passes the result back to the LLM as tool output

For approvals, the tool executes and the output is returned to the LLM.

For rejections, the rejection reason becomes the tool output. The LLM sees something like: *"Tool call rejected by user. Reason: I want to keep this resource."* It can then decide what to do next.

---

## The ask_user Flow

`ask_user` reuses the same infrastructure as tool approval. The difference is:

| Aspect | Tool Approval | ask_user |
|--------|--------------|----------|
| **Trigger** | Agent calls a gated tool | Agent explicitly calls `ask_user` |
| **UI** | Approve/Reject buttons | Question with input controls |
| **Response** | Boolean (approve/reject) + reason | Structured answer data |
| **Configuration** | Requires `requireApproval` in YAML | Always available, no config needed |

Both go through `request_confirmation()` → A2A event → UI → A2A response → unblock. The shared infrastructure means both features are reliable and consistent.

---

## Batch Approvals

When the LLM generates multiple tool calls in parallel (e.g., "create three ConfigMaps"), each tool call that requires approval generates its own confirmation event. The UI collects all pending confirmations and presents them together.

You can approve or reject each one individually. This means you can approve creating `config-a` and `config-b` but reject `config-c` — the agent handles each result independently.

---

## State Machine

An agent's execution follows this state machine when HITL is involved:

```
IDLE ──▶ PROCESSING ──▶ TOOL_CALL
                            │
                     ┌──────┴──────┐
                     │             │
              Not in list    In requireApproval
                     │             │
                     ▼             ▼
                  EXECUTE    INPUT_REQUIRED
                     │             │
                     │      ┌──────┴──────┐
                     │      │             │
                     │   Approved      Rejected
                     │      │             │
                     │      ▼             ▼
                     │   EXECUTE    TOOL_REJECTED
                     │      │             │
                     ▼      ▼             ▼
                  PROCESSING ◀────────────┘
                     │
                     ▼
                  COMPLETE
```

The `INPUT_REQUIRED` state is where the agent is blocked, waiting for human input. The agent pod stays running — it's not restarted or rescheduled. The blocking is cooperative, handled within the ADK runtime.

---

## Security Considerations

- **requireApproval is enforced server-side** — it's not a UI-only check. Even if someone bypasses the UI and sends an A2A message directly, the executor still blocks on the confirmation.
- **Rejection reasons are not executable** — they're passed as plain text context to the LLM. There's no code injection risk from rejection reasons.
- **Tool access is explicit** — an agent can only call tools listed in its `toolNames`. You can't escalate to tools that aren't assigned to the agent.
- **CRD validation is declarative** — the validation rules are baked into the CRD schema, so misconfigurations are caught at resource creation time, not at runtime.
