# Execution plan — Lodestar

## Milestones

| M | Name | Outcome | Gate |
|---|---|---|---|
| M0 | Walking skeleton | CSV → validated `Dataset` → one Streamlit page with one chart; `check.py` green in CI; plug-in CLI root | G5 |
| M1 | Core analytics | audience tags + context models + brands + static risks; stats + spend + health + incidents | G6.1 |
| M2 | Account view | claims, data-derived risks, `AccountView.build`, leak test; chart components | G6.2 |
| M3 | Surfaces | customer page, internal page, MCP server + API | G6.3 |
| M4 | Narrative and bench | providers, guard, EBR md + docx; bench, card, remaining CLI commands | G6.4 |
| M5 | Docs and ship | README, runbook, design document with screenshots, security review, evidence export, GitHub | G7/G8 |

## Task table

| Task | M | Title | Risk | Tier | Complexity | Budget (in-tok/calls/min) | Depends on | RTM |
|---|---|---|---|---|---|---|---|---|
| T-000 | M0 | Walking skeleton: package, data loader, gate, CI, CLI root, page stubs | medium | T2 | high | 80000/80/120 | — | FR-001, FR-002, FR-018, NFR-003, NFR-004, NFR-007, NFR-008, NFR-009 |
| T-001 | M1 | Audience tags, context models, brands scoring, static risks | high | T2 | high | 80000/80/120 | T-000 | FR-004, FR-007, FR-008, FR-009, NFR-005 |
| T-002 | M1 | Stats, spend summary, health score, incident detection | low | T2 | high | 80000/80/120 | T-000 | FR-003, FR-006, NFR-002 |
| T-003 | M2 | Claims check, data-derived risks, AccountView, leak test | high | T2 | high | 80000/80/120 | T-001, T-002 | FR-005, FR-007, FR-009, NFR-005 |
| T-004 | M2 | Plotly chart and tile components | low | T2 | normal | 40000/40/45 | T-002 | FR-010, FR-011 |
| T-005 | M3 | Customer account-health page | high | T2 | high | 80000/80/120 | T-003, T-004 | FR-011, NFR-005 |
| T-006 | M3 | Internal QBR page | medium | T2 | high | 80000/80/120 | T-003, T-004 | FR-010 |
| T-007 | M3 | MCP server and FastAPI API | high | T2 | high | 80000/80/120 | T-003 | FR-015, FR-016, NFR-005 |
| T-008 | M4 | Narrative providers, leak guard, EBR markdown and DOCX | high | T2 | high | 80000/80/120 | T-003 | FR-012, FR-013, FR-014, NFR-001 |
| T-009 | M4 | Bench, card, CLI commands (score, claims, risks, pipeline, bench) | low | T2 | normal | 40000/40/45 | T-007, T-008 | FR-017, FR-018, NFR-006 |
| T-010 | M5 | README, runbook, design document | low | T2 | normal | 40000/40/45 | T-009 | FR-019, FR-020 |
| T-011 | M5 | Security review of B1/B2 surfaces, dependency audit, ship report | high | T2 | normal | 40000/40/45 | T-010 | NFR-005, NFR-009 |
| T-012 | M5 | Polish follow-ups from verifier findings and first screenshots (added 2026-09-10) | low | T2 | normal | 40000/40/45 | T-009 | FR-010, FR-011, NFR-005 |
| T-013 | M5 | OVERVIEW.md and SHOWCASE.md per portfolio convention (added 2026-09-10) | low | T2 | normal | 40000/30/30 | T-009 | FR-019, FR-020 |

## Dependency graph

```
T-000 ─┬─ T-001 ─┬─ T-003 ─┬─ T-005 ─┐
       │         │         ├─ T-006 ─┤
       └─ T-002 ─┤         ├─ T-007 ─┼─ T-009 ─ T-010 ─ T-011
                 └─ T-004 ─┘         │
                           T-008 ────┘
```

## Wave schedule

| Wave | Tasks (disjoint scopes) | Max parallel |
|---|---|---|
| W0 | T-000 | 1 |
| W1 | T-001, T-002 | 2 |
| W2 | T-003, T-004 | 2 |
| W3 | T-005, T-006, T-007 | 3 |
| W4 | T-008 | 1 (orchestrator captures screenshots meanwhile) |
| W5 | T-009 | 1 |
| W6 | T-012 (then orchestrator screenshots) | 1 |
| W7 | T-010, T-013 (disjoint docs) | 2 |
| W8 | T-011 | 1 |

## Validation gates per milestone

Every task: pack's targeted commands, then `uv run python scripts/check.py`. M0: CI green on
GitHub. M2: leak test parametrised over every model passes with N > 0 internal fields counted.
M3: `AppTest` renders both pages; MCP in-memory session drives six tools. M4: bench twice
identical; `render.py --check` clean; EBR guard test with hostile provider passes. M5:
pip-audit clean or dispositioned; evidence chain verifies; screenshots committed.

## Budget summary

T2 implementers: 8 high (80k) + 4 normal (40k) = 800k planned input tokens; verifiers at 0.4 =
320k; security reviews (medium/high) ≈ 200k. T3 (this session): design docs and orchestration
only. T1: none planned.

## Risks to the plan

| Risk | Mitigation |
|---|---|
| Streamlit `AppTest` flakiness on Windows | tests use `AppTest.from_file` with `default_timeout=30`; page modules keep logic in core |
| `mcp` SDK API drift | pin `mcp>=1.9,<2`; use in-memory client per DRYDOCK pattern |
| python-docx table rendering of Markdown pipes | limited Markdown subset defined in LLD §12 |
| Implementer touches shared `main.py` | page stubs created in T-000; packs forbid `main.py` |
