# Requirements — Lodestar

## Functional requirements

| ID | Requirement | Acceptance criterion (testable) | Priority | Source |
|---|---|---|---|---|
| FR-001 | Load the 31-day CSV from a configurable path and validate its schema | Given `LODESTAR_CSV` unset, the default `data/northstar_flagship_30_day_metrics.csv` loads 31 validated rows; a file with a missing column raises `LodestarError(code=DATA_SCHEMA)` naming the column | Must | prompt: "where the application expects the CSV" |
| FR-002 | Compute derived daily metrics | `requests_per_ticket`, `tokens_per_request`, `cost_per_automated_ticket_usd`, `cost_per_1k_tokens_usd`, `week` label, `is_incident` (from operational note) exist for every row with values matching hand calculation for 2026-08-01 and 2026-08-31 | Must | design |
| FR-003 | Compute an account-health score with a transparent methodology | `HealthReport` exposes every metric score with floor, target, observed window value, pillar means, weights and composite; composite equals the weighted sum of pillar scores to 2 dp; band assigned per thresholds | Must | prompt D1 "transparent methodology" |
| FR-004 | Compute an internal relationship pillar and internal composite from `account_context.yaml` | Internal composite = 0.7 × operational + 0.3 × relationship; relationship inputs never appear in customer-audience output | Must | prompt D1 optional items |
| FR-005 | Verify the three headline claims against the telemetry | `claims_check()` returns three `ClaimCheck` items with status in {VERIFIED, VERIFIED_WITH_CAVEAT, REQUIRES_BASELINE} and the arithmetic used; automation claim is VERIFIED_WITH_CAVEAT (month mean 67.9%, exit 70.8%) | Must | prompt D3 §2 "metric context" |
| FR-006 | Detect incident days from the data, not only from notes | Days where p95 or error rate exceed a rolling threshold are flagged; 2026-08-09 and 2026-08-21 are flagged; no other day is | Must | prompt D2 "known caveats" |
| FR-007 | Produce a risk register with audience tags | Risks load from data-derived rules plus `account_context.yaml`; each has `visibility`; top-3 internal ranking is deterministic | Must | prompt D1 "top three risks" |
| FR-008 | Model the four-brand expansion pipeline with fit scoring | Each brand has descriptive fields (customer-visible) and scoring, confidence, blockers, commercial notes (internal); `rank_pilots()` is deterministic and returns Kestrel Athletics first under the shipped weights | Must | prompt D1 "expansion pipeline", D3 §5 |
| FR-009 | Enforce audience scoping in one place | `AccountView.for_audience(CUSTOMER)` contains zero fields tagged `internal`; a parametrised test walks every model and every MCP tool | Must | prompt D2 exclusion list |
| FR-010 | Internal QBR dashboard page | Streamlit page shows: internal health score with pillar breakdown and methodology expander; adoption, spend, reliability, latency, quality trend charts; top-3 risks; expansion pipeline table with confidence; dependencies / support burden / competitive / decisions-needed panel; next actions with owner and date | Must | prompt D1 |
| FR-011 | Customer account-health dashboard page | Streamlit page shows: automation, AHT, CSAT outcome tiles with baseline caveat; usage trend; availability, error, P50/P95; grounding, eval pass, escalation, caveats; spend trend and budget context; risks and recommended actions; expansion opportunity and next steps; no internal fields | Must | prompt D2 |
| FR-012 | Executive EBR brief generation | `lodestar ebr` writes Markdown and DOCX with the 8 required sections, figures pulled from the same core functions; passes the leak guard | Must | prompt D3 |
| FR-013 | LLM provider abstraction with offline default | Providers: `offline` (deterministic, default), `fireworks` (OpenAI-compatible chat completions), `anthropic` (optional extra); selection by `LODESTAR_LLM_PROVIDER`; missing key raises `LodestarError(code=PROVIDER_CONFIG)` before any network call | Must | portfolio convention; account context |
| FR-014 | LLM output leak guard | Any provider output containing an internal-only term or figure is rejected with `LodestarError(code=LEAK_GUARD)` and the offline text is used instead | Must | threat model B1/B2 |
| FR-015 | MCP server exposing account tools with audience parameter | Six tools; `audience` defaults to `customer`; customer calls return no internal fields; in-memory client test drives all six | Must | FDE showcase |
| FR-016 | FastAPI JSON API mirroring the MCP tools | `/api/{audience}/health`, `/trends`, `/risks`, `/pipeline`, `/claims`; OpenAPI served; customer routes leak nothing | Should | integration showcase |
| FR-017 | Offline bench writes `metrics/headline.json` | `lodestar bench` is deterministic; second run produces no diff; KPIs per LLD §9 | Must | portfolio convention |
| FR-018 | Plug-in CLI | `lodestar` discovers sub-apps from `lodestar/commands/*.py` without editing a registry; `lodestar --help` lists score, claims, risks, pipeline, ebr, bench, mcp, api, dashboard | Must | file-scope isolation |
| FR-019 | Design document with screenshots and methodology | `docs/design-document.md` covers every bullet of submission item 3 | Must | prompt submission §3 |
| FR-020 | README setup and run instructions | Covers every bullet of submission item 2 | Must | prompt submission §2 |

