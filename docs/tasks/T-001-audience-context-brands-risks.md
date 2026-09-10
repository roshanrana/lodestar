---
id: T-001
title: Audience tags, context models, brands scoring, static risks
milestone: M1
risk: high
tier: T2
complexity: high
reasoning: on
budget: {input_tokens: 80000, tool_calls: 80, wall_clock_min: 120}
depends_on: [T-000]
rtm: [FR-004, FR-007, FR-008, FR-009, NFR-005]
status: todo
---
# T-001 — Audience tags, context models, brands scoring, static risks

## Goal
The visibility mechanism exists and is fail-closed; the three YAML context files load into
typed models whose fields carry visibility tags; brands are fit-scored and ranked; static
risks and next actions load with their visibility.

## Spec references
- `docs/design/03-lld.md §10` — `Audience`, `Visibility`, `tag()`, `field_visibility()` (default INTERNAL when untagged), `scrub()` (recursive; internal fields → None / empty container), `internal_field_count()`.
- `§11` — `Context{assumptions, brands, account}`, `load_context(data_dir)`; unknown keys → `CONTEXT_INVALID`; `Risk` model; static risks from YAML; `fit_score = round(Σ w × dim × 20, 1)`; expected Kestrel 87.0, Little Compass 60.0, Lumen Beauty 58.0, Meridian Home 57.0; `rank_pilots()` fit desc then name.
- Visibility per field is documented in the header comments of `data/brands.yaml` and `data/account_context.yaml`; `data/assumptions.yaml` is wholly customer-visible.
- Data-derived risks (R-TECH-1/2, R-EXEC-2) and `top_risks()` ranking are T-003, not this task; expose `static_risks(ctx) -> tuple[Risk, ...]` and `next_actions(ctx)` here.

## Scope (files this task may touch)
- lodestar/audience.py, lodestar/context.py, lodestar/brands.py, lodestar/risks.py
- tests/test_audience.py, tests/test_context.py, tests/test_brands.py, tests/test_risks_static.py

## Acceptance criteria
- AC1: `field_visibility(Model, "untagged")` returns INTERNAL; `scrub(model, CUSTOMER)` on a nested fixture model removes every internal field at every depth and leaves customer fields byte-identical; `scrub(model, INTERNAL)` returns an equal object.
- AC2: `internal_field_count()` on the fixture equals the hand-counted number; on `BrandProfile` it equals 10 (stage, confidence_pct, est_monthly_inference_usd, est_annual_value_usd, blockers, competitive_note, margin_note, internal_owner, target_date, plus `fit_weights_note` if you add one — otherwise 9; state the number in the handoff).
- AC3: `load_context()` on `data/` succeeds; a YAML with an unknown key raises `CONTEXT_INVALID` naming file and field; every internal-tagged field on every context model is Optional (a test walks all models).
- AC4: `fit_score` and `rank_pilots` reproduce the four expected scores and the order Kestrel Athletics, Little Compass, Lumen Beauty, Meridian Home.
- AC5: `static_risks(ctx)` returns four risks with ids R-REL-1, R-COMP-1, R-EXEC-1, R-TECH-3 and the visibilities internal, internal, customer, customer; `next_actions(ctx)` returns 8 with 5 customer-visible.
- AC6: Relationship inputs load as `dict[str, float]` with mean 71.0 and are internal-tagged on `AccountContext`.

## Validation commands (targeted)
- `uv run pytest tests/test_audience.py tests/test_context.py tests/test_brands.py tests/test_risks_static.py -q`
- `uv run mypy lodestar`

## Verification checklist (for the Verifier)
- [ ] scope respected
- [ ] error taxonomy used (`CONTEXT_INVALID`)
- [ ] no sensitive fields logged
- [ ] tests named with RTM IDs
- [ ] no new dependencies
- [ ] every internal field is Optional and tagged; default visibility is INTERNAL

## Threat-model boundary touched
B1 (internal ↔ customer audience): controls C-01 visibility tags, C-02 single `scrub()`.

## Handoff (Implementer fills, ≤10 lines)
- Implemented: Visibility tags/scrub/count (audience.py), context loaders for assumptions/brands/account (context.py), brand fit scoring/ranking (brands.py), static risks + next actions (risks.py); internal_field_count(BrandProfile) = 9 (no fit_weights_note added).
- Files changed: lodestar/audience.py, lodestar/context.py, lodestar/brands.py, lodestar/risks.py, tests/test_audience.py, tests/test_context.py, tests/test_brands.py, tests/test_risks_static.py
- Tests run: `uv run pytest tests/test_audience.py tests/test_context.py tests/test_brands.py tests/test_risks_static.py -q` → 27 passed; full suite `uv run pytest -q` → 78 passed; `uv run mypy lodestar` → success; `uv run ruff check . && uv run ruff format --check .` → clean
- Deviations from pack: none
- Open questions: none
- Budget actual: in-tokens ~62000, tool calls ~40, attempts 1
