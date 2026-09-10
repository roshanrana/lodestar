# Problem brief — Lodestar

## Problem statement

Verbatim (Fireworks AI take-home, "Northstar Retail Group: EBR, Internal QBR & Account Health"):

> Use the same evidence to create three coordinated deliverables: an internal Fireworks QBR
> dashboard for account planning and decision-making; a customer-facing account-health
> dashboard for Northstar leadership; executive EBR content that communicates the flagship
> deployment's results and recommends a credible expansion plan. [...] Northstar Retail Group
> is a strategic account. One month ago, Fireworks helped move a fine-tuned customer-support
> chatbot from proof of concept to production for Northstar's flagship brand. The deployment
> now handles 70% of Tier-1 support volume, has reduced average handle time by 35%, has
> increased CSAT by 12 points. Northstar owns four additional retail brands [...] Your goal is
> to evaluate the health of the existing deployment, prepare an Executive Business Review,
> and make a credible case for expanding the solution to the other four brands.

Restated: one 31-day telemetry file describes one production deployment. From it, build one
application with two audience-scoped views and one executive brief, such that the internal
view is candid, the customer view is polished and never leaks internal judgement, and every
number in every surface traces to the same computation. The account-health score must have a
methodology a sceptical VP of Engineering can audit.

The name: a lodestar is the star a navigator steers by. Lodestar is the fixed point the
account team steers the Northstar account by.

## Business outcome and measures

| Outcome | Measure | Baseline | Target | By when |
|---|---|---|---|---|
| Reviewer can run both dashboards locally from README alone | Fresh-clone run succeeds with `uv sync` + one command, no API key | n/a | 100% | submission |
| Internal and customer views tell one story | Every figure on either view is produced by the same `lodestar` core function | n/a | 0 hand-typed figures in UI or EBR | submission |
| Customer view leaks nothing internal | Count of internal-tagged fields present in customer-audience output (UI models, API, MCP, EBR) | unknown | 0, enforced by test and bench | submission |
| Health score is auditable | Methodology page shows every input, floor, target, weight and the arithmetic | n/a | reproducible by hand from the CSV | submission |
| EBR is executive-ready and consistent | 8 required sections present; figures match the customer dashboard | n/a | 8 / 8 | submission |
| Portfolio: repo demonstrates FDE practice | Shipyard lifecycle evidence, MCP server, provider abstraction, measured card, CI gate | n/a | shipped to `roshanrana/lodestar` | 2026-09-10 |

## Users and actors

| Actor | Type | Needs | Volume |
|---|---|---|---|
| Fireworks account team, FDE, leadership | human, internal | Candid health, risks, decisions needed, next actions with owners, expansion pipeline and confidence | weekly |
| Northstar VP Customer Experience | human, customer | Outcomes (automation, AHT, CSAT), caveats, next steps | monthly / EBR |
| Northstar VP Engineering | human, customer | Availability, error rate, P50/P95, grounding, eval pass rate, spend, architecture for scale | monthly / EBR |
| Take-home reviewer | human | Run it, read it, judge the reasoning | once |
| LLM client (Claude Desktop / Code, any MCP host) | system | Query health, trends, risks, pipeline with audience scoping | ad hoc |
| CI | system | One gate command, deterministic, offline | every push |

## Regulatory and control context

- Jurisdictions / regulators: none binding; retail customer-support telemetry, synthetic.
- Internal frameworks in force: Shipyard lifecycle (this repo), owner's portfolio conventions
  (measured card, single `check` gate, CI runs the gate).
- External frameworks aligned to: none formally; audience separation follows the prompt's
  explicit exclusion list ("Do not expose internal sentiment, competitive intelligence, margin
  assumptions, expansion confidence, or other internal commercial judgments").
- Named gate approvers: | Gate | Role | Name |
  |---|---|---|
  | G0–G4, G7, G8 | Delivery lead / product owner | Roshan Rana (blanket autonomous authorisation, D-000) |
  | G5, G6.x | CI + fresh-context Verifier | automated |
- Evidence expectations: `docs/evidence/ledger.jsonl` hash-chained; verdict per task; bench
  writes `metrics/headline.json`; README card rendered from it and drift-checked.

## Data classification

| Data element | Class | PII/PCI/MNPI | Residency | Retention |
|---|---|---|---|---|
| `northstar_flagship_30_day_metrics.csv` (31 rows, daily aggregates) | internal (synthetic) | none | repo | permanent |
| `data/account_context.yaml` internal sections (relationship health, competitive notes, margin, confidence) | **internal-only** (fictional) | none | repo | permanent |
| `data/brands.yaml` (four brand profiles) | mixed: descriptive fields customer-visible; scoring and confidence internal-only | none | repo | permanent |
| `data/assumptions.yaml` (pre-launch baselines, unit costs) | customer-visible with "assumed" label | none | repo | permanent |
| LLM provider API keys | secret | n/a | env only, never repo | n/a |
| LLM prompts/outputs | internal; prompts carry only customer-audience aggregates | none | not persisted | n/a |

## Constraints

- Hosting / sovereignty: local only; reviewer's laptop; CI on GitHub-hosted Ubuntu.
- Approved technology catalogue: Python 3.12, `uv`, portfolio conventions from sibling repos
  (ruff, mypy strict, pytest with coverage floor, `scripts/check.py`, `metrics/render.py`).
- Approved AI providers: Fireworks AI (OpenAI-compatible endpoint) as the natural default for
  this account; Anthropic optional; deterministic offline provider mandatory and default.
- Build-time AI spend: high-capability model for design and orchestration only; Sonnet-class
  (effort high) ceiling for implementation and verification (owner instruction 2026-09-09).
- Hard dates: submission ZIP; target ship 2026-09-10.
- Team and skills: one owner plus agents.

## Integration landscape (summary)

Inputs: one CSV (path configurable), three YAML context files. Outputs: Streamlit app with two
pages, FastAPI JSON API, MCP server (stdio), CLI, Markdown + DOCX EBR, `metrics/headline.json`,
README card. No databases, no external services required at runtime.

## Open questions

1. Pre-deployment baselines for AHT and CSAT are not in the dataset (owner: Northstar CX ops;
   handled by documented assumption in `data/assumptions.yaml`, flagged on every surface).
2. Brand descriptions were referenced by the prompt but not supplied (owner: prompt author;
   handled by fictional profiles in `data/brands.yaml`, labelled as assumptions).
3. No availability SLO is stated (assumed 99.9% monthly; flagged as "proposed" to the customer).
4. Request volume grew 64% while ticket volume grew 9%; attribution unknown (surfaced as a
   risk and an action, not resolved).
