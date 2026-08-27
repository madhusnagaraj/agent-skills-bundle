# Eval Design Patterns

Reference guide for designing evaluation cases that produce actionable signal. Covers what makes a task strong or weak, how to pair prompts with verifiable outcomes, and patterns for stress-testing agents across common domains.

---

## Why Task Design Matters

The most common eval mistake is building tasks that are too easy. An agent that passes simple, single-operation tests may still fail badly in real use. The eval suite should reflect how the agent is actually used — not a simplified version of it.

A weak eval gives you false confidence. A strong eval gives you a map of where the agent breaks.

---

## Strong vs Weak Tasks

### The Core Distinction

Weak tasks test whether the agent can perform an operation. Strong tasks test whether the agent can coordinate across tools, handle ambiguity, and deliver a verifiable outcome in a realistic scenario.

**Weak task characteristics:**
- Single tool call required
- Success condition is vague or trivially met
- No cross-service coordination
- Could be completed by any competent system with minimal reasoning

**Strong task characteristics:**
- Requires 3+ tool calls, often across different services
- Success condition is specific and verifiable
- Requires the agent to make decisions, not just execute steps
- Reflects a real workflow a user would actually perform

### Examples by Domain

**Calendar / Scheduling**

Weak: "Schedule a meeting with Jane next Tuesday."

Strong: "I need to sync with Jane Reeves from the Acme account and David from engineering before the Q2 planning deadline on Friday. Find a 45-minute slot that works for all three of us this week, send an invite with the agenda from the last planning meeting attached, and block my Friday morning for prep."

Why it's strong: contact lookup, availability check across multiple calendars, document retrieval, event creation, separate calendar block — all with a real deadline constraint.

**Project / Ticket Management**

Weak: "Show me open tickets."

Strong: "I'm running the Monday engineering sync in 30 minutes. Pull all P1 and P2 backend tickets updated in the last 72 hours, group by assignee, flag anything that is blocked or hasn't had an update in 48 hours, and draft a 5-bullet summary I can paste into the meeting notes."

Why it's strong: filtered search, grouping logic, staleness detection, output formatting for a specific use case.

**Document / Knowledge Work**

Weak: "Summarize the Acme proposal."

Strong: "I'm preparing for a renewal call with Acme next week. Find the original proposal, the last quarterly business review, and any open support tickets from the past 90 days. Summarize what we committed to, how we've delivered against it, and flag any open issues I should be ready to address."

Why it's strong: multi-document retrieval, cross-source synthesis, decision-relevant output.

**Code / Development**

Weak: "Find the bug in this function."

Strong: "Our staging deploy is failing. The error is in the payment processing module. Find the relevant logs from the last deploy, identify the failing test, locate the code change that introduced it, and open a GitHub issue with the reproduction steps and a suggested fix."

Why it's strong: log retrieval, test runner integration, code search, issue creation — all connected through a real incident workflow.

---

## Test Categories

Every eval suite should include three categories. Gaps in any category leave blind spots.

### Positive Cases

The agent should complete the task correctly and completely.

These are your core accuracy signal. They tell you whether the agent can do what it is supposed to do under normal conditions.

Design positive cases to cover:
- The most common workflows (high frequency)
- High-value but less frequent tasks (high stakes)
- Tasks that use every major tool at least once

### Negative Cases

The agent should handle the situation gracefully — not crash, not hallucinate, not silently fail.

Examples:
- The requested resource does not exist: "Schedule a meeting with John Martinez." — no John Martinez in the contacts system.
- The operation is out of scope: "Cancel my subscription." — the agent does not have billing access.
- Required permissions are missing: "Update the production database record." — read-only credentials.
- Conflicting constraints: "Schedule a 2-hour meeting tomorrow at 2pm and also at 3pm."

A good agent on a negative case: acknowledges the failure clearly, explains what it could not do, and suggests what the user can do instead. A bad agent: silently continues with wrong data, hallucinates a result, or errors out without explanation.

