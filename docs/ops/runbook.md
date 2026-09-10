# Operations runbook — Lodestar

Everything here assumes `uv sync --all-extras` has been run once from the repo root and every
command is prefixed `uv run` (or run inside `uv run`'s environment). No command below needs an
API key unless you have deliberately switched the LLM provider.

## Start / stop

| Surface | Start | Default port | Stop |
|---|---|---|---|
| Dashboard | `uv run lodestar dashboard` | `LODESTAR_DASHBOARD_PORT` = 8501 | Ctrl+C in the terminal running it |
| API | `uv run lodestar api` | `LODESTAR_API_PORT` = 8765 | Ctrl+C in the terminal running it |
| MCP server | `uv run lodestar mcp` | stdio, no port | Ctrl+C, or let the MCP host close the subprocess |

`lodestar dashboard` runs `streamlit run app/main.py` as a subprocess; `--port` on the `dashboard`
command overrides `LODESTAR_DASHBOARD_PORT` for that run. `lodestar api --port` does the same for
the FastAPI app. The MCP server speaks over its own stdin/stdout, so a host (Claude Desktop,
Claude Code) manages its lifecycle as a subprocess — see `docs/mcp.md` for registration.

To confirm the API is up: `curl http://localhost:8765/healthz`. To confirm the dashboard is up,
open `http://localhost:8501` — the landing page renders within the NFR-002 budget of 10 seconds
on a laptop.

## Common failures and fixes

Every runtime failure raises a `LodestarError` with one of the closed codes in LLD §7
(`docs/design/03-lld.md`). The code name is always in the surfaced message.

| Error code | When it happens | Fix |
|---|---|---|
| `DATA_NOT_FOUND` | The CSV at `LODESTAR_CSV` (default `data/northstar_flagship_30_day_metrics.csv`) does not exist | Check the path, or unset `LODESTAR_CSV` to use the shipped default |
| `DATA_TOO_LARGE` | CSV is over 5 MB or has over 10 000 rows | Point `LODESTAR_CSV` at a smaller extract; the caps are a hardening control, not configurable |
| `DATA_SCHEMA` | A column is missing, extra, or out of the expected order | Match the header exactly to `data/northstar_flagship_30_day_metrics.csv`; the error names the missing/extra columns |
| `DATA_VALUE` | A cell cannot be parsed as its expected type | The error names the row number and column; fix that cell |
| `DATA_MULTI_DEPLOYMENT` | The CSV contains more than one distinct `deployment` value | Split the file so each run covers exactly one deployment (multi-deployment CSVs are out of scope, per the requirements) |
| `DATA_DUPLICATE_DATE` | Two rows share the same `date` | Deduplicate the source file; the error names the offending date |
| `CONTEXT_INVALID` | A YAML file under `LODESTAR_DATA_DIR` has an unknown key or a bad field | The error names the file and field; fix the YAML against the model it mirrors (`context.py`, `risks.py`, `brands.py`) |
| `PROVIDER_CONFIG` | The selected `LODESTAR_LLM_PROVIDER` is missing its required API key | Set `FIREWORKS_API_KEY` or `ANTHROPIC_API_KEY` for the provider you selected, or switch back to `offline` |
| `PROVIDER_HTTP` | The live provider returned a non-2xx response | Retryable: check the provider's status and the model name (`LODESTAR_FIREWORKS_MODEL` / `LODESTAR_ANTHROPIC_MODEL`); the EBR falls back to offline template prose if the guard rejects the output |
| `LEAK_GUARD` | Generated prose contained an internal term or an internal-tagged figure | Not user-fixable at runtime by design: the offline template text is used instead. If this appears in your own provider output during development, check what internal figures reached the prompt |
| `INTERNAL_DISABLED` | An `audience="internal"` call was made while `LODESTAR_INTERNAL_ENABLED=0` | Set `LODESTAR_INTERNAL_ENABLED=1` if internal access should be allowed in this environment |

## Switching providers

The LLM provider only affects the optional executive-narrative polish in `lodestar ebr`; every
other surface (dashboards, MCP, API, bench) never calls a provider. Selection is one environment
variable:

```bash
# offline (default) — deterministic, no network, no key
LODESTAR_LLM_PROVIDER=offline uv run lodestar ebr

# Fireworks (OpenAI-compatible chat completions)
LODESTAR_LLM_PROVIDER=fireworks FIREWORKS_API_KEY=... uv run lodestar ebr
# optional overrides:
#   LODESTAR_FIREWORKS_MODEL=accounts/fireworks/models/llama-v3p1-70b-instruct
#   LODESTAR_FIREWORKS_BASE_URL=https://api.fireworks.ai/inference/v1

# Anthropic (optional extra)
LODESTAR_LLM_PROVIDER=anthropic ANTHROPIC_API_KEY=... uv run lodestar ebr
# optional override: LODESTAR_ANTHROPIC_MODEL=claude-sonnet-5
```

Keys are read from the environment only and are never written to the repo, never logged, and
never included in `.env` (which is gitignored) — `.env.example` documents every variable name
with no values. A missing key for the selected provider raises `PROVIDER_CONFIG` before any
network call is attempted, so a misconfigured provider fails closed rather than falling back
silently.

## Regenerating bench, card, EBR, and screenshots

```bash
uv run lodestar bench                      # writes metrics/headline.json (deterministic; run twice to confirm no diff)
uv run python metrics/render.py            # regenerates docs/assets/metrics.svg and the README results block
uv run python metrics/render.py --check    # verifies the README/SVG have not drifted from headline.json (no write)
uv run lodestar ebr                        # writes docs/ebr/northstar-ebr-brief.md and .docx
```

Screenshots are not part of the gate and need the optional `screenshots` dependency group and a
running dashboard:

```bash
uv sync --all-extras --group screenshots
uv run playwright install chromium
uv run lodestar dashboard          # in one terminal, left running
uv run python scripts/screenshots.py --base http://localhost:8501   # in another terminal
```

This writes `docs/assets/home.png`, `docs/assets/customer-account-health.png` and
`docs/assets/internal-qbr.png` at 1440px wide in light theme, waiting for every Plotly chart on
each page to render before capturing.

## Running the gate

```bash
uv run python scripts/check.py
```

Runs, in order, and stops at the first failure: `ruff check`, `ruff format --check`, `mypy
lodestar` (host platform), `mypy lodestar --platform linux`, `pytest` with an 80% coverage floor
on `lodestar/`, the secrets-pattern scan, `lodestar bench`, a `git diff --exit-code` on
`metrics/headline.json` (bench determinism), and `metrics/render.py --check` (README/SVG drift).
It prints `all checks passed` only if every step succeeds. CI runs the identical command.

## Where evidence lives

`docs/evidence/ledger.jsonl` is a hash-chained, append-only log of every task's verification
evidence (one entry per task/gate event); `docs/evidence/README.md` describes its format.
`STATE.md` at the repo root carries the task log, budget ledger and gate log for the whole
Shipyard build. Task packs and their verifier verdicts live under `docs/tasks/`.
