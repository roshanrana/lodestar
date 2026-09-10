# Lodestar MCP server

`lodestar mcp` runs a [FastMCP](https://github.com/modelcontextprotocol/python-sdk) server named
`lodestar` over stdio. It exposes six read-only tools that all read from the same `AccountView`
pipeline (`lodestar/view.py`) used by the Streamlit dashboards and the FastAPI app
(`lodestar/api.py`), so every surface returns identical, audience-scrubbed numbers.

## Registering the server

### Claude Desktop

Add an entry to `claude_desktop_config.json` (Settings -> Developer -> Edit Config):

```json
{
  "mcpServers": {
    "lodestar": {
      "command": "uv",
      "args": ["run", "--directory", "<repo>", "lodestar", "mcp"]
    }
  }
}
```

Replace `<repo>` with the absolute path to this repository (e.g.
`C:\Code-Central\lodestar` or `/Users/you/lodestar`). Restart Claude Desktop after saving.

### Claude Code

Add the same server to your Claude Code MCP configuration (`.mcp.json` in the repo, or your
user-level MCP settings):

```json
{
  "mcpServers": {
    "lodestar": {
      "command": "uv",
      "args": ["run", "--directory", "<repo>", "lodestar", "mcp"]
    }
  }
}
```

Both hosts launch the server as a subprocess and speak MCP over its stdin/stdout, so no port or
network configuration is needed.

Alternatively, register it with the Claude Code CLI directly instead of hand-editing JSON:

```bash
claude mcp add lodestar -- uv run --directory <repo> lodestar mcp
```

This writes the same `command`/`args` pair shown above into your Claude Code MCP configuration.

## The audience rule

Every tool takes `audience: "internal" | "customer" = "customer"`. `AccountView.build` scrubs
every internal-tagged field to `None` for `audience="customer"` before it ever reaches a tool —
there is no separate customer code path to keep in sync.

Requesting `audience="internal"` while the server is configured with
`LODESTAR_INTERNAL_ENABLED=0` raises `LodestarError(Code.INTERNAL_DISABLED)`. FastMCP converts
that into an MCP **tool error result** (`isError: true`) rather than a raised exception; the
error text starts with the code name, so a caller can match on `"INTERNAL_DISABLED"` in
`result.content[0].text`. Internal audience access defaults to enabled (`LODESTAR_INTERNAL_ENABLED`
defaults to `1`) — see `lodestar/config.py` / LLD §8 for the full configuration matrix.

## Tools

| Tool | Args | Returns |
|---|---|---|
| `get_health_score` | `audience: "internal"\|"customer" = "customer"` | The operational `HealthReport` (scrubbed for `customer`). |
| `get_metric_trend` | `metric: <numeric DayRow field name>`, `audience = "customer"` | `{metric, dates, values, l7_mean, f7_mean}` — daily series plus the last-7/first-7 day means. |
| `get_claims_check` | — | The three headline-claim checks (`list[ClaimCheck]`); no internal fields exist on this model, so there is no audience argument. |
| `get_risks` | `audience = "customer"`, `top_n: int = 3` | The top `top_n` risks visible to `audience`, ranked by severity x likelihood. |
| `get_expansion_pipeline` | `audience = "customer"` | The ranked brand expansion pipeline (scrubbed for `customer`). |
| `get_next_actions` | `audience = "customer"` | The joint/internal next actions visible to `audience`. |

`metric` for `get_metric_trend` must be one of the numeric `DayRow` fields: `requests`,
`input_tokens`, `output_tokens`, `total_tokens`, `spend_usd`, `p50_latency_ms`, `p95_latency_ms`,
`error_rate_pct`, `availability_pct`, `tier1_tickets`, `automated_tier1_tickets`,
`automation_rate_pct`, `avg_handle_time_min`, `csat_score`, `grounded_answer_rate_pct`,
`quality_eval_pass_rate_pct`, `escalation_rate_pct`, `requests_per_ticket`, `tokens_per_request`,
`cost_per_automated_ticket_usd`, `cost_per_1m_tokens_usd`. An unrecognised value is rejected by
MCP input-schema validation before the tool runs (`isError: true`).

## Related: the HTTP API

`lodestar api` (see `lodestar/api.py`) exposes the same data over `GET /api/{audience}/...`
routes (`health`, `trends?metric=`, `claims`, `risks?top_n=`, `pipeline`, `actions`) plus a plain
`GET /healthz`, for hosts that prefer HTTP/JSON over MCP. Both surfaces share the same cached
`_view(audience)` in `lodestar/mcp_server.py`, so results never drift between them.
