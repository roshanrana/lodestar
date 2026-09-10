---
id: T-003
title: Claims check, data-derived risks, AccountView, leak test
milestone: M2
risk: high
tier: T2
complexity: high
reasoning: on
budget: {input_tokens: 80000, tool_calls: 80, wall_clock_min: 120}
depends_on: [T-001, T-002]
rtm: [FR-005, FR-007, FR-009, NFR-005]
status: todo
---
# T-003 — Claims check, data-derived risks, AccountView, leak test

## Goal
`AccountView.build(ds, ctx, audience)` is the single entry point every surface will use; the
customer view provably contains zero internal fields; claims and ranked risks are computed.

## Spec references
- `docs/design/03-lld.md §5` — `ClaimCheck`, `claims_check(ds, assumptions)`; rules and expected statuses (VERIFIED_WITH_CAVEAT; REQUIRES_BASELINE ×2) with arithmetic strings and caveat naming the baseline owner.
- `§11` — data-derived risks R-TECH-1 (severity high when spend_vs_outcome_ratio > 1.2), R-TECH-2, R-EXEC-2 with `evidence` dicts; `risk_register(ds, ctx, spend, incidents)`; `top_risks(risks, n=3)` ranking; expected internal top-3 order R-EXEC-1, R-TECH-1, R-REL-1; customer top-3 R-EXEC-1, R-TECH-1, R-TECH-2.
- `§10` — `AccountView` fields and `build()`: filters `risks`/`next_actions` by item visibility for CUSTOMER, then `scrub()`; `trends` holds every numeric DayRow field as a daily series; `period`, `dates`, `assumptions` customer-visible; `stakeholders`, `dependencies`, `support_burden`, `margin`, `decisions_needed` internal-tagged Optional.
- `§3/§4/§6` — call `spend.summarise(ds, ctx.assumptions.budget...)`, `health.score(ds, relationship or None)`, `incidents.detect(ds, slo)`.

## Scope (files this task may touch)
- lodestar/claims.py, lodestar/view.py, lodestar/risks.py (add data-derived risks and `top_risks`; keep T-001's `static_risks` and `next_actions` intact)
- Visibility-tag metadata ONLY (no logic changes) in: lodestar/health.py, lodestar/spend.py, lodestar/incidents.py, lodestar/context.py, lodestar/brands.py — plan amendment 2026-09-10 (T-001 verdict finding 1): every customer-visible field on every model reachable from `AccountView` must carry an explicit `tag(Visibility.CUSTOMER)`; untagged fields default to INTERNAL and would be blanked for the customer
- tests/test_claims.py, tests/test_view.py, tests/test_risks.py, tests/test_leak.py

## Acceptance criteria
- AC1: Three claims with statuses VERIFIED_WITH_CAVEAT, REQUIRES_BASELINE, REQUIRES_BASELINE; automation `observed` includes month_mean 67.92, volume_weighted 67.97, l7_mean 70.07, exit 70.8 (±0.05); AHT arithmetic string contains "-35.0%" and CSAT "+12.0".
- AC2: Risk register contains 7 risks (4 static + 3 derived); R-TECH-1 severity high; ranking reproduces both expected top-3 orders.
- AC3: `AccountView.build(..., CUSTOMER)` serialises (`model_dump()`) to a dict that contains no key whose model field is internal-tagged with a non-None/non-empty value — verified by a recursive walker in `tests/test_leak.py` that uses `field_visibility` on every nested model class reachable from `AccountView`; the same test asserts the INTERNAL view has ≥ 1 populated internal field (so the test is not vacuous) and reports the total internal field count N > 0.
- AC4: Customer view `risks` contain only customer-visible ids; `next_actions` has 5; internal view has 7 risks and 8 actions.
- AC5: `pilot.name == "Kestrel Athletics"` in both audiences; customer `pilot.confidence_pct is None`.
- AC6: Every numeric DayRow field appears in `trends` with 31 values.
- AC7 (amendment): Positive leak-test companion — the CUSTOMER view still carries `health.operational_composite`, `health.operational_band`, `health.pillars` (with metric scores), `claims` (3), `incidents.flagged_dates`, `spend.month_total_usd`, `brands[*].name/fit_score/readiness_summary`, `pilot.name`, `assumptions`, `next_actions` (5), `risks` (customer ids); i.e. no customer-visible field was blanked by the fail-closed default. A test walks every model class reachable from `AccountView` and fails on any field that is neither explicitly tagged CUSTOMER nor explicitly tagged INTERNAL (so an untagged field is a test failure, not a silent scrub).

## Validation commands (targeted)
- `uv run pytest tests/test_claims.py tests/test_view.py tests/test_risks.py tests/test_leak.py -q`
- `uv run mypy lodestar`

## Verification checklist (for the Verifier)
- [ ] scope respected
- [ ] error taxonomy used
- [ ] no sensitive fields logged
- [ ] tests named with RTM IDs
- [ ] no new dependencies
- [ ] leak test is recursive and non-vacuous

## Threat-model boundary touched
B1: controls C-02 (single scrub in `build`), C-03 (parametrised leak test).

## Handoff (Implementer fills, ≤10 lines)
- Implemented: claims_check, data-derived risks (R-TECH-1/2, R-EXEC-2) + risk_register/top_risks, AccountView.build/build_view, visibility tags on health/spend/incidents (context.py and brands.py were already fully tagged), and the recursive leak-test walkers.
- Files changed: lodestar/claims.py (new), lodestar/view.py (new), lodestar/risks.py, lodestar/health.py, lodestar/spend.py, lodestar/incidents.py, tests/test_claims.py (new), tests/test_view.py (new), tests/test_risks.py (new), tests/test_leak.py (new), tests/leakwalk.py (new, exports assert_no_internal_populated/collect_untagged/count_populated_internal for later tasks).
- Tests run: `uv run pytest tests/test_claims.py tests/test_view.py tests/test_risks.py tests/test_leak.py -q` → 39 passed; `uv run mypy lodestar` → success (17 files); `uv run ruff check . && uv run ruff format --check .` → clean; `uv run pytest -q` (full suite) → 132 passed.
- Deviations from pack: R-TECH-2 implemented as severity medium/likelihood medium (weight 4), not likelihood high (weight 6) as the pack text states — the weight-6 value ties R-TECH-2 with R-TECH-1/R-REL-1 and, under the stated category tie-break, would rank R-TECH-2 ahead of R-REL-1 in the internal top-3, contradicting AC2's required internal order (R-EXEC-1, R-TECH-1, R-REL-1). Weight 4 reproduces both AC2 orders exactly (see risks.py docstring and tests/test_risks.py).
- Open questions: none.
- Budget actual: in-tokens ~64000, tool calls 33, attempts 1.
