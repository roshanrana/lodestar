# Northstar Retail Group — Executive Business Review

**Prepared for:** VP Customer Experience, VP Engineering · **Prepared by:** Fireworks AI account team
**Deployment:** northstar-flagship-support-ft-v1 · **Period:** 2026-08-01 to 2026-08-31 (31 days)
**Source:** every figure below is computed from the shared 31-day telemetry; assumptions are marked.

## 1. Executive narrative

The flagship deployment crossed 70% Tier-1 automation in its final week and closed the month at 70.8%. Average handle time fell to 7.35 minutes and CSAT rose to 84.2 over the same 31-day period, while the operational health score finished at 80.2 out of 100, rated healthy.

These gains matter because they came from a single brand and one catalog running on one shared architecture, the conditions most within reach today. Spend stayed at 60.7% of the assumed monthly budget line, and inference cost per automated ticket held near $0.0124, showing the pattern scales in cost as well as in outcomes.

The right next step is a single, well-measured pilot rather than a wide rollout: starting with Kestrel Athletics on the same shared-base architecture preserves the discipline behind this month's results while building the integration playbook the remaining brands will follow in sequence.

## 2. Metric context

| Outcome | Figure | What the telemetry shows | Caveat |
|---|---|---|---|
| Tier-1 automation | 70.8% at month end | 67.9% average across the month; 70.1% in the final week | The 70% figure is a run-rate, not a month average. It was crossed on 2026-08-27. |
| Average handle time | 7.35 min | Fell 9.8% within the month (from 8.15 min) | The 35% reduction compares against an assumed pre-launch baseline of 11.3 min (Northstar CX operations to confirm). |
| CSAT | 84.2 | Rose 3.6 points within the month | The 12-point gain compares against an assumed pre-launch baseline of 72.2 (Northstar CX operations to confirm). |
| Availability | 99.9% (final week) | 27 of 31 days sat below the proposed 99.9% SLO; two incident days (2026-08-09 and 2026-08-21) | No SLO was agreed at launch; we propose one below. |
| Latency | P50 537 ms · P95 1463 ms | Both improved roughly a quarter over the month | P95 spiked to 2168 ms during the catalog-sync incident. |
| Quality | Grounding 96.0% · Eval pass 94.8% · Escalations 11.5% | All three improved every week; the 24 Aug index refresh is visible in grounding | Quality is measured by automated evaluation, not by human audit; a monthly human sample would close that gap. |
| Spend | $1,213 for the month | 60.7% of the assumed $2,000 budget line; $0.0124 per automated ticket | Request volume grew 44.6% while automated tickets grew 14.7%; attribution is the first joint action. |

**Operational health score: 80.2 / 100 (healthy).** Weakest pillar: Spend efficiency 65.8. Methodology is published on the account-health dashboard.

## 3. Expansion recommendation

Expand, in sequence, not all at once. The flagship result is real but young: one month, one brand, one catalog. The conditions that produced it are a clean product catalog, API-first help-desk tooling and an engaged operating team. Two of the four remaining brands share those conditions today; two do not yet. The recommendation is a single pilot that proves a repeatable playbook, then a staged rollout ordered by readiness rather than by size.

Proposed order: 1. Kestrel Athletics (Weeks 1-8 (pilot)); 2. Little Compass (Q1 after holiday peak); 3. Lumen Beauty (Q1-Q2, after safety-policy review); 4. Meridian Home (Q2, after catalog-RAG hardening).

## 4. Technical scaling plan

**Recommended architecture: one shared base fine-tune, one lightweight adapter per brand, brand-specific system prompts, and brand-scoped retrieval indexes, served on a single multi-adapter deployment.**

