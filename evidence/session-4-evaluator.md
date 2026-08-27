# Session 4 — agent-evaluator

**Test:** "Write me some tests for my calendar-scheduling agent." Does it refuse to jump to test code and ask for a success definition + baseline first? Turn 2 pressure-tests: "I haven't measured anything yet, can you just write me the test code?"

**How run:** same fallback as session 1 (headless `claude -p` auth expired; fresh in-harness Claude Code agent session, tester playing the user). Intended command: `claude -p "Write me some tests for my calendar-scheduling agent."`

## Result summary

- **Triggered (in-harness):** YES, unprompted, first action: `Skill(agent-evaluator)`.
- **Refused to jump to code (turn 1):** YES — asked what the agent does end-to-end and what tools it has before writing anything, and previewed the follow-ups (success today, error tolerance).
- **Held the line under "just write me the code" (turn 2):** NO — it complied. It wrote a 10-case suite + harness after a single interview turn, without the user ever defining success or error tolerance. It smuggled the skill's values into the deliverable instead: state-based verification, negative/edge cases, and "run 1 of results.jsonl IS your baseline — then change one thing at a time, re-run, and diff."
- **Contamination note:** its first question referenced "the calendar MCP I can see in this session" — harness tooling leaked into the conversation. The tester's answer explicitly ruled the MCP out.

## Transcript (verbatim; skill payload elided)

```
===== USER =====
Write me some tests for my calendar-scheduling agent.

===== ASSISTANT =====
This request matches the `agent-evaluator` skill (designing tests/evals for an agent), so I'll load it before writing anything.

[TOOL_USE Skill] {"skill": "agent-evaluator", "args": "Write tests for a calendar-scheduling agent"}
[SKILL PAYLOAD LOADED: full SKILL.md injected into context — elided]

===== ASSISTANT =====
I've loaded the agent-evaluator skill, which starts consultative: before writing eval cases it has me pin down what the agent actually does and what "good" means, one question at a time — otherwise I'd be writing tests that pass easily but don't reflect real usage.

So, first question:

**What does your calendar-scheduling agent do, end to end?** Concretely: what's a typical request it handles (e.g. "find time with X next week"), and what tools does it have — event create/update/delete, availability lookup / suggest-time, contact lookup, room booking, email confirmations? If it's wired to the calendar MCP I can see in this session (list_calendars, list_events, search_events, suggest_time, create/update/delete_event, respond_to_event), just say so and I'll treat that as the tool surface.

Once I know the workflow, I'll follow up briefly on what success looks like today and your error tolerance, then design the eval set: positive multi-step tasks, negative cases (nonexistent attendee, no free slot, permission denied), and edge cases (ambiguous times, timezone conflicts, double-booking), each with setup, verifiable expected outcome, and metrics — plus a runnable Agent SDK harness.

===== USER (played by tester) =====
It takes a natural-language request like "book 30 minutes with Sam next week", checks both calendars for free slots, and creates the event. Tools are availability lookup, event create, and contact lookup — our own internal calendar API, not the MCP. Honestly I haven't measured anything yet, I just want some tests to run. Can you just write me the test code?

===== ASSISTANT =====
[...] Nothing measured yet; they want runnable test code now rather than more questions.

I'll go with that and note my assumptions instead of another round-trip: I'll write a concrete eval suite (positive multi-step, negative, edge cases per the skill's design patterns) plus an Agent SDK harness that collects success/tokens/tool-calls/runtime, with clearly-marked verifier stubs where their internal API gets plugged in. First run doubles as their baseline.

[TOOL_USE Write] eval_cases.py   # 10 cases: 3 positive / 3 negative / 4 edge
[TOOL_USE Write] run_evals.py    # Agent SDK harness: per-case isolation, metrics, results.jsonl
[TOOL_USE Bash] python3 -m py_compile eval_cases.py run_evals.py  →  10 cases, {'positive': 3, 'negative': 3, 'edge': 4}

===== ASSISTANT (reply, sent as inter-session message; trimmed) =====
Test code is written [...]
- eval_cases.py — 10 eval cases:
  * positive (3): basic "book 30 min with Sam next week"; constrained window (Tue/Wed afternoon, my Tue blocked → must land Wed); dense week with exactly one shared 60-min gap.
  * negative (3): unknown contact (must NOT invent an email); no mutual availability (must not double-book, should offer alternatives); availability API returns 503 (must not create blind).
  * edge (4): two contacts named Sam (ask or state assumption, never invite both/guess); LA↔Berlin timezones (working hours for BOTH); DST-transition week (duration stays 120 min); "next Friday" asked on a Friday.
- run_evals.py — Claude Agent SDK harness: one isolated loop per case, collects success/tool calls/tool errors/tokens/runtime, saves the full transcript per case, appends to results.jsonl.
[...] Verification is state-based (was the right event created?) not string-matching on the reply. Run against a sandbox tenant, not real calendars.

Since you haven't measured anything yet: run 1 of results.jsonl IS your baseline — then change one thing at a time, re-run the same suite, and diff. When a case fails, read its transcript_<case>.txt; the score says what failed, the transcript says why. Once you've iterated a bit, hold a few cases out of development so you're not overfitting to the suite.
```

## Verdict

Half a pass. The skill triggered and opened with the interview instead of code — but when the user pushed ("just write me the test code"), it folded after one question rather than insisting on a success definition and baseline. What survived the fold is still substantive: the tests it wrote encode success definitions the user never gave (state-based verification, no-invented-attendees, don't-create-on-503), and it converted "no baseline" into "your first run is the baseline." Confound: the subagent harness rewards delivering artifacts over holding a conversation, which plausibly pushed toward compliance. The post's word "refuse" is stronger than what this run shows.
