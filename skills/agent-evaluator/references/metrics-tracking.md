# Metrics Tracking

Reference guide for defining, collecting, and interpreting evaluation metrics. Covers quantitative and qualitative metrics, baseline comparison, tracking over time, and how to handle cases where metrics disagree.

---

## Quantitative Metrics

These are measurable per run and comparable across versions. Collect them automatically via the eval harness.

| Metric | Definition | How to collect | Interpretation |
|---|---|---|---|
| **Accuracy / success rate** | % of tasks where `verify_outcome` returns true | Automated verifier per task | Core signal. If this is below 80%, other metrics are secondary. |
| **Runtime** | Wall-clock time from prompt submission to final response | `time.time()` before/after the agentic loop | User experience and cost. Spikes on specific tasks indicate inefficiency. |
| **Tokens consumed** | Total input + output tokens per task | `msg.usage` fields in the SDK response stream | Direct cost proxy. Compare across versions to catch regressions. |
| **Tool errors** | Count of tool calls that returned an error | Hook on PostToolUse, check for error type | High error counts indicate bad parameters, missing data, or fragile tools. |
| **Tool call count** | Total number of tool invocations per task | Increment counter on each tool_use message | Efficiency signal. High counts often indicate redundant calls or poor descriptions. |
| **Workflow completion rate** | % of tasks that reach the final step | Track a completion flag set at the last step | Distinguishes partial failures from complete failures. |

### How to Use the Table

Report all six metrics per run, not just accuracy. A change that improves accuracy while tripling token consumption may not be worth shipping. A change that keeps accuracy flat but halves tool call count is probably a win.

Track metrics in a results log (see template below). Compare every new run against the baseline and the previous run.

---

## Qualitative Metrics

These require human review but catch signal that numbers miss. Review qualitative metrics on a sample of runs — typically 10-20% of the eval suite, weighted toward failures and edge cases.

### User Corrections Needed

Review transcripts and flag any case where the agent's output required manual correction or clarification before it was usable. This captures errors that pass automated verification — for example, an email that was sent to the right person but with incorrect content.

Scoring: rate each run as `no correction needed`, `minor correction`, or `substantial correction / redo`.

### Consistency Across Runs

Run the same prompt 3-5 times without changing anything. Do you get the same outcome? The same tool call sequence? The same token count?

High consistency means the agent has a reliable, deterministic approach. Low consistency means ambiguity in the task description, the tool descriptions, or the prompt.

If two runs differ significantly on tool call sequence but both produce correct results, that is acceptable. If they produce different final results, that is a reliability problem.

### First-Try Success Rate

Could a new user — someone unfamiliar with the system internals — give a realistic prompt and succeed on the first try without needing to know tool names, IDs, or special syntax?

This is the hardest qualitative metric to achieve and the most important for real-world adoption.

### Reasoning Quality

Read the chain-of-thought, not just the final output. A correct result produced by bad reasoning is brittle — it will fail when conditions change slightly.

Look for: does the agent understand why it is doing each step? Does it identify and respond to failures appropriately? Does it make sensible decisions when constraints conflict?

---

## Baseline Comparison Framework

A baseline is a run of the eval suite on the current system, before any change is made. Every improvement claim must be compared against a baseline.

### Why Baselines Matter

Without a baseline, you cannot answer: "Is this better?" You can only say "this is what the new version does."

A baseline also reveals the current state of the system. If you have never run evals before, the baseline run is often the most valuable thing you produce — it shows where the agent already struggles.

### How to Establish a Baseline

1. Freeze the current version of all tools, prompts, and skills
2. Run the full eval suite
3. Record all six quantitative metrics per task
4. Log the run in the results log with the label `baseline`

From this point forward, every run is compared against baseline.

### What to Compare

For each new run, compute the delta against baseline for each metric:

```
accuracy delta     = new_accuracy - baseline_accuracy
token delta        = new_tokens - baseline_tokens  (positive = regression)
tool_call delta    = new_tool_calls - baseline_tool_calls  (positive = regression)
runtime delta      = new_runtime - baseline_runtime  (positive = regression)
error_rate delta   = new_errors - baseline_errors  (positive = regression)
```

