# AgentC2 Bug Report: MCP Tool Output Validation Rejects Valid Responses

**Reporter:** Corey Shelson  
**Date:** April 8, 2026  
**Workspace:** IVR (Island View Retreat)  
**Severity:** Blocking � agents cannot use MCP-type provider tools at runtime  
**Affects:** Any MCP-type provider (confirmed with Reddit MCP; Firecrawl likely affected too)

---

## Summary

Two related bugs prevent agents from using tools provided by MCP-type integrations:

1. **Tool output validation rejects valid MCP responses** � The platform's output validator expects a `result` field, but standard MCP servers return data in the `content` field. The Reddit data arrives successfully but gets rejected by validation.

2. **MCP tool discovery never persists to database** � `integration_tools_rediscover` always returns empty for MCP-type providers, leaving the integration tools table at 0 tools even after successful connection tests.

---

## Bug 1: Tool Output Validation Mismatch (Blocking)

### Steps to Reproduce

1. Create an agent with Reddit MCP tools assigned (`reddit_reddit_search_subreddit`, etc.)
2. Invoke the agent with a prompt that triggers tool use
3. Agent makes correct tool calls, Reddit MCP server responds with data
4. Platform rejects the response with validation error

### Actual Behavior

Agent run trace shows (run ID: `cmnqllg4c02a6nr01od9wgb4t`):

```
Step 1 � Tool Call:
  reddit_reddit_search_subreddit
  Args: {"subreddit": "canadatravel", "query": "cottage Ontario vacation", "limit": 50, "sort": "relevance"}

Step 2 � Tool Result (ERROR):
  "Tool output validation failed for reddit_reddit_search_subreddit. 
   The tool returned invalid output:
   - result: Required
   
   Returned output: {"content": [{"t... (truncated)"
```

The MCP server returns data in the standard MCP format (`{"content": [...]}`), but the validator expects a `result` field at the top level.

This error cascades � the agent retried with `reddit_reddit_search`, `reddit_reddit_get_subreddit_posts`, all with the same validation failure. Data is coming back from Reddit each time (visible in truncated error messages) but never reaches the agent.

### Expected Behavior

The platform should accept standard MCP tool responses in `content` format and pass the data through to the agent. The MCP protocol returns `{"content": [{type: "text", text: "..."}]}` � this is the correct format per the MCP specification.

### Environment

- Agent: `ivr-reddit-scout` (ID: `cmnqh6czs01aepc01jsvclxnl`)
- Model: `kimi/kimi-k2.5`
- MCP Server: `reddit-no-auth-mcp-server` (eliasbiondo)
- Connection test: PASSES (6 tools found, handshake in 6.8s)
- Tool calls: Agent makes them correctly with proper arguments
- Reddit response: Data returns (visible in truncated error output)

---

## Bug 2: MCP Tool Discovery Does Not Persist

### Steps to Reproduce

1. Create a connection for an MCP-type provider (e.g., Reddit with `authType: "none"`)
2. Run `integration_connection_test` � passes, reports 6 tools found
3. Run `integration_tools_rediscover` � returns `{"rediscovery": []}`
4. Run `integration_tools_list` � returns `{"totalTools": 0, "enabledTools": 0}`

### Actual Behavior

```
integration_connection_test ? "Found 6 tools" ?
integration_tools_rediscover ? {"rediscovery": []} (empty)
integration_tools_list ? {"totalTools": 0}
```

Tested multiple times: deleted connection, recreated, re-tested, re-ran rediscovery. Tools never persist to the database table.

### Expected Behavior

After `integration_connection_test` successfully discovers tools, `integration_tools_rediscover` should persist those tools to the integration tools table so they appear in `integration_tools_list` and can be toggled on/off.

### Comparison with Working Providers

OAuth-type providers (Google Search Console, Google Calendar, Google Drive) have tools properly registered in the integration tools table � these were auto-bootstrapped at workspace creation. MCP-type providers imported from local Cursor MCP config do not get their tools registered.

---

## Additional Context

### What Works

- Agent creation and invocation via API
- Kimi K2.5 model (note: requires `temperature: 1` exactly � any other value returns `"invalid temperature: only 1 is allowed for this model"`)
- MCP server process spawning and handshake
- Reddit MCP tool calls from agent (correct tool names, correct arguments)
- Reddit API responses (data returns successfully)
- OAuth-type provider tools (GSC, Calendar, Drive)

### What Doesn't Work

- Tool output validation for MCP-type providers (blocks data from reaching agent)
- Tool discovery persistence for MCP-type providers (tools not saved to DB)

### Run IDs for Investigation

| Run ID | What Happened |
|--------|---------------|
| `cmnqh6tag01akpc01u433c8q7` | Temperature error (before fix) |
| `cmnqh7c0x01atpc016x738h2d` | Temperature error (cached config) |
| `cmnqh7tn301b0pc01mnhwsou3` | Temperature error (timing) |
| `cmnqh8lam01bbpc01tmrrwzr8` | OpenRouter missing API key |
| `cmnqhb6xj01i5pc01y1tx7zhd` | Kimi works but tools not injected (0 tool calls) |
| **`cmnqllg4c02a6nr01od9wgb4t`** | **Key run � tools called, Reddit responds, output validation rejects** |

### Agent ID

`cmnqh6czs01aepc01jsvclxnl` (slug: `ivr-reddit-scout`)

### Connection ID

`cmnqhd4xn01iopc01jm1cxikk` (Reddit, no auth)
