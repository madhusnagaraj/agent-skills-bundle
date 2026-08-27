# Agent SDK Context Management

Reference guide for context management features in the Agent SDK. Covers sessions, skill loading, CLAUDE.md, subagents, and hooks — with Python and TypeScript examples.

---

## Sessions

Sessions allow an agent to maintain and resume context across multiple queries. Instead of starting fresh each time, the agent picks up where it left off.

### Resuming a session

Use `resume` to continue a previous session. The full conversation history is preserved.

**Python:**
```python
from claude_code_sdk import query, ClaudeAgentOptions

session_id = None

# First turn — captures session_id from the response
async for msg in query(
    prompt="Analyze the authentication module and identify potential issues",
    options=ClaudeAgentOptions(allowed_tools=["Read", "Grep", "Glob"])
):
    if hasattr(msg, 'session_id'):
        session_id = msg.session_id

# Second turn — resumes with full context from the first turn
async for msg in query(
    prompt="Now fix the most critical issue you found",
    options=ClaudeAgentOptions(
        resume=session_id,
        allowed_tools=["Read", "Edit"]
    )
):
    ...
```

**TypeScript:**
```typescript
import { query, ClaudeAgentOptions } from "@anthropic-ai/claude-code";

let sessionId: string | undefined;

// First turn
for await (const msg of query({
    prompt: "Analyze the authentication module and identify potential issues",
    options: { allowedTools: ["Read", "Grep", "Glob"] }
})) {
    if (msg.type === "system" && msg.session_id) {
        sessionId = msg.session_id;
    }
}

// Second turn — resumes with full context
for await (const msg of query({
    prompt: "Now fix the most critical issue you found",
    options: {
        resume: sessionId,
        allowedTools: ["Read", "Edit"]
    }
})) {
    // handle messages
}
```

### Forking a session

Fork a session to explore an alternative approach without affecting the original. Both branches share history up to the fork point.

**Python:**
```python
# Both of these resume from the same session_id — they fork from that point
async for msg in query(
    prompt="Try fixing this with a token refresh approach",
    options=ClaudeAgentOptions(resume=session_id)
):
    ...

async for msg in query(
    prompt="Try fixing this by extending the session lifetime instead",
    options=ClaudeAgentOptions(resume=session_id)
):
    ...
```

**TypeScript:**
```typescript
// Fork 1: try one approach
for await (const msg of query({
    prompt: "Try fixing this with a token refresh approach",
    options: { resume: sessionId }
})) { ... }

// Fork 2: try a different approach from the same point
for await (const msg of query({
    prompt: "Try fixing this by extending the session lifetime instead",
    options: { resume: sessionId }
})) { ... }
```

Sessions are the primary mechanism for maintaining context in multi-turn workflows. Use them instead of manually passing conversation history between calls.

---

## Skills (SKILL.md): Progressive Disclosure in Practice

Skills demonstrate the progressive disclosure pattern directly in how they are loaded.

**Frontmatter** (always loaded): The `name` and `description` fields in the skill's YAML frontmatter are loaded when the agent starts. This is the lightweight identifier — a small, fixed cost that enables discovery.

**Body** (loaded on demand): The full skill body is loaded when the agent decides the skill is relevant to the current task. The `description` field is what drives this decision — it is a precise trigger description that tells the agent when to activate the skill.

**References** (linked, not pre-loaded): Files in the `references/` directory are linked from the skill body. They are not automatically loaded — the agent loads them selectively when it needs deeper detail on a specific topic.

This three-tier structure keeps the always-present cost low (just the frontmatter description) while making the full guidance available when relevant.

### Writing trigger descriptions that work

The `description` field should be a precise statement of the conditions under which the skill is useful — not a marketing description of the skill. Use signal phrases the user is likely to actually say.

```yaml
# Good: precise triggers
description: Optimizes information in an agent's context window for maximum effectiveness.
  Use when user says "manage context", "optimize tokens", "agent memory", "compaction",
  "context window", or is dealing with long-running agents or context limits.

# Weaker: too generic
description: Helps with AI agent optimization and memory management.
```

---

## CLAUDE.md: Project-Level Pre-Loading

`CLAUDE.md` is loaded at session start for every agent run in the project directory. It is the pre-loaded tier of the hybrid retrieval model.

### What belongs in CLAUDE.md

Content that is:
- Stable (does not change per task or per user)
- Needed on nearly every run
- Compact enough that the budget cost is justified every time

Typical contents:
- Project overview and key conventions
- Directory structure (brief, not exhaustive)
- Build and test commands
- Active constraints or decisions the agent must know about
- Links to key files the agent should be aware of

### What does not belong in CLAUDE.md

- Dynamic content that changes per task
- Large reference documents (link to them instead; load on demand)
- Content only relevant to rare workflows
- Content that belongs in the system prompt instead

### The hybrid model

CLAUDE.md implements the pre-loaded tier. The agent's tools implement the just-in-time tier. Together, they form the hybrid retrieval model: stable foundational context is always available, dynamic or large content is loaded when needed.

---

## Subagents: Clean Context Isolation

Subagents give focused agents their own clean context windows, preventing exploration from polluting the orchestrator's context.

### Basic subagent setup

