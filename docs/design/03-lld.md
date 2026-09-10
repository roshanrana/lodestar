# Low-level design — Lodestar

Contracts in this document are FROZEN at G3. Task packs cite sections by number.

## §1 Repository layout

```
lodestar/                      core package (mypy strict)
  __init__.py                 __version__
  errors.py                   LodestarError + Code enum                       (§7)
  config.py                   Settings from env                              (§8)
  data.py                     DayRow, Dataset, load(), derive                (§2, §3)
  stats.py                    mean, median, window helpers                   (§3)
  health.py                   METHOD constants, HealthReport, score()        (§4)
  claims.py                   ClaimCheck, claims_check()                     (§5)
  incidents.py                incident detection                             (§6)
  audience.py                 Audience, Visibility, tag(), scrub()           (§10)
  context.py                  load_context(): assumptions, brands, account   (§11)
  risks.py                    Risk, risk_register(), top_risks()             (§11)
  brands.py                   BrandProfile, fit_score(), rank_pilots()       (§11)
  spend.py                    SpendSummary                                   (§3)
  view.py                     AccountView.build(ds, ctx, audience)           (§10)
  narrative/
    __init__.py
    provider.py               Provider protocol, offline/fireworks/anthropic (§12)
    guard.py                  leak guard                                     (§12)
    ebr.py                    render_ebr() -> md; write_docx()               (§12)
  mcp_server.py               FastMCP server, six tools                      (§13)
  api.py                      FastAPI app                                    (§13)
  bench.py                    run_bench() -> headline dict                   (§9)
  cli.py                      Typer root; auto-discovers commands/*          (§15)
  commands/                   one file per sub-app: score.py claims.py risks.py pipeline.py
                              ebr.py bench.py mcp.py api.py dashboard.py
app/
  main.py                     st.navigation entry                            (§14)
  pages/customer_health.py    Customer view                                  (§14)
  pages/internal_qbr.py       Internal view                                  (§14)
  components/charts.py        Plotly builders (pure functions)               (§14)
  components/tiles.py         KPI tile helpers                               (§14)
data/                         CSV + three YAML context files
docs/ebr/ebr_template.md      architect-authored prose with {{placeholders}} (§12)
metrics/render.py, card.json, headline.json
scripts/check.py, scripts/secrets_scan.py, scripts/shipyard/*
tests/                        pytest; names carry RTM ids
```

## §2 Data model (`lodestar/data.py`)

```python
class DayRow(BaseModel, frozen=True):
    date: datetime.date
    deployment: str
    requests: int; input_tokens: int; output_tokens: int; total_tokens: int
    spend_usd: float
    p50_latency_ms: int; p95_latency_ms: int
    error_rate_pct: float; availability_pct: float
    tier1_tickets: int; automated_tier1_tickets: int; automation_rate_pct: float
    avg_handle_time_min: float; csat_score: float
    grounded_answer_rate_pct: float; quality_eval_pass_rate_pct: float; escalation_rate_pct: float
    operational_note: str = ""
    # derived (computed in load(), not read from file)
    requests_per_ticket: float          # requests / tier1_tickets
    tokens_per_request: float           # total_tokens / requests
    cost_per_automated_ticket_usd: float  # spend_usd / automated_tier1_tickets
    cost_per_1m_tokens_usd: float       # spend_usd / total_tokens * 1_000_000
    week: int                           # 1-based: (day_index // 7) + 1 → days 1-7 week 1 … day 31 week 5
    note_kind: Literal["none", "incident", "improvement", "info"]
```

`note_kind`: `incident` if note matches `/spike|burst|outage|degrad|error/i`; `improvement` if
`/improv|refresh|upgrade|fix/i`; `info` if non-empty otherwise; `none` if empty. Order of checks
as listed (incident wins).

```python
class Dataset(BaseModel, frozen=True):
    rows: tuple[DayRow, ...]           # sorted by date ascending, unique dates
    deployment: str
    source_path: str
    def window(self, last_n: int) -> tuple[DayRow, ...]   # last n rows
    def first(self, n: int) -> tuple[DayRow, ...]
    def series(self, field: str) -> list[float]

def load(path: Path | None = None) -> Dataset
```

