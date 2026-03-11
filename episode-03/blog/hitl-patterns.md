# HITL Patterns and Best Practices for kagent

Practical patterns for designing agents with human-in-the-loop controls. When to gate, when not to, and how to write system prompts that work well with approval flows.

---

## Pattern 1: The Read-Free, Write-Gated Agent

The most common pattern. All read operations run freely. All write operations require approval.

```yaml
toolNames:
  # Read (free)
  - k8s_get_resources
  - k8s_describe_resource
  - k8s_get_pod_logs
  - k8s_get_events
  - k8s_get_resource_yaml
  - k8s_get_available_api_resources
  - k8s_get_cluster_configuration
  - k8s_check_service_connectivity
  # Write (gated)
  - k8s_apply_manifest
  - k8s_delete_resource
  - k8s_patch_resource
  - k8s_create_resource
  - k8s_label_resource
  - k8s_annotate_resource
requireApproval:
  - k8s_apply_manifest
  - k8s_delete_resource
  - k8s_patch_resource
  - k8s_create_resource
  - k8s_label_resource
  - k8s_annotate_resource
```

**When to use:** General-purpose agents, team-facing agents, any agent where you want visibility into every change.

---

## Pattern 2: The Destructive-Only Gate

A lighter touch. Only operations that are hard to reverse require approval. Creates and labels run freely.

```yaml
toolNames:
  - k8s_get_resources
  - k8s_describe_resource
  - k8s_apply_manifest
  - k8s_delete_resource
  - k8s_patch_resource
  - k8s_label_resource
requireApproval:
  - k8s_delete_resource    # hard to reverse
  - k8s_patch_resource     # can break running workloads
```

**When to use:** Agents operated by experienced users who trust creates but want a safety net on deletes and patches.

---

## Pattern 3: The Clarify-First Agent

An agent designed to always ask before acting. The system prompt is key here:

```yaml
systemMessage: |
  You are a careful Kubernetes assistant. You ALWAYS follow these rules:

  1. Before ANY modification, use ask_user to confirm the exact action
  2. Present the user with the specific YAML or command you plan to execute
  3. Only proceed after the user confirms
  4. If the user's request could affect multiple resources, list them
     and ask which ones to modify

  Never assume. Always ask.
```

Combined with `requireApproval`, this creates a two-gate system:
1. The agent asks you what to do (via `ask_user`)
2. The system asks you to approve the tool call (via `requireApproval`)

**When to use:** High-stakes environments, production clusters, compliance-heavy workflows.

---

## Pattern 4: The Scoped Agent

An agent with access to only a narrow set of tools. Less is more.

```yaml
toolNames:
  - k8s_get_resources
  - k8s_get_pod_logs
  - k8s_delete_resource
requireApproval:
  - k8s_delete_resource
```

This agent can only read resources, read logs, and delete things (with approval). It can't create, patch, or modify. The small surface area makes it easier to reason about what it can and can't do.

**When to use:** Cleanup agents, log analysis agents, single-purpose automation.

---

## System Prompt Best Practices

The system prompt matters a lot for HITL agents. Here's what works:

### Tell the agent to explain before acting

```
Before making any changes, explain what you plan to do and why.
```

This ensures the user sees the agent's reasoning **before** the approval prompt. Without this, the user sees "Approve k8s_delete_resource?" with no context about why.

### Tell the agent to use ask_user

```
If the user's request is ambiguous or could be interpreted multiple ways,
use the ask_user tool to clarify before proceeding.
```

Without this, agents tend to guess. With it, they ask. The ask is almost always better than the guess.

### Tell the agent how to handle rejections

```
If a tool call is rejected, acknowledge the user's decision and ask how
they'd like to proceed instead. Do not retry the same action.
```

Without this guidance, some models will retry the same tool call after rejection, which creates a frustrating loop.

### Tell the agent what it can and can't do

```
You have access to the following tools:
- k8s_get_resources: List Kubernetes resources (no approval needed)
- k8s_delete_resource: Delete resources (requires user approval)
```

Listing the tools and their approval status in the system prompt helps the model plan its approach. It knows to batch read operations and present a plan before triggering write operations.

---

## Anti-Patterns

### Gating everything

Don't put read-only tools in `requireApproval`. If the user has to click "Approve" to list pods, they'll stop using the agent. Gate writes, not reads.

### Not gating enough

If an agent can `k8s_delete_resource` without approval, it will eventually delete the wrong thing. When in doubt, gate it.

### Vague system prompts

A system prompt that says "be helpful" gives the agent no guidance on when to ask vs. when to act. Be specific about the behavior you want around approvals and clarification.

### Ignoring rejection reasons

If your agent doesn't acknowledge rejection reasons, users learn that rejecting is pointless and stop using the approval flow. The agent should explicitly reference the reason in its response.

---

## Mapping Tools to Risk Levels

A practical framework for deciding what to gate:

| Risk Level | Examples | Gate? |
|------------|----------|-------|
| **None** | `k8s_get_resources`, `k8s_describe_resource`, `k8s_get_pod_logs` | No |
| **Low** | `k8s_label_resource`, `k8s_annotate_resource` | Optional |
| **Medium** | `k8s_apply_manifest`, `k8s_create_resource`, `k8s_patch_resource` | Yes |
| **High** | `k8s_delete_resource`, `k8s_execute_command` | Always |

Start with gating medium and high. Add low-risk gates only if your environment requires it.

---

## Testing Your HITL Configuration

Before giving an agent to your team, test these scenarios:

1. **Approval flow** — Does the agent pause? Can you approve and see it succeed?
2. **Rejection flow** — Does the agent handle rejection gracefully? Does it reference your reason?
3. **ask_user flow** — Does the agent ask when things are ambiguous? Does it use your answer correctly?
4. **Edge case: multiple tool calls** — If the agent wants to call multiple gated tools, do all approvals appear? Can you approve some and reject others?
5. **Edge case: rejection then retry** — After rejection, does the agent try a different approach or loop on the same action?

If all five pass, your HITL configuration is solid.
