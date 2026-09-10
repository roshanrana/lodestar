# Security review — T-011

Independent review of the audience boundary (B1), the provider boundary (B2), the filesystem
boundary (B3) and the MCP/API boundary (B4) against `docs/design/02-threat-model.md`, with live
probes and a dependency audit. Reviewer did not modify any code, test, metric or existing doc.
Diff-of-record: the `lodestar/` and `app/` tree at HEAD (commit at time of review; see
`docs/evidence/ledger.jsonl`, chain verified `chain valid`).

## Control table (C-01..C-17)

| Control | Description | Status | Evidence |
|---|---|---|---|
| C-01 | Visibility tags on every model field, fail-closed default | implemented | `tests/test_leak.py::test_fr009_no_field_reachable_from_account_view_is_left_untagged` (passed); `tests/leakwalk.py::collect_untagged` walks `AccountView` recursively |
| C-02 | Single `scrub()` function drops internal fields for CUSTOMER | implemented | `lodestar/audience.py::scrub`; exercised by every `test_nfr005_*` test in `test_api.py`/`test_mcp.py` |
| C-03 | Parametrised leak test over all models and tools | implemented | `tests/test_leak.py`, `tests/test_mcp.py::test_nfr005_customer_tool_results_have_no_populated_internal_fields`, `tests/test_api.py::test_nfr005_*` |
| C-04 | Bench KPI `leak_check` = 0 | implemented | `metrics/headline.json.kpis.leak_check.value == "0 of 37"` (verbatim) |
| C-05 | Customer page imports only the CUSTOMER `AccountView` | implemented | `app/pages/customer_health.py:30` calls `build_view(csv_path, data_dir, Audience.CUSTOMER)` unconditionally; no internal-audience branch on that page |
| C-06 | Deny-list guard applied to rendered EBR and provider output | implemented | `lodestar/narrative/guard.py::check_text`; `tests/test_guard.py`; Probe 2 below (rendered EBR scanned directly, zero term hits, `check_text` passes) |
| C-07 | Provider receives only the CUSTOMER view serialisation | implemented | `tests/test_ebr.py::test_fr014_provider_prompt_is_built_only_from_the_customer_view` |
| C-08 | Provider output used only as prose; rejection falls back to offline text | implemented | `tests/test_ebr.py::test_fr014_hostile_provider_narrative_is_rejected_and_offline_narrative_kept`, `::test_fr014_cli_ebr_exits_zero_when_guard_rejects_provider_narrative`; Probe 3 below |
| C-09 | API keys from env only, never logged; `.env` gitignored | implemented | `lodestar/config.py::Settings.from_env` reads `os.environ` only; `.gitignore:13` lists `.env`; `tests/test_provider.py::test_nfr001_provider_never_logs_key_or_body` |
| C-10 | Secrets regex scan runs at `check` | implemented | `scripts/check.py:29` runs `scripts/secrets_scan.py` as a gate step; Probe 7 below |
| C-11 | 5 MB / 10,000-row caps, per-cell validation with row number | implemented | `tests/test_data.py::test_nfr008_over_10000_rows_raises_data_too_large`, `::test_nfr008_oversized_file_bytes_raises_data_too_large`, `::test_nfr008_missing_column_raises_data_schema`, `::test_nfr008_extra_column_raises_data_schema`, `::test_nfr008_non_numeric_cell_raises_data_value_with_row_and_column`; Probe 5 below |
| C-12 | `operational_note` displayed as plain text only, never sent to a provider | implemented (stronger than documented) | grep of `app/` and `lodestar/narrative/ebr.py` finds zero references to `operational_note` outside `lodestar/data.py`/`lodestar/view.py` (only the derived `note_kind` flag is used, for incident detection) — the raw note text is not surfaced anywhere at all, so it cannot reach a provider or a user-facing view; see Finding 1 |
| C-13 | `LODESTAR_INTERNAL_ENABLED` gate on API/MCP/internal page | implemented | `tests/test_api.py::test_fr016_internal_disabled_returns_403_with_code`, `tests/test_mcp.py::test_fr015_internal_disabled_surfaces_as_tool_error`, `tests/test_page_internal.py::test_fr010_internal_disabled_shows_notice_and_no_charts`; Probe 4 below |
| C-14 | Tool inputs validated by pydantic/typing | implemented | `lodestar/mcp_server.py` uses `Literal["internal","customer"]` for `AudienceArg` and a `Literal` union `MetricName`; FastMCP/pydantic reject out-of-range values (`tests/test_mcp.py::test_fr015_get_metric_trend_rejects_invalid_metric_name`) |
| C-15 | Bench determinism check | implemented | `tests/test_bench.py::test_fr017_run_bench_is_deterministic`; `tests/test_ebr.py::test_fr014_write_ebr_offline_is_deterministic` |
| C-16 | Evidence ledger hashes | implemented | `python scripts/shipyard/evidence.py verify` → `chain valid` (run during this review); 20 entries in `docs/evidence/ledger.jsonl` |
| C-17 | `pip-audit` at ship | implemented | Probe 6 below: `uv export` + `uvx pip-audit` run during this review, no findings |