Validation (all raise `LodestarError`): file missing → `DATA_NOT_FOUND`; size > 5 MB or rows >
10 000 → `DATA_TOO_LARGE`; missing/extra column → `DATA_SCHEMA` (detail names columns);
unparsable cell → `DATA_VALUE` (detail: row number, column); more than one distinct
`deployment` → `DATA_MULTI_DEPLOYMENT`; duplicate date → `DATA_DUPLICATE_DATE`. Expected column
order is exactly the CSV header in `data/northstar_flagship_30_day_metrics.csv`.

## §3 Statistics and windows (`lodestar/stats.py`, `lodestar/spend.py`)

- `F7` = first 7 rows; `L7` = last 7 rows; `MONTH` = all rows.
- Window value of a rate/latency metric = arithmetic mean of daily values.
- `cost_per_automated_ticket(window)` = Σ spend / Σ automated tickets (not mean of ratios).
- `spend_growth_ratio` = Σ spend(L7) / Σ spend(F7); `automated_growth_ratio` = Σ automated(L7)
  / Σ automated(F7); `spend_vs_outcome_ratio` = spend_growth_ratio / automated_growth_ratio.
- Anchors the verifier must reproduce (±0.01 unless noted): month Σ requests 789 563; Σ spend
  1 213.12; Σ automated 97 488; Σ tier1 143 430; month cost per automated ticket 0.01244;
  requests_per_ticket 2026-08-01 = 4.345, 2026-08-31 = 6.548; tokens_per_request 2026-08-01 =
  1398.0 (±0.1); L7 mean automation 70.07; L7 mean availability 99.8997 (±0.001); L7 mean p95
  1463.1; spend_growth_ratio 1.436; automated_growth_ratio 1.147; spend_vs_outcome_ratio 1.253.

```python
class SpendSummary(BaseModel, frozen=True):
    month_total_usd: float; budget_line_usd: float; pct_of_budget: float
    l7_daily_mean_usd: float; projected_30d_flat_usd: float   # l7_daily_mean * 30
    projected_30d_at_growth_usd: float                        # l7_daily_mean * spend_growth_ratio * 30
    cost_per_automated_ticket_month_usd: float; cost_per_automated_ticket_l7_usd: float
    cost_per_1m_tokens_month_usd: float
    spend_growth_ratio: float; automated_growth_ratio: float; requests_growth_ratio: float
    spend_vs_outcome_ratio: float
```

## §4 Health-score methodology (FROZEN, D-006) (`lodestar/health.py`)

Scoring window: L7. Trend reference: F7. Each metric scores 0–100 by linear interpolation
between `floor` (→0) and `target` (→100), clipped to [0, 100]. For lower-is-better metrics the
floor is the larger number. Pillar score = arithmetic mean of its metric scores. Operational
composite = Σ weight × pillar. Bands: `healthy` ≥ 80; `watch` 65–79.99; `at_risk` < 65.

| Pillar | Weight | Metric | Direction | Floor | Target |
|---|---|---|---|---|---|
| adoption | 0.20 | automation_rate_pct | higher | 55 | 75 |
| adoption | | escalation_rate_pct | lower | 20 | 10 |
| reliability | 0.25 | availability_pct | higher | 99.5 | 99.95 |
| reliability | | error_rate_pct | lower | 2.0 | 0.5 |
| latency | 0.15 | p95_latency_ms | lower | 2500 | 1200 |
| latency | | p50_latency_ms | lower | 1000 | 400 |
| quality | 0.25 | grounded_answer_rate_pct | higher | 85 | 97 |
| quality | | quality_eval_pass_rate_pct | higher | 85 | 97 |
| quality | | csat_score | higher | 70 | 88 |
| spend_efficiency | 0.15 | cost_per_automated_ticket_usd (window Σ/Σ) | lower | 0.03 | 0.01 |
| spend_efficiency | | spend_vs_outcome_ratio | lower | 1.5 | 1.0 |

