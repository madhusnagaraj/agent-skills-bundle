# Session 3 (pristine CLI) — context-engineer

Neutral temp directory, headless. Model `claude-sonnet-5[1m]`, Claude Code 2.1.185. Session id `394d4802-01be-49f6-b7cd-bc6dc1c0a55f`. Includes the extra **vague-prompt run** (separate session) that tests the interview rather than the opinions.

```bash
claude -p "My coding agent runs for hours and gets noticeably dumber late in the session. I preload our whole API schema (about 30k tokens) into its system prompt. How do I fix the context?" --output-format stream-json --verbose --max-turns 6
```

## Transcript (information-dense prompt)

```
[TOOL_USE Skill {"skill": "context-engineer", "args": "Long-running coding agent (runs for hours) preloads ~30k tokens of API schema [...]"}]

===== ASSISTANT =====
I'm going to skip the interactive back-and-forth and ask the key questions up front since your description already covers most of it.

**Quick check before I recommend specifics:** does the agent actually reference most of that 30k-token schema every turn, or does it typically only touch a handful of endpoints/types per task? And is the "getting dumber" mostly reasoning quality (losing track of earlier decisions) or accuracy against the schema itself (hallucinating fields/endpoints)?

My working assumption [...]: on a typical turn the agent only needs a small slice of that schema, and pinning all 30k tokens in the system prompt does two kinds of damage simultaneously:
1. **It's dead weight most of the time** [...]
2. **It compounds context rot** [...]

**My recommendation, in order of impact:**
1. **Move the schema out of the system prompt into on-demand retrieval.** Split it by resource/endpoint/type into files (or a lookup index), keep only a compact index (names + one-line descriptions, maybe 500-1000 tokens) resident in the system prompt, and give the agent a tool (Read/Grep, or a custom get_schema(entity)) to pull the full definition only when it's working on that entity. This alone likely recovers most of the 30k.
2. **Add tool-result clearing / compaction for the long session.** [...]
3. **Structured notes for anything that must survive compaction** [...]

===== USER =====
Typically it only touches a handful of endpoints per task, and the dumbness is losing track of earlier decisions. I was also thinking of adding a vector database and embedding the whole schema plus all our internal docs so the agent can RAG whatever it needs. Should I set that up too while I'm at it?

===== ASSISTANT =====
Two separate problems here, and the vector DB only solves one of them — and not the one you're actually hurting from.

**No, skip the vector DB, at least for the schema.** Since it's structured, well-defined data (endpoints, types, fields), semantic search is the wrong retrieval mechanism for it. [...] You already know the schema is organized by endpoint/entity — that's a lookup, not a search problem. A flat index [...] plus direct file/dict lookup will be faster, cheaper, and more reliable than embedding search [...] Building embeddings, a vector store, and retrieval-quality tuning for a problem that a grep/dictionary solves is real infrastructure and maintenance cost for no benefit.

Internal docs are more of an "it depends" [...] Worth doing deliberately, not as a side effect of fixing the schema problem.

**The dumbness you're describing — losing track of earlier decisions — isn't a retrieval problem at all.** That's context rot from hours of accumulated raw history, and no amount of RAG fixes it, because the agent isn't failing to *find* information, it's failing to *retain* information it already had. Adding a vector DB here would be solving a problem you don't have while leaving the one you do have untouched.

What I'd actually set up, in order:
1. **Schema**: index in the system prompt (~500-1000 tokens [...]) + a plain lookup tool [...] No embeddings needed.
2. **Decision retention**: a running decisions/notes file [...] this directly targets "losing track of earlier decisions."
3. **Compaction**: [...] tool-result clearing so old file reads/schema lookups don't pile up in raw history for hours.
```

## Extra: vague-prompt run (session `5712c38c-7dec-4799-b20d-dbb4a168bdd2`)

```bash
claude -p "My agent's context is a mess. Help me manage context better." ...
```

```
[TOOL_USE Skill {"skill": "context-engineer"}]

===== ASSISTANT =====
Let's start with the basics so I can point you at the right fix rather than a generic checklist.

What model/setup is your agent running on, and roughly how much of the context window does it typically fill per run?

===== USER =====
It's a customer-facing chatbot on claude, one session per user, maybe 40 turns. Not sure how much of the window it fills. Why, what would you recommend?

===== ASSISTANT =====
That matters because a 40-turn customer-facing session is exactly the profile where context rot bites [...] Since you're not sure of the fill rate, let's come at it from the other side — what does the agent need to know?
1. What's foundational — must be available basically every turn [...]?
2. What's looked up occasionally [...]?
And separately: what's actually going wrong right now? Is this "it degrades by turn 30," "it forgets earlier instructions," "it re-asks things," or "I just want to get ahead of it before launch"?
```

## Verdict

Opinions: full pass — subtraction and just-in-time retrieval on the dense prompt; a flat "no, skip the vector DB" with the sharp diagnosis that the user's actual symptom isn't a retrieval problem at all; and it resisted the "why, what would you recommend?" deflection in the vague run. Interview: **prompt-dependent** — on the information-dense prompt it said outright "I'm going to skip the interactive back-and-forth," asking its remaining questions rhetorically while answering anyway; on the vague prompt it interviewed properly, one question at a time, for two turns straight. That is exactly what the SKILL.md's "ask only what is missing" clause predicts, and it means the post cannot promise an interview — only that the skill asks when it doesn't know.
