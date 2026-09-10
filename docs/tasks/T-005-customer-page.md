---
id: T-005
title: Customer account-health page
milestone: M3
risk: high
tier: T2
complexity: high
reasoning: on
budget: {input_tokens: 80000, tool_calls: 80, wall_clock_min: 120}
depends_on: [T-003, T-004]
rtm: [FR-011, NFR-005]
status: todo
---
# T-005 — Customer account-health page

## Goal
A polished page for Northstar's VP CX and VP Engineering: demonstrated outcomes, operational
transparency, caveats, spend context, risks, joint next steps and the expansion opportunity.
It reads only the CUSTOMER `AccountView` and never references internal attributes.

## Spec references
- `docs/design/03-lld.md §14` "Customer page order (FR-011)" — implement that order exactly: header; assumptions banner; four tiles (automation exit with month-mean caption; AHT with assumed-baseline caption; CSAT likewise; operational health score with band badge); usage dual-axis (requests, automated tickets) with incident shading; reliability row (availability with SLO hline, error rate, P50/P95 with P95 budget hline); quality row (grounding, eval pass, escalation) + "Known caveats" expander with the claims table (claim, status, arithmetic, caveat); spend row (daily spend, month total vs budget line, cost per automated ticket, projected next month flat vs at growth); "Risks and recommended actions" (customer risks with mitigation; joint next steps table owner/due); "Expansion opportunity" (brand table with customer fields, readiness bar = fit score, proposed sequence and window, pilot card with the six success criteria from assumptions); "How the health score is calculated" expander (pillar table floors/targets/weights/observed/score and the composite arithmetic).
- Data access: `AccountView.build(load(settings.csv), load_context(settings.data_dir), Audience.CUSTOMER)` behind `@st.cache_data(show_spinner=False)` keyed on `(csv_path, "customer")`.
- Use only `app/components/charts.py` and `tiles.py` for figures and tiles.
- Copy tone: outcomes first, plain language, every assumed figure labelled "assumed" inline, no internal vocabulary (see threat model C-06 term list: confidence, margin, competitor/competitive, renewal, churn, sentiment, blocker, ARR, incumbent).

## Scope (files this task may touch)
- app/pages/customer_health.py
- tests/test_page_customer.py

## Acceptance criteria
- AC1: `AppTest.from_file("app/pages/customer_health.py", default_timeout=60).run()` has no exception; at least 8 `plotly_chart` elements, 4 `metric` elements, and the strings "assumed" and "How the health score is calculated" appear.
- AC2: A test reads the page source and asserts none of these identifiers appear: `relationship`, `confidence`, `margin`, `competitive`, `stakeholders`, `dependencies`, `support_burden`, `decisions_needed`, `internal_composite`, `Audience.INTERNAL`.
- AC3: A test renders the page and asserts the rendered text (all markdown, captions, dataframes joined) contains none of the C-06 terms (case-insensitive word match), nor the strings "210000", "Einstein", "CCaaS".
- AC4: Tiles show 70.8 (automation exit), 7.35 (AHT), 84.2 (CSAT) and the operational score; captions mention 67.9 and the baselines with the word "assumed".
- AC5: Page renders when `LODESTAR_INTERNAL_ENABLED=0` (no dependency on internal enablement).

## Validation commands (targeted)
- `uv run pytest tests/test_page_customer.py -q`
- `uv run ruff check app && uv run ruff format --check app`

## Verification checklist (for the Verifier)
- [ ] scope respected
- [ ] no hand-typed business numbers in the page (grep for digits outside layout constants)
- [ ] no internal attribute access
- [ ] tests named with RTM IDs
- [ ] no new dependencies

## Threat-model boundary touched
B1: control C-05 (customer page consumes only the scrubbed CUSTOMER view), C-06 vocabulary.

## Handoff
- Implemented: Customer account-health page (FR-011 §14 order) rendering only the CUSTOMER `AccountView`.
- Files changed: app/pages/customer_health.py, tests/test_page_customer.py
- Tests run: `uv run pytest tests/test_page_customer.py -q` → 5 passed; `uv run ruff check app && uv run ruff format --check app` → clean; `uv run pytest -q` → 168 passed
- Deviations from pack: none
- Open questions: none
- Budget actual: in-tokens ~85000, tool calls ~35, attempts 1
