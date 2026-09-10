---
id: T-012
title: Polish follow-ups from verifier findings and first screenshots
milestone: M5
risk: low
tier: T2
complexity: normal
reasoning: off
budget: {input_tokens: 40000, tool_calls: 40, wall_clock_min: 45}
depends_on: [T-009]
rtm: [FR-010, FR-011, NFR-005]
status: todo
---
# T-012 — Polish follow-ups

## Goal
Close the non-blocking findings carried from T-000, T-002 verdicts and the visual defects the
orchestrator observed in the first full-page captures, so the final screenshots and the
reviewer's first run look finished.

## Spec references
- `docs/design/03-lld.md §14` — page titles are "Customer · Account health" and "Internal · QBR" (middle dot U+00B7); chart contract: legend horizontal at top, `margin=dict(l=10, r=10, t=40, b=10)`; the legend must not overlap the figure title (raise the title or place the legend below the title, e.g. `title=dict(y=0.98, yanchor="top")` with `legend=dict(y=1.02, yanchor="bottom")`, or increase top margin to 60 and keep both).
- `docs/design/02-threat-model.md` C-06 — the Home page must not name internal categories; replace the Home sentence that mentions "relationship, pipeline, and margin considerations" with neutral wording ("an internal QBR view for the Fireworks account team").
- T-000 verdict finding 2: `lodestar/data.py` module docstring must state the actual validation order (header → row-count cap → per-cell parse → multi-deployment → duplicate date).
- T-000 verdict finding 3: tests in `tests/test_app_skeleton.py` tagged nfr002/fr010/fr011 must be re-tagged to T-000's own RTM ids (fr001 for load, fr018 for navigation) or made to assert what their names claim.
- T-002 verdict finding 1: `tests/test_incidents.py` must assert the literal `days_below_slo == 27` rather than recomputing with the production predicate.
- Internal page charts (§14 five trend charts): series with different scales must not share one y-axis. Use `dual_axis` for reliability (availability % left, error rate % right) and for quality (grounding and eval pass % left, CSAT right); adoption may stay single-axis. Keep incident and F7/L7 window shading.
- Internal page tiles: the "Weakest pillar" `st.metric` truncates long names ("Spend Efficie…"); show the score as the value (e.g. "65.8") with the pillar name in the label ("Weakest pillar · Spend efficiency") or caption so nothing truncates at 1440 px.
- Customer page: the "Known caveats" expander must be `expanded=True` by default (caveats are part of the story the VP CX should see without clicking); the methodology expander stays collapsed.
- T-008 handoff deviation: add `deployment: str` (customer-tagged, from `Dataset.deployment`) to `AccountView` in `lodestar/view.py` and make the EBR `{{deployment}}` placeholder use it (`lodestar/narrative/ebr.py`), then regenerate `docs/ebr/northstar-ebr-brief.md` and `.docx` with `uv run lodestar ebr --provider offline` so the committed brief shows the deployment id `northstar-flagship-support-ft-v1`.
- EBR editorial (architect): `{{availability_month}}` must be formatted to two decimals (99.85, not 99.9) in `lodestar/narrative/ebr.py` placeholders; the template text for talking point 3 was already updated by the architect; regenerate the brief as part of the deployment-id item above.
- T-008 verdict finding 3: rename `tests/test_ebr.py::test_ac6_fr014_...` to lead with the RTM id (`test_fr014_ac6_...`).
- T-002 verdict finding 2: add literal assertions for Σ automated 97 488 and Σ tier1 143 430 in `tests/test_stats.py` or `tests/test_spend.py`.

## Scope (files this task may touch)
- app/pages/customer_health.py (title string, caveats expander default), app/pages/internal_qbr.py (reliability and quality charts → dual_axis; weakest-pillar tile label), app/pages/home.py (one sentence), app/components/charts.py (legend/title layout only)
- lodestar/data.py (docstring only), lodestar/view.py (one new customer-tagged field), lodestar/narrative/ebr.py (placeholder source only), docs/ebr/northstar-ebr-brief.md, docs/ebr/northstar-ebr-brief.docx (regenerated), tests/test_ebr.py and tests/test_view.py (new assertions only)
- tests/test_app_skeleton.py, tests/test_incidents.py, tests/test_stats.py, tests/test_charts.py (layout assertion update if needed), tests/test_page_customer.py, tests/test_page_internal.py (new assertions only)

## Acceptance criteria
- AC1: Customer page H1 renders "Customer · Account health" (AppTest title element text equals that string); internal page unchanged.
- AC2: On every chart from `line_with_incidents` and `dual_axis`, the title's y position is above the legend's y position or the top margin is ≥ 60 (test asserts one of the two); visually no overlap in `docs/assets` captures taken after this task.
- AC3: Home page rendered text contains neither "margin" nor "relationship".
- AC4: Internal page reliability and quality charts have a secondary y-axis (AppTest: inspect the plotly_chart specs for `yaxis2`), and the weakest-pillar tile value/label contain no ellipsis truncation (label ≤ 32 chars).
- AC5: Customer page "Known caveats" expander is expanded by default (AppTest expander `.expanded` or the element proto).
- AC6: The committed EBR brief header shows "northstar-flagship-support-ft-v1" and the determinism test still passes.
- AC7: The four carried test findings are closed as specified; full gate `uv run python scripts/check.py` passes; `render.py --check` clean.

## Validation commands (targeted)
- `uv run pytest tests/test_app_skeleton.py tests/test_incidents.py tests/test_stats.py tests/test_charts.py tests/test_page_customer.py tests/test_page_internal.py -q`
- `uv run python scripts/check.py`

## Verification checklist (for the Verifier)
- [ ] scope respected
- [ ] no business logic changed (docstring, strings, layout, tests only)
- [ ] no sensitive fields logged
- [ ] tests named with RTM IDs
- [ ] no new dependencies

## Threat-model boundary touched
B1: C-06 vocabulary on the Home page.

## Handoff (Implementer fills, ≤10 lines)
- Implemented: all listed polish fixes (titles, chart legend/title margin, Home wording, internal reliability/quality charts to dual_axis, weakest-pillar tile, caveats expander default, AccountView.deployment + EBR placeholder + 2-decimal availability_month, regenerated EBR brief) plus the five carried test findings.
- Files changed: app/components/charts.py, app/pages/{customer_health,home,internal_qbr}.py, lodestar/{data,view}.py, lodestar/narrative/ebr.py, docs/ebr/northstar-ebr-brief.{md,docx}, tests/{test_app_skeleton,test_charts,test_ebr,test_incidents,test_page_customer,test_page_internal,test_stats,test_view}.py
- Tests run: `uv run pytest tests/test_app_skeleton.py tests/test_incidents.py tests/test_stats.py tests/test_charts.py tests/test_page_customer.py tests/test_page_internal.py -q` → 42 passed; `uv run pytest -q` (full) → 251 passed; `uv run python scripts/check.py` → all checks passed (97.47% coverage, no bench/card drift).
- Deviations from pack: dual_axis has no hline parameter, so the reliability chart's SLO reference line is dropped when switched to dual_axis (incident/window shading kept, per pack wording).
- Open questions: none
- Budget actual: in-tokens ~95000, tool calls 44, attempts 1
