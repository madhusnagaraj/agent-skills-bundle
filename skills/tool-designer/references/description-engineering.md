# Description Engineering

How to write tool descriptions that make agents more accurate, efficient, and reliable. Covers structure, parameter design, response format, and token efficiency.

---

## Why Descriptions Matter

Tool descriptions are the primary interface between the agent and the tool. They are not documentation for humans — they are instructions for a decision-making model.

A description that is vague, incomplete, or structured like API documentation causes:
- Wrong tool selections (the agent picks the wrong tool for the job)
- Bad parameter choices (wrong format, missing required context, wrong enum value)
- Unnecessary retries (agent tries, fails, tries again with a different tool)
- Cascading failures (wrong tool call leads to wrong result leads to wrong next step)

Description refinements alone have achieved SWE-bench SOTA. This is the highest-leverage place to invest time in tool design.

---

## Description Structure

Write descriptions in four parts, in this order:

1. **What it does** — One concrete sentence. Use active voice. Name the specific action.
2. **When to use it** (and when NOT to) — Disambiguate from similar tools. This is critical when the agent has multiple tools with overlapping surface area.
3. **Key capabilities** — What the agent can control. What parameters matter most. What this tool can and cannot do.
4. **Expected formats and boundaries** — What valid input looks like. What edge cases to know about.

Not every tool needs all four parts. Short tools can collapse to one or two sentences. Long or ambiguous tools need all four.

---

## Good vs. Bad Examples

### Search tool

**Bad:**
```
search_tickets: Search for support tickets.
```

Problems: Does not say what you can search by. Does not distinguish from list_tickets. Does not mention result limits or pagination.

**Good:**
```
search_tickets: Finds support tickets matching given criteria. Use when you need to locate
a ticket by customer, subject, status, or date — not when you already have a ticket_id
(use get_ticket instead). Supports partial text matching on subject and customer name.
Returns up to 20 results by default; use page_token for additional pages. Filters can
be combined: status + date_from narrows to recent open tickets.
```

---

### Create/action tool

**Bad:**
```
send_email: Sends an email.
```

Problems: No guidance on recipients format. No mention of cc/bcc. No clarification on HTML vs plain text. No indication of rate limits or sending constraints.

**Good:**
```
send_email: Sends an email from the configured sender address to one or more recipients.
Use for transactional messages, notifications, and follow-ups. recipient and cc fields
accept arrays of email addresses or "Name <email>" format. body supports plain text or
HTML (set content_type accordingly). Do not use for bulk sends over 50 recipients — use
send_bulk_email instead. subject is required; body defaults to empty string if omitted.
```

---

### Compound context tool

**Bad:**
```
get_customer_context: Gets customer context.
```

Problems: "Context" means nothing without specifics. Agent cannot know what is returned.

**Good:**
```
get_customer_context: Returns a bundle of customer information needed for most support and
sales workflows. Includes: profile (name, email, account tier), subscription status and
renewal date, last 5 transactions, open support tickets, and account notes. Use when you
need to understand a customer's situation before taking action — saves multiple individual
lookups. Use response_format="concise" for a summary suitable for decision-making;
"detailed" for full field data when you need to act on specific values.
```

---

## Parameter Naming: Poka-Yoke

Poka-yoke (error-proofing) in parameter design means making it difficult for the agent to pass wrong values.

**Use specific names:**
- `user_id` not `user` — "user" is ambiguous between a name, email, and ID
- `start_date` not `from` — "from" is a reserved word in some languages and ambiguous in context
- `status_filter` not `status` — clarifies this is a filter, not the current status of something being set
- `max_results` not `limit` — self-documenting; "limit" requires mental translation

**Always document the format:**
```
order_id: string
  # The unique order identifier. Format: ORD-XXXXXX (e.g., ORD-000123).
  # Find this via search_orders if you don't have it.

event_date: string
  # ISO 8601 date (e.g., 2024-01-15). Times accepted in UTC: 2024-01-15T14:30:00Z.

priority: "low" | "normal" | "high" | "urgent"
  # Defaults to "normal". Use "urgent" only for SLA-critical issues.
```

**Use enums aggressively:**
If only a fixed set of values are valid, enumerate them. Agents cannot reliably guess valid string values. Listing them:
- Prevents invalid calls
- Makes tool behavior predictable
- Helps the agent pick the right value based on context

