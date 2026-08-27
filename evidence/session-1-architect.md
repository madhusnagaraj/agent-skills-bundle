# Session 1 — agent-architect

**Test:** the post's exact "Watch one work" line. Does the skill trigger unprompted, interview one question at a time, and land on "this is a single prompt, not an agent" (Tier 0)?

## How this session was actually run (read this first)

The intended command was:

```bash
# from a neutral temp directory
claude -p "I want to build an agent that tags our support tickets by category and urgency." \
  --output-format stream-json --verbose --max-turns 6
# then: claude -p --resume <session-id> "<answer>"
```

That command **could not authenticate on this machine**: the stored OAuth token (keychain and `~/.claude/.credentials.json`) expired ~2026-07-12 and does not refresh headlessly; only the interactive session's host-side auth works. Re-auth requires a human `claude login`.

**Fallback actually used:** a fresh Claude Code agent session (same CLI harness, Claude Code 2.1.185, model `claude-fable-5` for the agent) was spawned with the user line as its entire prompt, in an isolated git worktree, with the four bundle skills installed at `~/.claude/skills/`. The tester played the user's side via inter-session messages. This tests the same skill mechanism (description match → Skill tool invocation → SKILL.md loaded), but it is **not** a pristine `claude -p` session: the harness has hundreds of other skills registered and repo CLAUDE.md context present. Trigger results below are labeled "in-harness".

## Result summary

- **Triggered (in-harness):** YES, unprompted, first action: `Skill(agent-architect)`.
- **Interviewed:** YES — asked before recommending. One question turn, but that turn bundled several sub-questions (input format? metadata? output destination? exact tag set?) — not strictly "one question at a time".
- **Landed Tier 0:** YES — "a single LLM call per ticket. No agent needed." Explicitly called the agent framing over-engineering.
- **Deviation from the post's script:** the post says "come back if it fails"; the real session instead shipped a working Tier-0 code sketch immediately (subagent finish-and-deliver pressure).

## Transcript (verbatim; skill payloads and long tool results elided with markers)

```
===== USER =====
I want to build an agent that tags our support tickets by category and urgency.

[system-reminder elided]

===== ASSISTANT =====
I'll load the agent-architect skill — this is exactly its territory ("build an agent" for a classification task), and it'll guide the architecture decision before we jump to code.

[TOOL_USE Skill] {"skill": "agent-architect", "args": "Support ticket tagging agent: classify tickets by category and urgency"}
[TOOL_RESULT] Launching skill: agent-architect

[SKILL PAYLOAD LOADED: full SKILL.md injected into context — elided]

===== ASSISTANT =====
Good news up front: "tag tickets by category and urgency" is a classification task, which usually needs far less machinery than the word "agent" suggests — often a single well-crafted LLM call per ticket. Before I recommend an architecture, I need a few specifics, one at a time.

First question: **What does a ticket look like when it arrives, and where does the tag need to go?**

Concretely — is the input just subject + body text, or does it include metadata (customer tier, product area, prior ticket history)? And is the output written back to a helpdesk (Zendesk, Intercom, etc.) via API, or into a database/queue? A concrete example ticket and the exact tag set you want (e.g. categories: billing / bug / how-to; urgency: low / medium / high) would let me sketch this precisely.

===== USER (played by tester) =====
Tickets are short text emails — a subject line and a couple of paragraphs of body, no extra metadata. After tagging, all that should exist is a category label (billing, bug, how-to, or account) and an urgency label (low, medium, high) on the ticket. And yes, the steps are always the same for every ticket: read it, pick a category, pick an urgency. Nothing else.

===== ASSISTANT =====
Restating the answer [...]: tickets are plain text (subject + short body, no metadata), the only output needed is two labels — category (billing / bug / how-to / account) and urgency (low / medium / high) — and the steps are identical for every ticket.

That settles the classification. Let me check the SDK recipes reference so any code sketch uses correct import paths.

[TOOL_USE Bash] ls ~/.claude/skills/agent-architect/references/ && head -100 .../agent-sdk-recipes.md
[TOOL_USE Skill] {"skill": "claude-api", "args": "model choice + structured output for a cheap high-volume classification task"}
[... three more reference lookups elided ...]

===== ASSISTANT =====
I have everything I need. Here's the architecture verdict [...]

## Recommendation: Tier 0 — a single LLM call per ticket. No agent needed.

Your task fails every test that would justify an agent or even a multi-step workflow: the input is plain text, no tools or lookups are needed, the output is two labels from fixed sets, and the steps never vary. "Read it, pick a category, pick an urgency" is one prompt with structured output. Building an agent loop, orchestrator, or pipeline for this would be pure over-engineering — more cost, more latency, more failure modes, zero accuracy gain.

### Design

One call to the Messages API per ticket, with **structured outputs** so the response is guaranteed to be valid JSON matching your label schema [...]

```python
class TicketTags(BaseModel):
    category: Literal["billing", "bug", "how-to", "account"]
    urgency: Literal["low", "medium", "high"]
[... full working sketch: messages.parse + system prompt with category definitions ...]
```

### Decisions you still get to make
- **Model / cost.** [...] I'd start with Opus, measure accuracy on ~50 real tickets, then try Haiku on the same set and compare.
- **Real-time vs. batch.** [...] the **Message Batches API** runs the identical requests at **50% cost** [...]
- **Failure handling.** API errors are the only real failure mode; a retry-then-flag-for-human policy is plenty. There's no "agent went off the rails" risk because there's no loop.

The one place to invest effort is the **category/urgency definitions in the system prompt** — crisp, mutually exclusive definitions with a tie-breaking rule matter far more than architecture here. If you later find quality lacking on ambiguous tickets, the escalation path is a few few-shot examples in the prompt — still Tier 0 — not an agent.
```

## Verdict

The core claim held: given the post's exact sentence, the skill fired on its own (in-harness) and talked the user down from "agent" to one prompt, in two turns. Two honest deviations: (a) the "one question at a time" instruction was followed in spirit (one question **turn**) but that turn packed 3–4 sub-questions; (b) it did not stop at "try a prompt first" — it delivered the prompt, plus model/cost/batch advice the user never asked for.
