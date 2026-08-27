# Session 2 (pristine CLI) — tool-designer

Neutral temp directory, headless. Model `claude-sonnet-5[1m]`, Claude Code 2.1.185. Session id `489c1d1d-0b2e-4075-9c44-612d4cb2989c`.

```bash
claude -p "I'm building a support agent. Design tools for it: I was planning list_tickets, get_ticket, list_customers, get_customer, get_customer_orders, list_kb_articles, search_kb, create_reply_draft, send_reply." --output-format stream-json --verbose --max-turns 6
```

## Transcript

```
===== ASSISTANT =====
Using tool-designer to design this tool set properly.

[TOOL_USE Skill {"skill": "tool-designer"}]

===== ASSISTANT =====
Before I design anything, I want to understand the actual workflow the agent will run — the proposed 9-tool list looks like a direct mirror of your database tables, and that's usually a sign the tool surface hasn't been shaped around what the agent actually needs to *do*.

First question: **What's a full example of the agent handling one support ticket, start to finish?** e.g. "customer emails asking where their order is → agent looks up X → checks Y → drafts a reply → sends it." Walk me through the concrete steps for your most common ticket type.

===== USER =====
Most common: customer emails asking about a refund or an order problem. The agent should read the ticket, look up the customer and their recent orders, search our KB for the relevant policy, draft a reply, and a human clicks send today but we may automate sending later. That's basically it.

===== ASSISTANT =====
Good, that's a clear golden path. A couple more before I design:

1. **The ticket already identifies the customer** (via email/account on the ticket), or does the agent ever need to search for "which customer sent this" separately? [...]
2. **Are there other ticket types besides refund/order-problem** [...]
3. **When you say "search KB for the relevant policy"** — is that always a free-text query [...] or does the agent sometimes need to browse/list KB articles by category? [...]

===== USER =====
1. The ticket has the customer's email on it, that's enough to identify them. 2. Also account/login questions and general how-to questions, those don't need orders. 3. Free-text search only, nobody browses. Now can you give me the tool list? And my teammate also wants a generic run_sql tool added for anything we missed.

===== ASSISTANT =====
**On `run_sql`: no, don't add it.** A few reasons that matter for a support agent specifically:

- **Injection/security surface** — even "read-only" SQL access from an LLM-driven agent is a standing risk. A crafted or malformed ticket body ending up anywhere near query construction is a real threat model here, not a hypothetical.
- **No error-handling contract** — the whole point of Phase 4 tool design is giving the agent actionable errors ("customer not found, try search_contacts"). A raw SQL error is unrecoverable noise the agent can't act on.
- **Unbounded blast radius** — "anything we missed" means an agent could join across tables it shouldn't reason about (other customers' PII, internal financials), with no scoping.
- **It's an admission the tool set is incomplete, not a fix for it** — if something's missing, the fix is a fifth narrowly-scoped tool, not a backdoor around all of them. [...]

## Tool list

Your 9 collapses to **4**, based on what you told me: ticket always carries the customer's email (no customer search needed), KB is free-text-only (no browsing), and orders only matter for a subset of ticket types.

**1. `get_ticket(ticket_id)`** [...]
**2. `get_customer_context(customer_email)`** — Returns customer profile *plus* their last 5 orders in one call. I bundled get_customer + get_customer_orders [...] Not `list_customers` — you confirmed the agent never searches for "who is this."
**3. `search_kb(query, max_results=5)`** [...] No `list_kb_articles` — you said nobody browses.
**4. `create_reply_draft(ticket_id, body, cited_articles=[])`** [...]

**`send_reply` — deliberately not in the agent's tool list yet.** Design it in the API/backend now if you want, but don't expose it to the agent while a human is clicking send. [...]

One open question before you build: is the agent always invoked per-ticket [...] or does it ever need to pull "what's in my queue"? If the latter, that's a search_tickets(status, assigned_to) tool, not a plain list_tickets — but only worth adding if that's a real workflow, not just because it mirrors a table.
```

## Verdict

Strongest run of the eight. Auto-triggered; interviewed genuinely (turn 1: one question; turn 2: three bundled follow-ups); consolidated **9 → 4** — more aggressive than the harness run's 9 → 6, because the interview answers justified the cuts; refused `run_sql` outright with four reasons; and pulled `send_reply` from the agent's surface entirely on safety grounds. This is the "expect to get a shorter one back" claim performing above the draft's own example.