## Non-functional requirements (budgets)

| ID | Attribute | Budget | Measured how | Source |
|---|---|---|---|---|
| NFR-001 | Offline operation | All of `check`, dashboards, bench, EBR (offline provider), MCP run with no network and no API key | CI has no secrets; test asserts no socket opened for offline provider | portfolio |
| NFR-002 | Startup | Dashboard first render ≤ 10 s on a laptop; core `load()` + `health()` ≤ 1 s | timed test on `health()` ≤ 1 s | reviewer experience |
| NFR-003 | Gate time | `uv run python scripts/check.py` ≤ 5 min on GitHub Ubuntu runner | CI duration | portfolio |
| NFR-004 | Test coverage | ≥ 80% lines on `lodestar/` | pytest-cov fail-under | portfolio |
| NFR-005 | Leakage | 0 internal-tagged fields in any customer-audience output | leak test + bench KPI | prompt D2 |
| NFR-006 | Determinism | `bench` twice → identical `headline.json`; `render.py --check` clean | check step | portfolio |
| NFR-007 | Type safety | mypy strict on `lodestar/`, host and `--platform linux` | check step | portfolio |
| NFR-008 | Input hardening | CSV > 5 MB or > 10 000 rows rejected; non-numeric cells rejected with row number | unit tests | threat model B3 |
| NFR-009 | Secrets | No key material in repo; `.env.example` only; `gitleaks`-style regex scan in check | check step | prompt submission §5 |

## Constraints

Python 3.12; `uv`; Streamlit ≥ 1.38 with Plotly; FastAPI; Python `mcp` SDK; `python-docx`;
`httpx` for providers (no vendor SDK required for Fireworks); `pyyaml`; pydantic v2; Typer.
No database. No Docker required. Windows and Linux hosts.

## Assumptions (each with an owner to confirm)

| ID | Assumption | Value | Owner | Surfaced where |
|---|---|---|---|---|
| A-1 | Pre-deployment average handle time | 11.3 min | Northstar CX ops | both dashboards, EBR §2 |
| A-2 | Pre-deployment CSAT | 72.2 | Northstar CX ops | both dashboards, EBR §2 |
| A-3 | Availability SLO | 99.9% monthly (proposed, not yet agreed) | joint | both dashboards |
| A-4 | Fully loaded human Tier-1 ticket cost | $4.50 | Northstar finance | internal only (margin/ROI framing), EBR uses "vs human handling" qualitatively unless customer confirms |
| A-5 | Monthly inference budget line for flagship | $2,000 | Northstar | customer spend panel "budget context" |
| A-6 | Four brand profiles | fictional, in `data/brands.yaml` | prompt author | brands table header, design doc |
| A-7 | Relationship health inputs | fictional, in `data/account_context.yaml` | Fireworks account team | internal only |

## Out of scope (explicit)

Authentication for the dashboards (audience separation is by page and by code, not by login);
persistence of LLM outputs; real Fireworks account data; multi-deployment CSVs (schema allows
one `deployment` value; more is a validation error with a clear message); PDF export.

## RTM seed

See `docs/rtm.md` (one row per FR/NFR, regenerated by `scripts/shipyard/evidence.py rtm`).
