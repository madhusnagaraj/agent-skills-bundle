# Evidence ledger — agent-skills-bundle

**Date:** 2026-08-26 · **CLI:** Claude Code 2.1.185 · **Model:** `claude-fable-5` · **Skills under test:** `~/.claude/skills/{agent-architect,tool-designer,context-engineer,agent-evaluator}`

## Claim under test

From the draft's "Watch one work" and install sections:

> "It does not produce code. It asks one question at a time [...] And then it tells you something slightly deflating: this is not an agent. It is one well-written prompt." (agent-architect)
> "Bring it your tool list before you build anything; expect to get a shorter one back." (tool-designer)
> "Its default advice is to remove things, not add them." (context-engineer)
> "It is written to refuse test code until you have said what success means, and to ask for a baseline." (agent-evaluator)
> "The last test is the automatic one: try the ticket line above and agent-architect should pick it up on its own." (auto-trigger)

## Method — including the deviation

**Intended:** four headless CLI sessions (`claude -p "<user line>"`, then `--resume` to play the user), from a neutral temp directory.

**What actually happened:** headless `claude -p` could not authenticate on the test machine — the stored OAuth token (keychain and `~/.claude/.credentials.json`) expired ~2026-07-12 and does not refresh headlessly; only the interactive session's host-side auth works, and re-auth needs a human `claude login`. Two attempts (sandboxed and unsandboxed, plus a cleaned-environment retry) all returned `401 OAuth access token has expired`.

**Fallback used:** four fresh Claude Code agent sessions in the same authenticated harness (isolated git worktrees, prompt = the user line verbatim, nothing else), with the tester playing the user's side via inter-session messages, 2–3 user turns each. Same skill mechanism end to end: description match → `Skill` tool invocation → SKILL.md loaded into context → behavior follows it. Skill invocation was verified in the raw JSONL transcripts, not self-report.

**Why that is not the same test:** (a) the harness registers hundreds of skills and plugin commands, vs. the handful a typical reader installs — trigger competition differs in an unknown direction; (b) the repo's CLAUDE.md and session context ride along — one session referenced a calendar MCP visible in the harness; (c) subagent sessions are biased toward finishing and delivering artifacts rather than holding a multi-turn interview — this visibly affected sessions 2 and 4.

## Result

| # | Skill | Auto-triggered (in-harness) | Interviewed first | Landed the expected answer | Held under pressure (turn 2) |
|---|---|---|---|---|---|
| 1 | agent-architect | **Yes** — `Skill(agent-architect)`, unprompted | **Partial** — asked before recommending, but bundled 3–4 sub-questions into its one question turn | **Yes** — "Tier 0 — a single LLM call per ticket. No agent needed." | n/a (2 turns to verdict) |
| 2 | tool-designer | **Yes** — unprompted | **No** — announced it was skipping the interview ("this is a background run") and stated assumptions instead | **Yes** — consolidated 9 → 6 with rationale | **Yes** — 7 requested additions → 3 added, 3 absorbed, `run_sql` declined on security/guardrail grounds; final 8 tools |
| 3 | context-engineer | **Yes** — unprompted | **No** — answered in one shot; the prompt already contained the skill's three "essentials," and it used the SKILL.md's "ask only what is missing" clause; flagged one assumption instead of asking | **Yes** — subtraction + just-in-time: 30k dump → 1–2k index + retrieval, compaction, notes file | **Yes** — flatly declined the vector-DB/RAG add-on; "measure first, embeddings only if lexical search demonstrably fails" |
| 4 | agent-evaluator | **Yes** — unprompted | **Yes** — refused to write code on turn 1; asked for the end-to-end workflow first | **Partial** — see next column | **No** — when told "I haven't measured anything, just write me the test code," it complied after one interview turn instead of insisting on a success definition; it embedded success criteria and "run 1 = your baseline" into the delivered suite |

Numbers worth keeping: 4/4 skills invoked unprompted from the naive line (in-harness); tool-designer went 9 → 6, then held 13 requested → 8 and refused 1 outright; agent-architect reached the "not an agent" verdict in 2 turns; agent-evaluator produced 10 cases (3 positive / 3 negative / 4 edge) when it folded.

## Verdict

**Narrow to:** the skills reliably trigger from the post's naive phrasings *in a Claude Code environment* and reliably deliver the opinionated answers — simplest tier, shorter tool list, subtraction over addition — including under explicit pressure to add complexity. The **interview discipline is softer than the post's language**: "one question at a time" was honored once (loosely), skipped twice with stated justification, and abandoned under a direct "just give me code" push. And the specific claim that `agent-architect` "should pick it up on its own" in a fresh `claude -p` / interactive CLI session is **untested in this run** — the trigger evidence is from an agent harness, not a clean install.

## What's real / what isn't yet

- **Real:** unprompted Skill invocation from all four naive prompts, verified in raw transcripts; the Tier-0 "this is one prompt, not an agent" verdict on the post's exact ticket line; 9→6 tool consolidation and the run_sql refusal; the anti-RAG pushback with a measure-first plan.
- **Real but weaker than the draft says:** "asks one question at a time" — observed as "asks before recommending," with bundling and two justified skips. "Written to refuse test code until you have said what success means" — it *starts* consultative but complies when pushed; the refusal is a default, not a guarantee.
- **Not shown:** auto-trigger in a pristine CLI session from a neutral directory (auth blocker; the harness result is suggestive, not equivalent — and the harness had ~200+ competing skills, arguably a *harder* trigger environment, but also a model/context setup readers won't have). The install-verification steps (`/skills` listing, `/agent-architect` direct invocation) were not exercised end-to-end.
- **Confounds:** n=1 session per skill; one model (`claude-fable-5` — most readers will run a different default); subagent finish-and-deliver bias directly explains the two interview skips and plausibly the evaluator's fold; the tester knew the desired outcomes when playing the user (cooperative answers, though turn-2 pressure lines were adversarial by design).
- **Suggested draft edits** (for the revision step, not applied here): soften "asks one question at a time" to "asks before it recommends"; soften "written to refuse test code until..." to "opens by interviewing instead of writing code" or state the fold honestly; keep the Tier-0 session narrative — it matches the observed session closely — but note transcripts are lightly abridged.

**De-identification check:** grep over `evidence.md` and `experiment/*.md` for employer/client/product/internal-system/account references and the local username/repo name — zero hits (home paths rewritten to `~`, scratchpad paths bracketed).

## What I'd test next

Re-run the same four scripts through real `claude -p` from a neutral directory once the machine is re-authenticated (`claude login`) — that is the single missing piece for the post's auto-trigger and install claims — plus one vague-prompt run of context-engineer to test the interview rather than the opinions, and one stubborn re-push on agent-evaluator ("no really, skip the questions") to see whether the fold is one-turn-deep or total.
