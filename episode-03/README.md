# Episode 03: Human-in-the-Loop with kagent

Build a Kubernetes-native AI agent with human oversight — requiring user approval before executing sensitive operations and allowing agents to ask clarifying questions mid-execution.

## What You'll Build

A Kubernetes management agent that:
- Can read, write, and delete files via an MCP filesystem server
- **Requires human approval** before write/delete operations
- **Asks the user questions** when it needs clarification
- Runs entirely on a local Kind cluster

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed and running
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) installed
- [kubectl](https://kubernetes.io/docs/tasks/tools/) installed
- [Helm](https://helm.sh/docs/intro/install/) installed
- An OpenAI API key (or another supported LLM provider)

---

## Step 1: Create a Kind Cluster

```bash
kind create cluster --name kagent-hitl
```

Verify the cluster is running:

```bash
kubectl cluster-info --context kind-kagent-hitl
```

## Step 2: Install kagent

Install the kagent CRDs:

```bash
helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
    --namespace kagent \
    --create-namespace
```

Set your OpenAI API key:

```bash
export OPENAI_API_KEY="your-api-key-here"
```

Install the kagent Helm chart:

```bash
helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
    --namespace kagent \
    --set providers.default=openAI \
    --set providers.openAI.apiKey=$OPENAI_API_KEY
```

Wait for all pods to be ready:

```bash
kubectl wait --for=condition=ready pod --all -n kagent --timeout=120s
```

## Step 3: Deploy the Human-in-the-Loop Agent

This is the core of the demo. The agent has access to filesystem tools but **requires approval** for destructive operations.

Save the following as `hitl-agent.yaml`:

```yaml
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: k8s-agent
  namespace: kagent
spec:
  type: Declarative
  declarative:
    modelConfig: openai-gpt4
    systemMessage: |
      You are a Kubernetes management agent. You help users manage files
      and resources. Before making any changes, explain what you plan to do.
      If the user's request is ambiguous, use the ask_user tool to clarify.
    tools:
      - type: McpServer
        mcpServer:
          name: filesystem-server
          toolNames:
            - read_file
            - write_file
            - delete_file
          requireApproval:
            - delete_file
            - write_file
```

Apply it:

```bash
kubectl apply -f hitl-agent.yaml
```

### What This Configuration Does

| Tool | Behavior |
|------|----------|
| `read_file` | Executes immediately — no approval needed |
| `write_file` | Pauses and waits for user approval |
| `delete_file` | Pauses and waits for user approval |
| `ask_user` | Built-in, always available — agent can ask you questions |

## Step 4: Access the kagent UI

Port-forward the kagent UI:

```bash
kubectl port-forward -n kagent svc/kagent-ui 8080:8080
```

Open your browser to [http://localhost:8080](http://localhost:8080).

## Step 5: Test the Human-in-the-Loop Flow

### Test 1: Tool Approval

1. Open the kagent UI and select the **k8s-agent**
2. Send a message: `Create a file called /tmp/hello.txt with the content "Hello from kagent"`
3. The agent will attempt to call `write_file` — execution **pauses**
4. You'll see **Approve / Reject** buttons in the UI
5. Click **Approve** to let the agent write the file
6. The agent confirms the file was created

### Test 2: Rejection with Reason

1. Send: `Delete the file /tmp/hello.txt`
2. The agent calls `delete_file` — execution **pauses**
3. Click **Reject**
4. Enter a reason: `"I want to keep this file for now"`
5. The agent sees your reason and responds accordingly

### Test 3: Ask User

1. Send: `Set up a config file for my application`
2. The agent doesn't know which application — it calls `ask_user`
3. You'll see an interactive question in the UI, e.g.:
   - "Which database should I use?" with choices like PostgreSQL, MySQL, SQLite
4. Select your answer and the agent continues with your input

### Test 4: Parallel Tool Approval

1. Send: `Create three files: /tmp/a.txt, /tmp/b.txt, and /tmp/c.txt`
2. The agent generates multiple `write_file` calls simultaneously
3. All pending approvals appear together in the UI
4. You can approve or reject each one individually, then submit

---

## How It Works Under the Hood

```
User --> UI --> A2A Message --> Executor --> ADK --> Tool
                                                      |
                                             request_confirmation()
                                                      |
                                             <-- input_required -->
                                                      |
User <-- UI <-- A2A Event <-- Executor <---- ADK <----'
  |
  v (Approve/Reject/Answer)
  |
User --> UI --> A2A Message --> Executor --> ADK --> Tool (resumes)
```

### Tool Approval Flow

1. Agent calls a tool marked with `requireApproval`
2. The `before_tool_callback` calls `request_confirmation()` and blocks
3. The ADK generates an `adk_request_confirmation` event
4. The UI displays Approve/Reject controls
5. Your decision is sent back as an A2A message
6. **Approved** — tool executes normally
7. **Rejected** — agent receives the rejection reason and adapts

### Ask User Flow

The `ask_user` tool reuses the same confirmation infrastructure:

1. Agent calls `ask_user` with questions and optional choices
2. Execution pauses via `request_confirmation()`
3. The UI renders interactive question cards (single-select, multi-select, or free-text)
4. Your answers are returned to the agent as structured data

### Question Types

| Type | Config | UI |
|------|--------|----|
| Single-select | `choices: [...]`, `multiple: false` | Choice chips (pick one) |
| Multi-select | `choices: [...]`, `multiple: true` | Choice chips (pick many) |
| Free-text | `choices: []` | Text input field |

---

## Cleanup

Delete the Kind cluster when you're done:

```bash
kind delete cluster --name kagent-hitl
```

---

## Key Takeaways

- **Tool Approval** gives you a safety net — sensitive tools require explicit human confirmation before executing
- **Ask User** lets agents gather information mid-execution instead of guessing
- **Parallel Approval** handles batch operations gracefully — approve or reject each tool call individually
- **Rejection Reasons** provide context back to the LLM so it can adjust its approach
- **No extra config needed** for `ask_user` — it's a built-in tool available on every agent
- Both mechanisms share the same `request_confirmation` infrastructure, keeping the architecture simple
