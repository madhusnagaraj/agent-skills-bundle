# Session 2 — tool-designer

**Test:** a 9-tool REST-mirror list for a support agent. Does the skill trigger, consolidate the list down, and push back on tool-count growth?

**How run:** same fallback as session 1 (see `session-1-architect.md` — headless `claude -p` auth was expired; fresh in-harness Claude Code agent session, tester playing the user via inter-session messages). Intended command: `claude -p "I'm building a support agent. Design tools for it: I was planning list_tickets, get_ticket, list_customers, get_customer, get_customer_orders, list_kb_articles, search_kb, create_reply_draft, send_reply."`

## Result summary

- **Triggered (in-harness):** YES, unprompted, first action: `Skill(tool-designer)`.
- **Interviewed:** NO — it explicitly skipped the skill's Phase-1 questions: "It calls for clarifying questions first, but since this is a background run I'll make reasonable assumptions." (Direct artifact of the subagent harness; an interactive session may behave differently — untested here.)
- **Consolidated:** YES — 9 → 6, with per-tool rationale; kept draft/send split deliberately as a safety boundary.
- **Pushed back on growth (turn 2):** YES — of 7 requested additions it added 3, absorbed 3, and **declined `run_sql`** with security/guardrail/tool-selection reasons.

## Transcript (verbatim; skill payload elided; long specs trimmed with [...])

```
===== USER =====
I'm building a support agent. Design tools for it: I was planning list_tickets, get_ticket, list_customers, get_customer, get_customer_orders, list_kb_articles, search_kb, create_reply_draft, send_reply.

===== ASSISTANT =====
This is squarely a tool-design task, so I'll load the tool-designer skill first to ground the recommendations in ACI principles.

[TOOL_USE Skill] {"skill": "tool-designer", "args": "Design tools for a customer support agent: proposed list_tickets, get_ticket, ..."}
[SKILL PAYLOAD LOADED: full SKILL.md injected into context — elided]

===== ASSISTANT =====
The skill loaded. It calls for clarifying questions first, but since this is a background run I'll make reasonable assumptions, state them, and deliver a full design — flagging where your answers would change it. Assumptions: a single helpdesk backing system, the common workflow is "ticket comes in → understand customer → find answer in KB → reply," and you want a human-review-friendly reply path (that's implied by having both draft and send).

## The headline: 9 tools → 6

Your list mirrors a REST API (list/get pairs per resource). Agents don't browse lists — they search — and they never want a customer without that customer's context. Consolidated surface:

| Yours | Becomes | Why |
|---|---|---|
| `list_tickets` | `tickets_search` | Agents always filter. A bare list of 500 tickets burns context and invites the agent to paginate instead of reason. |
| `get_ticket` | `tickets_get` (returns ticket **+ full message thread**) | The agent can't draft a reply without the thread; never make it a second call. |
| `list_customers`, `get_customer`, `get_customer_orders` | `customers_get_context` | These three are always called in a chain: find customer → profile → orders. One tool, one call, one context bundle. |
| `list_kb_articles`, `search_kb` | `kb_search` | Nobody lists a knowledge base; a `response_format` param covers "snippets vs full article." |
| `create_reply_draft`, `send_reply` | `replies_draft`, `replies_send` | **Kept separate deliberately** — draft/send is a safety boundary, not redundancy. `replies_send` should only accept a `draft_id`, so the agent physically cannot send un-drafted text. |

[... full per-tool specs with descriptions, parameters, and shaped error messages elided for length ...]

## Two decisions that are yours, not mine
1. **Autonomy of `replies_send`.** If humans approve every reply, drop `replies_send` from the agent's `allowed_tools` entirely [...] the strongest guardrail is a tool the agent doesn't have.
2. **Whether `tickets_get` should embed the full customer context.** [...]

===== USER (played by tester) =====
Great. A teammate reviewed this and wants us to also add: list_orders, get_order, update_ticket_status, add_internal_note, escalate_ticket, get_agent_availability, and a generic run_sql tool for anything we missed. Can you add those to the design?

===== ASSISTANT =====
[...] I'll fold these through the same consolidation discipline rather than appending all seven, and one of them (run_sql) I'll decline with reasons.

## Revised design: 7 requested → 3 added, final surface is 8 tools

**Added:** orders_get, tickets_update (merges update_ticket_status + add_internal_note — "the same triage gesture"), tickets_escalate (auto-routes; availability check folded in).

**Absorbed, not added:**
- `list_orders` → already covered by `customers_get_context` [...] An agent never wants orders detached from a customer.
- `add_internal_note` + `update_ticket_status` → one `tickets_update` [...]
- `get_agent_availability` → folded into `tickets_escalate`. The agent's goal is never "know the roster," it's "hand this off" [...] it removes a whole class of stale-read races.

**Declined: `run_sql`.** A generic "for anything we missed" escape hatch undoes the whole design:
1. It bypasses every guardrail — the draft-before-send gate, enum validation, shaped errors — since anything is reachable by query.
2. Security/PII: a support agent with raw SQL on prod is one injected ticket body away from exfiltrating the customers table.
3. It degrades tool selection: given an escape hatch, models drift toward it and skip the well-designed tools, and schema-guessing SQL fails in ways the agent can't self-correct.
If real gaps surface in testing, the fix is to name the gap and design a scoped tool for it — that's a feature of this process, not a limitation.
```

## Verdict

Consolidation and pushback both landed hard: 9 → 6 unprompted, then held the line at 8 against a 13-tool request and refused the SQL escape hatch with specific reasons. The clear miss: it **announced it was skipping the interview** because it detected it was running as a background agent — so this run proves the opinions, not the interview discipline.
