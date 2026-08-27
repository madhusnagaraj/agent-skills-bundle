# Agent SDK Tools Reference

Built-in tools, MCP integration, allowed_tools configuration, and custom tool registration patterns for the Claude Agent SDK.

---

## Built-in Tools

These tools are provided by the SDK. Use them directly — do not wrap them in custom tools unless you need to enforce specific constraints on their behavior.

| Tool | What it does | When to use it |
|---|---|---|
| `Read` | Reads file contents from the filesystem | When the agent needs to inspect source files, config, or data |
| `Write` | Creates or overwrites a file | When the agent produces a new output file |
| `Edit` | Makes targeted edits to existing files | When the agent modifies existing content without full replacement |
| `Bash` | Executes shell commands | When the agent needs to run scripts, tests, build commands, or CLI tools |
| `Glob` | Finds files matching a pattern | When the agent needs to discover which files exist |
| `Grep` | Searches file contents with regex | When the agent needs to find where something is defined or used |
| `WebSearch` | Searches the web for information | When the agent needs current information not in its training data |
| `WebFetch` | Fetches and reads a URL | When the agent needs to retrieve content from a specific webpage |
| `AskUserQuestion` | Prompts the user for input | When the agent needs a decision or clarification it cannot infer |

**Restrict built-in tools using `allowed_tools`**: Only include the tools the agent actually needs. An agent that reads and writes code does not need `WebSearch`. An agent that queries external APIs does not need `Bash`.

---

## MCP Server Integration

MCP (Model Context Protocol) servers expose tools from external services. Use MCP when integrating with well-known services — GitHub, Notion, Jira, Slack, Linear, and others have maintained MCP servers.

### Basic MCP configuration

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def run_with_mcp():
    async for msg in query(
        prompt="Search for open issues labeled 'bug' in the repository",
        options=ClaudeAgentOptions(
            mcp_servers={
                "github": {
                    "command": "npx",
                    "args": ["@modelcontextprotocol/server-github"]
                }
            }
        )
    ):
        if hasattr(msg, "content") and msg.content:
            print(msg.content)

asyncio.run(run_with_mcp())
```

**TypeScript**
```typescript
import { query, ClaudeAgentOptions } from "@anthropic-ai/claude-agent-sdk";

async function runWithMcp(): Promise<void> {
    for await (const msg of query(
        "Search for open issues labeled 'bug' in the repository",
        {
            mcpServers: {
                github: {
                    command: "npx",
                    args: ["@modelcontextprotocol/server-github"]
                }
            }
        } as ClaudeAgentOptions
    )) {
        if (msg.content) console.log(msg.content);
    }
}
```

### Multiple MCP servers

```python
options=ClaudeAgentOptions(
    mcp_servers={
        "github": {
            "command": "npx",
            "args": ["@modelcontextprotocol/server-github"]
        },
        "notion": {
            "command": "npx",
            "args": ["@modelcontextprotocol/server-notion"],
            "env": {"NOTION_API_KEY": os.environ["NOTION_API_KEY"]}
        },
        "slack": {
            "command": "npx",
            "args": ["@modelcontextprotocol/server-slack"],
            "env": {"SLACK_BOT_TOKEN": os.environ["SLACK_BOT_TOKEN"]}
        }
    }
)
```

### MCP with allowed_tools restriction

When using MCP servers, restrict which MCP tools the agent can access. MCP servers often expose many tools; agents perform better with a focused subset.

```python
options=ClaudeAgentOptions(
    mcp_servers={
        "github": {
            "command": "npx",
            "args": ["@modelcontextprotocol/server-github"]
        }
    },
    # Only allow specific GitHub tools, plus built-in Read for local context
    allowed_tools=["Read", "mcp__github__search_issues", "mcp__github__create_issue", "mcp__github__add_comment"]
)
```

MCP tool names follow the pattern: `mcp__{server_name}__{tool_name}`

---

## allowed_tools Configuration

`allowed_tools` is the most important safety control for agent tool access. Always define it explicitly.

```python
# Minimal: read-only agent for analysis
ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep"]
)

# Standard: code agent that can read and modify files
ClaudeAgentOptions(
    allowed_tools=["Read", "Write", "Edit", "Glob", "Grep", "Bash"]
)

# Research agent: web access, no file writes
ClaudeAgentOptions(
    allowed_tools=["Read", "WebSearch", "WebFetch"]
)

