# Threat model — Lodestar

## Assets

| Asset | Classification | Why it matters |
|---|---|---|
| Internal commercial judgement (relationship scores, competitive notes, margin, expansion confidence, blockers) | internal-only | prompt forbids exposure to the customer; a leak destroys trust and the exercise |
| Telemetry CSV and derived figures | internal (synthetic) | integrity of every surface depends on it |
| LLM provider API keys | secret | financial and reputational |
| Repo integrity (evidence ledger, headline.json) | internal | portfolio claims are "measured, not claimed" |

## Actors

Legitimate: reviewer, owner, Northstar VPs (customer page / EBR), Fireworks team (internal
page), MCP host. Adversarial: a customer-audience consumer probing for internal fields; a
crafted CSV; a compromised or hallucinating LLM provider returning internal terms or
instructions; a dependency with a CVE; a prompt-injection string in `operational_note`.

## Trust boundaries

| ID | Boundary |
|---|---|
| B1 | Internal audience ↔ customer audience (same process, same data, different outputs) |
| B2 | Lodestar ↔ LLM provider (network egress of prompt content; ingress of untrusted text) |
| B3 | Filesystem inputs (CSV, YAML) ↔ core |
| B4 | MCP host / HTTP client ↔ tool and route handlers |

## STRIDE table

| Boundary | Threat | Scenario | L | I | Controls | Residual | Accepted by |
|---|---|---|---|---|---|---|---|
| B1 | I | Customer page or MCP tool renders a field tagged internal | M | H | C-01 visibility tags on every model field; C-02 single `scrub()`; C-03 parametrised leak test over all models and tools; C-04 bench KPI `leak_check` = 0; C-05 customer page imports only `AccountView.customer()` | low | owner |
| B1 | I | Prose (EBR, narrative) mentions internal concepts by paraphrase | M | H | C-06 deny-list guard (`confidence`, `margin`, `competitor`, `renewal`, `churn`, `sentiment`, `blocker`, internal figures) applied to rendered EBR and to provider output; test with seeded leaks | medium: paraphrase can evade | owner |
| B2 | I | Prompt sends internal data to provider | L | M | C-07 provider receives only the CUSTOMER view serialisation; unit test asserts prompt payload contains no internal field | low | owner |
| B2 | T/E | Provider returns instructions ("ignore previous…") or internal-looking text | M | M | C-06 guard; C-08 output used only as prose in one section, never executed or parsed as commands; rejection falls back to offline text | low | owner |
| B2 | I | API key logged or committed | L | H | C-09 keys read from env only; never logged; `.env` gitignored; C-10 secrets regex scan in `check` | low | owner |
| B3 | D/T | Oversized or malformed CSV | M | L | C-11 5 MB / 10 000-row caps before parse; per-cell validation with row number; closed error codes | low | owner |
| B3 | E | `operational_note` contains injection text | M | L | C-12 notes are displayed as plain text, never sent to a provider, never interpreted | low | owner |
| B4 | S/E | MCP tool called with `audience="internal"` by an unauthenticated host | M | M | C-13 documented: the MCP server and API are local developer tools; internal audience requires `LODESTAR_INTERNAL_ENABLED=1` (default 1 locally, 0 in `.env.example` for shared deployments); C-14 tool inputs validated by pydantic | medium, by design of a local tool | owner |
| all | R | Figures in README/EBR not reproducible | L | M | C-15 bench determinism check; C-16 evidence ledger hashes | low | owner |
| deps | T | Vulnerable dependency | M | M | C-17 `pip-audit` at ship; pinned lower bounds; lockfile committed | low | owner |

## High-risk components

`lodestar.audience` (B1), `lodestar.narrative` (B2, B1) → `risk: high` packs get a fresh-context
T2 verifier plus a security review against B1/B2. `lodestar.data` (B3) → `risk: medium`.

## AI components (product)

Model inventory: the product itself calls an LLM only for optional prose polish; default is
offline. Evaluation: leak guard tests with seeded violations; prompt-injection test with a
hostile provider stub. Guardrails: C-06, C-07, C-08. Monitoring: n/a (local tool).

## Agentic build pipeline boundary

Implementers receive task packs only; verifiers are fresh contexts; no production credentials
exist; the repo's `operational_note` strings and YAML content are data, never instructions.

## Open items

None blocking. Paraphrase leakage (B1 row 2) is accepted as residual for a local tool and is
disclosed in the design document.
