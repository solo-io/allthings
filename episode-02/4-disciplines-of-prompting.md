
## The Four Disciplines of “Prompting” in 2026

### 1) Prompt craft (table stakes)
This is the classic skill:

- Clear instruction
- Examples + counterexamples
- Guardrails (what to do / what not to do)
- Explicit output format
- Rules for ambiguity (“if X conflicts with Y, do Z”)

It still matters — it just doesn’t differentiate you anymore.

### 2) Context engineering (your real leverage)
Your prompt might be 200 tokens.
Your context window might be 200k–1M.

That means your “prompt” is a rounding error.

Context engineering is the work of designing the information environment the agent runs inside:

- system prompts / agent instructions
- tool definitions + permissions
- RAG sources / docs / repos
- memory (what persists across runs)
- conventions (how this org writes, builds, tests, ships)

If you see someone getting 10x more out of the same model, they usually aren’t “better at wording.”
They built better context infrastructure.

### 3) Intent engineering (what the agent should *want*)
Context tells the agent **what to know**.
Intent tells the agent **what to optimize for**.

This is where teams break things at enterprise scale:

- speed vs quality
- cost vs correctness
- customer satisfaction vs ticket closure time
- “ship it” vs “fail safe”

If you don’t encode trade-offs and escalation triggers, the agent will “pick a metric” implicitly.

### 4) Specification engineering (blueprints for autonomous work)
Specs are what you write when you can’t rely on real-time correction.

A good spec is:

- self-contained
- structured
- internally consistent
- explicit about quality measurement

If your output keeps coming back “80% correct,” you don’t have a prompt problem.
You have a spec + evaluation problem.

#### What “specification engineering” actually means
The video’s argument is simple: once agents can run for hours or days, you can’t rely on live back-and-forth to fix mistakes.

So specification engineering becomes the skill of writing **agent-executable blueprints**:

- **Self-contained**: no hidden assumptions, undefined acronyms, or missing “obvious” org context.
- **Verifiable**: “done” is defined in checks that someone else can evaluate.
- **Constrained**: must/must-not/preferences are explicit (plus what should be escalated).
- **Decomposable**: the work can be broken into chunks that can be executed and verified independently.

The broader implication: your *documents* become infrastructure. Strategy docs, product docs, runbooks, and OKRs all start acting like specs that agents (and humans) can execute.
