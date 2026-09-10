---
id: T-009
title: Bench, card, CLI commands (score, claims, risks, pipeline, bench)
milestone: M4
risk: low
tier: T2
complexity: normal
reasoning: off
budget: {input_tokens: 40000, tool_calls: 40, wall_clock_min: 45}
depends_on: [T-007, T-008]
rtm: [FR-017, FR-018, NFR-006]
status: todo
---
# T-009 — Bench, card, CLI commands

## Goal
`lodestar bench` writes a deterministic `metrics/headline.json`; `metrics/render.py` renders the
README card; `check.py` gains bench-drift and card-drift steps; the remaining CLI commands exist.

## Spec references
- `docs/design/03-lld.md §9` — `run_bench()` KPIs (health_score, claims_verified, leak_check "0 of N" where N = `internal_field_count(AccountView)`, incidents_detected, weakest_pillar, cost_per_automated_ticket), bars (five pillar scores, max 100), facts rows (MCP tools exercised = 6 via in-memory session; API routes count from `api.app.routes` filtered to `/api/`; EBR sections 8 of 8 from an offline render; days below SLO 27 of 31; request growth vs automated growth; one `pending` row "Live provider narrative"). `card.json` already exists from T-000; `render.py` requires keys `kpis`, `bars`, `facts` and accents from {teal, blue, amber, violet, red}.
- `§15` — commands `score [--audience] [--json]`, `claims [--json]`, `risks [--audience] [--top 3]`, `pipeline [--audience]`, `bench [--check]` (`--check` exits 1 if the file would change). Output via `rich`-free plain `typer.echo` tables or JSON.
- `scripts/check.py` `STEPS`: append `("bench", [uv run lodestar bench])`, `("bench drift", git diff --exit-code -- metrics/headline.json)`, `("card drift", uv run python metrics/render.py --check)`.
- README block: `metrics/render.py` splices between `<!-- metrics:start -->` and `<!-- metrics:end -->`; add those markers with a `## Results` placeholder to README.md if absent (T-010 writes the rest of the README; touch only the marker block).

## Scope (files this task may touch)
- lodestar/bench.py
- lodestar/commands/score.py, lodestar/commands/claims.py, lodestar/commands/risks.py, lodestar/commands/pipeline.py, lodestar/commands/bench.py
- metrics/headline.json, docs/assets/metrics.svg (generated), README.md (marker block only)
- scripts/check.py (STEPS list only)
- tests/test_bench.py, tests/test_commands.py

## Acceptance criteria
- AC1: `run_bench()` twice returns equal dicts; `lodestar bench` then `lodestar bench --check` exits 0; a mutated headline makes `--check` exit 1.
- AC2: `python metrics/render.py` succeeds and `--check` is clean afterwards; the SVG and README block are committed.
- AC3: KPI values on the shipped data: health_score "80.2 / 100" (±0.3 rendered to one decimal), claims_verified "1 of 3 from telemetry alone", leak_check "0 of N" with N ≥ 10, incidents_detected "2 of 2", weakest_pillar "Spend efficiency 65.8" (±0.3), cost_per_automated_ticket "$0.0124".
- AC4: `lodestar score --json`, `claims --json`, `risks --audience customer`, `pipeline` all exit 0; customer JSON from `score --audience customer` has no populated internal fields (reuse the leak walker).
- AC5: `uv run python scripts/check.py` passes end to end.

## Validation commands (targeted)
- `uv run pytest tests/test_bench.py tests/test_commands.py -q`
- `uv run python scripts/check.py`

## Verification checklist (for the Verifier)
- [ ] scope respected (README touched only inside markers)
- [ ] headline.json has no timestamps or machine-specific paths
- [ ] no sensitive fields logged
- [ ] tests named with RTM IDs
- [ ] no new dependencies

## Threat-model boundary touched
none

## Handoff (Implementer fills, ≤10 lines)
- Implemented: lodestar/bench.py (run_bench/write_headline/check_headline) + score/claims/risks/pipeline/bench CLI commands; check.py and README wired; headline.json/metrics.svg generated.
- Files changed: lodestar/bench.py, lodestar/commands/{score,claims,risks,pipeline,bench}.py, scripts/check.py, README.md (new), metrics/headline.json, docs/assets/metrics.svg, tests/test_bench.py, tests/test_commands.py
- Tests run: `uv run pytest tests/test_bench.py tests/test_commands.py -q` → 27 passed; `uv run mypy lodestar` → success; `uv run ruff check . && uv run ruff format --check .` → clean; AC1-AC4 verified manually (bench idempotent, mutated headline fails --check, KPIs match §9 exactly, score/claims/risks/pipeline exit 0, customer JSON leak-clean).
- Deviations from pack: "request vs automated growth" fact reports the true computed ratio "×1.45" (requests_growth_ratio=1.4455 rounds to 1.45), not the pack's illustrative "×1.44"; automated ratio ×1.15 matches exactly.
- Open questions: AC5 (`uv run python scripts/check.py`) fails only at the pre-existing `pytest` step on `tests/test_ebr.py::test_fr014_committed_ebr_matches_a_fresh_offline_render` — confirmed via `git stash` that this failure predates this task (docs/ebr files are outside T-009 scope) and blocks the run before it reaches my new bench/bench-drift/card-drift steps. Those three new steps were verified green in isolation (`lodestar bench`, `git diff --exit-code -- metrics/headline.json`, `python metrics/render.py --check` all exit 0); the bench-drift step currently exits 0 only because headline.json is untracked (git diff shows nothing for untracked files) — determinism itself is proven by `run_bench()` called twice returning equal dicts, and the step will correctly gate drift once the file is committed.
- Budget actual: in-tokens ~95000, tool calls ~55, attempts 1.
