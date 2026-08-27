# Agent SDK Eval

Reference guide for running evaluations programmatically using the Agent SDK. Covers the basic eval harness, hook-based metric collection, multi-step session evals, batch running, and TypeScript equivalents for all patterns.

---

## Basic Eval Harness (Python)

The core pattern: one agentic loop per task, metric capture from the message stream, outcome verification at the end.

```python
import asyncio
import time
from claude_agent_sdk import query, ClaudeAgentOptions

async def run_eval(task_prompt, expected_outcome, allowed_tools):
    metrics = {"tokens": 0, "tool_calls": 0, "errors": 0, "duration": 0}
    start = time.time()

    result = None
    async for msg in query(
        prompt=task_prompt,
        options=ClaudeAgentOptions(allowed_tools=allowed_tools)
    ):
        if hasattr(msg, "usage"):
            metrics["tokens"] += msg.usage.get("output_tokens", 0)
        if hasattr(msg, "type") and msg.type == "tool_use":
            metrics["tool_calls"] += 1
        if hasattr(msg, "is_error") and msg.is_error:
            metrics["errors"] += 1
        if hasattr(msg, "result"):
            result = msg.result

    metrics["duration"] = time.time() - start
    metrics["success"] = verify_outcome(result, expected_outcome)
    return metrics
```

### Basic Eval Harness (TypeScript)

```typescript
import { query, ClaudeAgentOptions } from "claude-agent-sdk";

interface EvalMetrics {
  tokens: number;
  toolCalls: number;
  errors: number;
  duration: number;
  success: boolean;
}

async function runEval(
  taskPrompt: string,
  expectedOutcome: unknown,
  allowedTools: string[]
): Promise<EvalMetrics> {
  const metrics: EvalMetrics = {
    tokens: 0, toolCalls: 0, errors: 0, duration: 0, success: false
  };
  const start = Date.now();

  let result: unknown = null;
  for await (const msg of query(taskPrompt, {
    options: new ClaudeAgentOptions({ allowedTools })
  })) {
    if ("usage" in msg) {
      metrics.tokens += (msg.usage as any).output_tokens ?? 0;
    }
    if ("type" in msg && msg.type === "tool_use") {
      metrics.toolCalls += 1;
    }
    if ("isError" in msg && msg.isError) {
      metrics.errors += 1;
    }
    if ("result" in msg) {
      result = msg.result;
    }
  }

  metrics.duration = (Date.now() - start) / 1000;
  metrics.success = verifyOutcome(result, expectedOutcome);
  return metrics;
}
```

---

## Hooks for Tool-Level Metrics

Use the SDK's hook system to capture structured data on every tool call without modifying the agent's behavior. This gives you per-tool call counts, timing, and error rates.

### Python

```python
import time
from claude_agent_sdk import ClaudeAgentOptions, HookMatcher

tool_call_log = []

async def track_tool_usage(input_data, tool_use_id, context):
    tool_name = input_data.get("tool_name", "unknown")
    entry = {
        "tool": tool_name,
        "tool_use_id": tool_use_id,
        "timestamp": time.time(),
        "input": input_data,
    }
    tool_call_log.append(entry)
    return {}

async def track_tool_errors(input_data, output_data, tool_use_id, context):
    if output_data.get("is_error"):
        log_metric("tool_error", {
            "tool": input_data.get("tool_name"),
            "tool_use_id": tool_use_id,
            "error": output_data.get("content"),
        })
    return {}

options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [HookMatcher(matcher=".*", hooks=[track_tool_usage])],
        "PostToolUse": [HookMatcher(matcher=".*", hooks=[track_tool_errors])],
    }
)
```

### TypeScript

```typescript
import { ClaudeAgentOptions, HookMatcher } from "claude-agent-sdk";

const toolCallLog: Array<{tool: string; timestamp: number; input: unknown}> = [];

async function trackToolUsage(
  inputData: Record<string, unknown>,
  toolUseId: string,
  context: unknown
): Promise<Record<string, never>> {
  toolCallLog.push({
    tool: (inputData.tool_name as string) ?? "unknown",
    timestamp: Date.now(),
    input: inputData,
  });
  return {};
}

async function trackToolErrors(
  inputData: Record<string, unknown>,
  outputData: Record<string, unknown>,
  toolUseId: string,
  context: unknown
): Promise<Record<string, never>> {
  if (outputData.is_error) {
    logMetric("tool_error", {
      tool: inputData.tool_name,
      toolUseId,
      error: outputData.content,
    });
  }
  return {};
}

const options = new ClaudeAgentOptions({
  hooks: {
    PreToolUse: [new HookMatcher({ matcher: ".*", hooks: [trackToolUsage] })],
    PostToolUse: [new HookMatcher({ matcher: ".*", hooks: [trackToolErrors] })],
  },
});
```

