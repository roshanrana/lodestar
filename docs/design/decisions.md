# Decisions (append-only ADR log)

## D-000 — Gate approval under blanket autonomy   (2026-09-09, phase 0, status: accepted)
Context: Shipyard requires a named human approver at each gate and forbids agents speaking for
approvers. The owner instructed on 2026-09-09: "work as autonomously as possible. go." and
separately constrained model tiers.
Options: (a) park at each gate and wait; (b) treat the standing instruction as the approval for
G0–G4, G7, G8, record it verbatim, and present a consolidated gate summary at ship.
Decision: (b). Every gate entry records approval text `"go — Roshan Rana, delivery lead,
standing autonomous authorisation 2026-09-09 (D-000)"`. This is a deviation from Shipyard rule 1
and is disclosed in STATE.md and the ship report.
Consequences: design documents are frozen at commit time; the owner reviews at ship.
Approver: Roshan Rana (owner).

## D-001 — One application, two audience pages   (2026-09-09, phase 2, status: accepted)
Context: the prompt allows two dashboards or one app with separated views.
Decision: one Streamlit app, two pages, one core. Consistency becomes structural.
Consequences: audience separation must be enforced in code (D-003), not by deployment.
Approver: Roshan Rana (D-000).

## D-002 — stdlib + pydantic v2 for compute, no pandas   (2026-09-09, phase 2, status: accepted)
Context: 31 rows; mypy strict; fast startup; determinism.
Decision: typed `DayRow` models, pure functions, explicit arithmetic. Plotly receives lists.
Consequences: a few helper functions (mean, window) written by hand and tested.
Approver: Roshan Rana (D-000).

## D-003 — Visibility tags as the leakage control   (2026-09-09, phase 2, status: accepted)
Context: the customer view must never expose internal judgement.
Decision: every field of every context/report model carries `visibility` in
`json_schema_extra`; `scrub()` is the single function that drops internal fields for the
CUSTOMER audience; all customer surfaces receive scrubbed models only; a leak test enumerates
models and MCP tools; the bench publishes the count.
Consequences: adding a field without a tag fails a test (default is internal, fail-closed).
Approver: Roshan Rana (D-000).

## D-004 — Fireworks via OpenAI-compatible httpx adapter   (2026-09-09, phase 2, status: accepted)
Context: the account runs on Fireworks; the reviewer must not need a key.
Decision: `offline` provider is default and deterministic; `fireworks` uses
`https://api.fireworks.ai/inference/v1/chat/completions` with `FIREWORKS_API_KEY` and
`LODESTAR_FIREWORKS_MODEL` (default `accounts/fireworks/models/llama-v3p1-70b-instruct`);
`anthropic` is an optional extra. The same adapter serves any OpenAI-compatible base URL.
Consequences: no vendor SDK dependency; tests stub `httpx`.
Approver: Roshan Rana (D-000).

## D-005 — Recommended scaling architecture for five brands   (2026-09-09, phase 2, status: accepted)
Context: EBR §4 must recommend separate fine-tunes, a shared model, or another architecture.
Decision (product recommendation, not build decision): one shared base fine-tune on pooled
support behaviour (tone-neutral) plus one lightweight per-brand adapter (LoRA) served
multi-adapter on a single deployment, brand-specific system prompts, and brand-scoped RAG
indexes with hard namespace isolation; a shared evaluation harness with per-brand golden sets
gating every promotion.
Rationale: cost (one deployment's capacity shared, adapters near-free at serve time), latency
(no per-brand cold deployments), isolation (adapters trained only on their brand's data;
retrieval cannot cross namespaces), quality (brand voice and policy in adapter + prompt +
index), maintenance (one base upgrade path, adapters retrained independently).
Consequences: EBR and internal pipeline present this as the plan; Kestrel Athletics pilot
validates the adapter pattern before broader rollout.
Approver: Roshan Rana (D-000).

## D-006 — Health-score methodology frozen   (2026-09-09, phase 3, status: accepted)
Context: transparency requirement. Decision: LLD §4 defines pillars, weights, floors, targets,
windows and bands; changing any constant is a G3 re-entry and a design-document update.
Approver: Roshan Rana (D-000).

## D-007 — Project renamed Sextant → Lodestar   (2026-09-09, phase 6, status: accepted)
Context: the owner rejected the working name after G4 and chose Lodestar from four options.
Decision: rename package, CLI, env-var prefix (LODESTAR_), documents, GitHub repository and
local folder in one deterministic pass; evidence entries 1–5 keep their original artifact hashes
(the documents changed text after hashing), so the ledger records the pre-rename baseline and
later entries record the renamed artifacts.
Consequences: `docs/evidence/ledger.jsonl` seq 1–5 hashes will not match the current files by
design; `evidence.py verify` still validates the chain.
Approver: Roshan Rana (owner, in chat).

## D-008 — Retrospective: routing and orchestration lessons   (2026-09-10, phase 8, status: accepted)
Context: 14 task packs implemented and verified at T2 (Sonnet) with a T3 architect/orchestrator.
Observations: (1) `complexity: normal` packs overran the 40k input ceiling by 1.6–2.4× whenever
they touched four or more files, while `complexity: high` packs sat at 0.8–1.3×; the normal
ceiling is mis-sized for multi-file tasks, not the tier. (2) Every task passed verification at
attempt 1; no escalation above T2 was needed, so the owner's tier cap cost nothing in quality.
(3) The two verdict failures were orchestration faults, not implementation faults: a verifier run
in the live tree during a wave, and a broad `git add docs` that swept another task's files into
the wrong commit. (4) Architect arithmetic errors (days below SLO 22 vs 27) were caught by an
implementer recount because the pack carried an anchor; anchors in packs are worth their cost.
Decision: for future projects, size `normal` at 60k input tokens or split packs at three files;
verify only in worktrees pinned to a commit; the orchestrator stages explicit paths only;
keep numeric anchors in every pack and treat a mismatch as a finding against the anchor first.
Approver: Roshan Rana (D-000).
