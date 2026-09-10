---
id: T-007
title: MCP server and FastAPI API
milestone: M3
risk: high
tier: T2
complexity: high
reasoning: on
budget: {input_tokens: 80000, tool_calls: 80, wall_clock_min: 120}
depends_on: [T-003]
rtm: [FR-015, FR-016, NFR-005]
status: todo
---
# T-007 — MCP server and FastAPI API

## Goal
Any MCP host (Claude Desktop, Claude Code) and any HTTP client can query the account with
audience scoping enforced by the same `AccountView.build`.

## Spec references
- `docs/design/03-lld.md §13` — FastMCP server named `lodestar`; six tools with `audience: Literal["internal","customer"] = "customer"`; `get_metric_trend(metric: Literal[...numeric DayRow fields...], audience)` returns `{metric, dates, values, l7_mean, f7_mean}`; `get_risks(audience, top_n=3)`; internal with `LODESTAR_INTERNAL_ENABLED=0` → `INTERNAL_DISABLED` surfaced as a tool error (raise `LodestarError`; FastMCP converts to an error result). FastAPI: `GET /healthz`; `GET /api/{audience}/health|trends?metric=|claims|risks?top_n=|pipeline|actions`; `LodestarError` → 400 input/config, 403 INTERNAL_DISABLED, 502 PROVIDER_HTTP, body `{code, message, detail}`.
- `§15` — `lodestar/commands/mcp.py` (`NAME="mcp"`, runs the server over stdio) and `lodestar/commands/api.py` (`NAME="api"`, `uvicorn.run(app, port=settings.api_port)`).
- Pattern reference: the in-memory MCP client test approach used in sibling repos — `mcp.shared.memory.create_connected_server_and_client_session(server._mcp_server)` or FastMCP's equivalent in the pinned version; list tools, call each, parse JSON text content.
- Views are built once per process and cached with `functools.lru_cache` keyed on audience; provide `reset_cache()` for tests.

## Scope (files this task may touch)
- lodestar/mcp_server.py, lodestar/api.py
- lodestar/commands/mcp.py, lodestar/commands/api.py
- tests/test_mcp.py, tests/test_api.py
- docs/mcp.md (how to register the server in Claude Desktop / Claude Code `mcpServers` config with `uv run lodestar mcp`)

## Acceptance criteria
- AC1: In-memory MCP session lists exactly six tools and calls each successfully with default audience; customer results, walked recursively with `field_visibility`, contain zero populated internal fields (reuse the walker from `tests/test_leak.py` by importing a helper you place in `tests/conftest.py` — conftest is in T-000's scope; if you need it, add `tests/leakwalk.py` instead and list it in the handoff).
- AC2: `get_metric_trend("p95_latency_ms")` returns 31 values and l7_mean 1463.1 ±0.1; an invalid metric name is rejected.
- AC3: With `LODESTAR_INTERNAL_ENABLED=0`, `get_health_score(audience="internal")` returns an MCP error result whose text contains `INTERNAL_DISABLED`; API returns 403 with `code == "INTERNAL_DISABLED"`.
- AC4: `fastapi.testclient` covers all six routes for both audiences; `/api/customer/health` JSON has `internal_composite` null and `relationship_inputs` null; OpenAPI at `/openapi.json` lists the routes.
- AC5: `lodestar --help` lists `mcp` and `api`.

## Validation commands (targeted)
- `uv run pytest tests/test_mcp.py tests/test_api.py -q`
- `uv run mypy lodestar`

## Verification checklist (for the Verifier)
- [ ] scope respected
- [ ] error taxonomy used and mapped to HTTP codes as specified
- [ ] no sensitive fields logged
- [ ] tests named with RTM IDs
- [ ] no new dependencies (mcp, fastapi, uvicorn already declared)

## Threat-model boundary touched
B4 (MCP host / HTTP client ↔ handlers): C-13 internal gate, C-14 pydantic-validated inputs; B1: C-02 via `AccountView.build`.

## Handoff (Implementer fills, ≤10 lines)
- Implemented: FastMCP server (6 tools) + FastAPI app sharing a cached `_view(audience)`, plus `mcp`/`api` CLI commands and docs/mcp.md.
- Files changed: lodestar/mcp_server.py, lodestar/api.py, lodestar/commands/mcp.py, lodestar/commands/api.py, tests/test_mcp.py, tests/test_api.py, docs/mcp.md.
- Tests run: `uv run pytest tests/test_mcp.py tests/test_api.py -q` → 26 passed; `uv run pytest -q` → 168 passed; `uv run mypy lodestar` → success; `uv run ruff check . && uv run ruff format --check .` → clean.
- Deviations from pack: used `functools.cache` (built on `lru_cache(maxsize=None)`, same `cache_clear()` API) instead of `@functools.lru_cache` directly, to satisfy ruff UP033 while keeping identical memoization/reset semantics.
- Open questions: none.
- Budget actual: in-tokens ~90k, tool calls ~45, attempts 1.