Expected on the shipped CSV (verifier anchors, ±0.3): adoption 80.2; reliability 87.2; latency
78.5; quality 83.1; spend_efficiency 65.8; **operational composite 80.2 → healthy**.

Internal extension: `relationship` pillar = mean of the five `relationship` inputs in
`account_context.yaml` (expected 71.0). `internal_composite` = 0.7 × operational + 0.3 ×
relationship (expected 77.5 → watch). Relationship inputs, the internal composite and its band
are `internal` visibility.

```python
class MetricScore(BaseModel, frozen=True):
    metric: str; direction: Literal["higher","lower"]; floor: float; target: float
    observed_l7: float; observed_f7: float; score: float; delta_vs_f7: float
class PillarScore(BaseModel, frozen=True):
    pillar: str; weight: float; score: float; metrics: tuple[MetricScore, ...]
    trend: Literal["improving","flat","declining"]   # mean delta_vs_f7 in score points: >+2 improving, <-2 declining
class HealthReport(BaseModel, frozen=True):
    window_days: int = 7
    pillars: tuple[PillarScore, ...]
    operational_composite: float; operational_band: Literal["healthy","watch","at_risk"]
    weakest_pillar: str
    relationship_inputs: dict[str, float] | None = tag(internal)
    relationship_score: float | None = tag(internal)
    internal_composite: float | None = tag(internal)
    internal_band: str | None = tag(internal)
    methodology_note: str   # one paragraph, fixed text, describing the arithmetic above
def score(ds: Dataset, relationship: Mapping[str, float] | None = None) -> HealthReport
# relationship is None for the customer audience; view.py passes ctx.account.relationship for internal
```

## §5 Claims check (`lodestar/claims.py`)

```python
class ClaimCheck(BaseModel, frozen=True):
    claim: str                       # "Handles 70% of Tier-1 support volume" etc.
    status: Literal["VERIFIED","VERIFIED_WITH_CAVEAT","REQUIRES_BASELINE","NOT_SUPPORTED"]
    observed: dict[str, float]       # named figures used
    arithmetic: str                  # human-readable, e.g. "(7.35 - 11.3) / 11.3 = -35.0%"
    caveat: str
    assumption_used: str | None      # key path in assumptions.yaml or None
def claims_check(ds: Dataset, assumptions: AssumptionSet) -> tuple[ClaimCheck, ClaimCheck, ClaimCheck]
```

Rules: automation — `VERIFIED` if month mean ≥ 70; `VERIFIED_WITH_CAVEAT` if L7 mean ≥ 70 but
month mean < 70 (expected: month mean 67.92, volume-weighted 67.97, L7 70.07, exit 70.8);
else `NOT_SUPPORTED`. AHT — within-month change (7.35 vs 8.15 = −9.8%) reported; status
`REQUIRES_BASELINE` with arithmetic against the assumed baseline (−35.0%). CSAT — same pattern
(+3.6 within month; +12.0 vs assumed 72.2). Caveat text must state that the baseline is
assumed and name its owner.

## §6 Incident detection (`lodestar/incidents.py`)

A day is `flagged` if it has ≥ 3 prior days and either `error_rate_pct > 1.30 × median(error
of previous ≤7 days)` or `p95_latency_ms > 1.20 × median(p95 of previous ≤7 days)`.
Expected: flagged = {2026-08-09, 2026-08-21}; both have `note_kind == "incident"`; no other day
flagged. `detect(ds: Dataset, slo_availability_pct: float) -> IncidentSummary` returning
`IncidentSummary{flagged_dates, note_incident_dates, matched, false_positives, missed,
days_below_slo: int, slo_availability_pct: float}` where `days_below_slo` counts days with
availability < the SLO passed in (expected 27 of 31 below 99.9; only 08-27 and 08-29..31 clear it).

## §7 Error taxonomy (`lodestar/errors.py`)

