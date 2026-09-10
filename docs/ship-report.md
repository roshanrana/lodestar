# Ship report — Lodestar

Independent, evidence-based summary of what shipped, what was measured, what was written but
not run, review findings, and the deviations recorded across the build. Compiled by the T-011
security reviewer; no code, test, metric, or existing doc was modified to produce this report.

## What shipped

- **Surfaces**: one Streamlit app (`app/main.py`) with three pages — Home, "Customer · Account
  health", "Internal · QBR" — a FastAPI HTTP API (`lodestar/api.py`), a FastMCP server
  (`lodestar/mcp_server.py`, 6 tools), a Typer CLI (`lodestar/cli.py`, commands: `score`,
  `claims`, `risks`, `pipeline`, `ebr`, `bench`, `mcp`, `api`, `dashboard`), and a generated
  Executive Business Review (`docs/ebr/northstar-ebr-brief.md` and `.docx`).
- **Commands**: `lodestar bench` (writes `metrics/headline.json`, `metrics/card.json`),
  `lodestar ebr [--provider offline|fireworks|anthropic]`, `uv run python scripts/check.py`
  (gate: tests, secrets scan, and related checks), `python scripts/shipyard/evidence.py verify`.
- **Counts** (measured during this review, `uv run pytest -q` at HEAD):
  - Full test suite: **251 passed**, 0 failed (2 deprecation warnings, unrelated to this repo's
    code — `httpx`/`starlette` library warnings).
  - Targeted validation commands from the T-011 pack (`test_leak.py`, `test_guard.py`,
    `test_mcp.py`, `test_api.py`, `test_provider.py`, `test_data.py`): **85 passed**.
  - Task packs executed: 12 (T-000 through T-012, plus T-010/T-011/T-013 in flight at time of
    writing — see Deviations).
  - Verdicts on file: **11** (`docs/tasks/*.verdict.md`, T-000 through T-009 and T-012), all
    PASS (T-000 PASS after one environmental FAIL re-run).
  - Evidence ledger entries: **20** (`docs/evidence/ledger.jsonl`); chain verified valid by
    `python scripts/shipyard/evidence.py verify` → `chain valid`, run during this review.
  - Gates closed: G0-G5, G6.2 (132 tests, 97.7%), G6.3 (168 tests, 97.6%), G6.4 (246 tests,
    97.5%) — per `STATE.md`; the 251-test count above reflects tests added since G6.4 (T-012).
  - Dependency audit: 28 packages resolved via `uv export --all-extras`; `pip-audit` reports
    **0 known vulnerabilities**.
  - Secrets scan: `scripts/secrets_scan.py` → "no secrets found"; supplementary grep for
    `sk-ant-`, `fw_`, `AKIA`, `-----BEGIN` finds only the scanner's own pattern source and
    synthetic fixtures in `tests/test_config.py`.

## Measured figures (from `metrics/headline.json`, verbatim)

```json
{
  "kpis": {
    "health_score": {"label": "Operational health", "value": "80.2 / 100", "note": "Weighted L7 composite across five pillars; rated healthy."},
    "claims_verified": {"label": "Headline claims", "value": "1 of 3 from telemetry alone", "note": "3 of 3 reproduce with the stated baselines (assumed, owner Northstar CX ops)"},
    "leak_check": {"label": "Internal fields in customer view", "value": "0 of 37", "note": "Customer AccountView scrubbed of every internal-tagged field before it leaves AccountView.build."},
    "incidents_detected": {"label": "Incident days detected", "value": "2 of 2", "note": "Detector-flagged days matched against the dataset's own incident annotations."},
    "weakest_pillar": {"label": "Weakest pillar", "value": "Spend efficiency 65.8", "note": "Lowest-scoring of the five weighted pillars in the operational composite."},
    "cost_per_automated_ticket": {"label": "Inference cost per automated ticket", "value": "$0.0124", "note": "Month total spend divided by month total automated Tier-1 tickets."}
  },
  "bars": {
    "title": "Pillar scores (L7 window)",
    "rows": [
      {"label": "Adoption", "value": 80.2}, {"label": "Reliability", "value": 87.2},
      {"label": "Latency", "value": 78.5}, {"label": "Quality", "value": 83.1},
      {"label": "Spend efficiency", "value": 65.8}
    ]
  },
  "facts": {
    "rows": [
      {"label": "MCP tools exercised", "value": "6 of 6", "status": "ok"},
      {"label": "API routes", "value": "6 of 6", "status": "ok"},
      {"label": "EBR sections present", "value": "8 of 8", "status": "ok"},
      {"label": "Days below proposed SLO", "value": "27 of 31 at 99.9%", "status": "ok"},
      {"label": "Request vs automated growth", "value": "requests x1.45 vs automated tickets x1.15 (L7/F7)", "status": "ok"},
      {"label": "Live provider narrative", "value": "not run offline", "status": "pending"}
    ]
  }
}
```

(Reproduced from `metrics/headline.json` at review time; not recomputed or altered by this
report. "Days below proposed SLO" reflects the corrected anchor 27/31 — see Deviations.)

## Written but not run

