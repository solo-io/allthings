# Episode 03: Human-in-the-Loop with kagent

AI agents that can take action are powerful — but you don't always want them acting without your say-so. This tutorial walks you through building a Kubernetes-native AI agent that **pauses and asks for your approval** before doing anything destructive.

## What You'll Build

By the end of this tutorial, you'll have an agent running on a local Kind cluster that:

- **Reads cluster resources** freely (no approval needed)
- **Pauses for your approval** before creating, modifying, or deleting resources
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

This is the core of the tutorial. You'll create an agent that uses kagent's **built-in Kubernetes tools**, with **approval gates** on the destructive ones.

Save this as `hitl-agent.yaml`:

```yaml
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: hitl-agent
  namespace: kagent
spec:
  modelConfig: default-model-config
  systemMessage: |
    You are a Kubernetes management agent. You help users inspect and manage
    resources in the cluster. Before making any changes, explain what you
    plan to do. If the user's request is ambiguous, use the ask_user tool
    to clarify before proceeding.
  tools:
    - builtin:
        name: kagent.tools.k8s.GetResources
      type: Builtin
    - builtin:
        name: kagent.tools.k8s.DescribeResource
      type: Builtin
    - builtin:
        name: kagent.tools.k8s.GetPodLogs
      type: Builtin
    - builtin:
        name: kagent.tools.k8s.GetEvents
      type: Builtin
    - builtin:
        name: kagent.tools.k8s.GetResourceYAML
      type: Builtin
    - builtin:
        name: kagent.tools.k8s.ApplyManifest
      type: Builtin
    - builtin:
        name: kagent.tools.k8s.DeleteResource
      type: Builtin
    - builtin:
        name: kagent.tools.k8s.PatchResource
      type: Builtin
  requireApproval:
    - kagent.tools.k8s.ApplyManifest
    - kagent.tools.k8s.DeleteResource
    - kagent.tools.k8s.PatchResource
```

Create the agent:

```bash
kubectl create -f hitl-agent.yaml
```

**Here's what each tool does:**

| Tool | What Happens |
|------|-------------|
| `GetResources` | Runs immediately — lists pods, services, deployments, etc. |
| `DescribeResource` | Runs immediately — shows resource details |
| `GetPodLogs` | Runs immediately — reads pod logs |
| `GetEvents` | Runs immediately — shows cluster events |
| `GetResourceYAML` | Runs immediately — exports resource YAML |
| `ApplyManifest` | **Pauses for approval** — creates or updates resources |
| `DeleteResource` | **Pauses for approval** — deletes resources |
| `PatchResource` | **Pauses for approval** — modifies resources |
| `ask_user` | Built-in on every agent — asks you questions anytime |

The key is `requireApproval` — any tool listed there will pause execution until you explicitly approve it. Read-only tools run freely; write operations need your sign-off.

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

### Test 1: Read Without Approval

1. Select the **hitl-agent** in the UI
2. Type: `List all pods in the kagent namespace`
3. The agent calls `GetResources` — it runs **immediately** with no approval prompt
4. You see the pod listing right away

This shows that read-only tools are not gated.

### Test 2: Approve a Create

1. Type: `Create a ConfigMap called test-config in the default namespace with the key message set to "hello from kagent"`
2. The agent calls `ApplyManifest` — execution **pauses**
3. You'll see **Approve / Reject** buttons appear, along with the YAML it wants to apply
4. Click **Approve**
5. The agent creates the ConfigMap and confirms

### Test 3: Reject a Delete

1. Type: `Delete the ConfigMap test-config in the default namespace`
2. The agent calls `DeleteResource` — execution **pauses**
3. Click **Reject** and enter a reason: `I want to keep this ConfigMap for now`
4. The agent sees your reason and responds accordingly — it doesn't delete anything

### Test 4: Agent Asks You a Question

1. Type: `Set up a namespace for my application`
2. The request is vague, so the agent calls `ask_user` to clarify (e.g., "What should the namespace be called?")
3. Answer the question and the agent continues with your input

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
- **Read-only tools run freely**, write operations pause for approval
- **`ask_user`** is built-in on every agent — no extra config required
- **Rejection reasons** are sent back to the LLM so it can adjust its approach
