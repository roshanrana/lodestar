# Operational readiness review — Lodestar

Lodestar is a local, reviewer-run tool (no production deployment). The ORR therefore records
what a reviewer needs to run it reliably and what would be required before any shared hosting.

| Item | Owner | Status | Evidence |
|---|---|---|---|
| Single gate command exists and CI runs it | owner | done | `scripts/check.py`; `.github/workflows/check.yml`; GitHub Actions green on `main` |
| Runbook: start/stop, common failures, escalation | owner | done | `docs/ops/runbook.md` (error codes from LLD §7 mapped to fixes) |
| SLOs and dashboards | n/a (local tool) | n/a | Product SLOs for the *Northstar deployment* are proposed inside the dashboards (99.9% availability, P95 ≤ 1500 ms); none apply to Lodestar itself |
| Alerts routed; paging tested | n/a | n/a | local tool |
| Capacity vs projected load | owner | done | 31-row dataset; `health()` < 1 s (NFR-002 test); input caps 5 MB / 10 000 rows |
| Access review; break-glass | owner | done | No accounts. Internal view gated by `LODESTAR_INTERNAL_ENABLED` (default on locally, `0` in `.env.example`); before any shared hosting add authentication in front of the internal page, API and MCP server (documented limitation) |
| Secrets rotation | owner | done | No secrets in repo; provider keys read from env only; `scripts/secrets_scan.py` runs in the gate |
| Log retention per classification | owner | done | stdlib logging to stderr; never-log list in LLD §16 (keys, prompt bodies, internal fields) |
| Backup / restore; DR | n/a | n/a | Stateless; inputs are committed data files |
| Rollback rehearsed | owner | done | `git checkout <previous tag>` + `uv sync`; every commit on `main` passed the gate |
| Change record approved | owner | done | `docs/ops/change-record.md` |
| Model-risk sign-off (AI components) | owner | done | Product LLM use is optional prose polish only, offline by default; guard and injection tests (`tests/test_guard.py`, `tests/test_ebr.py`); paraphrase residual accepted in the threat model |

Go / no-go: **go** for submission and public repository. Not a go for shared hosting without
authentication in front of the internal surfaces.
