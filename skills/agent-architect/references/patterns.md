# Agent Architecture Patterns

Reference guide for the 6 core agentic patterns. Each entry covers: what it is, when to use it, when NOT to use it, how it flows, and generic examples.

---

## Pattern 1: Prompt Chaining

### What it is
A sequence of LLM calls where each call processes the output of the previous one. Optional programmatic gates sit between steps to validate outputs before passing them forward. If a gate fails, the chain can halt, retry the step, or branch to an error handler.

### Flow
```
Input -> [LLM Step 1] -> gate? -> [LLM Step 2] -> gate? -> [LLM Step 3] -> Output
```

### When to use it
- The task naturally decomposes into a fixed, ordered sequence of subtasks
- Earlier steps produce structured artifacts (outlines, plans, extracted data) that later steps consume
- You want to validate intermediate outputs before committing to the next step
- Accuracy matters more than latency — you can afford the extra round trips

### When NOT to use it
- The steps are not truly sequential (consider Parallelization instead)
- A single well-prompted call could handle the full task (most common case — try this first)
- The chain is so long that accumulated errors compound across steps
- You need to adapt the number or nature of steps based on the input (consider Orchestrator-Workers)

### Generic examples

**Marketing content pipeline**
1. Step 1: Extract key product features from a brief
2. Gate: Validate extracted features are complete
3. Step 2: Draft marketing copy using extracted features
4. Step 3: Translate copy to target language
5. Gate: Validate translation quality score

**Document drafting**
1. Step 1: Generate a detailed outline from source material
2. Gate: Human or programmatic review of outline
3. Step 2: Write each section using the approved outline
4. Step 3: Edit for tone and consistency

**Data enrichment pipeline**
1. Step 1: Extract entities from raw text
2. Gate: Check entity confidence scores
3. Step 2: Classify each extracted entity
4. Step 3: Generate structured JSON output

---

## Pattern 2: Routing

### What it is
An initial classification step examines the input and directs it to a specialized handler. Each handler is optimized for its category — it may use different tools, different prompts, or even a different model.

### Flow
```
Input -> [Classifier LLM] -> route decision -> [Handler A]
                                            -> [Handler B]
                                            -> [Handler C] -> Output
```

### When to use it
- Inputs fall into clearly distinct categories with meaningfully different handling requirements
- A single general handler produces worse results than specialized handlers
- You want to optimize cost by routing simple requests to cheaper/faster models and complex ones to more capable models
- Different input types require different tools or data sources

### When NOT to use it
- Categories are fuzzy or overlapping (classification errors will cascade)
- All inputs need the same handling — adding a routing step is pure overhead
- You have only two categories and a simple conditional in code would suffice (do not use LLM classification for this)

### Generic examples

**Customer support triage**
- Billing questions → Billing handler (with access to billing system tools)
- Technical issues → Technical handler (with access to docs and diagnostic tools)
- General questions → General handler (no specialized tools needed)
- Escalations → Human handoff handler

**Model-tier routing**
- Simple factual questions → fast, cheap model
- Complex reasoning tasks → capable, slower model
- Code generation → code-specialized model

**Content moderation**
- Safe content → pass through
- Borderline content → human review queue
- Clearly violating content → automatic rejection

---

## Pattern 3: Parallelization

### What it is
Two distinct sub-variants:

**Sectioning**: A large task is divided into independent subtasks that run concurrently. Each subtask processes a different slice of the work. Results are aggregated at the end.

**Voting**: The same task is run multiple times (often with different prompts or temperatures). Results are compared, and the best or most common result is selected. Reduces variance and catches errors that a single run might miss.

### Flow (Sectioning)
```
Input -> [Split] -> [Worker A] \
                -> [Worker B]  -> [Aggregate] -> Output
                -> [Worker C] /
```

### Flow (Voting)
```
Input -> [Attempt 1] \
      -> [Attempt 2]  -> [Arbitrate/Select] -> Output
      -> [Attempt 3] /
```

### When to use it
- Sectioning: Subtasks are genuinely independent (no data dependency between them)
- Sectioning: Latency reduction matters and tasks are I/O-bound or compute-bound
- Voting: You need higher confidence in the output and can afford the cost
- Voting: Tasks have a checkable correct answer (code, math, factual questions)
- Guardrails: One instance handles the main task while another concurrently checks for policy violations

### When NOT to use it
- Tasks are sequential and have data dependencies between steps (use Prompt Chaining)
- Voting is used without a clear arbitration strategy — you must know how to pick the winner
- Cost is a hard constraint and single-pass accuracy is acceptable
- The "parallel" tasks are fast enough that concurrency overhead outweighs the benefit

### Generic examples

**Sectioning: Large document analysis**
- Worker A: Analyze chapters 1-3
- Worker B: Analyze chapters 4-6
- Worker C: Analyze chapters 7-9
- Aggregator: Synthesize cross-chapter themes

**Sectioning: Guardrails**
- Main worker: Generate response
- Guard worker: Check response for policy violations (runs concurrently)
- Gate: Combine results — block if guard flags violation

**Voting: Code generation**
- Attempt 1, 2, 3: Generate solution independently
- Arbitrate: Run tests on each; pick the one that passes most tests

**Voting: Fact verification**
- Attempt 1, 2, 3: Answer the question independently with different prompts
- Arbitrate: Select the answer with highest agreement across attempts

---