| Code | Class | Retryable | Surface to user | Logged fields |
|---|---|---|---|---|
| DATA_NOT_FOUND | input | no | yes, with expected path | path |
| DATA_TOO_LARGE | input | no | yes | size, rows |
| DATA_SCHEMA | input | no | yes, columns | missing, extra |
| DATA_VALUE | input | no | yes, row and column | row, column |
| DATA_MULTI_DEPLOYMENT | input | no | yes | deployments |
| DATA_DUPLICATE_DATE | input | no | yes | date |
| CONTEXT_INVALID | input | no | yes, file and field | file, field |
| PROVIDER_CONFIG | config | no | yes, env var name | var name only, never value |
| PROVIDER_HTTP | upstream | yes | yes, status | status, provider |
| LEAK_GUARD | policy | no | yes, term category | term category, never the text |
| INTERNAL_DISABLED | policy | no | yes | audience |

`class LodestarError(Exception): code: Code; message: str; detail: dict[str, object]`.

## §8 Configuration matrix (`lodestar/config.py`)

| Env var | Default | Purpose |
|---|---|---|
| LODESTAR_CSV | data/northstar_flagship_30_day_metrics.csv | telemetry path |
| LODESTAR_DATA_DIR | data | YAML context directory |
| LODESTAR_INTERNAL_ENABLED | 1 | when 0, internal audience raises INTERNAL_DISABLED in API/MCP and the internal page shows a notice |
| LODESTAR_LLM_PROVIDER | offline | offline / fireworks / anthropic |
| FIREWORKS_API_KEY | — | required only for fireworks |
| LODESTAR_FIREWORKS_MODEL | accounts/fireworks/models/llama-v3p1-70b-instruct | |
| LODESTAR_FIREWORKS_BASE_URL | https://api.fireworks.ai/inference/v1 | any OpenAI-compatible base works |
| ANTHROPIC_API_KEY | — | required only for anthropic |
| LODESTAR_ANTHROPIC_MODEL | claude-sonnet-5 | |
| LODESTAR_API_PORT | 8765 | |
| LODESTAR_DASHBOARD_PORT | 8501 | |

`.env.example` lists all with comments; `LODESTAR_INTERNAL_ENABLED=0` there with a comment
that local development uses 1.

## §9 Bench and card (`lodestar/bench.py`, `metrics/`)

`run_bench()` returns the headline dict; `lodestar bench` writes `metrics/headline.json` with
`json.dumps(indent=2, sort_keys=False)` and a trailing newline. No timestamps. `card.json`:

```json
{"title": "Lodestar account fix", "subtitle": "Every figure computed from the 31-day Northstar telemetry by the same functions that drive both dashboards, the MCP server and the EBR.", "make_target": "make bench", "kpi_order": ["health_score","claims_verified","leak_check","incidents_detected","weakest_pillar","cost_per_automated_ticket"]}
```

KPIs (label / value format / accent): `health_score` "Operational health" `"80.2 / 100"` teal;
`claims_verified` "Headline claims" `"1 of 3 from telemetry alone"` amber (note: 3 of 3 with
stated baselines); `leak_check` "Internal fields in customer view" `"0 of N"` violet where N =
count of internal-tagged fields across all models; `incidents_detected` "Incident days
detected" `"2 of 2"` blue; `weakest_pillar` "Weakest pillar" `"Spend efficiency 65.8"` red;
`cost_per_automated_ticket` "Inference cost per automated ticket" `"$0.0124"` teal.
Bars: title "Pillar scores (L7 window)", one row per pillar, max 100, accents in pillar order
teal, blue, violet, amber, red. Facts rows (status ok): MCP tools exercised (6), API routes
(count), EBR sections present (8 of 8), days below proposed SLO (27 of 31), request growth vs
ticket growth ratios; status `pending`: "Live provider narrative" (not run offline).

## §10 Audience and visibility (`lodestar/audience.py`, `lodestar/view.py`)