| Concern | Separate fine-tuned model per brand | Shared model with brand context only | **Shared base + per-brand adapter (recommended)** |
|---|---|---|---|
| Cost | Five deployments, five capacity reservations | One deployment | One deployment; adapters add negligible serving cost |
| Latency | Cold or under-utilised small deployments per brand | Same as today | Same as today; adapter selection is per request |
| Data isolation | Strong | Weak: one model sees all brands' data | Strong: each adapter trains only on its brand; retrieval namespaces are isolated |
| Quality | Best brand voice, slowest to improve everywhere | Brand voice and policy leak across brands in prompts alone | Brand voice in adapter + prompt; policy in the index; per-brand golden evals gate every promotion |
| Maintenance | Five upgrade paths | One | One base upgrade path; adapters retrained independently |

## 5. Pilot proposal

**Pilot brand: Kestrel Athletics** (Athletic apparel and footwear; about 3100 Tier-1 tickets per day; Salesforce Service Cloud (API-first)). Readiness: Closest profile to the flagship, API-first tooling, engaged brand leadership.

Eight weeks: weeks 1–2 data export, ticket taxonomy and help-desk integration; weeks 3–4 brand adapter training, brand index build and golden-set sign-off by Kestrel Athletics's CX lead; weeks 5–6 shadow mode alongside agents; weeks 7–8 ramp from 25% to 100% of Tier-1.

Success at week 8 means all of: automation ≥ 60%; CSAT within 1 points of the brand's baseline; grounding ≥ 93%; P95 ≤ 1800 ms; availability ≥ 99.9%; escalation ≤ 15%. Evidence for a broader rollout: the pilot hits those thresholds using the shared-base-plus-adapter pattern with no flagship regression, and the integration playbook is written down and was followed.

## 6. Budget and timeline

| Phase | Window | Scope | Inference run-rate (order of magnitude) |
|---|---|---|---|
| Flagship steady state | now | Polaris | $1,382 per month at the current daily rate |
| Pilot | Weeks 1-8 (pilot) | Kestrel Athletics | Roughly two-thirds of flagship volume |
| Wave 2 | Q1 after holiday peak | Little Compass | Sized after the pilot; seasonal peak modelled explicitly |
| Wave 3 | Q1-Q2, after safety-policy review | Lumen Beauty, Meridian Home | Largest catalog last, after the retrieval pattern is proven at scale |

Framing: inference is not the cost that matters. At $0.0124 per automated ticket the flagship's inference spend is a rounding error against agent handling cost. The investment is integration and evaluation effort per brand, which the adapter pattern and a shared platform owner keep linear rather than multiplicative.

## 7. Executive talking points

1. The flagship crossed 70% Tier-1 automation in its final week and finished at 70.8%; quality and latency improved every week while doing it.
2. Handle-time and CSAT gains are real within the month and match the stated baselines; we are asking Northstar to confirm those baselines so the numbers are yours, not ours.
3. Availability averaged 99.85% and sat below the proposed 99.9% SLO on 27 of 31 days, with two same-day-mitigated incidents; we propose to formalise that SLO and report against it monthly.
4. Spend is 60.7% of budget, but usage is growing faster than the tickets it resolves; attributing that growth is our first joint action.
5. Expand through one pilot (Kestrel Athletics) on a shared-base-plus-adapter architecture, then sequence the remaining brands by readiness.

## 8. Biggest risk and mitigation

**The biggest risk is organisational, not technical: four brands with four help-desk stacks and four operating teams.** Without a shared platform owner on the Northstar side and a shared evaluation harness, the rollout fragments into four bespoke bots that each need their own care and feeding, and the flagship's economics do not carry over.

Mitigation: name a Northstar CX platform owner before the pilot starts; adopt the shared-base-plus-adapter architecture so brand differences live in adapters, prompts and indexes rather than in separate systems; require every brand to pass the same golden-set gate before ramp; and use the Kestrel Athletics pilot to write the playbook the other three brands will follow.

---
*Assumptions used: pre-launch AHT 11.3 min and CSAT 72.2 (Northstar CX ops to confirm); 99.9% availability SLO (proposed); $2,000 monthly budget line (Northstar finance to confirm). Brand profiles are illustrative pending Northstar's brand briefs.*
