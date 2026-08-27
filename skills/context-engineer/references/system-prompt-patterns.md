# System Prompt Patterns

Reference guide for system prompt architecture. Covers right-altitude calibration, structural organization, minimal sufficiency, example design, tool guidance, and the attention budget concept.

---

## The Right Altitude Principle

A system prompt must be specific enough to produce consistent behavior but minimal enough to leave room for the model to reason. This balance — the right altitude — is the central calibration challenge.

### Failure Mode 1: Too Prescriptive

Symptoms:
- System prompt exceeds 3,000-5,000 tokens
- Bullet lists covering every exception and edge case
- Rules that contradict each other when combined
- Agent behavior is brittle: correct on covered cases, wrong on anything novel

What goes wrong: when a prompt tries to enumerate every scenario, it implicitly treats the agent as a rule-follower rather than a reasoner. The agent matches rules mechanically instead of understanding intent. Novel inputs — which are inevitable in production — hit gaps in the rule set and produce inconsistent behavior.

### Failure Mode 2: Too Vague

Symptoms:
- System prompt is under 200 tokens and reads as a mission statement
- No examples, no constraints, no tool guidance
- Agent behavior varies widely across similar inputs

What goes wrong: without concrete signals, the model falls back on its pretraining defaults. These may be high quality in general but are not calibrated to your use case. Behavior is unpredictable and hard to iterate on.

### Finding the Right Altitude

The right altitude is usually:
- A clear identity statement (what the agent is, what it is not)
- Explicit constraints (the non-negotiables the agent must always respect)
- One or two canonical examples (the highest-leverage token spend)
- Targeted tool guidance when the agent has multiple tools with overlapping use cases

Start here. Add content only when you observe a specific, repeatable failure.

---

## Structural Organization

Organize system prompts into distinct, labeled sections. This matters for two reasons: it makes the prompt easier to maintain, and it helps the model locate relevant information when constructing a response.

XML tags and Markdown headers both work. Choose one and use it consistently throughout.

### Recommended Sections

**Identity**
What the agent is and is not. Its default posture. What it optimizes for.

```
You are a customer support agent for Acme Corp. You help users troubleshoot
product issues, process refunds, and escalate complex cases. You are not a
sales agent — do not upsell or recommend upgrades unless explicitly asked.
```

**Task context**
What problem the agent is solving. The inputs it receives and outputs it produces.

**Rules or constraints**
The non-negotiables. Keep this section short. If it is long, most of it belongs in the examples section.

**Tool guidance**
When to use which tool. This is critical when tools have overlapping capabilities — without guidance, agents waste tokens on decision overhead and sometimes select the wrong tool.

Example:
```
Use search_orders when you have a customer name or email but no order ID.
Use get_order when you already have an order_id. Never use search_orders
if you have the order_id — it is slower and returns less detail.
```

**Examples**
One or two end-to-end examples showing the agent handling a representative case. The highest-value section.

---

## Minimal Sufficiency

The minimal sufficient system prompt is the shortest prompt that produces acceptable behavior on the real distribution of inputs.

### Why it matters

Every token in the system prompt is loaded on every call, occupying context budget regardless of relevance. A system prompt with 4,000 tokens of edge-case coverage consumes 4,000 tokens even when the current task is the simple, common case.

### The process

1. Start with the strongest available model and the shortest plausible prompt.
2. Run the agent on a representative sample of real inputs.
3. Identify the most common failure mode.
4. Add the minimum content that addresses that failure mode.
5. Repeat.

This iterative approach produces leaner prompts than writing speculatively upfront. Speculative prompts inevitably contain large sections that address hypothetical problems that never materialize in production.

### What not to add

- Rules for cases that have not occurred yet
- Explanations of why a rule exists (unless the explanation changes behavior)
- Redundant restatements of the same constraint in different words
- Definitions of terms the model already understands

---

## Few-Shot Examples: The Highest-Value Token Spend

For an LLM, examples are the pictures worth a thousand words. A well-chosen example conveys more about desired behavior than a paragraph of rules because it shows the model what "correct" looks like end-to-end.

### Selection criteria

**Diverse, not exhaustive**
Cover genuinely different scenarios — different input types, different tool use patterns, different output formats. Do not include ten variations of the same case.

**Canonical, not edge cases**
Show the prototypical form of each important scenario. The model generalizes from canonical examples to edge cases more reliably than it does the reverse.

**End-to-end**
Include input, any tool calls (with results), reasoning, and final output. Partial examples — showing only the output — give the model less to generalize from.

**Annotated when useful**
A brief note on why the example is handled a certain way helps the model generalize the principle rather than just memorizing the pattern.

### What to avoid

- More than 3-4 examples in the system prompt (beyond this, the benefit per example diminishes)
- Examples that differ only in surface detail (customer names, product names) — these add tokens without adding diversity
- Exhaustive edge-case lists as a substitute for examples — the model cannot memorize them and the list bloats the prompt

### Practical sizing

For most agents, 1-2 examples in the system prompt is sufficient. If more coverage is needed, use few-shot examples in the human turn (passed dynamically) rather than bloating the system prompt.

---

## Tool Design for Context Efficiency

The tools an agent has affect context in two ways:
1. Tool definitions consume tokens on every call
2. More tools create decision ambiguity, which wastes context on overhead reasoning

### Design principles

**Minimal tool set**
Give the agent only the tools it needs. An agent with 8 well-chosen tools outperforms one with 30 tools of mixed relevance. Review the tool list periodically and remove tools that are rarely used.

**No overlapping capabilities**
If two tools can accomplish the same thing, the agent will consume tokens deciding which one to use — and may make the wrong choice. Consolidate overlapping tools or add explicit tool selection guidance to the system prompt.

**Self-contained tool results**
Design tools to return results that are usable without additional context. An agent should not need to cross-reference a tool result with something earlier in the conversation to interpret it.

**Descriptive parameters**
Parameter names and descriptions guide the agent's choices. Ambiguous parameter names cause wrong calls. Clear parameter names with explicit valid values (enums) reduce decision overhead.

---

## The Attention Budget Concept

The transformer attention mechanism creates pairwise relationships between every token in the context and every other token. As context grows, the model attends to more and more token pairs — and the signal-to-noise ratio drops.

This has practical consequences:

**Context rot**
Beyond a certain point, adding more tokens to the context degrades quality. The model loses track of information from early in the context, produces inconsistent outputs, and sometimes repeats work it has already done. The threshold varies by model and task, but the effect is real and observable in production.

**Front-loading matters**
Information near the beginning and end of the context window receives more reliable attention than information in the middle. If there is content the agent must always reference — key constraints, critical context — put it at the top of the system prompt, not buried in a long list of rules.

**Implications for system prompt design**
- Shorter, focused system prompts outperform longer, comprehensive ones beyond a certain length
- Redundant restatements of the same rule do not reinforce attention — they just add noise
- Critical constraints belong at the top, not distributed throughout

**Practical threshold**
If a system prompt exceeds 3,000-4,000 tokens without a strong justification (e.g., many code examples), it is almost certainly too long. Audit for redundancy, vague explanations, and edge cases that have never materialized.
