---
id: T-006
title: Internal QBR page
milestone: M3
risk: medium
tier: T2
complexity: high
reasoning: on
budget: {input_tokens: 80000, tool_calls: 80, wall_clock_min: 120}
depends_on: [T-003, T-004]
rtm: [FR-010]
status: todo
---
# T-006 — Internal QBR page

## Goal
A candid, actionable page for Fireworks account, engineering and leadership teams: internal
health with methodology, trends, top-3 risks, expansion pipeline with confidence, internal
dependencies / support burden / competitive notes / decisions needed, next actions with owners
and dates, stakeholder coverage.

## Spec references
- `docs/design/03-lld.md §14` "Internal page order (FR-010)" — implement exactly: red banner "INTERNAL — Fireworks only. Not for customer distribution."; if `settings.internal_enabled` is False render only a notice and stop; tiles (internal composite + band, operational composite + band, relationship score, weakest pillar); health breakdown table (pillar, weight, L7 value per metric, score, trend arrow) with methodology expander; five trend charts (adoption: automation + escalation; spend: daily spend + cost per automated ticket; reliability: availability with SLO hline + error rate; latency: P50/P95 with budget hline; quality: grounding, eval pass, CSAT) with incident and F7/L7 window shading; top-3 risks as cards (title, category, severity × likelihood, evidence key figures, mitigation); expansion pipeline table (brand, stage, fit, confidence, est. monthly inference, annual value, blockers, target date, owner) + `fit_confidence_scatter`; four-column panel: dependencies, support burden, competitive notes (from brands' `competitive_note` and R-COMP-1), decisions needed (decision, owner, by); next actions table (action, owner, due, visibility); stakeholder coverage table; claims table.
- Data access: `AccountView.build(..., Audience.INTERNAL)` behind `@st.cache_data` keyed on `(csv_path, "internal")`.
- Use only `app/components/charts.py` and `tiles.py` for figures and tiles.

## Scope (files this task may touch)
- app/pages/internal_qbr.py
- tests/test_page_internal.py

## Acceptance criteria
- AC1: `AppTest.from_file("app/pages/internal_qbr.py", default_timeout=60).run()` has no exception; ≥ 6 `plotly_chart`, 4 `metric`, and the banner text present.
- AC2: Rendered text contains the internal composite (77.5 ±0.3 formatted to one decimal), the string "Kestrel Athletics", "75%" or "75" near "confidence", and the three decision owners.
- AC3: With `LODESTAR_INTERNAL_ENABLED=0` (monkeypatched env before import), the page renders a notice and zero `plotly_chart`.
- AC4: Top-3 risk cards are R-EXEC-1, R-TECH-1, R-REL-1 in that order (assert by title text order).
- AC5: Next actions table has 8 rows with a visibility column.

## Validation commands (targeted)
- `uv run pytest tests/test_page_internal.py -q`
- `uv run ruff check app && uv run ruff format --check app`

## Verification checklist (for the Verifier)
- [ ] scope respected
- [ ] no hand-typed business numbers in the page
- [ ] internal-enabled gate respected
- [ ] tests named with RTM IDs
- [ ] no new dependencies

## Threat-model boundary touched
B1 (internal side): control C-13 (`LODESTAR_INTERNAL_ENABLED` gate).

## Handoff
- Implemented: Full internal QBR page (banner, gate, tiles, health breakdown+methodology, 5 trend charts, top-3 risk cards, expansion pipeline+scatter, dependencies/support/competitive/decisions panel, next actions, stakeholders, claims), all data via `AccountView.build(..., Audience.INTERNAL)` cached on `(csv_path, "internal")`.
- Files changed: app/pages/internal_qbr.py, tests/test_page_internal.py, docs/tasks/T-006-internal-page.md (handoff only)
- Tests run: `uv run pytest tests/test_page_internal.py -q` → 5 passed; `uv run ruff check`/`ruff format --check` on my two files → clean; full `uv run pytest -q` → 167 passed, 1 pre-existing failure in tests/test_page_customer.py (T-005's in-flight customer_health.py, outside my scope, untouched by me)
- Deviations from pack: none
- Open questions: none
- Budget actual: in-tokens ~90000, tool calls ~38, attempts 1
