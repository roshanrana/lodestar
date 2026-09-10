# Lodestar — Showcase

A guided tour of the features, with the commands to run. [OVERVIEW.md](OVERVIEW.md) has the
reasoning behind the design. Run everything from the repository root; `uv sync --all-extras` was
already done in this environment.

Every command below was run by the implementer while writing this file, and its observed output
is summarised in one line underneath it.

## 1. The gate

```bash
uv run python scripts/check.py
```
> Ran ruff lint/format, mypy strict (host and `--platform linux`), pytest, the secrets scan, `lodestar bench`, bench drift and card drift: **251 passed, 97.47% coverage, all checks passed.**

## 2. The dashboard, both pages

```bash
uv run lodestar dashboard --port 8511
```

Open <http://localhost:8511/> for the landing page, then:

- <http://localhost:8511/customer_health> — **Customer · Account health.** Look at the four
  outcome tiles (automation, AHT, CSAT, operational health) each with its caveat caption, the
  usage trend with incident shading, the reliability row with the proposed-SLO line, the "Known
  caveats" expander showing the claims table, and the methodology expander with the full pillar
  arithmetic. Nothing here mentions relationship health, margin, competitive notes, or expansion
  confidence — those fields are scrubbed before the page ever receives them.
- <http://localhost:8511/internal_qbr> — **Internal · QBR.** Look at the red
  "INTERNAL — Fireworks only" banner, the internal-composite and relationship-score tiles, the
  five pillar trend charts, the expansion-pipeline table (with fit, confidence and blockers), and
  the dependencies / support burden / competitive notes / decisions-needed panel — all internal
  visibility.