```python
class Audience(StrEnum): INTERNAL = "internal"; CUSTOMER = "customer"
class Visibility(StrEnum): INTERNAL = "internal"; CUSTOMER = "customer"
def tag(v: Visibility) -> dict   # returns json_schema_extra={"visibility": v}; use as Field(**tag(...)) or Field(default, **tag(...))
def field_visibility(model_cls, name) -> Visibility   # default INTERNAL when untagged (fail closed)
def scrub[T: BaseModel](obj: T, audience: Audience) -> T
```

`scrub` returns the object unchanged for INTERNAL. For CUSTOMER it walks the model recursively
(nested models, tuples/lists of models, dicts of models) and returns a new instance with
internal-tagged fields set to `None` (Optional fields) or an empty container; non-optional
scalar internal fields are a design error caught by `test_fr009_all_internal_fields_optional`.
`internal_field_count(model_cls) -> int` counts internal-tagged fields recursively (bench N).

```python
class AccountView(BaseModel, frozen=True):
    audience: Audience
    generated_from: str                          # CSV path basename
    period: tuple[date, date]
    health: HealthReport
    claims: tuple[ClaimCheck, ...]
    incidents: IncidentSummary
    spend: SpendSummary
    trends: dict[str, list[float]]               # metric name -> daily series (customer-visible)
    dates: list[date]
    risks: tuple[Risk, ...]                       # already filtered by visibility
    top_risks: tuple[Risk, ...]                   # 3
    brands: tuple[BrandProfile, ...]              # scrubbed per audience
    pilot: BrandProfile                           # top-ranked
    next_actions: tuple[NextAction, ...]          # filtered by visibility
    assumptions: AssumptionSet                    # customer-visible
    stakeholders: tuple[Stakeholder, ...] | None = tag(internal)
    dependencies: tuple[Dependency, ...] | None = tag(internal)
    support_burden: SupportBurden | None = tag(internal)
    margin: MarginNote | None = tag(internal)
    decisions_needed: tuple[Decision, ...] | None = tag(internal)
    @classmethod
    def build(cls, ds: Dataset, ctx: Context, audience: Audience) -> "AccountView"
```

`build` for CUSTOMER: filters `risks`/`next_actions` by item visibility, then applies `scrub`.
Every surface (pages, API, MCP, EBR) obtains data only via `AccountView.build`.

## §11 Context, risks, brands (`context.py`, `risks.py`, `brands.py`)

`Context{assumptions: AssumptionSet, brands: BrandCatalog, account: AccountContext}` loaded from
`LODESTAR_DATA_DIR`; unknown keys → `CONTEXT_INVALID`. Models mirror the YAML files; tags per the
comments in each YAML file.

Risks: `Risk{id, title, category: technical|relationship|execution|commercial|competitive,
severity: low|medium|high, likelihood: low|medium|high, visibility, detail, mitigation,
evidence: dict[str, float]}`. Data-derived risks generated in code:
- `R-TECH-1` "Usage growth is decoupled from ticket volume" (customer) — evidence: requests
  growth ratio, automated growth ratio, requests_per_ticket first/last, spend growth. severity
  high if spend_vs_outcome_ratio > 1.2.
- `R-TECH-2` "Availability below the proposed 99.9% SLO on most days" (customer) — evidence:
  days_below_slo, incident dates. severity medium.
- `R-EXEC-2` "Headline outcomes depend on baselines not in telemetry" (customer) — evidence:
  within-month deltas. severity medium.
Ranking for `top_risks(3)`: severity (high=3) × likelihood (3/2/1), ties by category order
technical, execution, relationship, commercial, competitive, then id. Expected internal top-3:
R-TECH-1, R-EXEC-1, R-REL-1 (R-EXEC-1 high/high = 9 outranks R-TECH-1 high/medium = 6; order:
R-EXEC-1, R-TECH-1, R-REL-1). Customer top-3: R-EXEC-1, R-TECH-1, R-TECH-2.

Brands: `fit_score(brand, weights) = round(Σ w × dim × 20, 1)`; expected Kestrel 87.0, Little
Compass 60.0, Lumen Beauty 58.0, Meridian Home 57.0; `rank_pilots()` sorts by fit desc then
name. Customer-visible `BrandProfile` fields per the YAML header comment; the rest Optional and
internal-tagged.

