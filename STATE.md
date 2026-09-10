# STATE — Lodestar
Phase: 8 — shipped 2026-09-10   Milestone: M5 (closed)   Wave: —   Updated: 2026-09-10T00:45:00Z

**Gate command:** `uv run python scripts/check.py` (or `make check`) — exists from T-000 onward.
**Routing:** T3 (this session) designs and orchestrates only; all implementation, verification
and security review at T2 Sonnet, effort high (owner instruction 2026-09-09).

## Now / next
- Shipped: `roshanrana/lodestar` release 0.1.0; gate green locally and in CI (251 tests, 97.5% coverage); evidence ledger 25 entries, chain valid; export in `docs/evidence/pack-2026-09-10/`.
- Published 2026-09-10: GitHub profile README rows + repo topics; LinkedIn Projects entry (thumbnail = social card) and feed post (lnkd.in/g_bcy5fY); resume `Roshan_Rana-Resume_FDE.docx/.pdf` updated (entry at the bottom of Engineering Portfolio, page count held at 4). Publishing flow captured in the user-level `portfolio-publish` skill.
- All 14 task packs PASS (T-000 … T-013); T-010 carries a provenance addendum; T-011 security review CLEAR.
- Backlog (none load-bearing): `dual_axis` hline option so the internal reliability chart regains its SLO line; SHOWCASE §6 should print the `/api/internal/health` curl it describes; a recorded live-provider (Fireworks) narrative run to replace the pending card row; authentication in front of internal surfaces before any shared hosting.

## Blocked
| Task | Since | Reason | Needs |
|---|---|---|---|

## Deviations from plan
| Date | Task | Deviation | Recorded in |
|---|---|---|---|
| 2026-09-09 | gates | Gates G0–G4 approved under the owner's standing autonomous authorisation rather than per-gate replies | decisions.md D-000 |
| 2026-09-09 | naming | Owner rejected "Sextant"; renamed to Lodestar after T-000 handoff (package, CLI, docs, repo) | this file, decisions.md D-007 |
| 2026-09-10 | T-000 | Verifier ran in the live tree concurrently with W1 implementers; environmental FAIL. Process change: every verification now runs in a git worktree pinned to the task's commit (`.worktrees/lodestar-T###`) | T-000.verdict.md addendum |
| 2026-09-10 | T-002 | LLD §6/§9 anchor "22 of 31 days below SLO" was an architect arithmetic error; corrected to 27 after the implementer's recount; packs T-008/T-009 updated | commit 41162c1 |
| 2026-09-10 | T-003 | Scope widened to visibility-tag metadata in health/spend/incidents/context/brands after T-001 verifier found untagged customer fields on HealthReport (fail-closed default would blank them); AC7 positive leak test added | T-003 pack, this row |
| 2026-09-10 | plan | T-012 polish pack added (M5, after T-009) for verifier non-blocking findings and visual defects; screenshots tooling (Playwright) added as an optional dependency group outside the gate | 04-execution-plan.md, this row |
| 2026-09-10 | plan | T-013 added: docs/OVERVIEW.md and docs/SHOWCASE.md required by the profile-README convention ("each has an OVERVIEW.md and a SHOWCASE.md") but missing from the original plan | 04-execution-plan.md, this row |
| 2026-09-10 | T-010 | Three T-010 deliverables were committed under the T-012 verdict commit (7d36723) by an orchestrator `git add docs`; content unchanged since and verified; attribution corrected in T-010.verdict.md addendum. Rule: stage explicit paths only while any Implementer is active | T-010.verdict.md |
| 2026-09-09 | dataset | Brand descriptions referenced by the prompt were not supplied; fictional profiles authored in `data/brands.yaml` | 00-problem-brief.md open questions |