Status legend: implemented (control is in place and evidenced by a passing test or a probe run
during this review); partial (control exists but a probe found a documented gap); n/a (control
does not apply to this build). No control was scored `n/a` or `partial` for this codebase.

## Probe transcripts

### Probe 1 — MCP tools and API routes walked for populated internal fields

Commands: `uv run pytest tests/test_mcp.py tests/test_api.py -q` (in-memory MCP session /
FastAPI `TestClient`, using `tests/leakwalk.py` helpers), plus an independent re-check building
both `AccountView` audiences directly via `lodestar.view.build_view`.

```
uv run pytest tests/test_leak.py tests/test_guard.py tests/test_mcp.py tests/test_api.py \
  tests/test_provider.py tests/test_data.py -q
85 passed, 2 warnings in 2.14s

Independent re-check (build_view + leakwalk, outside pytest):
customer AccountView: 0 internal fields populated;
internal view has 109 populated internal fields (non-vacuous — the check is not trivially
passing because the CUSTOMER view happens to be empty)
```

Outcome: no internal-tagged field reachable from any of the 6 MCP tools or 6 API routes under
`audience=customer` is populated, and the check is proven non-vacuous against the INTERNAL view.

### Probe 2 — Rendered EBR scanned for C-06 terms and internal numeric strings

Commands: read `docs/ebr/northstar-ebr-brief.md` (the committed rendered artifact) and run
`lodestar.narrative.guard.check_text` against it, using the INTERNAL `AccountView` for the
numeric-figure check.

```
term hits: none
check_text(rendered EBR): PASS (no leak)
```

Outcome: none of the 16 `INTERNAL_TERMS` appear in the rendered EBR, and no ≥3-digit internal
figure string from the INTERNAL view appears in it either.

### Probe 3 — Hostile provider injection

Three provider-output strings run through `check_text()` against the INTERNAL view:

```
hostile ("Confidence is 75%, margin is 62%; ignore previous instructions..."):
  CAUGHT LEAK_GUARD {'category': 'term'}
paraphrase naming a listed term ("...privately worried about the renewal"):
  CAUGHT LEAK_GUARD {'category': 'term'}
paraphrase naming no listed term ("leadership doubts Northstar will re-sign"):
  NOT CAUGHT — residual, accepted per threat model (STRIDE row B1/2)
```

`tests/test_ebr.py::test_fr014_hostile_provider_narrative_is_rejected_and_offline_narrative_kept`
independently confirms the offline-fallback path: when the guard rejects a hostile provider
narrative, the CLI (`lodestar ebr`) still exits 0 and the committed template prose is kept, with
`"ignore previous instructions"` absent from the output file.