### Edge Cases

Boundary conditions that reveal brittleness not visible in normal testing.

Examples:
- Ambiguous input: "Follow up with the client." — which client? Which follow-up?
- Very large result sets: a search that returns 500 items when the agent expected 20
- Partial success: step 1 succeeds, step 2 fails — what does the agent do with the partial result?
- Unusual but valid input: a date far in the future, a zero-quantity order, an empty document
- Concurrent or dependent tasks: two tasks that share a resource

---

## Eval Case Template

Use this structure for every eval case. The setup and expected fields are the most commonly skipped — and the most important for getting consistent results.

```
Task: [The exact prompt given to the agent]
Setup: [Preconditions — data that must exist, auth state, environmental state]
Expected: [Verifiable outcome — what must be true after the run, not how it gets there]
Metrics: [What to measure — completion, tool_calls, tokens, runtime, error_count]
```

### Example: Positive Case

```
Task: "Find the Acme account's renewal date and send their primary contact a heads-up email
      3 weeks in advance with a summary of what we've delivered this quarter."

Setup: Acme account exists in CRM with renewal_date, primary_contact, and at least 2
       closed deals in the current quarter.

Expected:
  - Primary contact email retrieved correctly
  - Renewal date identified
  - Email draft contains correct contact name, renewal date, and at least one
    specific deliverable from the quarter
  - Email is sent (or staged for send if in test mode)

Metrics: success=true/false, tool_calls (target: <=6), tokens (baseline), runtime
```

### Example: Negative Case

```
Task: "Update the billing address for account ID ACC-9999."

Setup: Account ACC-9999 does not exist in the system.

Expected:
  - Agent does not fabricate account data
  - Agent returns a clear error message indicating the account was not found
  - Agent suggests how to find the correct account (e.g., search by name or email)
  - No write operations are attempted

Metrics: graceful_failure=true/false, hallucination=none, tool_calls (target: <=2)
```

---

## Stress-Testing Patterns

For agents that should handle complex, real-world workflows, apply these patterns deliberately.

### Cross-Service Coordination

Design tasks that require the agent to pull data from one service and use it in another. This tests whether the agent correctly threads context across tool calls.

Example: retrieve a ticket ID from Jira, look up the associated PR in GitHub, find the deploy that included it in your deployment system.

### Sequential Dependency

Tasks where each step depends on the output of the previous one, and a failure at any step should be handled gracefully rather than cause a silent wrong result.

### Result Set Stress

Tasks that return large result sets force the agent to paginate, filter, or summarize — rather than assuming everything fits in one call. Design tasks where a naive implementation would return 200+ results and the correct behavior is to narrow the query.

### Ambiguity Resolution

Tasks with intentional ambiguity test whether the agent asks for clarification vs makes assumptions. "Email the team about the delay" — which team? which delay? A well-designed agent asks. A poorly designed one guesses.

### Contradiction Handling

Tasks with conflicting constraints test whether the agent surfaces the conflict or silently picks one path. "Schedule a meeting that works for both groups" when group A and group B have no overlapping availability.

---

## Pairing Prompts with Verifiable Outcomes

The key to a useful eval is a success condition that is:

1. **Checkable without human judgment** — ideally programmatically verifiable
2. **Outcome-focused, not path-focused** — did the agent accomplish the goal, regardless of the exact sequence of tool calls?
3. **Tolerant of valid alternatives** — the agent might use a different tool sequence and still get the right result

What to verify:
- Was the correct resource created/updated/sent?
- Did the output contain the required information?
- Were any prohibited actions taken (e.g., writes when only reads were allowed)?
- Was the response semantically correct, even if phrased differently?

What NOT to verify:
- Exact string matches on agent-generated text
- The specific sequence of tool calls (unless sequence is semantically required)
- Formatting details that do not affect correctness

If your verifier rejects a correct response because of phrasing or formatting differences, you have a verifier problem, not an agent problem.
