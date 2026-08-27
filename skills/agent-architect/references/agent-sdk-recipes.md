# Agent SDK Recipes

Working code examples for each pattern using the Claude Agent SDK. Python and TypeScript versions for each.

---

## Imports

**Python**
```python
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition, HookMatcher
```

**TypeScript**
```typescript
import { query, ClaudeAgentOptions, AgentDefinition, HookMatcher } from "@anthropic-ai/claude-agent-sdk";
```

---

## Pattern 1: Prompt Chaining (Sessions)

Chain LLM calls by resuming a session. This carries context forward without manually passing text between steps.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def chained_pipeline(document: str) -> str:
    session_id = None

    # Step 1: Extract key points
    async for msg in query(
        prompt=f"Extract the 5 most important points from this document:\n\n{document}",
        options=ClaudeAgentOptions(allowed_tools=["Read"])
    ):
        # Capture session_id from the first message to resume later
        if hasattr(msg, "subtype") and msg.subtype == "init":
            session_id = msg.session_id

    # Programmatic gate: validate session_id was captured
    if not session_id:
        raise RuntimeError("Session initialization failed")

    # Step 2: Draft a summary using the extracted points (context preserved via session)
    summary = ""
    async for msg in query(
        prompt="Now write a 2-paragraph executive summary based on those key points.",
        options=ClaudeAgentOptions(resume=session_id)
    ):
        if hasattr(msg, "content"):
            summary = msg.content

    # Step 3: Suggest a title
    async for msg in query(
        prompt="Suggest a concise, professional title for this summary.",
        options=ClaudeAgentOptions(resume=session_id)
    ):
        if hasattr(msg, "content"):
            title = msg.content

    return f"# {title}\n\n{summary}"

asyncio.run(chained_pipeline("...your document..."))
```

**TypeScript**
```typescript
import { query, ClaudeAgentOptions } from "@anthropic-ai/claude-agent-sdk";

async function chainedPipeline(document: string): Promise<string> {
    let sessionId: string | undefined;

    // Step 1: Extract key points
    for await (const msg of query(
        `Extract the 5 most important points from this document:\n\n${document}`,
        { allowedTools: ["Read"] } as ClaudeAgentOptions
    )) {
        if (msg.subtype === "init") {
            sessionId = msg.sessionId;
        }
    }

    if (!sessionId) throw new Error("Session initialization failed");

    // Step 2: Draft summary (context preserved via session)
    let summary = "";
    for await (const msg of query(
        "Now write a 2-paragraph executive summary based on those key points.",
        { resume: sessionId } as ClaudeAgentOptions
    )) {
        if (msg.content) summary = msg.content;
    }

    return summary;
}
```

---

## Pattern 2: Routing (Classifier + Specialized Handlers)

Use subagents as specialized handlers. The orchestrator classifies the input and delegates to the right agent.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async def routed_handler(user_request: str) -> str:
    result = ""

    async for msg in query(
        prompt=(
            f"Classify this request and delegate to the appropriate specialist:\n\n"
            f"{user_request}\n\n"
            "If it's a bug fix request, use the bug-fixer agent. "
            "If it's a new feature request, use the feature-builder agent. "
            "If it's a documentation request, use the doc-writer agent."
        ),
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep", "Agent"],
            agents={
                "bug-fixer": AgentDefinition(
                    description="Diagnoses and fixes bugs in existing code",
                    prompt="Find the root cause of the bug and apply a minimal, correct fix.",
                    tools=["Read", "Edit", "Bash"]
                ),
                "feature-builder": AgentDefinition(
                    description="Implements new features based on requirements",
                    prompt="Implement the requested feature following existing code conventions.",
                    tools=["Read", "Write", "Edit", "Bash"]
                ),
                "doc-writer": AgentDefinition(
                    description="Writes and updates technical documentation",
                    prompt="Write clear, accurate documentation for the specified component.",
                    tools=["Read", "Write", "Edit"]
                ),
            }
        )
    ):
        if hasattr(msg, "content"):
            result = msg.content

    return result

asyncio.run(routed_handler("The login function throws a TypeError when the password field is empty"))
```

