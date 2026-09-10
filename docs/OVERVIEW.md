# Lodestar — Overview

**What it is:** a small Python application that turns one 31-day support-deployment telemetry
file into three coordinated deliverables — a candid internal account-planning view, a polished
customer-facing account-health view, and an executive business review — all computed by the same
core functions, so the three can never disagree on a number and the customer view can never see
what it should not.

**Read this if** you want the design reasoning. [SHOWCASE.md](SHOWCASE.md) tours the features
with commands you can run yourself.

## The setting

One account team supports one production deployment — a fine-tuned customer-support assistant
now handling a majority of Tier-1 volume for one Northstar Retail Group brand — and has to speak
about it to three different audiences from the same 31 rows of daily telemetry: the Fireworks
account team, who need the unvarnished internal picture (relationship health, margin, competitive
notes, expansion confidence) to plan the account; Northstar's own VP Customer Experience and VP
Engineering, who need outcomes and caveats but nothing about Fireworks' internal commercial
judgement; and an executive audience who need an eight-section business review whose figures
match what the customer dashboard already shows them. One telemetry file, three audiences, and a
hard rule that the customer-facing surfaces must never leak the internal ones.

## What Lodestar is

A single Python package (`lodestar`) is the only place any number is computed. Four thin surfaces
read from it: a two-page Streamlit app, a FastAPI JSON API, an MCP server (stdio), and a Typer
CLI that also drives the bench and the EBR generator. Nothing is hand-typed on any surface —
every figure in the dashboards, the API, the MCP tool results and the EBR markdown traces back to
one `AccountView.build(dataset, context, audience)` call.

## The design bets

**One core, four surfaces.** Two dashboards were allowed by the brief; Lodestar builds one
Streamlit app with two pages instead, because "the internal and customer stories are consistent"
becomes a structural property of sharing one core rather than a discipline two separate codebases
would have to maintain by hand.

**Audience tags fail closed.** Every field on every report model carries a `visibility` tag
(`internal` or `customer`) via pydantic's `json_schema_extra`. A single `scrub()` function walks
the object graph and blanks every internal-tagged field for the customer audience. A field left
untagged defaults to `internal` — so a developer who forgets to tag a new field gets it silently
hidden from customers rather than silently leaked to them, and a test fails immediately because
the field is now untagged rather than explicitly scoped.

**Offline-first providers.** The default narrative provider is deterministic and makes no network
call at all, so a reviewer can run every command in this repository, including EBR generation,
with no API key. A live provider (Fireworks, via an OpenAI-compatible `httpx` adapter, or
Anthropic) is opt-in through `LODESTAR_LLM_PROVIDER`, and even its output is not trusted directly:
a deny-list leak guard scans any provider-generated prose (and the final rendered EBR) for
internal terms and internal-only numeric figures before it can reach a customer-facing document.

**A measured card, not an asserted one.** `lodestar bench` recomputes six KPIs from the same core
functions and writes them to `metrics/headline.json`, deterministically (no timestamps, sorted
keys). The README's results card is rendered from that file and drift-checked in the same gate
that runs the tests — the numbers below are not typed into a document, they are read out of an
artifact the gate itself produced.

## What is measured

Six KPIs, from `metrics/headline.json`, produced by `uv run lodestar bench`:

| KPI | Value | What it means |
|---|---|---|
| Operational health | **80.2 / 100** (healthy) | Weighted L7 composite across five pillars (adoption, reliability, latency, quality, spend efficiency). |
| Headline claims | **1 of 3 from telemetry alone** | 3 of 3 reproduce once the stated pre-deployment baselines (assumed, owner Northstar CX ops) are applied. |
| Internal fields in customer view | **0 of 37** | The customer `AccountView` has zero populated internal-tagged fields, proven by a recursive walk of the model graph, not a hand-picked field list. |
| Incident days detected | **2 of 2** | Rolling-threshold detector flags match the dataset's own incident annotations exactly, with no false positives. |
| Weakest pillar | **Spend efficiency 65.8** | The lowest-scoring of the five weighted pillars in the operational composite. |
| Inference cost per automated ticket | **$0.0124** | Month total spend divided by month total automated Tier-1 tickets. |

The health-score methodology (five pillars, per-metric floors and targets, L7/F7 windows, band
thresholds) is frozen by decision D-006 and published in full — with the worked arithmetic — on
the account-health dashboard's methodology expander and in `docs/design/03-lld.md` §4.

## What is deliberately not claimed

The 70% Tier-1 automation figure is a claim about the deployment's exit-week run rate (70.8% on
2026-08-27), not the month average (67.9%) — the bench reports it as `VERIFIED_WITH_CAVEAT`, and
the claims check shows both numbers side by side rather than picking the more favourable one. The
35% average-handle-time reduction and the 12-point CSAT gain both require an assumed
pre-deployment baseline that does not exist in the telemetry (`REQUIRES_BASELINE`); the assumed
values (11.3 min, 72.2) are labelled as assumed with their owner (Northstar CX ops) everywhere
they appear, and the within-month change is reported alongside them so the reader can see what
the data alone supports. No availability SLO was agreed at launch — 99.9% is proposed, not
adopted, and 27 of the 31 days sit below it. The four additional brand profiles are fictional
placeholders (the take-home prompt referenced brand descriptions that were never supplied), and
every surface that shows them says so. The divergence between request growth and ticket growth
across the month is surfaced as an open risk, not resolved with a guess at its cause. The
live-provider narrative row on the results card is marked `pending` — nothing about a live
model's output quality is measured offline, only the offline path is.

## How it was built

Lodestar was built under a Shipyard-governed lifecycle: a frozen problem brief, requirements,
high- and low-level design, and a threat model, each gated (G0–G4) under the delivery lead's
standing autonomous authorisation (decision D-000), followed by a sequenced execution plan of
small, independently-verifiable tasks (T-000 onward). Implementation and verification ran at a
fixed model tier (Sonnet, effort high) per an explicit build-time routing decision, with a
separate, higher-capability tier reserved for design and orchestration only. Every task is
verified in a fresh-context pass: after an early verification run shared a live working tree with
a concurrent implementer and produced an environmental false failure, every subsequent
verification runs in its own git worktree pinned to the task's commit
(`.worktrees/lodestar-T###`), so a verdict reflects exactly the code at that commit and nothing
an adjacent task changed underneath it. Every gate and task verdict is appended to a hash-chained
evidence ledger (`docs/evidence/ledger.jsonl`), so the sequence of what was approved, by what
role, over which artifacts, cannot be reordered or edited after the fact without breaking the
chain. `STATE.md` and `docs/design/decisions.md` carry the full task log, budget ledger, and the
append-only decision record, including the deviations this process needed along the way.