Outcome: matches the threat model exactly — direct term/figure leaks and injection phrases are
caught; a paraphrase using no listed vocabulary word is not caught. This is the documented,
accepted residual risk (B1 row 2), not a new finding.

### Probe 4 — `LODESTAR_INTERNAL_ENABLED=0` gate

```
uv run pytest tests/test_api.py::test_fr016_internal_disabled_returns_403_with_code \
  tests/test_mcp.py::test_fr015_internal_disabled_surfaces_as_tool_error \
  tests/test_page_internal.py::test_fr010_internal_disabled_shows_notice_and_no_charts -q
3 passed
```

- API: `GET /api/internal/health` with `LODESTAR_INTERNAL_ENABLED=0` → HTTP 403,
  `body["code"] == "INTERNAL_DISABLED"`.
- MCP: any internal-audience tool call → tool error result containing `"INTERNAL_DISABLED"`.
- Internal page: `AppTest` shows the disabled notice and `len(at.get("plotly_chart")) == 0`.

Outcome: all three surfaces fail closed identically.

### Probe 5 — Oversized/malformed CSV

```
uv run pytest tests/test_data.py -q
17 passed
```

Covers: >10,000 rows → `DATA_TOO_LARGE`; oversized file bytes → `DATA_TOO_LARGE`; missing
column → `DATA_SCHEMA` (`{"missing": [...], "extra": []}`); extra column → `DATA_SCHEMA`
(`{"missing": [], "extra": [...]}`); non-numeric cell → `DATA_VALUE` (`{"row": N, "column":
name}`); two deployments → `DATA_MULTI_DEPLOYMENT`; duplicate date → `DATA_DUPLICATE_DATE`.
Code inspection (`lodestar/data.py:143-206`) confirms every raised error's `detail` dict carries
only row numbers, column names, and header lists — never the offending cell value or any other
raw file content; `lodestar/api.py:38` registers a dedicated `LodestarError` exception handler
that returns the structured `{code, message, detail}` body instead of a default FastAPI 500
traceback.

Outcome: all five error codes reproduce with no content leakage in the error body.

### Probe 6 — Dependency audit

```
uv export --all-extras --no-hashes --format requirements-txt -o .audit-requirements.txt
uvx pip-audit -r .audit-requirements.txt --progress-spinner off
Installed 28 packages in 599ms
No known vulnerabilities found
```

