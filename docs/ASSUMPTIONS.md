# Assumptions — Lodestar

Everything the telemetry could not tell us, and every choice we made where the prompt left
room. Each entry says what was assumed, why, who can replace it with a fact, and where it
shows up. Values marked **assumed** are labelled that way on every surface they touch
(dashboards, EBR, CLI, MCP, API).

The single source for the numeric ones is `data/assumptions.yaml`. Fictional context lives in
`data/brands.yaml` and `data/account_context.yaml`. Change a value there and every surface,
the bench and the EBR regenerate from it.

---

## 1. Data and telemetry

| # | Assumption | Basis | Owner / how to replace | Where it appears |
|---|---|---|---|---|
| D-1 | The CSV is the complete record for the month: 31 daily rows, one deployment (`northstar-flagship-support-ft-v1`), no missing days. | The file as supplied. A second deployment or a duplicate date is a validation error, not a merge. | Fireworks telemetry | `lodestar/data.py` validation; README "Where the CSV goes" |
| D-2 | Rate columns (`automation_rate_pct`, `error_rate_pct`, `availability_pct`, …) are already daily aggregates computed upstream; we do not recompute them from counts except where a count-based figure is more honest (volume-weighted automation, Σ/Σ cost per ticket). | Column semantics are not documented; the supplied rates reconcile with the counts to within rounding. | Fireworks telemetry | LLD §3; claims table shows both month mean (67.92%) and volume-weighted (67.97%) |
| D-3 | `spend_usd` is inference spend only (no services, no platform fee) and is the customer-billed figure. | Column name; magnitude ($1,213 for the month) matches token volume at ~$1.12 per million tokens. | Fireworks finance | Spend panels; EBR §2 and §6 |
| D-4 | `tier1_tickets` and `automated_tier1_tickets` are per-day totals for the flagship brand only. | Deployment is brand-scoped. | Northstar CX ops | Automation tiles; claims check |
| D-5 | `operational_note` is free text written by an operator and is treated strictly as data: displayed as plain text, never sent to a model, never interpreted. Its keywords only classify a day as `incident`, `improvement` or `info`. | Threat model B3 / control C-12. | — | `note_kind` derivation; incident cross-check |
| D-6 | Request volume growing 64% while tickets grew 9% is real and unexplained; we surface it as a risk and an action rather than assuming a cause. | No conversation-level data in the file. | Fireworks FDE + Northstar CX platform lead (action due 2026-09-19) | Risk R-TECH-1 on both views; EBR talking point 4 |

## 2. Baselines, targets and budget lines (not in the telemetry)

| # | Assumption | Value | Basis | Owner / how to replace | Where it appears |
|---|---|---|---|---|---|
| A-1 | Pre-deployment average handle time | **11.3 min** (assumed) | Chosen so the month-end 7.35 min reproduces the stated 35% reduction. Within the month AHT fell 9.8%. | Northstar CX ops, from ACD exports (action due 2026-09-16) | AHT tile caption; claims table (`REQUIRES_BASELINE`); EBR §2 |
| A-2 | Pre-deployment CSAT | **72.2** (assumed) | Chosen so the month-end 84.2 reproduces the stated 12-point gain. Within the month CSAT rose 3.6. | Northstar CX ops, from the survey tool | CSAT tile caption; claims table; EBR §2 |
| A-3 | Availability SLO | **99.9% monthly** (proposed, not agreed) | No SLO was set at launch. 99.9% is the conventional interactive-service target; the month sat below it on 27 of 31 days. | Joint: Northstar VP Engineering + Fireworks (action due 2026-09-23) | Availability chart reference line; risk R-TECH-2; EBR talking point 3 |
| A-4 | P95 latency budget | **1,500 ms** (proposed) | Interactive chat comfort threshold; month-end P95 is 1,455 ms. | Joint | Latency chart reference line; health-score target is stricter (1,200) |
| A-5 | Monthly inference budget line for the flagship | **$2,000** (assumed) | Used only to express spend as a percentage (60.7%). Nothing depends on it. | Northstar finance | Spend panel "% of assumed budget"; EBR §2 |
| A-6 | Fully loaded human Tier-1 ticket cost | **$4.50** (assumed) | Order-of-magnitude framing only; never shown to the customer as savings. | Northstar finance | Internal view only; EBR §6 says "a rounding error against agent handling cost" without a figure |

## 3. Brands (fictional)

The prompt referred to "a short description of each additional brand" that was not supplied.
The four brands, and the flagship's name, are invented so the expansion analysis has something
concrete to score. They are labelled illustrative wherever displayed.

