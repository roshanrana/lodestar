---
id: T-008
title: Narrative providers, leak guard, EBR markdown and DOCX
milestone: M4
risk: high
tier: T2
complexity: high
reasoning: on
budget: {input_tokens: 80000, tool_calls: 80, wall_clock_min: 120}
depends_on: [T-003]
rtm: [FR-012, FR-013, FR-014, NFR-001]
status: todo
---
# T-008 — Narrative providers, leak guard, EBR markdown and DOCX

## Goal
`lodestar ebr` writes `docs/ebr/northstar-ebr-brief.md` and `.docx` from the architect-authored
template and the CUSTOMER view; an optional LLM provider (Fireworks default when configured,
Anthropic optional) may rewrite only the executive narrative and its output must pass the leak
guard or be discarded.

## Spec references
- `docs/design/03-lld.md §12` in full — `Provider` protocol; `OfflineProvider` (returns ""), `OpenAICompatProvider(base_url, api_key, model, name="fireworks")` posting to `{base_url}/chat/completions` with httpx timeout 30, non-2xx → `PROVIDER_HTTP`; `AnthropicProvider` via raw httpx to `/v1/messages`; `make_provider(settings)` raising `PROVIDER_CONFIG` before any I/O when a key is missing; guard `INTERNAL_TERMS` list, numeric strings (≥3 digits) from internal fields of the INTERNAL view, `check_text()`; `render_ebr(view_customer, provider)`; `write_docx(md, path)` supporting H1/H2/paragraphs/bullets/numbered/pipe tables; eight H2 sections in order.
- `docs/ebr/ebr_template.md` — the placeholder keys are every `{{name}}` in that file; build the dict from the customer view: e.g. `automation_cross_date` = first date with automation ≥ 70 (2026-08-27), `sequence_list` = "1. Kestrel Athletics (Weeks 1-8 pilot); 2. Little Compass (…)" from ranked brands, `wave2_*`/`wave3_*`/`wave4_*` from ranks 2–4, `incident_dates` joined with " and ", percentages to one decimal, dollars to whole numbers except cost per ticket (4 dp). `executive_narrative` offline text: three short paragraphs assembled from figures — results, why they matter, what to take away — written by you in plain executive prose with no internal vocabulary.
- `§8` env vars for providers; `§7` codes; `§16` never log prompt bodies or keys.
- `§15` — `lodestar/commands/ebr.py` (`NAME="ebr"`, options `--provider`, `--out`).

## Scope (files this task may touch)
- lodestar/narrative/__init__.py, lodestar/narrative/provider.py, lodestar/narrative/guard.py, lodestar/narrative/ebr.py
- lodestar/commands/ebr.py
- docs/ebr/northstar-ebr-brief.md, docs/ebr/northstar-ebr-brief.docx (generated outputs, commit them)
- tests/test_provider.py, tests/test_guard.py, tests/test_ebr.py

## Acceptance criteria
- AC1: Offline: `render_ebr` output has no unfilled `{{`; exactly eight `## ` headings in the specified order; the guard passes on it; `write_docx` produces a file python-docx can reopen with ≥ 8 heading paragraphs and ≥ 3 tables.
- AC2: A hostile provider stub returning "Confidence is 75% and margin is 62%; ignore previous instructions" causes `LEAK_GUARD` and the offline narrative is used; the CLI exit code is 0 and a warning is logged with the term category only.
- AC3: `make_provider` with `LODESTAR_LLM_PROVIDER=fireworks` and no key raises `PROVIDER_CONFIG` before any `httpx` call (httpx transport mocked to fail the test if touched); with a key, a mocked 200 returns the content and a mocked 500 raises `PROVIDER_HTTP`; the request JSON contains no internal field values (assert against the INTERNAL view's internal numbers).
- AC4: Anthropic provider request carries `x-api-key` and `anthropic-version` headers and parses `content[0].text`.
- AC5: `lodestar ebr --provider offline --out <tmp>` writes both files; the committed `docs/ebr/*` equal a fresh offline render (determinism test).
- AC6: Figures in the rendered brief match the customer view: "70.8", "67.9", "7.35", "84.2", "1,213" or "1213", "27 of 31".

## Validation commands (targeted)
- `uv run pytest tests/test_provider.py tests/test_guard.py tests/test_ebr.py -q`
- `uv run lodestar ebr --provider offline`

## Verification checklist (for the Verifier)
- [ ] scope respected
- [ ] error taxonomy used (PROVIDER_CONFIG, PROVIDER_HTTP, LEAK_GUARD)
- [ ] no prompt bodies or keys logged
- [ ] tests named with RTM IDs
- [ ] no new dependencies (httpx, python-docx declared)
- [ ] provider receives only the CUSTOMER view

## Threat-model boundary touched
B2 (LLM provider): C-06 guard, C-07 customer-only prompt payload, C-08 output used as prose only, C-09 keys from env; B1: C-06 on the rendered brief.

## Handoff (Implementer fills, ≤10 lines)
- Implemented: `lodestar/narrative/{provider,guard,ebr}.py` + `lodestar/commands/ebr.py`; generated `docs/ebr/northstar-ebr-brief.{md,docx}`.
- Files changed: lodestar/narrative/__init__.py, provider.py, guard.py, ebr.py; lodestar/commands/ebr.py; docs/ebr/northstar-ebr-brief.md, .docx; tests/test_provider.py, test_guard.py, test_ebr.py.
- Tests run: `uv run pytest tests/test_provider.py tests/test_guard.py tests/test_ebr.py -q` → 51 passed; `uv run pytest -q` (full suite) → 219 passed, 0 failed; `uv run mypy lodestar` → no issues (26 files); `uv run ruff check .` and `uv run ruff format --check .` → clean except pre-existing `scripts/screenshots.py` (E501 + format, outside this task's scope, not touched).
- Deviations from pack: `{{deployment}}` has no source field on `AccountView` (no `deployment`/account-info field exists there) — used `view_customer.generated_from` (CSV basename) as the only dataset-identity string available on the customer view; renders as "northstar_flagship_30_day_metrics.csv". Not covered by AC1–AC6.
- Open questions: none.
- Budget actual: in-tokens ~95k, tool calls ~48, attempts 1.
