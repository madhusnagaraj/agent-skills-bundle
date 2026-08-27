# Session 4 (pristine CLI) — agent-evaluator

Neutral temp directory, headless. Model `claude-sonnet-5[1m]`, Claude Code 2.1.185. Session id `4aab2d43-96a6-42f6-92e6-bddd9d820945`; retry session `e32fed18-9474-4b1f-9d3c-b511d6074871`.

```bash
claude -p "Write me some tests for my calendar-scheduling agent." --output-format stream-json --verbose --max-turns 6
```

## THE FINDING: the skill did NOT auto-trigger

```
===== ASSISTANT =====
[TOOL_USE Bash {"command": "pwd && ls -la", "description": "Check current directory contents"}]

===== ASSISTANT =====
The current directory is empty — there's no calendar-scheduling agent code here for me to write tests against.

A couple of things I need from you to write useful tests:
1. **Where's the code?** Is this a different working directory, or is the agent not built yet [...]?
2. **What's the agent built with?** — e.g. Claude Agent SDK (Python/TS), LangGraph, a custom loop [...]
3. **What does it actually do?** [...]

Once I can see the real code, I'll follow `superpowers:test-driven-development` conventions for this repo (test framework, mocking approach for the Calendar API, etc.) rather than guessing a structure.
```

No `Skill(agent-evaluator)` call. The session parsed "write me some tests" as a unit-testing coding task, checked the directory for code, and reached for a *different* installed skill collection (superpowers TDD). Note the test machine has many other skills installed; a reader with only the four bundle skills faces less trigger competition — direction of that effect on a clean machine is unknown.

## Turn 2 — once "evals, not unit tests" was said, it triggered

```
===== USER =====
The agent isn't in this directory and I don't want unit tests against code — I want you to design tests/evals for the agent's behavior. It takes a natural-language request like 'book 30 minutes with Sam next week', checks both calendars for free slots, and creates the event, via our own internal calendar API. I haven't measured anything yet, just write me the test code.

===== ASSISTANT =====
This calls for the agent-evaluator skill since we're designing behavioral evals rather than unit tests.

[TOOL_USE Skill {"skill": "agent-evaluator", "args": "Design evals for a calendar-scheduling agent. [...]"}]

===== ASSISTANT =====
A few essentials I need before I can write code that actually runs (not going to belabor the full consultative process since you want code, but these are blocking, not optional):
1. **Language/stack** — Python or TypeScript? [...]
2. **Internal calendar API shape** — what are the actual function/tool signatures [...]? I need real signatures to mock accurately, or I'll invent an interface that doesn't match yours.
3. **"Sam" resolution** — is there a contacts/directory lookup tool [...]?
If you'd rather I just pick reasonable interface names and you'll adapt the mock later, say so [...] but tell me stack (Python/TS) at minimum since that decides everything else.
```

## Turn 3 — the stubborn double-push

```
===== USER =====
Python, and it's a function call: schedule(request_text) -> event dict. No really, skip the rest of the questions — I don't want to define success metrics or baselines, just give me the tests now.

===== ASSISTANT =====
[writes fake_calendar.py and test_scheduling_agent.py; py_compile check]

I don't have execution approval in this sandbox, so I did a careful manual review instead of running pytest [...]

**What's covered, and why each one earns its place:**
- **Happy path + no-overlap** [...] asserts the chosen slot doesn't overlap *either* party's busy blocks (not just "an event got created")
- **Tool-call ordering** — asserts get_free_busy was called for both attendees *before* create_event; catches an agent that skips checking and just books
- **Duration parsing** — parametrized over "30 minutes" / "half an hour" / "an hour" / "15 min"
- **No availability** — asserts create_event was never called [...] rather than the agent inventing a slot
- **Unknown / ambiguous "Sam"** [...]
- **"Next week" boundary** [...]
- **Timezone edge case** [...]
- **Grounding check** (test_return_value_matches_what_was_actually_booked) — the direct hallucination test [...]
- **Double-booking on repeat** [...]

**Two things you must adapt before this runs:** [import path; DI/monkeypatch target]
Also: FakeCalendarAPI assumes signatures [...] that I invented since you skipped that question [...]
```

## Retry: naming the skill directly (fresh session, fresh neutral dir)

```bash
claude -p "Use the agent-evaluator skill: write me some tests for my calendar-scheduling agent."
```

```
[TOOL_USE Skill {"skill": "agent-evaluator", "args": "calendar-scheduling agent — write tests"}]

===== ASSISTANT =====
I'll start by understanding the agent before designing tests — no test code yet.

What does the calendar-scheduling agent actually do end-to-end? Specifically: what triggers it (a user request, an email, a Slack message?), what tools does it call (Google Calendar, email, contacts?), and what does a typical successful run look like from start to finish?
```

Direct invocation works and produces the on-script behavior ("no test code yet").

## Verdict

Auto-trigger: **failed** on the naive line, in both runs of this scenario (harness run triggered; pristine CLI did not) — "write me some tests for my agent" is genuinely ambiguous between unit tests and evals, and the skill's description ("evaluate agent", "test agent", "agent evals") lost to the plain coding reading plus a competing TDD skill. Once the user said "evals" or named the skill, it fired and opened correctly. Refusal discipline: it held the interview through one "just write me the test code" (better than the harness run), then folded completely on the explicit double-push — no success-metrics conversation, no baseline demand, straight to code. The delivered tests do encode success criteria (grounding, ordering, no-invented-slots), but the post's "refuses until you have said what success means" is not what happens under a determined user.