- **Live Fireworks/Anthropic narrative**: the `fireworks` and `anthropic` providers
  (`lodestar/narrative/provider.py`) are implemented and unit-tested against mocked HTTP
  transports (`tests/test_provider.py`, `tests/test_ebr.py`), but no live network call to either
  vendor's API was made during the build or during this review — `headline.json.facts` marks
  "Live provider narrative" `status: "pending"` accordingly. The offline provider is the default
  and is what every reproducible figure in this report is computed against.
- **Screenshots**: `scripts/screenshots.py` (Playwright, an optional `screenshots` dependency
  group outside the gate) is a manual/on-demand tool, not part of `scripts/check.py`. Final
  screenshots were captured once during T-012 per `STATE.md`'s task log, not re-run by this
  review.
- **This review's own probes** were run against the offline/local build only (in-memory MCP
  session, FastAPI `TestClient`, direct model construction) — no probe in this report exercised
  a live network boundary, consistent with the threat model's B2 controls being unit- and
  stub-tested rather than live-tested.
- Nothing else in this task's scope was written but left unexecuted; `docs/security-review.md`
  documents every control as implemented with a passing test or a probe actually run during this
  review (see that file for the full control table and probe transcripts).

## Review findings and dispositions

Full transcripts and the C-01..C-17 control table are in `docs/security-review.md`. Summary:

- **No CRITICAL, HIGH, or MEDIUM findings.**
- 3 INFO findings, all disposed of with no code change required:
  1. `operational_note` is not currently rendered anywhere in the app (stronger than the threat
     model's C-12 description of it being "displayed as plain text") — a documentation-wording
     item, not a gap.
  2. The paraphrase-injection probe reproduces the threat model's own documented residual risk
     (B1 row 2, "paraphrase can evade") exactly as expected; confirms the rating is current.
  3. A prior verdict finding (`T-001.verdict.md` #1, untagged `HealthReport` fields) is now
     resolved — code inspection confirms all six named fields carry explicit `Visibility.CUSTOMER`
     tags in the current `lodestar/health.py`.
- Dependency audit (`pip-audit`, Probe 6) and secrets scan (Probe 7) are both clean; no
  disposition needed for either.
- See `docs/security-review.md` "What I would do next" for non-blocking follow-up suggestions
  (tightening the C-12 doc wording; a future semantic/paraphrase check as backlog hardening).

## Deviations

| Deviation | Detail | Recorded in |
|---|---|---|
| D-000 gate autonomy | Gates G0-G4, G7, G8 approved under the owner's standing autonomous authorisation ("work as autonomously as possible. go.") rather than per-gate replies; every gate entry records the approval text verbatim. | `decisions.md` D-000, `STATE.md` gate log |
| D-007 rename | Project renamed Sextant -> Lodestar after G4 (owner rejected "Sextant"); package, CLI, env-var prefix, docs, and repo renamed in one pass. Evidence ledger entries 1-5 keep their original pre-rename artifact hashes by design; `evidence.py verify` still validates the chain (confirmed `chain valid` during this review). | `decisions.md` D-007, `STATE.md` |
| T-000 environmental FAIL | The first verifier run shared the live working tree with Wave-1 implementers, causing an environmental FAIL. Process changed so every verification now runs in a git worktree pinned to the task's commit (`.worktrees/lodestar-T###`). T-000 then passed on an isolated re-run. | `docs/tasks/T-000.verdict.md` addendum, `STATE.md` |
| SLO-anchor correction 22->27 | The LLD §6/§9 anchor "22 of 31 days below SLO" was an architect arithmetic error; corrected to 27 after the implementer's recount, with dependent packs (T-008, T-009) updated. `metrics/headline.json` reflects the corrected value (27 of 31). | `STATE.md` (2026-09-10, T-002 row), `docs/tasks/T-002.verdict.md` |
| T-012 dual_axis SLO line backlog | The internal reliability chart's SLO reference line was dropped when that chart moved to the `dual_axis` helper, which has no `hline` parameter (the customer page's chart still shows the line). Accepted as a non-blocking backlog item, not scored against the T-012 verdict. | `STATE.md` backlog row, `docs/tasks/T-012.verdict.md` finding 1 |
| Budget drift on normal-complexity packs | `complexity: normal` packs consistently overran their 40k-token planned budget (T-004 1.63x, T-009 2.38x, T-012 2.38x) while `complexity: high` packs stayed near 1.0x. Logged as a routing-review item (D-008) for the retrospective; did not block any gate. | `STATE.md` budget ledger and routing-overrides section |

This report's own scope: T-011 touched only `docs/security-review.md` and this file; no code,
test, metric, or other existing doc was modified, and no new dependency was added. No probe in
this review revealed a gap requiring a new `tests/test_security_probes.py` file — every probe's
behavior is already covered by an existing, passing test (see `docs/security-review.md`'s
control table for the specific test per control).

## Links

- Evidence ledger: `docs/evidence/ledger.jsonl` (20 entries, `chain valid` per
  `python scripts/shipyard/evidence.py verify`, run during this review).
- Security review (full control table, probe transcripts, findings): `docs/security-review.md`.
