# High-level design — Lodestar

## Architecture style and rationale

A single Python package (`lodestar`) is the only place numbers are computed. Four thin
surfaces consume it: a Streamlit app (two pages), a FastAPI JSON API, an MCP server, and a
Typer CLI that also drives the bench and the EBR generator. Audience scoping is a property of
the data model (every field carries a visibility tag) and is applied by one function, so a
leak is a bug in one place and is testable in one place.

Why not two separate dashboards: the prompt allows either; one application with two pages
makes "same evidence, consistent story" structurally true rather than a matter of discipline.

## System context (C4 L1)

```
 CSV + YAML context ──► lodestar core ──► Streamlit pages (internal / customer)
                            │        ├──► FastAPI /api/{audience}/...
                            │        ├──► MCP server (stdio) tools(audience=...)
                            │        ├──► CLI: score / claims / risks / pipeline
                            │        ├──► EBR generator ──► docs/ebr/*.md, *.docx
                            │        └──► bench ──► metrics/headline.json ──► render.py ──► README card
                            └──► narrative providers: offline (default) | fireworks | anthropic
```

## Component breakdown (C4 L2)

| Component | Responsibility | Tier at build | Data it holds |
|---|---|---|---|
| `lodestar.data` | load, validate, derive; `Dataset` (frozen) | T2 | CSV rows |
| `lodestar.health` | pillar/metric scoring, bands, internal composite | T2 | none |
| `lodestar.claims` | headline-claim verification with assumptions | T2 | assumptions.yaml |
| `lodestar.risks` | data-derived + context risks, ranking | T2 | account_context.yaml |
| `lodestar.brands` | brand profiles, fit scoring, pilot ranking | T2 | brands.yaml |
| `lodestar.audience` | `Audience`, `Visibility`, `scrub()`, `AccountView` | T2 | none |
| `lodestar.narrative` | provider protocol, three providers, leak guard, EBR assembler, DOCX | T2 | template md |
| `lodestar.mcp_server` | six tools over `AccountView` | T2 | none |
| `lodestar.api` | FastAPI routes over `AccountView` | T2 | none |
| `lodestar.bench` | KPIs → headline.json | T2 | none |
| `lodestar.commands.*` | Typer sub-apps, auto-discovered | T2 | none |
| `app/` | Streamlit pages and Plotly components | T2 | none |
| `metrics/render.py` | portfolio card renderer (copied verbatim from sibling repos) | T0 | none |

## Data architecture

Immutable. `Dataset` wraps a list of validated `DayRow` models plus derived fields; every
compute function is pure `(Dataset, context) -> model`. Context YAML files are loaded once
into pydantic models whose fields carry `visibility` metadata via `Field(json_schema_extra=
{"visibility": "internal"})`. No writes except `metrics/headline.json`, `docs/ebr/*`.

## Critical flows

1. **Health score (FR-003/004).** load → derive → window L7/F7 → per-metric score → pillar
   mean → weighted composite → band; internal adds relationship pillar. Rendered on both pages;
   customer page shows operational composite only.
2. **Audience scrub (FR-009).** Any surface asks `AccountView.build(ds, ctx, audience)`.
   `scrub()` walks the model tree, drops internal-tagged fields for CUSTOMER, and returns a
   new model. MCP/API/EBR/customer page never receive an unscrubbed object.
3. **EBR generation (FR-012/013/014).** Build CUSTOMER view → fill template placeholders →
   optional provider polish of the executive-narrative section only → leak guard on provider
   output → write md and docx.
4. **MCP tool call (FR-015).** Host calls `get_health_score(audience="customer")` → scrub →
   JSON. Internal audience is opt-in per call and documented as such.
5. **Bench and card (FR-017).** `lodestar bench` computes KPIs (health, claims, leak count,
   incidents, lowest pillar, cost per automated ticket) → headline.json → render.py → SVG +
   README block → `--check` in CI.

## Cross-cutting concerns

Errors: one `LodestarError(code, message, detail)` with a closed code set (LLD §7). Config:
environment variables with defaults (LLD §8); `.env.example` committed. Logging: stdlib
logging, never logs API keys, never logs provider prompt bodies at INFO. Idempotency: all
generators overwrite deterministically.

## NFR design

| NFR | How met |
|---|---|
| NFR-001 offline | offline provider default; tests monkeypatch `httpx` to fail on any call |
| NFR-002 startup | pure pandas-free stdlib + pydantic compute on 31 rows; Streamlit caches `load()` |
| NFR-003 gate | no model calls, no Docker; pytest under a minute |
| NFR-005 leakage | visibility tags + scrub + parametrised leak test + bench KPI |
| NFR-006 determinism | no timestamps in headline.json; sorted keys |
| NFR-008 hardening | size/row caps before parse; per-cell numeric validation |

## Tech-stack decision

| Layer | Options | Trade-off | **Recommendation** | Rationale |
|---|---|---|---|---|
| Dashboard | Streamlit + Plotly / FastAPI + HTMX + Plotly.js / Next.js + FastAPI | Streamlit fastest to polish and simplest for a reviewer; HTMX more "engineered" but slower; Next.js heavy for a take-home | **Streamlit + Plotly** | reviewer runs `uv run lodestar dashboard`; polish comes from layout discipline, not framework |
| Compute | pandas / stdlib + pydantic | pandas is familiar but adds 100 MB and mypy friction; 31 rows need neither | **stdlib + pydantic v2** | fast startup, strict typing, deterministic |
| LLM access | Fireworks Python SDK / OpenAI SDK / httpx | SDKs add deps and version drift; httpx is enough for chat completions | **httpx against OpenAI-compatible endpoint** | Fireworks is the account's platform; one adapter also serves vLLM/Ollama |
| Agent integration | MCP stdio server / none | MCP is how LLM hosts consume this today | **Python `mcp` SDK, FastMCP** | mirrors LedgerLens/DRYDOCK conventions |
| EBR output | Markdown only / DOCX / PPTX | DOCX satisfies "brief" and opens anywhere; PPTX adds a second template to maintain | **Markdown + DOCX via python-docx** | executive-ready, diffable source |
| Gate | Makefile / `scripts/check.py` | no GNU make on the owner's host | **`scripts/check.py` with Makefile wrapper** | portfolio convention |

## Build-time model routing

`config/model-routing.yaml`: T3 = this session (design, orchestration, second-strike
diagnosis); T2 = Sonnet, effort high (implementation, verification, security review);
T1 = Haiku (evidence lines, commit messages) where used; T0 scripts everywhere else.
Owner instruction 2026-09-09: nothing above Sonnet for coding or testing.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Streamlit page code drifts from core numbers (hand-typed values) | pages call core only; a test greps `app/` for numeric literals outside layout constants |
| Leak through prose (narrative mentions "confidence") | deny-list guard on provider output and on the rendered EBR |
| Implementer expands scope (shared `cli.py`) | plug-in command discovery; packs list files |
| Reviewer on Windows | check runs on host and `--platform linux`; paths via `pathlib` |

## Decisions

D-000 autonomy and gate approval; D-001 single app, two pages; D-002 stdlib + pydantic over
pandas; D-003 visibility tags as the leakage control; D-004 Fireworks via OpenAI-compatible
httpx; D-005 recommended scaling architecture (shared base + per-brand LoRA + brand-scoped
RAG). All in `decisions.md`.