| # | Assumption | Detail | Replace with |
|---|---|---|---|
| B-1 | Flagship brand is **Polaris** (apparel and outdoor; 24k SKUs; ~4,600 Tier-1 tickets/day; Zendesk). | Sized to match the CSV's ticket volume. | Northstar's brand brief |
| B-2 | **Kestrel Athletics** — athletic apparel; 22k SKUs; 3,100 tickets/day; Salesforce Service Cloud; engaged GM. | Designed as the closest profile to the flagship: modern API-first tooling, similar catalog shape. Ranks first (fit 87.0). | Brand brief |
| B-3 | **Little Compass** — kids and baby; 28k SKUs; 2,400 tickets/day; Freshdesk; volume triples at seasonal peaks; recall handling stays human. | Ranks second (60.0); sequenced after the holiday peak. | Brand brief |
| B-4 | **Lumen Beauty** — cosmetics; 9k SKUs; 1,900 tickets/day; Gorgias on Shopify; allergen and usage questions border on medical advice; sceptical GM. | Ranks third (58.0); needs an answer-safety policy first. | Brand brief |
| B-5 | **Meridian Home** — furniture; 140k SKUs; 6,200 tickets/day; Zendesk with undocumented order-management middleware; bulky returns. | Largest value, hardest catalog; ranks fourth (57.0) and is sequenced last. | Brand brief |
| B-6 | Fit scoring weights: volume 0.20, catalog simplicity 0.20, tooling readiness 0.25, policy simplicity 0.15, data readiness 0.10, sponsor readiness 0.10; each dimension 1–5; fit = weighted mean × 20. | Tooling readiness weighted highest because integration effort, not model quality, dominated the flagship's month-1 support burden. | Fireworks FDE, after the pilot | `data/brands.yaml`; pipeline tables; design document |

## 4. Account context (internal only, fictional)

These exist so the internal QBR has something candid to show. None of it reaches the customer
view; the leak test and bench prove that.

| # | Assumption | Value |
|---|---|---|
| C-1 | Relationship inputs (0–100): executive sponsor engagement 80, champion strength 85, stakeholder coverage 55, renewal outlook 75, competitive position 60 → relationship score 71.0. | Internal composite = 0.7 × operational (80.2) + 0.3 × relationship (71.0) = 77.5 (watch). |
| C-2 | Stakeholder map: VP CX strong, VP Engineering light, CX platform lead strong, finance none, Kestrel GM medium, Meridian ops director none. | Drives risk R-REL-1 and the internal next actions. |
| C-3 | Contract renewal 2027-07-31; current ARR $210k; serving gross margin estimate 62%; 30 unbilled services hours in month 1. | Commercial framing only. |
| C-4 | Month-1 support burden: 46 FDE hours, 2 incidents, 7 tickets, 1 page. | Basis for "expansion services effort is the margin risk, not inference." |
| C-5 | Competitive pressure: incumbent contact-center vendor bundling agent-assist; a Salesforce bundled bot demoed to Kestrel; Gorgias and Freshdesk native assistants as paths of least resistance. | Internal risks R-COMP-1 and per-brand competitive notes. |
| C-6 | Internal dependencies: multi-adapter serving on the account's deployment class committed for Q4; catalog-sync resilience fix in progress; Salesforce sandbox requested 2026-08-26; baselines requested. | Internal dependencies panel and decisions needed. |

## 5. Health-score methodology choices

| # | Choice | Rationale |
|---|---|---|
| H-1 | Score the **last 7 days (L7)**, trend against the **first 7 days (F7)**. | A one-month-old deployment's current state matters more than its average; a 7-day window absorbs day-of-week effects and both incident days fall outside L7, so they show in trends and the reliability pillar's F7 comparison rather than dominating the headline. Whole-month scoring would read 77, not 80. |
| H-2 | Linear interpolation between a published floor (0) and target (100), clipped. | Anyone can recompute it by hand; no hidden curve. |
| H-3 | Weights: reliability 0.25, quality 0.25, adoption 0.20, latency 0.15, spend efficiency 0.15. | For a support bot, being up and being right outrank being fast; adoption is the customer's headline outcome; spend gets a seat because usage is decoupling from outcomes. |
| H-4 | Floors and targets (adoption 55→75%; escalation 20→10%; availability 99.5→99.95%; error 2.0→0.5%; P95 2,500→1,200 ms; P50 1,000→400 ms; grounding and eval pass 85→97%; CSAT 70→88; cost per automated ticket $0.03→$0.01; spend-vs-outcome ratio 1.5→1.0). | Set so a clearly good month scores in the 80s and a clearly bad one in the 50s; they are opinions, published as such, and frozen (D-006) so the score cannot be tuned after the fact. |
| H-5 | Bands: healthy ≥ 80, watch 65–79.99, at risk < 65. | Conventional traffic-light thresholds. |
| H-6 | Spend efficiency measures cost per automated ticket (Σ spend / Σ automated tickets) and a spend-growth vs outcome-growth ratio, not absolute spend. | Absolute spend should grow with success; the question is whether it grows faster than the outcome it buys (it does: ×1.44 vs ×1.15). |
| H-7 | The relationship pillar is internal-only and never enters the customer's number. | The customer sees an operational score they can audit; relationship judgement is Fireworks' own. |