> Started the dashboard on port 8511 (the orchestrator's own instance on 8501 was left running).
> `curl -s -o /dev/null -w "%{http_code}"` against `/`, `/customer_health` and `/internal_qbr` all
> returned **200**, then the process was stopped.

## 3. The CLI

```bash
uv run lodestar score
```
> `Operational health: 80.2 / 100 (healthy)` with all five pillars listed (adoption 80.2,
> reliability 87.2, latency 78.5, quality 83.1, spend_efficiency 65.8, the last one `declining`).

```bash
uv run lodestar score --audience internal
```
> Same operational figures — the internal-only relationship pillar is not printed by this command
> at all, only the shared operational composite.

```bash
uv run lodestar claims
```
> Printed all three headline claims: automation `VERIFIED_WITH_CAVEAT` (month mean 67.92% below
> 70%, but L7 mean 70.07% and exit-day 70.8% clear it), AHT and CSAT both `REQUIRES_BASELINE`
> against the assumed pre-deployment baselines.

```bash
uv run lodestar risks --top 2
```
> Printed the top 2 risks by severity × likelihood: `R-EXEC-1` (four brands, four stacks, four
> teams) and `R-TECH-1` (usage growth decoupled from ticket volume), each with its mitigation.

```bash
uv run lodestar pipeline
```
> Printed all four brands ranked by fit score: Kestrel Athletics 87.0 (pilot), Little Compass
> 60.0, Lumen Beauty 58.0, Meridian Home 57.0, each with its proposed window.

```bash
uv run lodestar bench --check
```
> `metrics\headline.json is current` — exit 0.

## 4. EBR generation, offline and with a provider

```bash
uv run lodestar ebr --provider offline
```
> Wrote `docs\ebr\northstar-ebr-brief.md` and `docs\ebr\northstar-ebr-brief.docx` using the
> template's own prose (no network call).

```bash
uv run lodestar ebr --provider fireworks
```
> No `FIREWORKS_API_KEY` is set in this environment, which is the reviewer's default state too.
> Exited 1 with `PROVIDER_CONFIG: missing required environment variable FIREWORKS_API_KEY` —
> the config check runs before any HTTP call, per LLD §12. Set `FIREWORKS_API_KEY` (and
> optionally `LODESTAR_FIREWORKS_MODEL`) to exercise the live path instead; the leak guard still
> scans whatever the provider returns before it reaches the rendered EBR.

## 5. MCP registration and a tool call from Claude Code

Register the server with the Claude Code CLI:

```bash
claude mcp add lodestar -- uv run --directory <repo> lodestar mcp
```

This launches `uv run lodestar mcp` as a stdio subprocess; full registration reference (Claude
Desktop and Claude Code) is in [mcp.md](mcp.md). To see the tool contract without a host attached,
this in-memory session drives the same `FastMCP` server object the `lodestar mcp` command runs,
the way `tests/test_mcp.py` does:

```python
import asyncio, json
from mcp.shared.memory import create_connected_server_and_client_session
from lodestar import mcp_server

async def main():
    async with create_connected_server_and_client_session(mcp_server.server._mcp_server) as client:
        tools = await client.list_tools()
        print("tools:", sorted(t.name for t in tools.tools))
        result = await client.call_tool("get_health_score", {"audience": "customer"})
        payload = json.loads(result.content[0].text)
        print("operational_composite:", payload["operational_composite"])
        print("relationship_score present:", payload.get("relationship_score"))

asyncio.run(main())
```
> Ran this snippet. Output: `tools: ['get_claims_check', 'get_expansion_pipeline',
> 'get_health_score', 'get_metric_trend', 'get_next_actions', 'get_risks']` (all 6),
> `operational_composite: 80.23454208755983` (rounds to the published 80.2), and
> `relationship_score present: None` — the internal-only field is absent from the
> `audience="customer"` tool result, over the same in-process MCP wire protocol a real host uses.

## 6. The API, with curl

```bash
uv run lodestar api --port 8766
```

```bash
curl -s http://127.0.0.1:8766/healthz
curl -s http://127.0.0.1:8766/api/customer/health
curl -s "http://127.0.0.1:8766/api/customer/trends?metric=csat_score"
curl -s http://127.0.0.1:8766/api/customer/claims
curl -s http://127.0.0.1:8766/api/customer/risks
curl -s http://127.0.0.1:8766/api/customer/pipeline
curl -s http://127.0.0.1:8766/api/customer/actions
```
> All seven requests returned **200**: `/healthz` gave `{"status":"ok"}`; `/api/customer/health`
> and `/api/internal/health` returned the same `HealthReport` JSON the CLI and dashboard use
> (internal fields present only on the internal route); the remaining customer routes each
> returned their expected JSON body. The API process was then stopped.

## 7. The leak test and the guard demo

```bash
uv run pytest tests/test_leak.py tests/test_guard.py -q
```
> **34 passed.** `test_leak.py` proves the customer `AccountView` has zero populated
> internal-tagged fields while the internal view has at least one (so the negative check is not
> vacuous), and that every field reachable from `AccountView` is explicitly tagged one way or the
> other. `test_guard.py` proves the deny-list guard rejects each of the 16 internal terms
> (confidence, margin, competitor/competitive, renewal, churn, sentiment, blocker, arr, gross
> margin, unbilled, incumbent, and four named vendors), on a word-boundary match that leaves
> unrelated words like "arrival" alone, including inside a hostile prompt-injection attempt.

## 8. Screenshots regeneration

```bash
uv sync --all-extras --group screenshots
uv run playwright install chromium   # one-time
uv run lodestar dashboard --port 8511
uv run python scripts/screenshots.py --base http://localhost:8511
```
> Ran this against a dashboard instance on port 8511 (leaving the orchestrator's 8501 instance
> untouched). Wrote `docs/assets/home.png` (75 KB), `docs/assets/customer-account-health.png`
> (497 KB), and `docs/assets/internal-qbr.png` (591 KB) at 1440px, light theme, after every
> Plotly chart on each page had rendered. The 8511 dashboard process was then stopped.

## 9. Query the code graph

```bash
graphify update .
graphify affected "scrub" --depth 2
```
> `graphify update .` re-extracted 197 files in ~7s (AST only, no LLM, offline) and rebuilt
> `graphify-out/` with 1901 nodes, 3622 edges, 164 communities. `graphify affected "scrub"
> --depth 2` listed every caller and importer of the audience-scrubbing boundary — `view.py`,
> `build_view()`, eight `scrub`-specific tests in `test_audience.py`, and every module that
> imports `view.py` (`customer_health.py`, `internal_qbr.py`, `bench.py`, `mcp_server.py`,
> `ebr.py`, the four `commands/*` modules, and five more test files) — the exact blast radius a
> change to `scrub()` should make you re-check. See [`docs/graph/README.md`](graph/README.md)
> for the full query reference.
