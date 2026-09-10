# Change record — Lodestar release 0.1.0

| Field | Value |
|---|---|
| Requested by | Roshan Rana (owner) — Fireworks AI take-home submission and portfolio |
| Implemented by | Sonnet-class Implementer agents, one task pack each (T-000 … T-013) |
| Verified by | Fresh-context Sonnet-class Verifier agents in isolated git worktrees (verdicts in `docs/tasks/*.verdict.md`); Security Reviewer (T-011) |
| Approved by | Roshan Rana, delivery lead, under the standing autonomous authorisation recorded in `docs/design/decisions.md` D-000 |

## Description

First release of Lodestar: one telemetry core, two audience-scoped Streamlit views, a FastAPI
JSON API, an MCP server with six tools, a Typer plug-in CLI, provider-agnostic EBR generation
(offline default; Fireworks and Anthropic optional) behind a leak guard, an offline bench that
writes the results card, and the Shipyard design and evidence set.

## Business justification

Answers the Fireworks AI take-home ("Northstar Retail Group: EBR, Internal QBR & Account
Health") with a reviewer-runnable application and demonstrates forward-deployed engineering
practice: frozen design, audience separation enforced in code, measured claims, independent
verification.

## Systems affected

New public repository `roshanrana/lodestar`; profile README `roshanrana/roshanrana` gains one
row in each table. No other system.

## Risk assessment

Low. Local tool, synthetic data, no credentials, no external calls by default. Residual risks
accepted in `docs/design/02-threat-model.md` (paraphrase leakage; internal surfaces unauthenticated
by design for a local tool).

## Implementation steps

1. `git clone https://github.com/roshanrana/lodestar && cd lodestar`
2. `uv sync --all-extras`
3. `uv run python scripts/check.py` (expected: all checks passed)
4. `uv run lodestar dashboard` → http://localhost:8501

## Verification steps

Gate green locally and in CI; `python scripts/shipyard/evidence.py verify` → chain valid;
`docs/ship-report.md` measured figures equal `metrics/headline.json`.

## Rollback plan

`git checkout <previous commit>` then `uv sync`. Trigger: any gate failure on `main`.

## Schedule and communication

Submitted 2026-09-10 as a ZIP built from `git archive HEAD` plus the committed EBR outputs.
Evidence pack: `docs/evidence/` (ledger and export).