**TypeScript**
```typescript
import { query, ClaudeAgentOptions, AgentDefinition } from "@anthropic-ai/claude-agent-sdk";

async function routedHandler(userRequest: string): Promise<string> {
    let result = "";

    for await (const msg of query(
        `Classify this request and delegate to the appropriate specialist:\n\n${userRequest}`,
        {
            allowedTools: ["Read", "Glob", "Grep", "Agent"],
            agents: {
                "bug-fixer": {
                    description: "Diagnoses and fixes bugs in existing code",
                    prompt: "Find the root cause of the bug and apply a minimal, correct fix.",
                    tools: ["Read", "Edit", "Bash"]
                } as AgentDefinition,
                "feature-builder": {
                    description: "Implements new features based on requirements",
                    prompt: "Implement the requested feature following existing code conventions.",
                    tools: ["Read", "Write", "Edit", "Bash"]
                } as AgentDefinition,
            }
        } as ClaudeAgentOptions
    )) {
        if (msg.content) result = msg.content;
    }

    return result;
}
```

---

## Pattern 3: Parallelization (Multiple Subagents)

Define multiple specialized worker agents and let the orchestrator dispatch to them concurrently.

**Python — Sectioning (independent subtasks)**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async def parallel_review(file_path: str) -> str:
    report = ""

    async for msg in query(
        prompt=(
            f"Perform a comprehensive review of {file_path}. "
            "Use the security-reviewer, performance-reviewer, and style-reviewer agents "
            "to analyze the file concurrently, then synthesize their findings into a single report."
        ),
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep", "Agent"],
            agents={
                "security-reviewer": AgentDefinition(
                    description="Reviews code for security vulnerabilities",
                    prompt=(
                        "Analyze the given file for security issues: injection vulnerabilities, "
                        "insecure dependencies, exposed secrets, improper input validation. "
                        "Return a structured list of findings with severity levels."
                    ),
                    tools=["Read", "Grep"]
                ),
                "performance-reviewer": AgentDefinition(
                    description="Reviews code for performance issues",
                    prompt=(
                        "Analyze the given file for performance issues: unnecessary iterations, "
                        "blocking I/O, memory leaks, inefficient data structures. "
                        "Return a structured list of findings with estimated impact."
                    ),
                    tools=["Read", "Grep"]
                ),
                "style-reviewer": AgentDefinition(
                    description="Reviews code for style and maintainability",
                    prompt=(
                        "Analyze the given file for style and maintainability issues: naming conventions, "
                        "function complexity, missing documentation, code duplication. "
                        "Return a structured list of suggestions."
                    ),
                    tools=["Read", "Grep"]
                ),
            }
        )
    ):
        if hasattr(msg, "content"):
            report = msg.content

    return report

asyncio.run(parallel_review("src/auth/login.py"))
```

**Python — Voting (multiple attempts, pick best)**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async def voting_generation(requirement: str) -> str:
    best_result = ""

    async for msg in query(
        prompt=(
            f"Generate three independent solutions for this requirement:\n\n{requirement}\n\n"
            "Use solution-a, solution-b, and solution-c agents to generate candidates independently. "
            "Then evaluate all three against correctness, readability, and edge case handling. "
            "Return only the best solution with an explanation of why it was selected."
        ),
        options=ClaudeAgentOptions(
            allowed_tools=["Bash", "Agent"],
            agents={
                "solution-a": AgentDefinition(
                    description="Generates solution candidate A",
                    prompt="Generate a complete, working solution. Prioritize simplicity.",
                    tools=["Read", "Write"]
                ),
                "solution-b": AgentDefinition(
                    description="Generates solution candidate B",
                    prompt="Generate a complete, working solution. Prioritize robustness and error handling.",
                    tools=["Read", "Write"]
                ),
                "solution-c": AgentDefinition(
                    description="Generates solution candidate C",
                    prompt="Generate a complete, working solution. Prioritize performance.",
                    tools=["Read", "Write"]
                ),
            }
        )
    ):
        if hasattr(msg, "content"):
            best_result = msg.content

    return best_result
```

**TypeScript**
```typescript
import { query, ClaudeAgentOptions, AgentDefinition } from "@anthropic-ai/claude-agent-sdk";

async function parallelReview(filePath: string): Promise<string> {
    let report = "";

    for await (const msg of query(
        `Perform a comprehensive review of ${filePath} using all three reviewer agents concurrently.`,
        {
            allowedTools: ["Read", "Glob", "Grep", "Agent"],
            agents: {
                "security-reviewer": {
                    description: "Reviews code for security vulnerabilities",
                    prompt: "Analyze for injection flaws, exposed secrets, and improper validation.",
                    tools: ["Read", "Grep"]
                } as AgentDefinition,
                "performance-reviewer": {
                    description: "Reviews code for performance issues",
                    prompt: "Analyze for blocking I/O, memory issues, and inefficient patterns.",
                    tools: ["Read", "Grep"]
                } as AgentDefinition,
            }
        } as ClaudeAgentOptions
    )) {
        if (msg.content) report = msg.content;
    }

    return report;
}
```