## Task log
<!-- one line per event: ts task outcome attempt tier in≈tokens calls commit -->
2026-09-09T23:45Z T-000 HANDOFF v1 T2 in≈103k calls 52 (uncommitted; verifier dispatched)
2026-09-09T23:50Z rename Sextant→Lodestar applied by orchestrator (T0 script); gate green 25 tests 95%
2026-09-10T00:20Z T-000 verdict FAIL v1 (environmental: verifier shared the live tree with W1 implementers)
2026-09-10T00:35Z T-000 PASS after isolated worktree re-run at e1720f4b7c9e; evidence seq 6
2026-09-10T08:30Z T-013 PASS seq 24; T-011 CLEAR seq 22; G7 seq 23; G8 seq 25; evidence exported; shipped
2026-09-10T07:50Z T-010 verdict FAIL on provenance (orchestrator swept T-010 files into commit 7d36723 via `git add docs`); content PASS; addendum + evidence seq 21
2026-09-10T07:35Z T-013 HANDOFF v1 T2 in≈85k calls 27 commit 5e9c78ba0ed4 (verifier in worktree)
2026-09-10T07:05Z T-010 HANDOFF v1 T2 in≈95k calls 37 commit 78f0a5933136 (verifier in worktree); T-011 and T-013 dispatched; profile README row committed locally in roshanrana (push at ship)
2026-09-10T06:35Z T-012 PASS seq 20
2026-09-10T06:10Z T-012 HANDOFF v1 T2 in≈95k calls 44 commit 8daf11b11981 (verifier in worktree); final screenshots captured with scripts/screenshots.py; T-010 dispatched
2026-09-10T05:45Z T-009 PASS seq 18; G6.4 seq 19 (M4 closed)
2026-09-10T05:20Z T-009 HANDOFF v1 T2 in≈95k calls 55 commit 7264b19c6e6e (verifier in worktree); deviation accepted: growth fact ×1.45 (true value); orchestrator regenerated EBR after its own template edit had broken determinism
2026-09-10T05:21Z T-012 dispatched
2026-09-10T04:55Z T-008 PASS seq 17; non-blocking: test name prefix (→ T-012), render_ebr third param documented
2026-09-10T04:30Z T-008 HANDOFF v1 T2 in≈95k calls 48 (verifier in worktree); deviation: {{deployment}} placeholder uses CSV basename → AccountView.deployment field added to T-012
2026-09-10T03:50Z T-006 PASS seq 13; T-005 PASS seq 14; T-007 PASS seq 15; G6.3 seq 16 (M3 closed)
2026-09-10T04:00Z orchestrator: .streamlit/config.toml (light theme, minimal toolbar) and scripts/screenshots.py (Playwright, optional `screenshots` dependency group) added; T-012 polish pack created from verdict findings and first captures
2026-09-10T03:20Z T-005 HANDOFF v1 T2 in≈85k calls 35 commit 6ee74713e57d (verifier in worktree)
2026-09-10T03:22Z T-007 HANDOFF v1 T2 in≈90k calls 45 commit 768b2253c59e (verifier in worktree); deviation accepted: functools.cache
2026-09-10T03:05Z T-006 HANDOFF v1 T2 in≈90k calls 38 commit f153826694bc (verifier in worktree); T-008 dispatched
2026-09-10T02:45Z T-003 PASS v1 (verifier T2, worktree); evidence seq 11; G6.2 seq 12 (M2 closed)
2026-09-10T02:25Z T-003 HANDOFF v1 T2 in≈64k calls 33 commit de44de92aafa (verifier in worktree); deviation accepted: R-TECH-2 likelihood medium so AC2 ordering holds
2026-09-10T02:26Z W3 dispatched: T-005, T-006, T-007 (before T-003 verdict; accepted risk, gate green at handoff)
2026-09-10T02:05Z T-004 PASS v1 (verifier T2, worktree); evidence seq 10; no findings
2026-09-10T01:55Z T-004 HANDOFF v1 T2 in≈65k calls 20 commit b4bfb8b36dac (verifier dispatched in worktree)
2026-09-10T01:35Z T-001 PASS v1 (verifier T2, worktree); evidence seq 9; finding 1 → T-003 scope amendment
2026-09-10T01:20Z T-002 PASS v1 (verifier T2, worktree); evidence seq 8; 2 non-blocking findings → T-010 follow-ups
2026-09-10T01:05Z T-001 HANDOFF v1 T2 in≈62k calls 40 commit 018d780c1a28 (verifier dispatched in worktree)
2026-09-10T01:00Z G5 CI green on 1c60060; evidence seq 7
2026-09-10T00:40Z T-002 HANDOFF v1 T2 in≈85k calls 27 commit e67be262425f (verifier dispatched in worktree)