**Python:**
```python
from claude_code_sdk import query, ClaudeAgentOptions, AgentDefinition

async for msg in query(
    prompt="Research how other projects handle rate limiting and summarize the best approaches",
    options=ClaudeAgentOptions(
        allowed_tools=["Read", "Glob", "Grep", "WebSearch", "Agent"],
        agents={
            "researcher": AgentDefinition(
                description="Deep research on a specific topic, returning a condensed summary",
                prompt=(
                    "Research thoroughly using all available tools. "
                    "Return a condensed summary under 2,000 tokens covering: "
                    "key findings, uncertainties, and recommended next steps. "
                    "Do not include raw source material or verbose quotes."
                ),
                tools=["Read", "Glob", "Grep", "WebSearch"]
            )
        }
    )
):
    ...
```

**TypeScript:**
```typescript
import { query, ClaudeAgentOptions, AgentDefinition } from "@anthropic-ai/claude-code";

for await (const msg of query({
    prompt: "Research how other projects handle rate limiting and summarize the best approaches",
    options: {
        allowedTools: ["Read", "Glob", "Grep", "WebSearch", "Agent"],
        agents: {
            researcher: {
                description: "Deep research on a specific topic, returning a condensed summary",
                prompt: (
                    "Research thoroughly using all available tools. " +
                    "Return a condensed summary under 2,000 tokens covering: " +
                    "key findings, uncertainties, and recommended next steps. " +
                    "Do not include raw source material or verbose quotes."
                ),
                tools: ["Read", "Glob", "Grep", "WebSearch"]
            } satisfies AgentDefinition
        }
    }
})) {
    // handle messages
}
```

### Parallel subagents

For independent research questions, launch subagents in parallel:

**Python:**
```python
import asyncio

async def run_research(question: str, session_id: str | None = None):
    results = []
    async for msg in query(
        prompt=question,
        options=ClaudeAgentOptions(
            resume=session_id,
            allowed_tools=["Read", "Glob", "Grep", "WebSearch", "Agent"],
            agents={"researcher": AgentDefinition(...)}
        )
    ):
        results.append(msg)
    return results

# Run three research tasks in parallel
questions = [
    "Research existing rate limiting patterns in this codebase",
    "Research how the Stripe SDK handles rate limiting",
    "Research exponential backoff best practices"
]

results = await asyncio.gather(*[run_research(q) for q in questions])
```

---

## Hooks for Context Management

Hooks intercept tool use events and can trigger custom logic — including context budget tracking, tool result clearing, or compaction triggers.

### Context budget tracking

Track how much context each tool call consumes and trigger compaction when a threshold is reached.

**Python:**
```python
from claude_code_sdk import query, ClaudeAgentOptions, HookMatcher

async def context_budget_tracker(tool_name: str, tool_result: dict) -> dict:
    """Track context usage and flag when budget is running low."""
    # Estimate token count of the result
    result_text = str(tool_result)
    estimated_tokens = len(result_text) // 4  # rough estimate

    # Log or record for monitoring
    print(f"Tool {tool_name} returned ~{estimated_tokens} tokens")

    # Return the result unchanged (or modify it to truncate if needed)
    return tool_result

async for msg in query(
    prompt="Analyze the codebase",
    options=ClaudeAgentOptions(
        allowed_tools=["Read", "Grep", "Glob"],
        hooks={
            "PostToolUse": [
                HookMatcher(
                    matcher="Read|Grep",
                    hooks=[context_budget_tracker]
                )
            ]
        }
    )
):
    ...
```

**TypeScript:**
```typescript
import { query, ClaudeAgentOptions, HookMatcher } from "@anthropic-ai/claude-code";

async function contextBudgetTracker(toolName: string, toolResult: unknown): Promise<unknown> {
    const resultText = JSON.stringify(toolResult);
    const estimatedTokens = Math.floor(resultText.length / 4);
    console.log(`Tool ${toolName} returned ~${estimatedTokens} tokens`);
    return toolResult;
}

for await (const msg of query({
    prompt: "Analyze the codebase",
    options: {
        allowedTools: ["Read", "Grep", "Glob"],
        hooks: {
            PostToolUse: [
                {
                    matcher: "Read|Grep",
                    hooks: [contextBudgetTracker]
                } satisfies HookMatcher
            ]
        }
    }
})) {
    // handle messages
}
```

### Automatic result truncation

Truncate large tool results before they enter the context:

**Python:**
```python
MAX_RESULT_TOKENS = 2000
MAX_CHARS = MAX_RESULT_TOKENS * 4

async def truncate_large_results(tool_name: str, tool_result: dict) -> dict:
    """Truncate tool results that exceed the token budget."""
    if "content" in tool_result:
        content = tool_result["content"]
        if len(content) > MAX_CHARS:
            tool_result["content"] = (
                content[:MAX_CHARS] +
                f"\n\n[Truncated: result was {len(content)} chars, "
                f"showing first {MAX_CHARS}. Use more specific queries to get targeted content.]"
            )
    return tool_result
```

### Compaction trigger

Trigger compaction when the estimated context usage exceeds a threshold:

**Python:**
```python
total_tokens_used = 0

async def compaction_trigger(tool_name: str, tool_result: dict) -> dict:
    global total_tokens_used
    estimated = len(str(tool_result)) // 4
    total_tokens_used += estimated

    if total_tokens_used > 50_000:  # threshold: adjust per model
        print(f"Context budget warning: ~{total_tokens_used} tokens used. Consider compaction.")

    return tool_result
```

Hooks make context management programmable. Combined with sessions (for continuity) and subagents (for isolation), they give you precise control over what occupies the context window and when.