---

## Session-Based Eval for Multi-Step Tasks

Some eval cases span multiple turns or simulate a user conversation over time. Use session IDs to continue context across calls.

### Python

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def run_multistep_eval(steps, allowed_tools):
    session_id = None
    step_results = []

    for i, step in enumerate(steps):
        opts = ClaudeAgentOptions(
            allowed_tools=allowed_tools,
            resume=session_id if session_id else None
        )

        step_result = None
        async for msg in query(prompt=step["prompt"], options=opts):
            # Capture session ID from first step
            if i == 0 and hasattr(msg, "subtype") and msg.subtype == "init":
                session_id = msg.session_id
            if hasattr(msg, "result"):
                step_result = msg.result

        step_results.append({
            "step": step["name"],
            "result": step_result,
            "success": verify_outcome(step_result, step["expected"])
        })

    return step_results

# Usage
steps = [
    {"name": "setup",   "prompt": "Create a new project called Horizon",  "expected": {"project_created": True}},
    {"name": "add",     "prompt": "Add Alice and Bob as collaborators",    "expected": {"collaborators": ["Alice", "Bob"]}},
    {"name": "verify",  "prompt": "Confirm the project settings look right", "expected": {"settings_confirmed": True}},
]
results = asyncio.run(run_multistep_eval(steps, allowed_tools=["project_tools"]))
```

### TypeScript

```typescript
import { query, ClaudeAgentOptions } from "claude-agent-sdk";

interface EvalStep {
  name: string;
  prompt: string;
  expected: unknown;
}

async function runMultistepEval(steps: EvalStep[], allowedTools: string[]) {
  let sessionId: string | undefined;
  const stepResults = [];

  for (let i = 0; i < steps.length; i++) {
    const step = steps[i];
    const opts = new ClaudeAgentOptions({
      allowedTools,
      resume: sessionId,
    });

    let stepResult: unknown = null;
    for await (const msg of query(step.prompt, { options: opts })) {
      if (i === 0 && "subtype" in msg && msg.subtype === "init") {
        sessionId = (msg as any).session_id;
      }
      if ("result" in msg) {
        stepResult = msg.result;
      }
    }

    stepResults.push({
      step: step.name,
      result: stepResult,
      success: verifyOutcome(stepResult, step.expected),
    });
  }

  return stepResults;
}
```

---

## Batch Eval Runner

Run a full suite of eval cases sequentially and aggregate results.

### Python

```python
import asyncio
import json
from datetime import datetime

async def run_eval_suite(tasks):
    results = []
    for task in tasks:
        print(f"Running: {task['name']}")
        try:
            metrics = await run_eval(
                task_prompt=task["prompt"],
                expected_outcome=task["expected"],
                allowed_tools=task["tools"]
            )
            results.append({"task": task["name"], **metrics})
        except Exception as e:
            results.append({
                "task": task["name"],
                "success": False,
                "error": str(e),
                "tokens": 0,
                "tool_calls": 0,
                "errors": 1,
                "duration": 0,
            })
    return results

def summarize_results(results):
    total = len(results)
    passed = sum(1 for r in results if r.get("success"))
    return {
        "date": datetime.now().isoformat(),
        "total_tasks": total,
        "passed": passed,
        "accuracy": round(passed / total * 100, 1) if total else 0,
        "avg_tokens": round(sum(r.get("tokens", 0) for r in results) / total, 0) if total else 0,
        "avg_tool_calls": round(sum(r.get("tool_calls", 0) for r in results) / total, 1) if total else 0,
        "avg_runtime": round(sum(r.get("duration", 0) for r in results) / total, 2) if total else 0,
        "total_errors": sum(r.get("errors", 0) for r in results),
        "per_task": results,
    }

# Usage
tasks = [
    {
        "name": "schedule_complex_meeting",
        "prompt": "Schedule a meeting with Jane and David...",
        "expected": {"event_created": True, "attendees": ["jane@example.com", "david@example.com"]},
        "tools": ["calendar", "contacts"],
    },
]

