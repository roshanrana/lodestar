---
id: T-010
title: README, runbook, design document
milestone: M5
risk: low
tier: T2
complexity: normal
reasoning: off
budget: {input_tokens: 40000, tool_calls: 40, wall_clock_min: 45}
depends_on: [T-009]
rtm: [FR-019, FR-020]
status: todo
---
# T-010 — README, runbook, design document

## Goal
A reviewer can run everything from the README; the design document answers every bullet of
the prompt's submission item 3 with the screenshots the orchestrator has placed in
`docs/assets/`.

## Spec references
- Prompt submission §2 (README must cover): required dependencies and versions; installation; how to start the dashboard; environment variables; where the CSV is expected; how to access each view; known limitations.
- Prompt submission §3 (design document must cover): screenshots of both dashboards; audience and purpose of each view; why these metrics and visualisations; how the health score is calculated (reproduce LLD §4 table and the arithmetic with the shipped numbers); how internal and customer content differs; what was intentionally excluded from each view; assumptions, caveats, design trade-offs (include the paraphrase-leak residual from the threat model and the F7/L7 window choice).
- README shape: mirror sibling repos — title, CI badge (`roshanrana/lodestar` check workflow), one-paragraph pitch, "At a glance" table, `## Results` card block (already spliced by render.py; do not edit inside markers), Quick start, Views, MCP, API, CLI, Configuration, Repository layout, How it was built (Shipyard lifecycle: design docs, task packs, verdicts, evidence ledger; model tiering), Known limitations, License.
- `docs/ops/runbook.md`: start/stop, common failures mapped to error codes (LLD §7), switching providers, regenerating bench/card/EBR.
- Screenshots: `docs/assets/customer-*.png`, `docs/assets/internal-*.png` exist before this task starts; reference them all.

## Scope (files this task may touch)
- README.md (outside the metrics markers), docs/design-document.md, docs/ops/runbook.md, docs/mcp.md (extend only)

## Acceptance criteria
- AC1: README contains every submission §2 bullet as a heading or table row; the commands in Quick start are exactly those that work (`uv sync --all-extras`, `uv run lodestar dashboard`, `uv run lodestar ebr`, `uv run lodestar mcp`, `uv run lodestar api`, `uv run python scripts/check.py`).
- AC2: design-document.md has a section per submission §3 bullet, embeds every PNG in `docs/assets/` that starts with `customer-` or `internal-`, and reproduces the health-score table with the shipped pillar scores and composite.
- AC3: `uv run python scripts/check.py` still passes (render.py --check clean: README edited only outside markers).
- AC4: No internal vocabulary appears in the sections describing the customer view except when explicitly listed under "intentionally excluded".

## Validation commands (targeted)
- `uv run python metrics/render.py --check`
- `uv run python scripts/check.py`

## Verification checklist (for the Verifier)
- [ ] scope respected
- [ ] every referenced image exists
- [ ] commands verified by running them
- [ ] no new dependencies

## Threat-model boundary touched
none

## Handoff (Implementer fills, ≤10 lines)
- Implemented: README (outside markers), docs/design-document.md, docs/ops/runbook.md, and a docs/mcp.md extension (Claude Code `claude mcp add` form).
- Files changed: README.md, docs/design-document.md (new), docs/ops/runbook.md (new), docs/mcp.md.
- Tests run: `uv run python metrics/render.py --check` → clean; `uv run python scripts/check.py` → 251 passed, 97.47% coverage, all checks passed.
- Deviations from pack: none.
- Open questions: none.
- Budget actual: in-tokens ~95000, tool calls 37, attempts 1.
