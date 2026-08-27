# Compaction Strategies

Detailed guide for managing context across long-running agents. Covers tool result clearing, conversation summarization, structured note-taking, and sub-agent isolation — with selection criteria and implementation guidance.

---

## Why Compaction Matters

A long-running agent accumulates context across turns: tool results, intermediate reasoning, user messages, and assistant responses. Eventually it approaches the context limit. Without a compaction strategy, the agent either crashes (hits the hard limit) or degrades gracefully — losing track of earlier content, repeating work, and producing increasingly inconsistent outputs.

Context rot — quality degradation as the window fills — begins well before the hard limit. Plan for it.

---

## Technique 1: Tool Result Clearing

**What it is**: Remove raw tool results from the conversation history once they have been acted on.

**Why it is the lowest-hanging fruit**: Tool results are often large. A single file read might produce 2,000-5,000 tokens. Search results, API responses, and shell output compound quickly. Most of this content is only needed for the immediate next response — once the agent has acted on it, the raw result adds noise without adding value.

**What to clear**:
- File reads that produced an action (the file was read, code was written — the raw content is no longer needed)
- Search results where the agent selected a result and proceeded
- API responses that were summarized in the agent's response
- Repeated reads of the same resource (keep the most recent; clear the duplicates)

**What to keep**:
- Tool results that are still being actively referenced
- The most recent read of any file the agent may need to re-reference
- Error messages that explain why a particular approach failed (the agent may need to avoid repeating it)

**Implementation note**: Tool result clearing is typically done programmatically — either by a hook that fires after tool use or by a compaction function that scans history and identifies clearable results. The Agent SDK's `PostToolUse` hook is well-suited for this.

**Risk**: Very low. Clearing old tool results almost never degrades quality. It is the safest compaction technique and should be the first one applied.

---

## Technique 2: Conversation Summarization

**What it is**: Pass the conversation history to a model and ask it to produce a condensed summary. Replace the full history with the summary.

**When to use it**: When the agent has completed a significant phase of work and the detailed turn-by-turn history is no longer needed — but the conclusions, decisions, and open issues are.

### Summarization Principles

**Maximize recall first, then precision**

The primary risk of summarization is information loss. A summary that omits a critical architectural decision or an unresolved bug can cause the agent to make inconsistent choices later. Prioritize completeness on the first pass; compress on the second.

**What to preserve**:
- Architectural and design decisions — especially the reasoning, not just the conclusion
- Unresolved bugs, open questions, and known limitations
- Key implementation details the agent will reference later
- Commitments made to the user or system

**What to discard**:
- Redundant tool outputs (if the same file was read three times, the repeated reads are noise)
- Verbose raw API or search results that were already processed
- Back-and-forth exchanges that reached a clear conclusion
- Exploratory reasoning that was superseded by a later decision

### Summarization Prompt Pattern

A simple and effective pattern:

```
The following is a conversation between an AI agent and a user.
Summarize the conversation preserving:
- All decisions made and the reasoning behind them
- All open questions, bugs, or unresolved issues
- Key implementation details the agent will need going forward
- Any commitments made to the user

Discard:
- Redundant or repeated tool outputs
- Intermediate reasoning that was revised or superseded
- Verbose content already incorporated into a decision

Conversation:
[full history]

Summary:
```

### When to trigger summarization

- When remaining context budget drops below a threshold (e.g., 20% remaining)
- At natural phase boundaries in long tasks (e.g., after research phase, before implementation phase)
- Proactively at session end, to produce a handoff summary for the next session

---

## Technique 3: Structured Note-Taking

**What it is**: The agent writes persistent notes to a file or store outside the context window. Notes are retrieved at session start or on demand.

**Why it works**: Notes decouple the agent's working memory from its context window. The context window is ephemeral — it disappears at the end of the session and degrades as it fills. Notes are durable — they persist across sessions and are retrieved selectively.

### Note Structure

Effective agent notes are:
- **Brief**: 3-5 sentences per entry, not paragraphs
- **Actionable**: Record decisions and next steps, not just observations
- **Dated or sequenced**: So the agent can understand what is recent vs stale
- **Structured**: Consistent headings make retrieval easier