## §12 Narrative providers and EBR (`lodestar/narrative/`)

```python
class Provider(Protocol):
    name: str
    def complete(self, system: str, user: str, max_tokens: int = 600) -> str
class OfflineProvider: returns "" (caller keeps template prose); name "offline"
class OpenAICompatProvider(base_url, api_key, model, name="fireworks"):
    POST {base_url}/chat/completions {"model","messages":[{system},{user}],"max_tokens","temperature":0.2}
    via httpx.Client(timeout=30); non-2xx → PROVIDER_HTTP
class AnthropicProvider(api_key, model): POST https://api.anthropic.com/v1/messages, headers
    x-api-key, anthropic-version 2023-06-01; import nothing vendor-specific
def make_provider(settings) -> Provider   # missing key → PROVIDER_CONFIG before any I/O
```

Leak guard (`guard.py`): `INTERNAL_TERMS = ("confidence", "margin", "competitor", "competitive",
"renewal", "churn", "sentiment", "blocker", "arr", "gross margin", "unbilled", "incumbent",
"ccaas", "einstein", "gorgias ai", "freddy")` matched case-insensitively on word boundaries;
plus every numeric string from internal-tagged fields of the INTERNAL view (e.g. "210000",
"62"→ only when ≥ 3 digits). `check_text(text, view_internal) -> None | raises LEAK_GUARD`.
Applied to provider output and to the final rendered EBR Markdown.