## Budget ledger
| Task | Tier | Planned in-tok | Actual | Calls | Attempts | Outcome |
|---|---|---|---|---|---|---|
| T-000 | T2 | 80000 | ~103000 (1.29×) | 52 | 1 | PASS |
| T-002 | T2 | 80000 | ~85000 (1.06×) | 27 | 1 | PASS |
| T-001 | T2 | 80000 | ~62000 (0.78×) | 40 | 1 | PASS |
| T-004 | T2 | 40000 | ~65000 (1.63×) | 20 | 1 | PASS |
| T-003 | T2 | 80000 | ~64000 (0.80×) | 33 | 1 | PASS |
| T-006 | T2 | 80000 | ~90000 (1.13×) | 38 | 1 | PASS |
| T-005 | T2 | 80000 | ~85000 (1.06×) | 35 | 1 | PASS |
| T-007 | T2 | 80000 | ~90000 (1.13×) | 45 | 1 | PASS |
| T-008 | T2 | 80000 | ~95000 (1.19×) | 48 | 1 | PASS |
| T-009 | T2 | 40000 | ~95000 (2.38×) | 55 | 1 | PASS |
| T-012 | T2 | 40000 | ~95000 (2.38×) | 44 | 1 | PASS |
| T-010 | T2 | 40000 | ~95000 (2.38×) | 37 | 1 | PASS |
| T-013 | T2 | 40000 | ~85000 (2.13×) | 27 | 1 | PASS |
| T-011 | T2 | 40000 | ~62000 (1.55×) | 32 | 1 | CLEAR |

## Gate log
| Gate | Date | Approver | Evidence seq |
|---|---|---|---|
| G0 | 2026-09-09 | Roshan Rana, delivery lead (D-000) | 1 |
| G1 | 2026-09-09 | Roshan Rana, delivery lead (D-000) | 2 |
| G2 | 2026-09-09 | Roshan Rana, delivery lead (D-000) | 3 |
| G3 | 2026-09-09 | Roshan Rana, delivery lead (D-000) | 4 |
| G4 | 2026-09-09 | Roshan Rana, delivery lead (D-000) — "approved" | 5 |
| G5 | 2026-09-10 | CI (GitHub Actions run 34421244802, success) | 7 |
| G6.2 | 2026-09-10 | Full gate in isolated worktree at de44de92aafa (132 tests, 97.7%) | 12 |
| G6.3 | 2026-09-10 | Full gate in isolated worktree at 768b2253c59e (168 tests, 97.6%) | 16 |
| G6.4 | 2026-09-10 | Full gate in isolated worktree at 7264b19c6e6e (246 tests, 97.5%) | 19 |
| G7 | 2026-09-10 | Roshan Rana, delivery lead (D-000); security review CLEAR, pip-audit clean | 23 |
| G8 | 2026-09-10 | Roshan Rana, delivery lead (D-000); ORR, change record, ship report | 25 |

## Routing overrides
- None. T2 packs with `complexity: high` run with reasoning on per config.
- Budget note: `complexity: normal` packs (T-004 1.63×, T-009 2.38×) consistently overran the 40k ceiling while `high` packs sat near 1.0×; the normal ceiling is too tight for tasks that touch four or more files. Routing review item for the retrospective (D-008 at ship).