## 6. Interpreting the three headline claims

| # | Claim | How we read it | Status shown |
|---|---|---|---|
| K-1 | "Handles 70% of Tier-1 support volume" | True as a run-rate: crossed on 2026-08-27, 70.1% in the final week, 70.8% on the last day. The month average is 67.9%. | `VERIFIED_WITH_CAVEAT` |
| K-2 | "Reduced average handle time by 35%" | Requires a pre-launch baseline the telemetry does not contain; with the assumed 11.3 min it reproduces exactly. Within-month change is 9.8%. | `REQUIRES_BASELINE` |
| K-3 | "Increased CSAT by 12 points" | Same pattern; assumed baseline 72.2 reproduces it. Within-month change is +3.6. | `REQUIRES_BASELINE` |

## 7. Incident detection

| # | Choice | Rationale |
|---|---|---|
| I-1 | A day is flagged when error rate exceeds 1.30 × the median of the previous ≤7 days, or P95 exceeds 1.20 × that median, with at least 3 prior days. | Simple, explainable, and it recovers exactly the two operator-noted incident days (08-09 catalog sync, 08-21 promotion burst) with no false positives. Thresholds were chosen for this dataset and should be re-validated on more months. |
| I-2 | The 08-24 note ("RAG index refresh improved grounding") is classified as an improvement, not an incident. | Keyword rule; visible in the grounding trend. |

## 8. Expansion, architecture and pilot

| # | Assumption | Basis |
|---|---|---|
| E-1 | Recommended architecture: one shared base fine-tune, one lightweight adapter per brand served multi-adapter on a single deployment, brand-specific system prompts, brand-scoped retrieval indexes with hard namespace isolation, shared evaluation harness with per-brand golden sets. | Cost (no per-brand deployments), latency (no cold small deployments), isolation (adapters trained only on their brand; retrieval cannot cross namespaces), quality (brand voice in adapter + prompt), maintenance (one base upgrade path). Assumes multi-adapter serving is available on the account's deployment class (internal dependency C-6). |
| E-2 | Pilot: Kestrel Athletics, 8 weeks (2 integration, 2 adapter and index, 2 shadow, 2 ramp 25%→100%). | Closest profile to the flagship and API-first tooling. |
| E-3 | Pilot success at week 8: automation ≥ 60%; CSAT within −1 point of the brand baseline; grounding ≥ 93%; P95 ≤ 1,800 ms; availability ≥ 99.9%; escalation ≤ 15%. | Deliberately below the flagship's month-1 exit figures: a new brand should clear the flagship's *launch* bar, not its *month-end* bar. |
| E-4 | Sequence after the pilot: Little Compass (post-holiday), Lumen Beauty (after answer-safety review), Meridian Home (after catalog-RAG hardening). | Readiness first, size last. |
| E-5 | Inference run-rate for all five brands stays in the low thousands of dollars per month; the real investment is integration and evaluation effort per brand. | Flagship spend $0.0124 per automated ticket; brand volumes from §3. |
| E-6 | The biggest risk is organisational (four help-desk stacks, four teams) rather than technical. | Month-1 effort went to integration and operations, not model quality. |

## 9. Tooling and environment

| # | Assumption |
|---|---|
| T-1 | Reviewer has Python 3.12 and `uv`; Windows or Linux; no Docker, no database, no API key needed. |
| T-2 | Streamlit is acceptable for a take-home dashboard; polish comes from layout discipline, and the compute lives in a typed core so it could front any UI. |
| T-3 | The default LLM provider is deterministic offline. Fireworks is reached through its OpenAI-compatible endpoint with `FIREWORKS_API_KEY`; Anthropic is optional. Live-provider narrative was **not** exercised in the bench (the card says so). |
| T-4 | Internal surfaces (internal page, `internal` audience on API and MCP) are unauthenticated because this is a local tool; `LODESTAR_INTERNAL_ENABLED=0` is the switch for any shared deployment, and authentication would be required before hosting. |
| T-5 | Paraphrase leakage through model prose is a residual risk: the guard blocks listed terms and internal figures, not every way of implying them. Accepted and disclosed in the threat model. |

## 10. Process

| # | Assumption |
|---|---|
| P-1 | The owner's standing instruction ("work as autonomously as possible; go") was treated as gate approval for G0–G4, G7 and G8 and recorded verbatim in the evidence ledger (decision D-000). |
| P-2 | Design and orchestration ran on a frontier model; every implementation, verification and security review ran on Sonnet-class models. No task needed a second attempt or an escalation. |
| P-3 | Verification is independent when it happens in a fresh context in a git worktree pinned to the task's commit; this became the rule after the first verifier tripped on another task's in-progress files. |
