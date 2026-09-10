---
id: T-011
title: Security review of B1/B2 surfaces, dependency audit, ship report
milestone: M5
risk: high
tier: T2
complexity: normal
reasoning: off
budget: {input_tokens: 40000, tool_calls: 40, wall_clock_min: 45}
depends_on: [T-010]
rtm: [NFR-005, NFR-009]
status: todo
---
# T-011 — Security review, dependency audit, ship report

## Goal
An independent review of the audience boundary and provider boundary with live probes, a
dependency audit, and a ship report listing what was measured and what was not.

## Spec references
- `docs/design/02-threat-model.md` STRIDE rows for B1, B2, B4 and controls C-01..C-17.
- Probes to run and record: (1) customer MCP tools and API routes walked for internal fields; (2) EBR rendered text scanned for C-06 terms and internal numbers; (3) hostile provider injection test; (4) `LODESTAR_INTERNAL_ENABLED=0` gate on API, MCP and internal page; (5) oversized CSV and malformed cells; (6) `uv export --all-extras --no-hashes --format requirements-txt -o .audit-requirements.txt && uvx pip-audit -r .audit-requirements.txt`; (7) `python scripts/secrets_scan.py`.
- Ship report template: what shipped; measured figures (from headline.json); what was written but not run (live provider narrative; screenshots are manual); review findings and dispositions; deviations (D-000).

## Scope (files this task may touch)
- docs/security-review.md, docs/ship-report.md
- tests/test_security_probes.py (new tests only if a probe found a gap; otherwise document the existing tests that cover each probe)

## Acceptance criteria
- AC1: `docs/security-review.md` lists each control C-01..C-17 with status (implemented / partial / n-a) and the test or probe that evidences it.
- AC2: pip-audit output recorded; any finding either fixed (bump in `pyproject.toml` is out of scope → record as blocked for the orchestrator) or dispositioned with reason.
- AC3: Probes 1–5 executed with commands and outcomes pasted (≤ 20 lines each); any gap becomes a numbered finding with severity.
- AC4: `docs/ship-report.md` exists with the four sections above and links to the evidence ledger.

## Validation commands (targeted)
- `uv run pytest tests/test_leak.py tests/test_guard.py tests/test_mcp.py tests/test_api.py -q`
- `uv run python scripts/secrets_scan.py`

## Verification checklist (for the Verifier)
- [ ] scope respected
- [ ] every control has evidence or an explicit gap
- [ ] no code changes made by the reviewer
- [ ] no new dependencies

## Threat-model boundary touched
B1, B2, B4 (review of all three).

## Handoff
- Implemented: security review of B1/B2/B4 with 8 live probes, dependency audit (pip-audit clean), and ship report — all 17 controls (C-01..C-17) implemented with passing evidence; no code gap found.
- Files changed: docs/security-review.md (new), docs/ship-report.md (new)
- Tests run: `uv run pytest tests/test_leak.py tests/test_guard.py tests/test_mcp.py tests/test_api.py tests/test_provider.py tests/test_data.py -q` → 85 passed; `uv run pytest -q` (full suite) → 251 passed; `uv run python scripts/secrets_scan.py` → no secrets found; `uvx pip-audit -r .audit-requirements.txt` → no known vulnerabilities
- Deviations from pack: none
- Open questions: none
- Budget actual: in-tokens ~62000, tool calls 32, attempts 1
