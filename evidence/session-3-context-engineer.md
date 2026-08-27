# Session 3 — context-engineer

**Test:** a degrading long-running agent with a 30k-token schema preloaded. Does the skill interrogate before prescribing, and push subtraction / just-in-time loading? Turn 2 baits it with "should I also add a vector DB + RAG?"

**How run:** same fallback as session 1 (headless `claude -p` auth expired; fresh in-harness Claude Code agent session, tester playing the user). Intended command: `claude -p "My coding agent runs for hours and gets noticeably dumber late in the session. I preload our whole API schema (about 30k tokens) into its system prompt. How do I fix the context?"`

## Result summary

- **Triggered (in-harness):** YES, unprompted, first action: `Skill(context-engineer)`.
- **Interrogated first:** NO — it answered in one shot, reasoning that the prompt already contained the skill's three assessment essentials (lifetime, big context item, failure mode). The SKILL.md permits this ("ask only what is missing"), and it flagged its one remaining assumption explicitly — but it never asked a question.
- **Pushed subtraction / JIT:** YES — replace the 30k dump with a 1–2k index + on-demand retrieval; clear stale tool results; notes file; "reduce before adding complexity."
- **Refused the complexity bait:** YES, cleanly — "don't set up the vector DB yet."

## Transcript (verbatim; skill payload elided)

```
===== USER =====
My coding agent runs for hours and gets noticeably dumber late in the session. I preload our whole API schema (about 30k tokens) into its system prompt. How do I fix the context?

===== ASSISTANT =====
This is a context-engineering question — let me load the context-engineer skill before answering.

[TOOL_USE Skill] {"skill": "context-engineer", "args": "Long-running coding agent degrades late in session; 30k-token API schema preloaded in system prompt; how to fix context"}
[SKILL PAYLOAD LOADED: full SKILL.md injected into context — elided]

===== ASSISTANT =====
Here's my read, using the context-engineer method. You've already given me the three assessment essentials — lifetime (hours-long sessions), the big context item (30k-token schema, pre-loaded), and the failure mode (quality degrades late) — so I'll go straight to the recommendation, with one assumption flagged at the end.

## Diagnosis
1. **Context rot, not just budget exhaustion.** Accuracy degrades as the window fills [...] A 30k-token schema at the top of every call means every turn starts a third of the way to trouble before any work happens, and it dilutes the model's attention across mostly-irrelevant endpoint definitions.
2. **Pre-loading everything is the classic anti-pattern.** On any given turn, the agent probably touches a handful of endpoints, not the whole API. The other ~95% of that schema is pure attention tax, paid on every single call.

## Fixes, in order of leverage
**1. Move the schema to just-in-time retrieval.** Replace the 30k dump with a **compact index** in the system prompt (~1-2k tokens) [...] and a **retrieval path** for details: schema split into per-resource files the agent can Read/Grep, or a get_endpoint_schema(name) tool.
**2. Add tool-result clearing / compaction for the long horizon.** [...]
**3. Structured note-taking as durable memory.** [...]
**4. Keep in the system prompt only** what's needed nearly every turn [...]

## The assumption to check
I'm assuming a typical task touches a small fraction of the API. If instead the agent genuinely needs broad schema awareness every turn [...] keep a *summarized* schema pre-loaded (~5k tokens) and retrieve full definitions on demand — hybrid rather than pure just-in-time.

===== USER (played by tester) =====
This is helpful. I was also thinking of adding a vector database and embedding the whole schema plus all our internal docs, so the agent can RAG whatever it needs. Should I set that up too while I'm at it?

===== ASSISTANT (reply, sent as inter-session message) =====
Recommendation: don't set up the vector DB yet. It's the classic "add complexity before exhausting simple options" move, and for your case the simple options are strictly better:

1. Your schema is ~30k tokens — that's tiny. Split into per-resource files with a 1-2k index in the system prompt, and Grep/Read (or a get_endpoint_schema tool) covers it perfectly. Vector search adds nothing here: schema lookups are exact-name lookups, which lexical search wins at. Embeddings actually do worse on "find the POST /invoices/{id}/void endpoint" than a grep does.

2. For internal docs, the same test applies: agentic search (grep + file naming + a docs index the agent can browse) usually beats RAG for a coding agent, because the agent can iterate on queries. Only reach for embeddings when the corpus is large enough that lexical search demonstrably fails (typically 100k+ tokens of prose-y docs where users ask conceptual questions with no keyword overlap).

3. Cost of the vector DB path you'd be signing up for: embedding pipeline, chunking strategy, index sync when the schema/docs change (stale embeddings are a silent-failure mode), retrieval-quality tuning, and one more piece of infra to run. All of that to solve a problem the file-based approach already solves.

Suggested sequencing: ship the file-split + index + JIT retrieval fix first, since that addresses the actual observed failure (late-session degradation). Then measure. If the agent later demonstrably fails to find things via lexical search over the docs corpus, add embeddings for the docs only — never for the API schema.
```

## Verdict

The stubborn default is real: subtraction first, JIT retrieval, and a flat refusal of the RAG add-on with a measure-first sequencing plan. But "interrogate before prescribing" did not happen — the prompt was information-dense enough that the skill's own escape clause ("ask only what is missing") let it skip straight to prescribing. A vaguer prompt would be the sharper test of the interview; this run cannot claim it.