A change is a net win if it improves accuracy without significant regression on cost metrics, OR if it improves cost metrics without accuracy regression.

---

## When Metrics Disagree

Disagreeing metrics are common and require judgment, not just arithmetic. Here are the most frequent patterns and how to think through them.

### Accuracy Up, Tokens Way Up

The agent is succeeding more often, but consuming significantly more tokens per run.

Questions to ask:
- Is the accuracy gain large enough to justify the cost? A 2% accuracy improvement for 40% more tokens is probably not worth it.
- Is the token increase coming from a specific task type? If token spikes are isolated to a few tasks, there may be a targeted fix.
- Is the agent fetching more data than it needs? High token counts often trace back to a list-vs-search issue or over-fetching in a context bundle.

**Recommendation**: dig into the high-token transcripts before accepting the trade-off.

### Accuracy Flat, Tool Calls Down

The agent is completing tasks with fewer tool calls, but accuracy has not changed.

This is usually a win — same outcome, lower cost, lower latency. But verify that:
- The reduction is not due to skipping necessary steps (review a sample of transcripts)
- Negative case and edge case handling has not degraded

**Recommendation**: accept the change if transcripts look clean and qualitative metrics hold.

### Positive Cases Improve, Negative Cases Degrade

A common failure mode when optimizing for the happy path. The agent gets better at tasks it was already doing, but worse at gracefully handling failures.

This is a real regression. Negative case degradation means users will hit hard failures in production instead of helpful error messages.

**Recommendation**: do not ship. Fix the negative case handling before accepting the change.

### High Variance Across Runs

Metrics fluctuate significantly between runs of the same eval suite without any changes.

This usually indicates non-determinism in the agent's behavior — typically from ambiguous tool descriptions or task prompts. High variance makes it impossible to measure improvements reliably.

**Recommendation**: before continuing iteration, run the baseline 3 times and compute variance. If variance is high, stabilize the agent's behavior first.

---

## Results Log Template

Keep a results log across all eval runs. Review it after every iteration to spot trends.

```
Date:        YYYY-MM-DD
Version:     [tag or description of what changed]
Change:      [one sentence: what was changed and why]

Metrics (averaged across eval suite):
  accuracy:            XX%     (baseline: XX%  |  delta: +/- XX%)
  tokens per task:     XXXX    (baseline: XXXX |  delta: +/- XX%)
  tool calls per task: X.X     (baseline: X.X  |  delta: +/- X.X)
  runtime per task:    Xs      (baseline: Xs   |  delta: +/- Xs)
  tool error rate:     X%      (baseline: X%   |  delta: +/- X%)
  completion rate:     XX%     (baseline: XX%  |  delta: +/- XX%)

Qualitative notes:
  [Key findings from transcript review]
  [Any regressions on qualitative metrics]
  [Notable behaviors not captured by quantitative metrics]

Decision: [Ship / Iterate / Revert — and why]
```

---

## Avoiding Over-Strict Verifiers

A verifier that rejects correct responses is one of the most common causes of false failure signals.

### What Over-Strict Looks Like

- Exact string match on agent-generated text: "Summary not found" vs "No summary was produced" — both correct, one fails
- Checking for a specific tool call sequence when multiple sequences produce the correct result
- Rejecting responses with additional correct information: the agent answered the question and provided useful context, but the verifier only expected the raw answer
- Checking capitalization, punctuation, or phrasing in outputs where these do not affect correctness

### How to Write Better Verifiers

1. **Check for semantic correctness, not exact match.** Did the agent find the right contact? Did the event get created with the right participants? Did the output contain the required information, regardless of phrasing?

2. **Use partial match for text outputs.** Check that the response contains required entities (names, dates, IDs) rather than matching the full text.

3. **Separate factual correctness from format.** If format matters, check it separately and score it separately.

4. **Test your verifiers.** Write 3-5 valid responses that express the same outcome differently and confirm your verifier accepts all of them. Write 3-5 clearly wrong responses and confirm your verifier rejects all of them.

If your verifier is causing confusion, describe the problem to Claude and ask for a less brittle implementation. Claude is effective at suggesting verification logic that is correct without being over-specified.
