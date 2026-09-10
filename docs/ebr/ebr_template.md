# Northstar Retail Group — Executive Business Review

**Prepared for:** VP Customer Experience, VP Engineering · **Prepared by:** Fireworks AI account team
**Deployment:** {{deployment}} · **Period:** {{period_start}} to {{period_end}} ({{days}} days)
**Source:** every figure below is computed from the shared 31-day telemetry; assumptions are marked.

## 1. Executive narrative

{{executive_narrative}}

## 2. Metric context

| Outcome | Figure | What the telemetry shows | Caveat |
|---|---|---|---|
| Tier-1 automation | {{automation_exit}}% at month end | {{automation_month_mean}}% average across the month; {{automation_l7}}% in the final week | The 70% figure is a run-rate, not a month average. It was crossed on {{automation_cross_date}}. |
| Average handle time | {{aht_last}} min | Fell {{aht_within_month_pct}}% within the month (from {{aht_first}} min) | The 35% reduction compares against an assumed pre-launch baseline of {{aht_baseline}} min ({{aht_baseline_owner}} to confirm). |
| CSAT | {{csat_last}} | Rose {{csat_within_month_delta}} points within the month | The 12-point gain compares against an assumed pre-launch baseline of {{csat_baseline}} ({{csat_baseline_owner}} to confirm). |
| Availability | {{availability_l7}}% (final week) | {{days_below_slo}} of {{days}} days sat below the proposed {{slo_availability}}% SLO; two incident days ({{incident_dates}}) | No SLO was agreed at launch; we propose one below. |
| Latency | P50 {{p50_l7}} ms · P95 {{p95_l7}} ms | Both improved roughly a quarter over the month | P95 spiked to {{p95_max}} ms during the catalog-sync incident. |
| Quality | Grounding {{grounding_l7}}% · Eval pass {{eval_l7}}% · Escalations {{escalation_l7}}% | All three improved every week; the 24 Aug index refresh is visible in grounding | Quality is measured by automated evaluation, not by human audit; a monthly human sample would close that gap. |
| Spend | ${{spend_month_total}} for the month | {{pct_of_budget}}% of the assumed ${{budget_line}} budget line; ${{cost_per_automated_ticket}} per automated ticket | Request volume grew {{requests_growth_pct}}% while automated tickets grew {{automated_growth_pct}}%; attribution is the first joint action. |

**Operational health score: {{health_score}} / 100 ({{health_band}}).** Weakest pillar: {{weakest_pillar}}. Methodology is published on the account-health dashboard.

## 3. Expansion recommendation

Expand, in sequence, not all at once. The flagship result is real but young: one month, one brand, one catalog. The conditions that produced it are a clean product catalog, API-first help-desk tooling and an engaged operating team. Two of the four remaining brands share those conditions today; two do not yet. The recommendation is a single pilot that proves a repeatable playbook, then a staged rollout ordered by readiness rather than by size.

Proposed order: {{sequence_list}}.

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

**Pilot brand: {{pilot_name}}** ({{pilot_category}}; about {{pilot_tickets}} Tier-1 tickets per day; {{pilot_tooling}}). Readiness: {{pilot_readiness}}.

Eight weeks: weeks 1–2 data export, ticket taxonomy and help-desk integration; weeks 3–4 brand adapter training, brand index build and golden-set sign-off by {{pilot_name}}'s CX lead; weeks 5–6 shadow mode alongside agents; weeks 7–8 ramp from 25% to 100% of Tier-1.

Success at week 8 means all of: automation ≥ {{pilot_automation_min}}%; CSAT within {{pilot_csat_delta}} points of the brand's baseline; grounding ≥ {{pilot_grounding_min}}%; P95 ≤ {{pilot_p95_max}} ms; availability ≥ {{pilot_availability_min}}%; escalation ≤ {{pilot_escalation_max}}%. Evidence for a broader rollout: the pilot hits those thresholds using the shared-base-plus-adapter pattern with no flagship regression, and the integration playbook is written down and was followed.

## 6. Budget and timeline

| Phase | Window | Scope | Inference run-rate (order of magnitude) |
|---|---|---|---|
| Flagship steady state | now | Polaris | ${{projected_30d_flat}} per month at the current daily rate |
| Pilot | {{pilot_window}} | {{pilot_name}} | Roughly two-thirds of flagship volume |
| Wave 2 | {{wave2_window}} | {{wave2_name}} | Sized after the pilot; seasonal peak modelled explicitly |
| Wave 3 | {{wave3_window}} | {{wave3_name}}, {{wave4_name}} | Largest catalog last, after the retrieval pattern is proven at scale |

Framing: inference is not the cost that matters. At ${{cost_per_automated_ticket}} per automated ticket the flagship's inference spend is a rounding error against agent handling cost. The investment is integration and evaluation effort per brand, which the adapter pattern and a shared platform owner keep linear rather than multiplicative.

## 7. Executive talking points

1. The flagship crossed 70% Tier-1 automation in its final week and finished at {{automation_exit}}%; quality and latency improved every week while doing it.
2. Handle-time and CSAT gains are real within the month and match the stated baselines; we are asking Northstar to confirm those baselines so the numbers are yours, not ours.
3. Availability averaged {{availability_month}}% and sat below the proposed {{slo_availability}}% SLO on {{days_below_slo}} of {{days}} days, with two same-day-mitigated incidents; we propose to formalise that SLO and report against it monthly.
4. Spend is {{pct_of_budget}}% of budget, but usage is growing faster than the tickets it resolves; attributing that growth is our first joint action.
5. Expand through one pilot ({{pilot_name}}) on a shared-base-plus-adapter architecture, then sequence the remaining brands by readiness.

## 8. Biggest risk and mitigation

**The biggest risk is organisational, not technical: four brands with four help-desk stacks and four operating teams.** Without a shared platform owner on the Northstar side and a shared evaluation harness, the rollout fragments into four bespoke bots that each need their own care and feeding, and the flagship's economics do not carry over.

Mitigation: name a Northstar CX platform owner before the pilot starts; adopt the shared-base-plus-adapter architecture so brand differences live in adapters, prompts and indexes rather than in separate systems; require every brand to pass the same golden-set gate before ramp; and use the {{pilot_name}} pilot to write the playbook the other three brands will follow.

---
*Assumptions used: pre-launch AHT {{aht_baseline}} min and CSAT {{csat_baseline}} (Northstar CX ops to confirm); {{slo_availability}}% availability SLO (proposed); ${{budget_line}} monthly budget line (Northstar finance to confirm). Brand profiles are illustrative pending Northstar's brand briefs.*