# Integration agent: custom tools plus minimal built-ins
ClaudeAgentOptions(
    allowed_tools=["search_orders", "get_customer_context", "update_order_status", "Read"]
)
```

**Rule**: Start with the minimum. Add tools only when the agent demonstrably needs them. Agents with fewer tools make fewer mistakes and are easier to debug.

---

## Custom Tool Registration

Register custom tools when you need: compound operations, domain-specific parameter validation, shaped responses optimized for the agent, or business logic that wraps multiple API calls.

**Python — function-based registration**
```python
from claude_agent_sdk import query, ClaudeAgentOptions, tool

@tool
def search_orders(
    query: str,
    status: str = None,
    date_from: str = None,
    page_size: int = 20
) -> dict:
    """
    Searches orders by customer name, email, or order ID. Use when you need to find
    orders matching criteria — not when you already have an order_id (use get_order).
    status accepts: 'open', 'shipped', 'delivered', 'cancelled'. date_from: ISO 8601
    (e.g. 2024-01-15). Returns up to page_size results (default 20, max 100).
    """
    # Business logic: call internal API, shape response for agent
    results = internal_orders_api.search(
        query=query,
        status=status,
        date_from=date_from,
        limit=min(page_size, 100)
    )
    return {
        "orders": [shape_order_for_agent(o) for o in results.items],
        "total_count": results.total,
        "page_token": results.next_token,
        "truncation_note": (
            f"Showing {len(results.items)} of {results.total} results. "
            "Use page_token to get more, or add filters to narrow."
            if results.total > len(results.items) else None
        )
    }

async def main():
    async for msg in query(
        prompt="Find all open orders for Acme Corp from the last 30 days",
        options=ClaudeAgentOptions(
            allowed_tools=["search_orders", "get_order", "update_order_status"]
        )
    ):
        if hasattr(msg, "content") and msg.content:
            print(msg.content)
```

**TypeScript — tool registration**
```typescript
import { query, ClaudeAgentOptions, defineTool } from "@anthropic-ai/claude-agent-sdk";

const searchOrders = defineTool({
    name: "search_orders",
    description: `Searches orders by customer name, email, or order ID. Use when you need
to find orders matching criteria — not when you already have an order_id (use get_order).
status accepts: 'open', 'shipped', 'delivered', 'cancelled'. date_from: ISO 8601 (e.g. 2024-01-15).
Returns up to page_size results (default 20, max 100).`,
    parameters: {
        type: "object",
        properties: {
            query: { type: "string", description: "Customer name, email, or order ID" },
            status: { type: "string", enum: ["open", "shipped", "delivered", "cancelled"] },
            date_from: { type: "string", description: "ISO 8601 start date, e.g. 2024-01-15" },
            page_size: { type: "integer", default: 20, maximum: 100 }
        },
        required: ["query"]
    },
    handler: async ({ query, status, date_from, page_size = 20 }) => {
        const results = await internalOrdersApi.search({ query, status, date_from, limit: page_size });
        return {
            orders: results.items.map(shapeOrderForAgent),
            total_count: results.total,
            page_token: results.nextToken,
            truncation_note: results.total > results.items.length
                ? `Showing ${results.items.length} of ${results.total} results. Use page_token for more.`
                : undefined
        };
    }
});
```

---

## When to Use Built-in vs. MCP vs. Custom

| Scenario | Use |
|---|---|
| File operations on local filesystem | Built-in: `Read`, `Write`, `Edit`, `Glob`, `Grep` |
| Running shell commands, scripts, tests | Built-in: `Bash` |
| Web research, fetching URLs | Built-in: `WebSearch`, `WebFetch` |
| Asking the user a question | Built-in: `AskUserQuestion` |
| Well-known external service (GitHub, Notion, Jira) | MCP server for that service |
| Internal APIs or proprietary systems | Custom tool |
| External API with complex parameter validation | Custom tool (wraps the API, validates locally) |
| Compound operation combining multiple API calls | Custom tool (the combination is the value) |
| Shaped responses for agent context efficiency | Custom tool (return semantic fields, not raw API response) |

**Decision rule**: If a built-in or MCP tool does what you need, use it. Build a custom tool only when you need to add value that is not there out of the box — validation, shaping, combining, or domain-specific error handling.