## Pattern 4: Orchestrator-Workers

### What it is
A central LLM (the orchestrator) dynamically breaks down a complex task, delegates subtasks to worker LLMs or tools, collects results, and synthesizes the final output. The orchestrator decides what tasks to create and which worker to send them to, adapting as results come in.

Unlike Prompt Chaining (which has fixed steps), the orchestrator pattern handles tasks where the number and nature of subtasks are not known upfront.

### Flow
```
Input -> [Orchestrator]
              |
    +---------+---------+
    |         |         |
[Worker A] [Worker B] [Worker C]  (dynamically assigned)
    |         |         |
    +---------+---------+
              |
         [Orchestrator synthesizes]
              |
           Output
```

### When to use it
- The full set of subtasks cannot be determined until partway through the work
- Different subtasks require genuinely different tools, skills, or context
- The orchestrator needs to adapt its plan based on intermediate results
- Complex, open-ended tasks that span multiple systems or files

### When NOT to use it
- The subtasks are fixed and known upfront (use Prompt Chaining instead — simpler and more predictable)
- The task is actually straightforward but feels complex on the surface — always try simpler patterns first
- The orchestration logic is so simple it could be a conditional in code

### Generic examples

**Multi-file codebase refactor**
- Orchestrator: Identify all files affected by the change
- Delegates to file-updater workers: one per affected file
- Orchestrator: Review changes for consistency, request corrections if needed

**Multi-source research synthesis**
- Orchestrator: Identify relevant sources to query
- Delegates to search workers: one query per source
- Orchestrator: Synthesize findings, identify gaps, request follow-up queries

**Automated test repair**
- Orchestrator: Run test suite, identify failures
- Delegates to fix workers: one per failing test
- Orchestrator: Re-run tests, handle any cascading failures

---

## Pattern 5: Evaluator-Optimizer

### What it is
An iterative loop where one LLM generates an output, another LLM (or the same one with a different prompt) evaluates that output against defined criteria, and the feedback is used to refine the output. The loop continues until the evaluation passes or a maximum iteration count is reached.

### Flow
```
Input -> [Generator] -> Output v1
                           |
                      [Evaluator] -> Pass? -> Final Output
                           |
                         Fail
                           |
                      Feedback -> [Generator] -> Output v2 -> ...
```

### When to use it
- There are explicit, articulable quality criteria the evaluator can check against
- Iteration demonstrably improves output quality for this type of task
- The task benefits from a "second opinion" or self-critique perspective
- You have a clear stopping condition (passes evaluation OR max iterations reached)

### When NOT to use it
- You cannot clearly define what "good" looks like — the loop will not converge
- The first pass is already high quality — iteration adds cost without benefit
- The evaluation criteria are subjective and inconsistent — the evaluator will give conflicting feedback
- Latency is a hard constraint — each iteration adds a full round trip

### Generic examples

**Literary translation refinement**
- Generator: Translate passage to target language
- Evaluator: Check for idiom accuracy, cultural nuance, register consistency
- Loop: Refine until all criteria pass or 3 iterations reached

**API design review**
- Generator: Draft API schema
- Evaluator: Check for REST conventions, naming consistency, completeness
- Loop: Refine until evaluation passes

**Technical writing**
- Generator: Draft documentation section
- Evaluator: Check for accuracy, completeness, clarity for target audience
- Loop: Refine until quality bar is met

---

## Pattern 6: Autonomous Agent

### What it is
An LLM that operates in a loop, using tools to interact with its environment, observing results, and deciding the next action — all without predefined steps. The human initiates with a goal; the agent plans and executes independently.

The agent:
1. Receives a goal
2. Plans an approach (or adapts an existing plan)
3. Selects and calls a tool
4. Observes the result
5. Updates its plan
6. Repeats until the goal is met or a stopping condition is reached
7. Pauses at defined checkpoints for human feedback

### When to use it
- The task genuinely cannot be decomposed into predictable steps upfront
- The environment is dynamic and the agent must adapt to what it finds
- The task requires many tool calls with branching logic based on intermediate results
- You are willing to invest in extensive sandboxed testing and guardrails

### When NOT to use it
- The task has fixed, predictable steps — use a workflow pattern instead
- You cannot define stopping conditions and failure modes — do not deploy without these
- The tools have irreversible side effects (file deletion, API writes) and you have not validated the agent's judgment extensively
- Latency and cost are important constraints — autonomous agents are expensive

### Critical requirements before deploying
- **Sandboxed testing**: Run the agent on representative inputs in a safe environment before production
- **Stopping conditions**: Define clearly what "done" looks like AND what failure looks like
- **Human checkpoints**: At minimum, require human approval before any irreversible action
- **Tool minimization**: Only give the agent the tools it actually needs
- **Audit logging**: Log every tool call and result for debugging and review
- **Cost controls**: Set hard limits on token consumption per run

### Generic examples

**Autonomous debugging agent**
- Goal: Fix all failing tests in the project
- Tools: Read, Edit, Bash (run tests)
- Loop: Run tests → identify failures → read relevant code → apply fix → re-run → repeat
- Checkpoint: Pause before committing changes

**Research agent**
- Goal: Gather information on topic X from multiple sources
- Tools: Web search, URL fetch, Write (save notes)
- Loop: Search → read results → identify gaps → search again → synthesize
- Checkpoint: Pause when synthesis draft is ready for human review
