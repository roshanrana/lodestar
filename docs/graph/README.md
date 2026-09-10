# Code graph

Lodestar carries a queryable code knowledge graph built with [graphify](https://pypi.org/project/graphifyy/)
(tree-sitter AST extraction over 37 languages, no LLM required for code). It answers "what is X?",
"how do A and B connect?", and "what depends on this?" with file:line citations in a few hundred
tokens — far cheaper than grepping the repo cold. The graph itself (`graphify-out/graph.json`,
`graph.html`, `manifest.json`, `cache/`) is derived and gitignored; only the human-readable
`graphify-out/GRAPH_REPORT.md` is committed. Rebuild it locally any time with `graphify update .`
(AST-only, offline, a few seconds).

## Build

```bash
graphify update .
```

Requires `graphifyy` (`uv tool install graphifyy`). Current build: 197 files, 1901 nodes, 3622
edges, 164 communities, ~7s on this machine. `.graphifyignore` excludes `.venv/`, `.worktrees/`,
`graphify-out/`, `docs/assets/`, `data/` (CSV/YAML fixtures), `.chrome-profile/`, and `uv.lock` so
the graph stays about the code, not generated or vendored content.

## Query

```bash
graphify query "<question>"       # scoped subgraph answering a task question
graphify explain "<concept>"      # focused view of one node's connections
graphify path "<A>" "<B>"         # shortest relation path between two nodes
graphify affected "<symbol>" --depth 2   # blast radius of changing a symbol
```

### Three real queries against this repo

**`graphify explain "AccountView"`** — the customer-facing view model, 15 connections:

```
Node: AccountView
  Source:    app/pages/customer_health.py L27
  Community: 12
  Degree:    15

Connections (15):
  --> Audience [uses] [INFERRED]
  --> Settings [uses] [INFERRED]
  --> HealthReport [uses] [INFERRED]
  <-- _render_kpi_row() [references] [EXTRACTED]
  <-- _load_customer_view() [references] [EXTRACTED]
  <-- _render_methodology() [references] [EXTRACTED]
  ... (9 more _render_* references)
```

**`graphify path "customer_health.py" "load()"`** — how the customer page reaches data loading:

```
warning: target match was ambiguous (top score 47002.1, runner-up 47002.1)
Shortest path (2 hops):
  customer_health.py --imports--> build_view() --calls--> load()
```

The one thing worth flagging: `load()` is ambiguous (multiple equally-scored candidates), so the
path picked one; confirm the target with `graphify explain "load"` first for anything
security-sensitive.

**`graphify affected "scrub" --depth 2`** — blast radius of changing the audience-scrubbing
function (the customer/internal data boundary):

```
Affected nodes for scrub()
- view.py [imports] lodestar/view.py:L1
- build_view() [calls] lodestar/view.py:L129
- test_audience.py [imports] tests/test_audience.py:L1
  (6 more test_fr009_*/test_nfr005_* scrub tests)
- customer_health.py, internal_qbr.py [imports_from]
- bench.py, mcp_server.py, ebr.py, commands/{claims,pipeline,risks,score}.py [imports_from]
- test_ebr.py, test_guard.py, test_leak.py, test_page_internal.py, test_view.py [imports_from]
```

Every consumer of the audience-scrubbing boundary in one call — the exact list a change to
`scrub()` should make you re-check.

## Hooks (not committed)

`graphify install --project --platform claude` installs a `.claude/skills/graphify/` skill (this
repo has it, and it is committed) plus, optionally, PreToolUse hooks in `.claude/settings.json`
that intercept Read/Grep so an agent is nudged toward the graph first. Those hooks are a local,
per-developer opt-in — they are not part of this repo's `.claude/settings.json` and are not
committed, since they'd affect every agent session in the repo, not just yours.

## Shipyard usage

Implementers query the graph (`graphify query`, `graphify explain`) before opening files for a
task pack. Verifiers run `graphify affected "<symbol>" --depth 2` on every symbol a diff touches;
anything the diff changes outside that list is a finding.