Example note schema:

```markdown
## Session 2026-03-18

### Completed
- Implemented the database migration for user_preferences table
- Fixed auth token expiry bug (was off by 1000ms due to clock skew)

### In Progress
- Payment webhook handler: parsing works, persistence is not yet tested

### Open Issues
- Rate limiting on the Stripe API: need to implement exponential backoff
- Migration script fails on empty tables — workaround: skip if row_count == 0

### Next Steps
1. Write integration tests for webhook handler
2. Test migration on staging with empty tables
3. Review retry logic with team before shipping
```

### File-Based Note Systems

For code-focused agents, a file-based note system works well:

- `AGENT_NOTES.md` or `PROGRESS.md` at the project root
- Agent reads it at session start (pre-loaded or via the first tool call)
- Agent appends entries after each significant action
- Agent updates "In Progress" and "Open Issues" sections as they evolve

This pattern mirrors how human developers use a running work log. It is simple to implement and requires no external infrastructure.

### Retrieval Patterns

Notes can be loaded:
- **Always** (pre-loaded in CLAUDE.md or at session start): Good for ongoing projects where every session builds on the last
- **On demand** (agent reads the notes file when it needs to orient itself): Good for agents that sometimes start fresh

---

## Technique 4: Sub-Agent Isolation

**What it is**: Launch focused sub-agents with clean context windows to handle exploration-heavy subtasks. The orchestrator receives a condensed summary instead of bearing the full cost of exploration.

**Why it works**: Exploration is expensive. Reading through a large codebase, researching a topic across multiple sources, or analyzing a dataset can consume tens of thousands of tokens. If the orchestrator does this work directly, the exploration fills its context window and leaves little room for the actual decision-making and output generation.

Sub-agent isolation separates the two concerns:
- **Sub-agent**: explores extensively in its own clean context window
- **Orchestrator**: receives a condensed summary (1-2K tokens) and reasons from that

### Sub-Agent Summary Contract

Every sub-agent should be given explicit instructions about what to return:

```
Research [specific question] thoroughly. Then return a condensed summary under 2,000 tokens covering:
- The key findings directly relevant to [specific question]
- Any uncertainties or gaps that require further investigation
- Recommended next steps

Do not include raw source material, verbose quotes, or exploratory dead ends in the summary.
```

This contract matters. Without it, sub-agents return verbose outputs that defeat the purpose of isolation.

### What sub-agents are good for

- Deep research on a specific topic with multiple sources
- Large codebase analysis ("what does this module do?")
- Parallel investigation of multiple independent questions
- Any task where you know upfront that exploration will be expensive

### What sub-agents are not good for

- Short tasks where the context cost of launching a sub-agent is comparable to just doing the work directly
- Tasks where the sub-agent needs tight coordination with the orchestrator at each step (use a shared session instead)
- Tasks where the sub-agent's full output is needed by the orchestrator (the whole point is summarization)

---

## Progressive Disclosure

A principle that unifies the techniques above: each interaction should reveal the context needed for the next decision — no more, no less.

Applied to tool use: retrieve information when it is needed, at the level of detail it is needed, and clear it when it has been consumed.

Applied to summarization: preserve the decisions and open issues (what the agent needs to continue) and discard the exploratory path that led there (what the agent no longer needs).

Applied to sub-agents: the sub-agent explores fully, then distills — revealing to the orchestrator only the signal relevant to the next decision.

---

## Selection Criteria

| Situation | Recommended technique |
|---|---|
| Large tool results filling the context | Tool result clearing |
| Conversational agent approaching context limit | Conversation summarization |
| Multi-session iterative development task | Structured note-taking |
| Parallel research or large-scale exploration | Sub-agent isolation |
| Long-running agent (general case) | Compaction + note-taking |
| Complex multi-phase investigation | Sub-agent isolation + orchestrator notes |
| Agent hitting context limit mid-session | Tool result clearing first, then summarization |

When in doubt, apply tool result clearing first — it is safe, easy, and often recovers enough budget to avoid needing more aggressive techniques.