---

## Pattern 4: Orchestrator-Workers (Dynamic Delegation)

The orchestrator decides at runtime which workers to call and what to delegate. Workers are specialized but reusable.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async def orchestrated_refactor(task_description: str) -> str:
    result = ""

    async for msg in query(
        prompt=(
            f"Complete this task by breaking it into subtasks and delegating to the appropriate workers:\n\n"
            f"{task_description}\n\n"
            "Use file-reader to understand existing code, file-editor to make changes, "
            "and test-runner to verify changes. Plan your approach first, then execute."
        ),
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Write", "Edit", "Bash", "Glob", "Grep", "Agent"],
            agents={
                "file-reader": AgentDefinition(
                    description="Reads and analyzes files to understand current state",
                    prompt=(
                        "Read the specified files and return a clear summary of their current "
                        "structure, purpose, and any relevant patterns."
                    ),
                    tools=["Read", "Glob", "Grep"]
                ),
                "file-editor": AgentDefinition(
                    description="Applies specific, described changes to a single file",
                    prompt=(
                        "Apply exactly the changes described. Do not change anything else. "
                        "Preserve existing formatting and conventions."
                    ),
                    tools=["Read", "Edit"]
                ),
                "test-runner": AgentDefinition(
                    description="Runs the test suite and reports results",
                    prompt=(
                        "Run the test suite and return a structured report: which tests passed, "
                        "which failed, and the full error output for any failures."
                    ),
                    tools=["Bash"]
                ),
            }
        )
    ):
        if hasattr(msg, "content"):
            result = msg.content

    return result

asyncio.run(orchestrated_refactor(
    "Rename the `userId` field to `user_id` across all Python files, update all tests, and verify nothing breaks."
))
```

**TypeScript**
```typescript
import { query, ClaudeAgentOptions, AgentDefinition } from "@anthropic-ai/claude-agent-sdk";

async function orchestratedRefactor(taskDescription: string): Promise<string> {
    let result = "";

    for await (const msg of query(
        `Complete this task by breaking it into subtasks and delegating to the appropriate workers:\n\n${taskDescription}`,
        {
            allowedTools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep", "Agent"],
            agents: {
                "file-reader": {
                    description: "Reads and analyzes files to understand current state",
                    prompt: "Read the specified files and return a clear summary of their structure and purpose.",
                    tools: ["Read", "Glob", "Grep"]
                } as AgentDefinition,
                "file-editor": {
                    description: "Applies specific changes to a single file",
                    prompt: "Apply exactly the changes described. Preserve existing formatting.",
                    tools: ["Read", "Edit"]
                } as AgentDefinition,
                "test-runner": {
                    description: "Runs the test suite and reports results",
                    prompt: "Run tests and return structured pass/fail report with error details.",
                    tools: ["Bash"]
                } as AgentDefinition,
            }
        } as ClaudeAgentOptions
    )) {
        if (msg.content) result = msg.content;
    }

    return result;
}
```

---

## Pattern 5: Evaluator-Optimizer (Generate-Evaluate-Refine Loop)

Use hooks to intercept tool outputs and trigger quality evaluation after each write. The loop continues until quality criteria are met.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, HookMatcher

# Quality check hook — runs after every Write tool call
async def quality_check_hook(tool_result):
    """
    Evaluate the written output against quality criteria.
    Return feedback string if quality is insufficient, empty string if it passes.
    """
    content = tool_result.get("content", "")

    # Example criteria: minimum length, required sections
    issues = []
    if len(content) < 200:
        issues.append("Output is too short — expand with more detail")
    if "## Summary" not in content:
        issues.append("Missing required ## Summary section")

    if issues:
        return f"Quality check failed:\n" + "\n".join(f"- {i}" for i in issues)
    return ""  # Pass — no feedback needed

async def evaluator_optimizer(prompt: str, output_path: str) -> str:
    result = ""

    async for msg in query(
        prompt=(
            f"{prompt}\n\n"
            f"Write your output to {output_path}. "
            "After writing, review it against these criteria: "
            "at least 200 words, includes a ## Summary section, "
            "and covers all key points. Revise until all criteria are met."
        ),
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Write", "Edit"],
            hooks={
                "PostToolUse": [
                    HookMatcher(
                        matcher="Write",
                        hooks=[quality_check_hook]
                    )
                ]
            }
        )
    ):
        if hasattr(msg, "content"):
            result = msg.content

    return result

asyncio.run(evaluator_optimizer(
    "Write a technical overview of the authentication module",
    "docs/auth-overview.md"
))
```

