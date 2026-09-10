---
id: T-013
title: OVERVIEW.md and SHOWCASE.md per portfolio convention
milestone: M5
risk: low
tier: T2
complexity: normal
reasoning: off
budget: {input_tokens: 40000, tool_calls: 30, wall_clock_min: 30}
depends_on: [T-009]
rtm: [FR-019, FR-020]
status: todo
---
# T-013 — OVERVIEW.md and SHOWCASE.md

## Goal
Every portfolio repository carries `docs/OVERVIEW.md` (what it is and why) and
`docs/SHOWCASE.md` (a guided tour of the features with the commands to run). Lodestar gets both,
consistent with the README, the design document and the EBR.

## Spec references
- Portfolio convention (profile README): "Each has an `OVERVIEW.md` (what it is and why) and a `SHOWCASE.md` (a guided tour of the features, with the commands to run) under `docs/`."
- `docs/design/00-problem-brief.md` (problem statement, outcomes), `02-hld.md` (architecture and stack decision), `03-lld.md §4` (health score), `§9` (bench KPIs), `§10` (audience scoping), `§12` (providers and guard), `§13` (MCP tools and API routes), `metrics/headline.json` (measured figures), `STATE.md` (how it was built).
- Tone and structure of a sibling repo's OVERVIEW/SHOWCASE (if present at `C:\Code-Central\drydock\docs\OVERVIEW.md` and `SHOWCASE.md`, read them for shape only; do not copy prose).

## Scope (files this task may touch)
- docs/OVERVIEW.md, docs/SHOWCASE.md

## Acceptance criteria
- AC1: `docs/OVERVIEW.md` covers: the operational problem (one account, three audiences, one telemetry file), what Lodestar is, the design bets (one core, audience tags fail closed, offline-first providers, measured card), what is measured (six KPIs from headline.json with values), what is deliberately not claimed, and how it was built (Shipyard lifecycle, model tiering, fresh-context verification in git worktrees, evidence ledger).
- AC2: `docs/SHOWCASE.md` is a guided tour with runnable commands in order: gate, dashboard (both pages and what to look at), CLI (`score`, `claims`, `risks`, `pipeline`, `bench --check`), EBR generation offline and with a provider, MCP registration and an example tool call from Claude Code, API routes with `curl` examples, the leak test and guard demo (`uv run pytest tests/test_leak.py tests/test_guard.py -q`), screenshots regeneration. Every command was executed by the implementer and its observed output summarised in one line beneath it.
- AC3: No figure appears that is not in `metrics/headline.json`, the LLD or the EBR; no internal vocabulary (C-06 list) is attributed to the customer view.
- AC4: `uv run python scripts/check.py` still passes (no code touched).

## Validation commands (targeted)
- `uv run python scripts/check.py`

## Verification checklist (for the Verifier)
- [ ] scope respected
- [ ] every command in SHOWCASE.md runs as written from a fresh shell in the repo root
- [ ] figures traceable to headline.json / LLD / EBR
- [ ] no new dependencies

## Threat-model boundary touched
none

## Handoff
- Implemented: docs/OVERVIEW.md (problem, design bets, six measured KPIs, deliberate non-claims, Shipyard build process) and docs/SHOWCASE.md (guided tour: gate, dashboard both pages, CLI, EBR offline+provider-config-error, MCP in-memory tool call, API curl, leak/guard tests, screenshot regen), each command actually executed.
- Files changed: docs/OVERVIEW.md (new), docs/SHOWCASE.md (new)
- Tests run: `uv run python scripts/check.py` → 251 passed, 97.47% coverage, all checks passed (twice, before and after writing docs); `uv run pytest tests/test_leak.py tests/test_guard.py -q` → 34 passed
- Deviations from pack: EBR "with a provider" demonstrated via `--provider fireworks` without FIREWORKS_API_KEY set (no key available), showing the deterministic PROVIDER_CONFIG error raised before any I/O per LLD §12; live-provider narrative itself was not exercised.
- Open questions: none
- Budget actual: in-tokens ~85000, tool calls 27, attempts 1
