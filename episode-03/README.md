# Episode 03: Human-in-the-Loop with kagent

AI agents that can take action are powerful — but you don't always want them acting without your say-so. This tutorial walks you through building a Kubernetes-native AI agent that **pauses and asks for your approval** before doing anything destructive.

## What You'll Build

By the end of this tutorial, you'll have an agent running on a local Kind cluster that:

- **Reads files** freely (no approval needed)
- **Pauses for your approval** before writing or deleting files
- **Asks you questions** when it needs more information

---

## Prerequisites

Make sure you have these installed before starting:

| Tool | Install Link |
|------|-------------|
| Docker | [get-docker](https://docs.docker.com/get-docker/) |
| Kind | [quick-start](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) |
| kubectl | [install-tools](https://kubernetes.io/docs/tasks/tools/) |
| Helm | [install](https://helm.sh/docs/intro/install/) |

You'll also need an **OpenAI API key**.

---

## Step 1 — Create a Kind Cluster

Spin up a local Kubernetes cluster:

```bash
kind create cluster --name kagent-hitl
```

Verify it's running:

```bash
kubectl cluster-info --context kind-kagent-hitl
```

You should see the control plane address printed. If so, you're good to go.

---

## Step 2 — Install kagent

There are two Helm charts to install: the CRDs first, then kagent itself.

**Install the CRDs:**

```bash
helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
    --namespace kagent \
    --create-namespace
```

**Set your API key:**

```bash
export OPENAI_API_KEY="your-api-key-here"
```

**Install kagent:**

```bash
helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
    --namespace kagent \
    --set providers.default=openAI \
    --set providers.openAI.apiKey=$OPENAI_API_KEY
```

**Wait for everything to come up:**

```bash
kubectl wait --for=condition=ready pod --all -n kagent --timeout=120s
```

Once all pods report `condition met`, move on.

---

## Step 3 — Deploy the Agent

This is the core of the tutorial. You'll create an agent with access to filesystem tools, but with **approval gates** on the dangerous ones.

Save this as `hitl-agent.yaml`:

```yaml
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: hitl-agent
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

Create the agent:

```bash
kubectl create -f hitl-agent.yaml
```

**Here's what each tool does:**

| Tool | What Happens |
|------|-------------|
| `read_file` | Runs immediately — no approval needed |
| `write_file` | Pauses and waits for you to approve or reject |
| `delete_file` | Pauses and waits for you to approve or reject |
| `ask_user` | Built-in — the agent can ask you questions anytime |

The key line is `requireApproval` — any tool listed there will pause execution until you explicitly approve it.

---

## Step 4 — Open the UI

Port-forward the kagent dashboard:

```bash
kubectl port-forward -n kagent svc/kagent-ui 8080:8080
```

Open [http://localhost:8080](http://localhost:8080) in your browser. You should see the kagent UI with your **hitl-agent** listed.

---

## Step 5 — Try It Out

Now for the fun part. Run through these tests to see human-in-the-loop in action.

### Test 1: Approve a Write

1. Select the **hitl-agent** in the UI
2. Type: `Create a file called /tmp/hello.txt with the content "Hello from kagent"`
3. The agent tries to call `write_file` — execution **pauses**
4. You'll see **Approve / Reject** buttons appear
5. Click **Approve**
6. The agent writes the file and confirms

### Test 2: Reject a Delete

1. Type: `Delete the file /tmp/hello.txt`
2. The agent calls `delete_file` — execution **pauses**
3. Click **Reject** and enter a reason: `I want to keep this file for now`
4. The agent sees your reason and responds accordingly — it doesn't delete anything

### Test 3: Agent Asks You a Question

1. Type: `Set up a config file for my application`
2. The request is vague, so the agent calls `ask_user` to clarify
3. You'll see a question appear in the UI (e.g., "Which database should I use?")
4. Answer the question and the agent continues with your input

### Test 4: Batch Approvals

1. Type: `Create three files: /tmp/a.txt, /tmp/b.txt, and /tmp/c.txt`
2. The agent generates multiple `write_file` calls at once
3. All pending approvals appear together
4. You can approve or reject each one individually

---

## How It Works

The flow is straightforward:

```
You send a message
    → Agent decides which tool to call
        → Is the tool in requireApproval?
            → YES: Execution pauses, you see Approve/Reject in the UI
                → Approve: tool runs normally
                → Reject: agent receives your reason and adapts
            → NO: Tool runs immediately
```

The `ask_user` tool works the same way — the agent pauses, you answer, and it continues. Both use the same underlying confirmation mechanism, which keeps things simple.

---

## Cleanup

Delete the cluster when you're done:

```bash
kind delete cluster --name kagent-hitl
```

---

## Key Takeaways

- **`requireApproval`** is all you need — list the tools that need human sign-off
- **`ask_user`** is built-in on every agent — no extra config required
- **Rejection reasons** are sent back to the LLM so it can adjust its approach
- **Batch approvals** let you review multiple tool calls at once