**Mark required vs optional:**
```
Required: query
Optional: status (default: all), date_from (default: 30 days ago), page_size (default: 20, max: 100)
```

---

## Response Format Design

Design response formats for how agents will use the data, not for how the API returns it.

### Use semantic field names

Agents reason in natural language. Field names should match how a human would describe the value.

| Low-level (avoid) | Semantic (prefer) |
|---|---|
| `cust_id_ref` | `customer_id` |
| `mime_type` | `file_type` (e.g., "PDF", "image") |
| `status_code_int` | `status` (e.g., "active", "cancelled") |
| `ts_created` | `created_at` or `created_date` |
| `usr_acct_tier` | `account_tier` (e.g., "free", "pro", "enterprise") |
| `flg_is_active` | `is_active` (boolean) |

### Concise vs. detailed modes

When a tool can return either a summary or full detail, give the agent control via a `response_format` parameter:

```
response_format: "concise" | "detailed"
```

In **concise** mode, return only the fields needed to make a decision:
```json
{
  "customer_id": "C-001234",
  "name": "Acme Corp",
  "account_tier": "enterprise",
  "open_tickets": 3,
  "renewal_in_days": 14
}
```

In **detailed** mode, return everything needed to act:
```json
{
  "customer_id": "C-001234",
  "name": "Acme Corp",
  "account_tier": "enterprise",
  "billing_email": "billing@acme.com",
  "subscription": { "plan": "enterprise_annual", "renewal_date": "2025-02-01", ... },
  "contacts": [...],
  "open_tickets": [...],
  "recent_transactions": [...]
}
```

This keeps token usage proportional to the agent's actual need.

---

## Truncation Patterns

When results are truncated, tell the agent what happened and what to do next. Never silently truncate.

**Bad truncation:**
```json
{ "results": [...20 items...] }
```
Agent does not know if it got all results or a partial set.

**Good truncation:**
```json
{
  "results": [...20 items...],
  "total_count": 143,
  "page_token": "eyJvZmZzZXQiOjIwfQ==",
  "truncation_note": "Showing 20 of 143 results. Pass page_token to get the next page, or add filters to narrow results."
}
```

**For context bundles** — when a single large field is truncated:
```json
{
  "comments": [...5 most recent...],
  "comments_note": "Showing 5 of 47 comments (most recent first). Use get_ticket_comments(ticket_id, page_token) for earlier comments."
}
```

---

## Token Efficiency

Every tool call consumes context. Design responses to be token-efficient by default.

**Pagination defaults**: Default to 20 results, not 100. The agent can request more if needed. Returning 100 items when 3 are relevant wastes context.

**Filter aggressively**: When the agent provides filters, apply them strictly. Do not return nearby matches when the filter clearly rules them out.

**Response structure**: JSON is usually the most compact structured format. Avoid verbose XML. Use Markdown only for human-readable content (documentation, notes, messages) that the agent will relay to a user.

**Field selection**: In concise mode, omit fields that are null, empty, or default. An empty array for `cc_recipients` in every email response wastes tokens.

**Summary instead of list**: When the agent needs to know about a collection but not act on each item, return a count and summary rather than the full list.
```json
{
  "open_tickets_count": 7,
  "oldest_open_ticket_age_days": 14,
  "tickets_summary": "3 billing issues, 2 technical, 2 general inquiries"
}
```
instead of returning all 7 tickets when the agent only asked "does this customer have open tickets?"

---

## Common Description Mistakes

**1. Describing the API, not the capability**
"Calls the POST /v2/tickets endpoint" tells the agent how it works, not when to use it or what it accomplishes.

**2. Omitting disambiguation**
When two tools have similar names or purposes, not saying "use X when Y, use Z when W" guarantees confusion.

**3. Not specifying formats**
"Pass a date" forces the agent to guess. "ISO 8601 format, e.g. 2024-01-15" prevents a malformed call.

**4. Missing limits**
If a tool will error for large inputs, document the limit. "Max 100 items per call" prevents a runtime error on item 101.

**5. Vague "optional" parameters**
"Filters results by status (optional)" — optional how? What is the default behavior when omitted? Does it return all statuses or only active ones?

**6. No example for ID formats**
IDs have formats. If you do not document the format and provide an example, agents will substitute a name, email, or guessed value.