results = asyncio.run(run_eval_suite(tasks))
summary = summarize_results(results)
print(json.dumps(summary, indent=2))
```

### TypeScript

```typescript
import { runEval } from "./eval-harness";

interface EvalTask {
  name: string;
  prompt: string;
  expected: unknown;
  tools: string[];
}

async function runEvalSuite(tasks: EvalTask[]) {
  const results = [];

  for (const task of tasks) {
    console.log(`Running: ${task.name}`);
    try {
      const metrics = await runEval(task.prompt, task.expected, task.tools);
      results.push({ task: task.name, ...metrics });
    } catch (e) {
      results.push({
        task: task.name,
        success: false,
        error: String(e),
        tokens: 0,
        toolCalls: 0,
        errors: 1,
        duration: 0,
      });
    }
  }

  return results;
}

function summarizeResults(results: ReturnType<typeof runEvalSuite> extends Promise<infer T> ? T : never) {
  const total = results.length;
  const passed = results.filter((r) => r.success).length;
  return {
    date: new Date().toISOString(),
    totalTasks: total,
    passed,
    accuracy: total ? Math.round((passed / total) * 1000) / 10 : 0,
    avgTokens: total ? Math.round(results.reduce((s, r) => s + r.tokens, 0) / total) : 0,
    avgToolCalls: total ? Math.round((results.reduce((s, r) => s + r.toolCalls, 0) / total) * 10) / 10 : 0,
    avgRuntime: total ? Math.round((results.reduce((s, r) => s + r.duration, 0) / total) * 100) / 100 : 0,
    perTask: results,
  };
}
```

---

## Verifier Patterns

A verifier checks whether the agent's result meets the expected outcome. Write verifiers that check for semantic correctness, not exact match.

### Python

```python
def verify_outcome(result, expected):
    """
    Generic verifier. Extend with domain-specific checks.
    Returns True if result satisfies expected criteria.
    """
    if result is None:
        return False

    # Check required fields are present
    if isinstance(expected, dict):
        for key, value in expected.items():
            if key not in result:
                return False
            # For lists: check all expected items are present (order-independent)
            if isinstance(value, list) and isinstance(result.get(key), list):
                if not all(item in result[key] for item in value):
                    return False
            # For booleans: exact match
            elif isinstance(value, bool):
                if result.get(key) != value:
                    return False
            # For strings: substring match (not exact — avoids over-strict failures)
            elif isinstance(value, str):
                if value.lower() not in str(result.get(key, "")).lower():
                    return False

    return True
```

### TypeScript

```typescript
function verifyOutcome(result: unknown, expected: unknown): boolean {
  if (result === null || result === undefined) return false;

  if (typeof expected === "object" && expected !== null && !Array.isArray(expected)) {
    const resultObj = result as Record<string, unknown>;
    for (const [key, value] of Object.entries(expected)) {
      if (!(key in resultObj)) return false;
      if (Array.isArray(value) && Array.isArray(resultObj[key])) {
        if (!value.every((item) => (resultObj[key] as unknown[]).includes(item))) return false;
      } else if (typeof value === "boolean") {
        if (resultObj[key] !== value) return false;
      } else if (typeof value === "string") {
        if (!String(resultObj[key] ?? "").toLowerCase().includes(value.toLowerCase())) return false;
      }
    }
  }

  return true;
}
```

---

## Running Evals in CI

To automate eval runs on every code change, add a script to your CI pipeline that:

1. Runs the eval suite against a test environment (not production)
2. Compares results against the stored baseline
3. Fails the build if accuracy drops below a defined threshold
4. Posts a summary to the PR or commit

```python
import sys

ACCURACY_THRESHOLD = 85.0  # fail if accuracy drops below this

async def main():
    results = await run_eval_suite(tasks)
    summary = summarize_results(results)

    print(f"Accuracy: {summary['accuracy']}%")
    print(f"Avg tokens: {summary['avg_tokens']}")
    print(f"Avg tool calls: {summary['avg_tool_calls']}")

    if summary["accuracy"] < ACCURACY_THRESHOLD:
        print(f"FAIL: accuracy {summary['accuracy']}% below threshold {ACCURACY_THRESHOLD}%")
        sys.exit(1)
    else:
        print("PASS")
        sys.exit(0)

asyncio.run(main())
```

Set `ACCURACY_THRESHOLD` conservatively at first (70-75%) and raise it as the agent matures. A threshold set too high will block valid changes; a threshold set too low provides no safety net.
