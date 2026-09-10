---
id: T-002
title: Stats, spend summary, health score, incident detection
milestone: M1
risk: low
tier: T2
complexity: high
reasoning: on
budget: {input_tokens: 80000, tool_calls: 80, wall_clock_min: 120}
depends_on: [T-000]
rtm: [FR-003, FR-006, NFR-002]
status: todo
---
# T-002 — Stats, spend summary, health score, incident detection

## Goal
The frozen health-score methodology, the spend summary and incident detection are implemented
as pure functions over `Dataset` and reproduce the anchor figures in the LLD.

## Spec references
- `docs/design/03-lld.md §3` — window definitions (F7, L7, MONTH), mean-of-daily vs Σ/Σ rules, growth ratios, `SpendSummary` fields, and the anchor figures (Σ requests 789 563; Σ spend 1 213.12; L7 mean automation 70.07; L7 availability 99.8997; spend_growth_ratio 1.436; automated_growth_ratio 1.147; spend_vs_outcome_ratio 1.253; month cost per automated ticket 0.01244).
- `§4` — pillar table (weights, floors, targets, directions), linear interpolation clipped to [0,100], bands, `MetricScore`/`PillarScore`/`HealthReport` models, `score(ds, relationship=None)`; expected pillar scores adoption 80.2, reliability 87.2, latency 78.5, quality 83.1, spend_efficiency 65.8, operational composite 80.2 healthy; with relationship inputs (mean 71.0) internal composite 77.5 watch. `HealthReport.methodology_note` is fixed prose describing the arithmetic.
- `§6` — `detect(ds, slo_availability_pct)`; rule: ≥3 prior days and (error > 1.30 × median of previous ≤7 days or p95 > 1.20 × median of previous ≤7 days); expected flagged {2026-08-09, 2026-08-21}; `days_below_slo` = 27 at 99.9.
- `§10` — internal fields on `HealthReport` use `Field(None, **tag(Visibility.INTERNAL))` imported from `lodestar.audience` (T-001 provides it; if it is not yet merged when you start, define the `tag` import lazily behind `TYPE_CHECKING` guards is NOT acceptable — instead wait for T-001 or implement against the exact signature `tag(v: Visibility) -> dict[str, Any]` and `Visibility` StrEnum, which are frozen).

## Scope (files this task may touch)
- lodestar/stats.py, lodestar/spend.py, lodestar/health.py, lodestar/incidents.py
- tests/test_stats.py, tests/test_spend.py, tests/test_health.py, tests/test_incidents.py

## Acceptance criteria
- AC1: All §3 anchors reproduced within stated tolerances by named tests.
- AC2: Each metric score equals the hand computation for at least two metrics in tests (automation L7 70.07 → 75.4 ±0.3; p95 L7 1463.1 → 79.8 ±0.3); clipping tested at both ends with synthetic rows.
- AC3: Pillar and composite anchors reproduced ±0.3; band thresholds tested at 79.99/80/64.99/65.
- AC4: `score(ds)` has `relationship_score is None`; `score(ds, relationship)` gives 71.0 and 77.5 ±0.3, band watch.
- AC5: `detect()` flags exactly the two dates; `matched == 2`, `false_positives == 0`, `missed == 0`, `days_below_slo == 27`.
- AC6: `score()` on the shipped dataset completes in < 1 s (timed test, generous 1 s bound).
- AC7: `SpendSummary` fields all populated; `projected_30d_flat_usd == l7_daily_mean * 30`; `pct_of_budget` computed against a budget value passed as an argument (`summarise(ds, budget_line_usd: float)`).

## Validation commands (targeted)
- `uv run pytest tests/test_stats.py tests/test_spend.py tests/test_health.py tests/test_incidents.py -q`
- `uv run mypy lodestar`

## Verification checklist (for the Verifier)
- [ ] scope respected
- [ ] constants match LLD §4 exactly (weights sum to 1.0; floors/targets verbatim)
- [ ] no sensitive fields logged
- [ ] tests named with RTM IDs
- [ ] no new dependencies

## Threat-model boundary touched
none

## Handoff (Implementer fills, ≤10 lines)
- Implemented: stats.py/spend.py/health.py/incidents.py per §3/§4/§6, all anchors reproduced.
- Files changed: lodestar/{stats,spend,health,incidents}.py; tests/test_{stats,spend,health,incidents}.py
- Tests run: `uv run pytest tests/test_stats.py tests/test_spend.py tests/test_health.py tests/test_incidents.py -q` → 26 passed; full suite 51 passed; mypy strict clean; ruff check/format clean.
- Deviations from pack: (1) per orchestrator amendment, HealthReport internal fields use literal `Field(default=None, json_schema_extra={"visibility": "internal"})` instead of importing `tag`/`Visibility` from audience.py. (2) spend_vs_outcome_ratio has no natural F7-window value (it already compares L7 vs F7); used 1.0 (neutral, no differential growth) as its trend-reference observed_f7 — undocumented in LLD, low risk since no anchor tests this delta.
- Open questions: (1) AC5/§6 anchor states days_below_slo=22 of 31 at SLO 99.9%, but the shipped CSV genuinely yields 27 (verified by direct recount; flagged incident dates and matched/false_positives/missed all reproduce exactly). Implemented `detect()` literally per the §6 formula; test asserts the real recomputed value (27) rather than hard-coding 22. Needs Verifier/architect call on whether the CSV or the LLD anchor is stale.
- Budget actual: in-tokens ~85000, tool calls 27, attempts 1.
