# Agent Architecture Decision Framework

A structured approach to picking the right architecture. Always start at the top of the tree and only move down when there is a clear, concrete reason to add complexity.

---

## Decision Tree

```
Is a single LLM call enough?
├── YES  →  Use an optimized prompt with clear instructions and good examples.
│           Stop here. Do not add steps.
│
├── MAYBE  →  Try the single-call approach first.
│             Run it on real inputs. Measure quality.
│             Add complexity only if it demonstrably fails.
│
└── NO  →  Does the task decompose into a fixed set of known steps?
    │
    ├── YES  →  Are those steps independent of each other?
    │   │       (No step needs the output of another to start)
    │   │
    │   ├── YES  →  PARALLELIZATION
    │   │           Run steps concurrently. Aggregate results.
    │   │           Sub-variant: Sectioning (different subtasks) or Voting (same task, N attempts)
    │   │
    │   └── NO  →   PROMPT CHAINING
    │               Run steps sequentially. Each step processes prior output.
    │               Add validation gates between steps where errors are likely.
    │
    ├── PARTIALLY  →  Is there a natural classification point at the start?
    │   │             (Can you categorize the input and handle each category differently?)
    │   │
    │   ├── YES  →  ROUTING
    │   │           Classify first, then dispatch to a specialized handler per category.
    │   │
    │   └── NO  →   Does output quality improve meaningfully with iteration?
    │               (Is there a clear quality bar you can define and check?)
    │
    │               ├── YES  →  EVALUATOR-OPTIMIZER
    │               │           Generate → evaluate against criteria → refine → repeat.
    │               │           Requires: explicit criteria, clear stopping condition.
    │               │
    │               └── NO  →   ORCHESTRATOR-WORKERS
    │                           Central LLM dynamically plans and delegates.
    │                           Use when neither chaining nor routing can handle the variability.
    │
    └── NO (steps genuinely cannot be predicted upfront)
        │
        └──  AUTONOMOUS AGENT
             LLM operates in a tool-use loop with environmental feedback.
             Use only when: task is truly open-ended, tools have been tested extensively,
             stopping conditions are defined, human checkpoints are in place.
```

---

## Cost and Latency Trade-offs

| Pattern | Relative Cost | Relative Latency | Notes |
|---|---|---|---|
| Single LLM call | Lowest | Lowest | Baseline. Always try this first. |
| Prompt Chaining | Low-Medium | Medium | Each step adds one round trip. |
| Routing | Low | Low-Medium | Adds one classifier call overhead. |
| Parallelization (Sectioning) | Medium-High | Low | Cost scales with workers; latency stays flat. |
| Parallelization (Voting) | High | Low-Medium | N × single-call cost; latency is one iteration. |
| Orchestrator-Workers | High | Medium-High | Dynamic overhead; unpredictable call count. |
| Evaluator-Optimizer | High | High | Each iteration adds full round trip. Cap iterations. |
| Autonomous Agent | Highest | Highest | Unbounded by default; must set hard limits. |

---

## Start Simple: Guardrails

Apply these checks before recommending any multi-step architecture.

**Guardrail 1: Can a better prompt solve this?**
Before adding steps, try improving the single-call prompt. Add examples (few-shot), clarify the output format, or break the instructions into clearer sections. Most "I need a pipeline" problems are actually "I need a better prompt" problems.

**Guardrail 2: Is the complexity coming from the task or from the implementation?**
Sometimes a task seems complex because the current prompt is vague. Clarify the requirements first. A clear requirement often maps cleanly to a single LLM call.

**Guardrail 3: What is the cost of being wrong?**
Each layer of complexity adds a new failure mode. A 5-step pipeline can fail at any step. An autonomous agent can fail in ways a human cannot anticipate. Match the architecture complexity to the actual risk profile of the task.

**Guardrail 4: Can you explain why each step exists?**
Every step in a workflow should have a clear, concrete justification. "We might need it" is not a justification. If you cannot say exactly why a step exists and what would break without it, remove it.

**Guardrail 5: Have you tested the simpler approach?**
Do not graduate to Orchestrator-Workers or Autonomous Agents without evidence that simpler patterns failed. Run experiments. Measure output quality. Let data drive architecture decisions.

---

## When to Add Complexity

Add a step or pattern upgrade only when ALL of these are true:

1. You have tested the simpler approach on representative inputs
2. The simpler approach produces measurably insufficient results
3. You can articulate exactly what the additional step fixes
4. The cost and latency trade-off is acceptable for the use case
5. You have a plan to handle failure at the new layer of complexity

---

## When to Remove Complexity

Remove a step or downgrade a pattern when ANY of these are true:

1. Removing the step does not change output quality in your tests
2. The step was added speculatively and has never been the critical path
3. The pattern was chosen based on perceived complexity, not demonstrated need
4. Latency or cost of the current pattern is unacceptable and a simpler pattern is close in quality

---

## Framework Guidance

### Use direct API or SDK calls, not heavyweight frameworks

Most agent frameworks add abstraction layers that obscure what is actually happening. Before adopting a framework, understand what it does under the hood:
- What does it do on each "agent step"?
- How does it manage context and memory?
- What happens when a tool call fails?
- How does it handle token limits?

If you cannot answer these questions, you do not understand the system you are building. Start with direct SDK calls. Add framework abstractions only when you have a specific, demonstrated need for them.

### Understand what "agentic" actually means

An agentic system is one where an LLM makes decisions that affect the sequence and nature of subsequent operations. This is powerful but introduces:
- **Unpredictability**: The path through the system is not fixed
- **Compounding errors**: A wrong decision early cascades through later steps
- **Debugging difficulty**: Reproducing a failure requires the same input AND the same LLM decisions

These are real costs. Accept them only when the flexibility they enable is genuinely necessary.

### Trust grows incrementally

Start with read-only agents. Let them observe and report. Move to write access only after extensive validation of their judgment. Move to autonomous write-execute loops only after that. There is no shortcut to building justified trust in an agent's decision-making.

---

## Pattern Selection Summary

| If you need to... | Use... |
|---|---|
| Process one input into one output | Single LLM call |
| Do step A, then step B using A's output | Prompt Chaining |
| Handle different input types differently | Routing |
| Run independent tasks faster | Parallelization (Sectioning) |
| Reduce variance or increase confidence | Parallelization (Voting) |
| Handle tasks where steps are unknown upfront | Orchestrator-Workers |
| Iteratively improve output to a quality bar | Evaluator-Optimizer |
| Handle open-ended tasks requiring tool-use loop | Autonomous Agent |
