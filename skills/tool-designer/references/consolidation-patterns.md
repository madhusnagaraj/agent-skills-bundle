# Tool Consolidation Patterns

Reference guide for deciding when and how to consolidate agent tools. Covers the reasoning behind consolidation, core patterns, anti-patterns, and decision criteria.

---

## Why Consolidation Matters

Agents have limited context windows. Every tool call consumes tokens — the prompt, the parameters, and the response. When an agent must chain many small tools to accomplish a single logical operation, it:

- Consumes context window space on intermediate data that is not directly useful
- Accumulates error risk at each step (any failure breaks the chain)
- Increases latency proportionally with each call
- Forces the agent to reason about API structure rather than the problem

The goal is to design tools that match how agents think about tasks — not how the underlying APIs happen to be organized.

**Rule of thumb**: If an agent would always call Tool A, then Tool B with A's result, then Tool C with B's result — and this sequence never varies — that is one tool, not three.

---

## Core Patterns

### Pattern 1: Search Over List

Never expose a "list all" tool when the agent will need to filter or find a specific item.

Agents do not browse. They solve a problem, which means they search for a specific thing. A `list_all` tool forces the agent to retrieve a large result set, then reason over it to find what it actually wants. This wastes context and introduces filtering logic that is better done server-side.

**Anti-pattern:**
```
list_contacts() -> returns 500 contacts
# Agent then has to read through results to find the right one
```

**Better:**
```
search_contacts(query, filters) -> returns top matches
# Agent can directly express what it needs
```

**More examples:**
- `search_projects(name_contains, status, owner)` — not `list_projects()`
- `find_messages(from, subject_contains, date_after)` — not `list_messages()`
- `search_inventory(product_name, category, in_stock_only)` — not `list_inventory()`

**When list-style tools are acceptable:**
- The domain genuinely has few items (list_supported_currencies returns 3 values)
- The agent always needs all items (list_required_fields for a form)
- The items are configuration or metadata, not operational data

---

### Pattern 2: Compound Action

When an operation always requires the same setup steps, bake those steps into a single tool.

This is especially common when creating or scheduling something requires first checking availability, permissions, or existing state.

**Anti-pattern:**
```
# Agent must call all of these in sequence every time:
search_users(name)         # Step 1: find user IDs
list_calendar_events(user) # Step 2: find their availability
create_calendar_event(...)  # Step 3: create the event
send_notification(user)     # Step 4: notify attendees
```

**Better:**
```
schedule_meeting(
    title, attendees, duration_minutes,
    preferred_date, preferred_time_range
)
# Internally handles: availability check, conflict resolution, creation, notification
```

**More examples:**
- `onboard_user(email, role, team)` — replaces: create_user + assign_role + add_to_team + send_welcome_email
- `archive_project(project_id, reason)` — replaces: update_status + notify_members + move_to_archive + create_audit_log
- `resolve_ticket(ticket_id, resolution_note)` — replaces: update_ticket + post_comment + notify_requester + close_ticket

**Decision test**: Ask "would the agent ever do Step 1 without also doing Steps 2, 3, and 4?" If no, combine them.

---

### Pattern 3: Context Bundle

When an agent always needs multiple pieces of information to reason about something, return them together in one call.

Agents frequently need surrounding context before they can take action. Retrieving context through individual calls multiplies latency and context usage.

**Anti-pattern:**
```
# Agent needs to call all of these before it can answer "what should I do next?"
get_ticket(ticket_id)
list_comments(ticket_id)
get_user(ticket.assignee_id)
list_related_tickets(ticket_id)
get_customer(ticket.customer_id)
```

**Better:**
```
get_ticket_context(ticket_id)
# Returns: ticket details, recent comments, assignee info,
#          customer profile, and related open tickets — one call
```

**More examples:**
- `get_pull_request_context(pr_id)` — returns PR details, diff summary, review comments, CI status, related issues
- `get_order_context(order_id)` — returns order, line items, shipping status, payment status, customer info, and open issues
- `get_customer_360(customer_id)` — returns profile, subscription status, recent activity, open tickets, and account notes

**Response format consideration**: Context bundles can be large. Design a `response_format: "concise" | "detailed"` parameter so the agent can request a summary when it only needs to decide the next step, and full detail when it needs to act.

---

## Anti-Patterns

### Thin API Wrappers

A tool that does nothing but call one API endpoint with the same parameters is rarely worth exposing to an agent. It adds cognitive overhead without adding value.

**Symptom**: The tool description is "Calls the GET /users/{id} endpoint."

**Problem**: The agent now has to know the API structure, not just the task. It has to understand what a "user" means in this API, what format `{id}` takes, and what fields are returned.

**Fix**: Either add value through parameter validation, result shaping, and error translation — or use the API directly in a compound tool that adds real agent-visible value.

---

### List-All Sprawl

A set of tools that all follow the pattern `list_X()` with no filtering.

**Symptom**: `list_users()`, `list_projects()`, `list_issues()`, `list_comments()` — all return full unfiltered collections.

**Problem**: The agent has no way to express what it is looking for. It retrieves everything and reasons over it — burning context on data it does not need.

**Fix**: Replace with search/filter tools. If browsing is a genuine use case, add it as a separate tool with an explicit "I want all of them" semantic.

---

### Overlapping Responsibilities

Two tools that do nearly the same thing, with slightly different parameter sets.

**Symptom**: `find_user_by_email(email)`, `find_user_by_name(name)`, `lookup_user(id)` — three tools for one logical operation.

**Problem**: The agent must decide which tool to use, may pick the wrong one, and the tool selection itself adds noise.

**Fix**: `search_users(query, search_by: "email" | "name" | "id")` — one tool, the agent specifies intent through a parameter.

---

### Premature Decomposition

Breaking a single logical action into sub-steps because that is how it works internally.

**Symptom**: `validate_order()`, `reserve_inventory()`, `charge_payment()`, `confirm_order()` — four steps that always happen together.

**Problem**: The agent must know to call all four in sequence. Any mis-step or mis-order causes partial state.

**Fix**: `place_order(items, payment_method, shipping_address)` — one tool handles the full transaction. Expose the sub-steps only if there is a real use case where an agent would run them independently.

---

## Decision Criteria

Use this checklist when deciding whether to combine or keep separate:

**Combine when:**
- The agent would always call both tools in the same sequence
- The first tool's output is only used to call the second tool
- The combined tool has a cleaner, more actionable name
- Error handling is simpler in the combined version (one failure mode instead of four)

**Keep separate when:**
- The operations are used in genuinely different contexts
- One operation is significantly slower and sometimes the agent only needs the fast one
- The combined tool would have more than 5-6 parameters (complexity signal)
- The operations have different permission requirements
- The operations have meaningfully different failure modes that require different handling

---

## Namespacing Strategies

Namespacing prevents tool name collisions and helps agents find the right tool when many are available.

### Prefix by service
Group all tools for a service under the same prefix. Use when the agent integrates with multiple external services.
```
github_search_issues
github_create_issue
github_get_pull_request
jira_search_issues
jira_create_issue
jira_transition_issue
```

### Prefix by resource
Group tools by the primary resource they operate on. Use when there are many operations on a few key resources.
```
orders_search
orders_get
orders_create
orders_cancel
customers_search
customers_get
customers_update
```

### Prefix by action type
Group tools by what they do. Use sparingly — usually works best as a secondary prefix.
```
search_orders
search_customers
search_inventory
create_order
create_ticket
create_notification
```

**Guidance**: Pick one strategy and apply it consistently. Mixing strategies confuses agents and developers alike. When in doubt, prefix by service or resource.