**TypeScript**
```typescript
import { query, ClaudeAgentOptions, HookMatcher } from "@anthropic-ai/claude-agent-sdk";

// Quality check hook — runs after every Write tool call
async function qualityCheckHook(toolResult: Record<string, unknown>): Promise<string> {
    const content = (toolResult.content as string) ?? "";
    const issues: string[] = [];

    if (content.length < 200) issues.push("Output is too short — expand with more detail");
    if (!content.includes("## Summary")) issues.push("Missing required ## Summary section");

    return issues.length > 0
        ? `Quality check failed:\n${issues.map(i => `- ${i}`).join("\n")}`
        : "";
}

async function evaluatorOptimizer(prompt: string, outputPath: string): Promise<string> {
    let result = "";

    for await (const msg of query(
        `${prompt}\n\nWrite your output to ${outputPath}. Revise until all quality criteria are met.`,
        {
            allowedTools: ["Read", "Write", "Edit"],
            hooks: {
                PostToolUse: [
                    {
                        matcher: "Write",
                        hooks: [qualityCheckHook]
                    } as HookMatcher
                ]
            }
        } as ClaudeAgentOptions
    )) {
        if (msg.content) result = msg.content;
    }

    return result;
}
```

---

## Pattern 6: Autonomous Agent (Full Tool Loop)

Minimal setup — give the agent tools and a goal. Use `permission_mode` to control how much autonomy it has.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def autonomous_agent(goal: str) -> None:
    """
    Runs an autonomous agent with full read/write/execute capabilities.
    Uses acceptEdits mode — agent can write files without per-edit confirmation.
    Always run in a sandboxed environment first.
    """
    async for msg in query(
        prompt=(
            f"{goal}\n\n"
            "Work through this systematically. Before making any irreversible change, "
            "state what you are about to do and why. "
            "Stop and summarize when you have completed the goal or reached a decision point "
            "that requires human input."
        ),
        options=ClaudeAgentOptions(
            permission_mode="acceptEdits",
            allowed_tools=["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
        )
    ):
        # Stream progress messages to the user
        if hasattr(msg, "content") and msg.content:
            print(msg.content)

asyncio.run(autonomous_agent("Find all failing tests in this project and fix them."))
```

**Python — with human checkpoint**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def autonomous_agent_with_checkpoint(goal: str) -> None:
    """
    Autonomous agent that pauses for human approval before committing changes.
    Uses default permission_mode (requires confirmation for edits).
    """
    async for msg in query(
        prompt=(
            f"{goal}\n\n"
            "Plan your approach first. Execute read-only operations to gather information. "
            "Before writing or modifying any file, describe the change and wait for approval."
        ),
        options=ClaudeAgentOptions(
            # Default permission_mode — prompts for confirmation on writes
            allowed_tools=["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
        )
    ):
        if hasattr(msg, "content") and msg.content:
            print(msg.content)
```

**TypeScript**
```typescript
import { query, ClaudeAgentOptions } from "@anthropic-ai/claude-agent-sdk";

async function autonomousAgent(goal: string): Promise<void> {
    for await (const msg of query(
        `${goal}\n\nWork systematically. Stop and summarize when complete or when human input is needed.`,
        {
            permissionMode: "acceptEdits",
            allowedTools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
        } as ClaudeAgentOptions
    )) {
        if (msg.content) {
            console.log(msg.content);
        }
    }
}

autonomousAgent("Find all failing tests in this project and fix them.");
```

---

## Choosing the Right SDK Features

| Pattern | Key SDK Feature | Why |
|---|---|---|
| Prompt Chaining | `resume: session_id` | Preserves context across sequential calls |
| Routing | `agents` dict + orchestrator prompt | Orchestrator classifies, delegates to named agents |
| Parallelization | `agents` dict + "concurrently" in prompt | Orchestrator dispatches multiple agents at once |
| Orchestrator-Workers | `agents` dict + `Agent` tool | Orchestrator dynamically selects which workers to call |
| Evaluator-Optimizer | `hooks.PostToolUse` | Intercepts writes to trigger quality evaluation |
| Autonomous Agent | `permission_mode`, full `allowed_tools` | Gives agent control it needs with appropriate guardrails |