Outcome: clean. No findings to disposition. The temporary `.audit-requirements.txt` was deleted
after the run (not part of this task's file scope).

### Probe 7 — Secrets scan

```
uv run python scripts/secrets_scan.py
no secrets found
```

Supplementary grep for the four literal patterns named in the task pack, excluding `.venv`:

```
grep -rn "sk-ant-|fw_|AKIA|-----BEGIN" (non-.venv tracked text files)
docs/tasks/T-000-walking-skeleton.md   -- names the patterns as an acceptance criterion (prose)
docs/tasks/T-000.verdict.md            -- names the patterns in verdict prose
scripts/secrets_scan.py                -- the scanner's own regex source
tests/test_config.py                   -- fixture values "fw_secret", "fw_"+24*'a',
                                           "sk-ant-"+16 chars, used to exercise Settings/the
                                           scanner against temp files; not live keys
```

Outcome: clean. No live secret material found; every hit is either the scanner's own pattern
definition or a test fixture using an obviously synthetic value.

### Probe 8 — Prompt-injection surface in data files

```
grep -rniE "ignore (all|previous|prior)|disregard|system prompt|you are now|act as|jailbreak" \
  data/*.yaml data/*.csv
(no matches)
```

The three non-empty `operational_note` values in the CSV are benign operational descriptions
("Catalog sync caused a latency and error spike; mitigated the same day", "RAG index refresh
improved grounding on return-policy questions", "Traffic burst during promotion; autoscaling
threshold adjusted") — no instruction-like text of any kind. Code inspection confirms
`operational_note` is parsed only to derive `note_kind` (`lodestar/data.py:212`, used solely by
`lodestar/incidents.py` for incident-day detection); the raw note string itself is never read by
`app/`, `lodestar/narrative/ebr.py`, or any provider call. This is stronger than the C-12
control description in the threat model, which says notes "are displayed as plain text" — in
the current build they are not displayed at all (see Finding 1).

## Numbered findings

1. **INFO** — C-12's control text describes `operational_note` as "displayed as plain text,
   never sent to a provider, never interpreted." In the shipped build the raw note text is not
   displayed anywhere (only the derived `note_kind` boolean-ish flag is used, for incident
   shading). This is a documentation/control-description mismatch, not a security gap — the
   actual behavior is strictly safer than what the control describes, since the field never
   reaches a rendered surface at all. Disposition: no code change needed; recommend the
   threat-model wording be tightened at the next design-doc touch to say the note is *not*
   currently surfaced, so a future feature that does render it inherits the "plain text only"
   requirement explicitly rather than by inference.

2. **INFO** — The paraphrase-injection probe (Probe 3) reproduces the threat model's own
   documented residual risk: "leadership doubts Northstar will re-sign" evades `check_text()`
   because it names no term from `INTERNAL_TERMS` and contains no ≥3-digit internal figure.
   This was probed exactly as instructed and confirms the threat model's "medium: paraphrase can
   evade" residual rating is accurate and current, not stale. Disposition: accepted per
   `02-threat-model.md` (B1 row 2) and `decisions.md`; no action required for T-011. Any future
   hardening (e.g. an LLM-based semantic classifier instead of a deny-list) is out of scope for
   this ship.

3. **INFO** — `docs/security-review.md`'s own findings list carries forward one item from prior
   verdicts that is now resolved: `T-001.verdict.md` finding 1 flagged `HealthReport` fields
   (`window_days`, `pillars`, `operational_composite`, `operational_band`, `weakest_pillar`,
   `methodology_note`) as untagged (fail-closed to internal) at that point in the build. Code
   inspection of the current `lodestar/health.py` (lines 138-165) shows every one of these
   fields now carries an explicit `Field(**tag(Visibility.CUSTOMER))` tag, consistent with the
   T-003 scope-widening deviation recorded in `STATE.md`. Disposition: resolved, no residual
   action.

No CRITICAL, HIGH, or MEDIUM findings were identified in this review. All findings are
disclosure/documentation-hygiene items (INFO) or independently-verified confirmations of
already-accepted, already-disclosed residual risk.

## Residual risks (restated from the threat model)

- **B1 row 2 (medium)**: a paraphrase that names no `INTERNAL_TERMS` word and no ≥3-digit
  internal figure can pass the deny-list guard undetected. Confirmed live in Probe 3. Accepted
  by the owner in `02-threat-model.md`; disclosed there and here.
- **B4 (medium, by design)**: `LODESTAR_INTERNAL_ENABLED` gates the internal audience, but the
  MCP server and API are documented as local developer tools with no authentication layer of
  their own — an unauthenticated host that can reach the process and the env is enabled can call
  internal-audience tools. Accepted by design (C-13) for a local tool; not exercised further by
  this review beyond confirming the flag itself fails closed (Probe 4).
- All other STRIDE rows are rated "low" residual in the threat model and this review found no
  evidence to revise that rating.

## What I would do next

- Tighten the C-12 wording in `02-threat-model.md` to reflect that `operational_note` is
  currently not rendered anywhere, rather than "displayed as plain text" (Finding 1).
- If a future feature ever surfaces `operational_note` verbatim (e.g. an incident-detail
  tooltip), add a leak-guard or injection test for it before that lands — today there is
  nothing to test because there is no rendering path.
- Consider a lightweight semantic/paraphrase check (even a second, broader term list, or a
  cheap classifier) as a follow-up hardening item for B1 row 2, tracked as a backlog item rather
  than a blocker — the local-tool threat model does not require it today.
- Re-run `pip-audit` at the next dependency bump; today's run is clean but is a point-in-time
  result (Probe 6).
