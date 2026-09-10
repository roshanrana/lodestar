---
id: T-000
title: Walking skeleton: package, data loader, gate, CI, CLI root, page stubs
milestone: M0
risk: medium
tier: T2
complexity: high
reasoning: on
budget: {input_tokens: 80000, tool_calls: 80, wall_clock_min: 120}
depends_on: []
rtm: [FR-001, FR-002, FR-018, NFR-003, NFR-004, NFR-007, NFR-008, NFR-009]
status: todo
---
# T-000 — Walking skeleton

## Goal
A fresh clone runs `uv sync --all-extras`, `uv run python scripts/check.py` passes (ruff, ruff
format, mypy strict host + linux, pytest ≥80% coverage, secrets scan), and `uv run lodestar
dashboard` opens a Streamlit app whose Home page shows one usage chart from the real CSV.

## Spec references
- `docs/design/03-lld.md §1` repository layout (create exactly these paths; later packs fill them).
- `§2` DayRow / Dataset / load() with derived fields and the validation error codes.
- `§7` error taxonomy — implement `lodestar/errors.py` in full now (all codes), even those unused yet.
- `§8` configuration matrix — implement `lodestar/config.py` `Settings` (frozen pydantic model built by `Settings.from_env()`), and `.env.example`.
- `§14` first paragraph — `app/main.py` with `st.navigation`, `app/pages/home.py`, and stub `customer_health.py` / `internal_qbr.py` that render a title and the same usage line chart.
- `§15` plug-in CLI discovery — `lodestar/cli.py`, `lodestar/commands/__init__.py`, `lodestar/commands/dashboard.py` (runs `streamlit run app/main.py --server.port <port>` via subprocess) only.
- `§17` test strategy — test names carry RTM ids.
- Portfolio conventions: copy `scripts/check.py` and `Makefile` shape from the excerpt below; `metrics/render.py` already exists and must be excluded from ruff/mypy like sibling repos.

check.py steps (in order): ruff check; ruff format --check; mypy lodestar; mypy lodestar --platform linux; pytest --cov=lodestar --cov-report=term-missing --cov-fail-under=80; python scripts/secrets_scan.py; later packs append bench/card steps — leave a clearly marked list `STEPS`.

## Scope (files this task may touch)
- pyproject.toml, uv.lock, .python-version, .gitignore, .gitattributes, Makefile, .env.example
- .github/workflows/check.yml
- scripts/check.py, scripts/secrets_scan.py
- lodestar/__init__.py, lodestar/errors.py, lodestar/config.py, lodestar/data.py, lodestar/cli.py
- lodestar/commands/__init__.py, lodestar/commands/dashboard.py
- lodestar/py.typed
- app/main.py, app/pages/home.py, app/pages/customer_health.py, app/pages/internal_qbr.py, app/components/__init__.py
- tests/__init__.py, tests/conftest.py, tests/test_data.py, tests/test_config.py, tests/test_cli.py, tests/test_app_skeleton.py
- metrics/card.json (write the exact JSON from LLD §9)

## Acceptance criteria
- AC1: `uv run python scripts/check.py` exits 0 on Windows host; the same command is what `.github/workflows/check.yml` runs on ubuntu-latest (setup-uv, `uv python install 3.12`, `uv sync --all-extras`).
- AC2: `load()` on the shipped CSV returns 31 rows sorted by date, `deployment == "northstar-flagship-support-ft-v1"`, and derived values `requests_per_ticket` 4.345 / 6.548 (first/last, ±0.001), `tokens_per_request` 1398.0 (first, ±0.1), `cost_per_1m_tokens_usd` 1.116 (first, ±0.001), `week` 1 for day 1 and 5 for day 31, `note_kind` incident/incident/improvement for 08-09/08-21/08-24 and `none` elsewhere.
- AC3: Each error code in LLD §2 is raised by a test fixture: missing file, > 10 000 rows, missing column, extra column, non-numeric cell (detail has row and column), two deployments, duplicate date.
- AC4: `uv run lodestar --help` lists `dashboard`; adding a new module under `lodestar/commands/` with `app` and `NAME` makes it appear without editing `cli.py` (test creates a temp module via `monkeypatch.syspath_prepend` or asserts discovery over a fake package).
- AC5: `streamlit.testing.v1.AppTest.from_file("app/pages/home.py")` runs without exception and renders at least one `plotly_chart`; the two stub pages do the same.
- AC6: `scripts/secrets_scan.py` fails on a file containing `FIREWORKS_API_KEY=fw_` followed by 20+ word characters or `sk-ant-` patterns (test with a temp file) and passes on the repo.
- AC7: `.env.example` contains every variable in LLD §8 with a comment; `.gitignore` covers `.venv/`, `.env`, caches, `htmlcov/`, `.coverage`.
- AC8: mypy strict passes for `lodestar/`; `app/` is excluded from mypy but included in ruff.

## Validation commands (targeted)
- `uv run pytest tests -q`
- `uv run python scripts/check.py`
- `uv run lodestar --help`

## Verification checklist (for the Verifier)
- [ ] scope respected (no files outside the list; `metrics/render.py` untouched)
- [ ] error taxonomy used (no bare `ValueError` escapes `load()`)
- [ ] no sensitive fields logged
- [ ] tests named with RTM IDs
- [ ] dependencies limited to: pydantic>=2.7, typer>=0.12, streamlit>=1.38, plotly>=5.22, pyyaml>=6, httpx>=0.27, fastapi>=0.111, uvicorn>=0.30, mcp>=1.9,<2, python-docx>=1.1; dev: ruff, mypy, pytest, pytest-cov, types-PyYAML
- [ ] CI workflow present and identical in intent to sibling repos

## Threat-model boundary touched
B3 (filesystem inputs): size and row caps enforced before parsing; per-cell validation with row numbers; `operational_note` stored as plain text.

## Handoff (Implementer fills, ≤10 lines)
- Implemented: package skeleton (errors, config, data model/loader, plug-in CLI, dashboard command), CI, check/secrets-scan scripts, Streamlit main+3 page stubs, tests for all ACs.
- Files changed: pyproject.toml, uv.lock, .python-version, Makefile, .env.example, .github/workflows/check.yml, scripts/check.py, scripts/secrets_scan.py, lodestar/{__init__,errors,config,data,cli,py.typed}.py, lodestar/commands/{__init__,dashboard}.py, app/main.py, app/pages/{home,customer_health,internal_qbr}.py, app/components/__init__.py, tests/{__init__,conftest,test_data,test_config,test_cli,test_app_skeleton}.py, metrics/card.json.
- Tests run: `uv run python scripts/check.py` → ruff/format/mypy(host+linux)/pytest/secrets-scan all green; 25 passed, coverage 94.58% (≥80% required).
- Deviations from pack: dropped `readme = "README.md"` from pyproject (file doesn't exist, out of scope); added `docs` and `scripts/shipyard` to ruff `extend-exclude` (pre-existing, out-of-scope files ruff otherwise fails on — same treatment as `metrics/render.py`).
- Open questions: none.
- Budget actual: in-tokens ~103000, tool calls 52, attempts 1.