EBR (`ebr.py`): `render_ebr(view_customer, provider) -> str` fills `docs/ebr/ebr_template.md`
placeholders `{{name}}` from a flat dict built from the customer view (see the template for the
full key list); if provider is not offline, the `{{executive_narrative}}` block is replaced by
provider output after the guard; otherwise the template's own prose stands.
`write_docx(md, path)` converts headings (#, ##), paragraphs, bullet lists, numbered lists and
pipe tables to python-docx; no images. Outputs: `docs/ebr/northstar-ebr-brief.md` and `.docx`.
Sections (H2, exact order): 1 Executive narrative; 2 Metric context; 3 Expansion
recommendation; 4 Technical scaling plan; 5 Pilot proposal; 6 Budget and timeline; 7 Executive
talking points; 8 Biggest risk and mitigation.

## §13 MCP tools and API (`mcp_server.py`, `api.py`)

FastMCP server name `lodestar`. Tools (all return JSON-serialisable dicts from scrubbed models;
`audience: Literal["internal","customer"] = "customer"`; internal with
`LODESTAR_INTERNAL_ENABLED=0` → INTERNAL_DISABLED as a tool error):

| Tool | Args | Returns |
|---|---|---|
| get_health_score | audience | HealthReport (scrubbed) |
| get_metric_trend | metric (Literal of DayRow numeric fields), audience | {metric, dates, values, l7_mean, f7_mean} |
| get_claims_check | — | list[ClaimCheck] |
| get_risks | audience, top_n: int = 3 | list[Risk] |
| get_expansion_pipeline | audience | list[BrandProfile] ranked |
| get_next_actions | audience | list[NextAction] |

API: `GET /healthz`; `GET /api/{audience}/health|trends?metric=|claims|risks?top_n=|pipeline|
actions`; errors map `LodestarError` → 400 (input/config), 403 (INTERNAL_DISABLED), 502
(PROVIDER_HTTP) with body `{code, message, detail}`.

## §14 Streamlit pages (`app/`)

`app/main.py`: `st.set_page_config(page_title="Lodestar", layout="wide")`; `st.navigation`
with three pages: Home (`app/pages/home.py`), "Customer · Account health", "Internal · QBR".
The walking skeleton (T-000) creates `main.py`, `home.py` and both page files as stubs that
render a title and one usage chart; later packs replace the page bodies without touching
`main.py`.
Home explains the two views, the data source and the assumptions banner. Both pages call
`AccountView.build` via `@st.cache_data` on `(csv_path, audience)`; every number rendered comes
from the view object. Internal page renders a red banner "INTERNAL — Fireworks only. Not for
customer distribution." and, when `LODESTAR_INTERNAL_ENABLED=0`, only a notice.

Customer page order (FR-011): header with period and data source; assumptions banner listing
A-1..A-3 with owners; row of four tiles (Tier-1 automation exit rate with month mean caption;
AHT with assumed-baseline caption; CSAT likewise; operational health score with band); usage
trend (requests and automated tickets, dual axis, incident shading); reliability row
(availability with SLO line, error rate, P50/P95 with P95 budget line); quality row (grounding,
eval pass, escalation) and "Known caveats" expander (claims table with status and arithmetic);
spend row (daily spend, month total vs budget line, cost per automated ticket, projected next
month flat vs at growth); risks and recommended actions (customer risks, joint next steps
table); expansion opportunity (brand table customer fields, readiness bar, proposed sequence,
pilot card with success criteria); methodology expander (pillar table with floors/targets/
weights and the arithmetic). No relationship, confidence, margin, competitive or stakeholder
content anywhere; a test imports the page module and asserts it never references those
attributes.

Internal page order (FR-010): banner; tiles (internal composite and band, operational
composite, relationship score, weakest pillar); health breakdown table with trend arrows and
methodology expander; five trend charts (adoption, spend, reliability, latency, quality) with
incident shading and F7/L7 shading; top-3 risks cards with evidence and mitigation; expansion
pipeline table (stage, fit, confidence, est. monthly inference, annual value, blockers, target
date, owner) with fit-vs-confidence scatter; panel with four columns: dependencies, support
burden, competitive notes, decisions needed (owner, by); next actions table (all, with
visibility column); stakeholder coverage table; claims table.

`components/charts.py` exposes pure functions returning `plotly.graph_objects.Figure`:
`line_with_incidents(dates, series: dict[str, list[float]], incident_dates, title, y_title,
hline: float | None = None, hline_label="")`, `dual_axis(...)`, `pillar_bar(report)`,
`fit_confidence_scatter(brands)`. Palette: primary `#2563eb`, secondary `#0d9488`, warn
`#d97706`, bad `#dc2626`, muted `#64748b`; template `plotly_white`; incident shading
`rgba(220,38,38,0.08)`; window shading `rgba(37,99,235,0.06)`.

## §15 CLI plug-in discovery (`lodestar/cli.py`, `lodestar/commands/`)

`cli.py` creates `app = typer.Typer(no_args_is_help=True)` and, at import, iterates
`pkgutil.iter_modules(lodestar.commands.__path__)` sorted by name; each module must expose
`app: typer.Typer` and `NAME: str`; `root.add_typer(module.app, name=module.NAME)`. Commands:
`score [--audience] [--json]`, `claims [--json]`, `risks [--audience] [--top 3]`, `pipeline
[--audience]`, `ebr [--provider] [--out docs/ebr]`, `bench [--check]`, `mcp` (stdio), `api
[--port]`, `dashboard [--port]` (runs `streamlit run app/main.py` via subprocess).
`pyproject` script: `lodestar = "lodestar.cli:app"`.

## §16 Observability and never-log list

Stdlib logging, logger `lodestar`. INFO: dataset loaded (rows, path), provider selected (name,
model), guard rejections (category). Never logged: API keys, provider prompt or completion
bodies, any internal-tagged field value at INFO or above.

## §17 Test strategy

pytest, `tests/` mirrors modules; every test function name starts with `test_fr###_` or
`test_nfr###_`; fixtures: shipped CSV (real), `tiny_csv` factory for error cases, `hostile
provider` stub returning internal terms and injection text, `httpx` blocked in offline tests.
Coverage floor 80% on `lodestar/`. `app/` is exercised by importing modules with
`streamlit.testing.v1.AppTest` for both pages (render without exception, no internal
attribute access on the customer page). Contract tests: MCP via in-memory client session;
API via `fastapi.testclient`. Determinism: bench twice equal.
