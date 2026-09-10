---
id: T-004
title: Plotly chart and tile components
milestone: M2
risk: low
tier: T2
complexity: normal
reasoning: off
budget: {input_tokens: 40000, tool_calls: 40, wall_clock_min: 45}
depends_on: [T-002]
rtm: [FR-010, FR-011]
status: todo
---
# T-004 — Plotly chart and tile components

## Goal
Pure functions in `app/components/` return consistent, polished Plotly figures and Streamlit
tile helpers that both pages will use, so the pages contain layout only.

## Spec references
- `docs/design/03-lld.md §14` last paragraph — `charts.py` functions: `line_with_incidents(dates, series, incident_dates, title, y_title, hline=None, hline_label="")`, `dual_axis(dates, left: dict, right: dict, incident_dates, title, left_title, right_title)`, `pillar_bar(report: HealthReport)`, `fit_confidence_scatter(brands)`; palette primary `#2563eb`, secondary `#0d9488`, warn `#d97706`, bad `#dc2626`, muted `#64748b`; template `plotly_white`; incident shading `rgba(220,38,38,0.08)` as full-height vrects; optional window shading `rgba(37,99,235,0.06)` for F7/L7 via `windows: list[tuple[date, date, str]] | None`.
- `tiles.py`: `kpi_tile(col, label, value, caption, delta=None, tone: Literal["good","warn","bad","neutral"]="neutral")` rendering `st.metric` plus a small caption; `band_badge(band)` returning coloured markdown for healthy/watch/at_risk; `assumption_banner(assumptions: AssumptionSet)` listing A-1..A-3 with owners.
- Figures must set `margin=dict(l=10, r=10, t=40, b=10)`, `height=320`, legend horizontal at top, hover unified on x.

## Scope (files this task may touch)
- app/components/charts.py, app/components/tiles.py
- tests/test_charts.py

## Acceptance criteria
- AC1: Each chart function returns a `plotly.graph_objects.Figure`; `line_with_incidents` adds one vrect per incident date and one hline when `hline` is given (tested by inspecting `fig.layout.shapes`).
- AC2: `pillar_bar` shows five bars with values equal to the report's pillar scores and a 80/65 band reference; `fit_confidence_scatter` accepts brands whose `confidence_pct` may be None (customer) and then plots fit only.
- AC3: `dual_axis` produces two y-axes and both series.
- AC4: Palette and template constants applied on every figure (test asserts `fig.layout.template.layout` origin is `plotly_white` via `fig.layout.template` name comparison or the function sets `template="plotly_white"` and test checks `fig.layout.template is not None`).
- AC5: `tiles.py` functions run inside `AppTest` without exception (smoke test using a tiny page file written to tmp_path).

## Validation commands (targeted)
- `uv run pytest tests/test_charts.py -q`
- `uv run ruff check app`

## Verification checklist (for the Verifier)
- [ ] scope respected
- [ ] no numeric business values hard-coded (only layout constants)
- [ ] no sensitive fields logged
- [ ] tests named with RTM IDs
- [ ] no new dependencies

## Threat-model boundary touched
none

## Handoff (Implementer fills, ≤10 lines)
- Implemented: charts.py (line_with_incidents, dual_axis, pillar_bar, fit_confidence_scatter) + tiles.py (kpi_tile, band_badge, assumption_banner) per LLD §14 palette/layout contract.
- Files changed: app/components/charts.py (new), app/components/tiles.py (new), tests/test_charts.py (new).
- Tests run: `uv run pytest tests/test_charts.py -q` → 15 passed; `uv run pytest -q` (full suite) → 93 passed; `uv run ruff check app tests` → clean; `uv run ruff format --check app tests` → clean.
- Deviations from pack: none.
- Open questions: none.
- Budget actual: in-tokens ~65000, tool calls 20, attempts 1.
