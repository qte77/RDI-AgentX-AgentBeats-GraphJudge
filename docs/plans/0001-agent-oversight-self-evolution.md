# 0001 — Agent Oversight & Self-Evolution

## Status (read this first — onboarding for the next session)

**Nothing shipped yet. This is the inception commit of the arc.** Two independent workstreams,
both extending this repo's existing three-tier evaluator (Tier 1 Graph / Tier 2 LLM-Judge / Tier
3 reserved for text-similarity) rather than replacing anything:

- **Workstream 1 — Oversight & mid-turn steering.** A new Tier 4 evaluator that watches a
  long-running, tool-calling agent's trace for drift (claimed-done-without-evidence, abnormal
  graph-centrality shifts, an LLM-judge flag) and, on detection, pushes a correction into the
  *same in-progress task* via GPT-6 Astra's Responses API mid-turn-steering channel — not a
  block-and-regenerate model like Arize's guardrails, a continue-and-correct one.
- **Workstream 2 — Self-evolution (this repo's own README Phase 2/3, unblocked at small scale).**
  ART training on captured traces, then an exploratory spike toward a self-evolving GreenAgent
  (DGM-style). No CoreWeave-sponsored compute assumed — scoped to whatever compute is actually
  available (see B2 in the table below).

**What's next, in order:** A1 → A2 → A3 → A4 → A5 → A6 (all agent-runnable, no blockers) → **B1-B4
(one owner sitting)** → C1-C4. See the remaining-work table — it is the single source of truth for
what's open; don't let this status section drift from it.

**The loop:** RED (write the failing test modeling desired behavior) → GREEN (minimal
implementation) → `make validate` (ruff + pyright + complexipy + coverage) → commit by topic →
repeat. Only non-trivial modules get unit tests — the drift-detector logic, the steering-adapter's
protocol handling, the ART-harness's reward computation. Wiring, config, and doc updates don't get
tests; they're covered by `make validate` + e2e per this repo's own `docs/best-practices/`.

**Owner-gates (batch these into one sitting — see table for B1-B4):** an OpenAI API key + spend
cap (real Astra calls cost money now, no hackathon credits), the Phase 2 compute decision, sign-off
on the chosen drift signals, and the PR merge itself (never agent auto-merge).

**Commands:**

```bash
make quick_validate   # fast inner loop: ruff + type_check
make validate          # full gate: ruff + type_check + complexity + test_coverage — must pass before every commit
make test_all           # uv run pytest — takes ~27 min locally (see watch-out below)
```

**Watch-outs:**

- `make test_all` took **26:43** for 375 tests during pre-flight verification (2026-09-07) — budget
  for this; A5 investigates the cause but don't let it block A1-A4.
- `.github/workflows/pytest.yaml` is **`workflow_dispatch`-only** — it does not run automatically on
  push or PR. Don't trust a green PR checks list to mean tests ran; run `make validate` locally and
  say so in the PR body.
- Ruff's security ruleset (bandit-equivalent `S` rules) is **not enabled yet** — tracked as open
  issue **qte77/RDI-AgentBeats-MAS-GraphJudge#15**. Folded into A1 below since the user's explicit
  requirement for this arc is "lint and sec" from the first line of new code, not retrofitted later.
- The 6 open `dependabot/*` remote branches are **active pending dependency PRs, not stale** — do
  not delete them as part of any "clean up stale branches" pass; check per-PR CI/merge status
  first, per this session's git-safety practice of investigating before discarding anything.
- **Subagents delegated any item below must run in a git worktree**, not directly against this
  checkout — use `Agent({ isolation: "worktree", ... })` so parallel work can't collide.
- Cross-repo dependency: Workstream 1's "driven task" is a real `agentic-job-offer-to-application-kit`
  workflow. **Unverified as of this writing** whether that repo's Workflow-tool execution already
  emits A2A-compatible traces this repo's executor can ingest, or whether a bridge/adapter is
  needed — resolve this as the first step of A3, don't assume either way.

## Source map (so the next session doesn't re-explore)

- **Tier 1 (Graph)**: `src/green/evals/graph.py` — `GraphEvaluator.evaluate()` L125-218,
  `GraphMetricPlugin` ABC L17-23, 8 built-in plugins L26-114 (degree/betweenness/closeness/
  eigenvector centrality, pagerank, density, clustering, connected components), extension point
  `register_plugin()` L116-123.
- **Tier 2 (LLM-Judge)**: `src/green/evals/llm_judge.py` — `llm_evaluate()` L233-256 (calls
  `_call_llm` L214, falls back to `rule_based_evaluate()` L109 on failure/empty response),
  `build_prompt()` L29-93, `get_llm_client()` L96-106. **`openai.AsyncOpenAI` (L12) is already a
  dependency** — Workstream 1 extends this client usage rather than adding a new one.
- **Tier 2 (Latency)**: `src/green/evals/system.py` — `evaluate_latency()` L25-80.
- **Extension point for new tiers**: `src/green/evals/base.py` — `BaseEvaluator` ABC, `tier`
  property defaults to `3` (L34-36). **A new oversight/drift tier is Tier 4** (Tier 3 is earmarked
  for the text-similarity/PeerRead plugin per the original README table, not for this). Follow the
  same wrapper pattern as `_LLMJudgeEvaluator`/`_LatencyEvaluator` in `src/green/server.py` L108-129.
- **Orchestration**: `src/green/executor.py` — `Executor.evaluate_all()` L281-335 runs
  Tier1→Tier2(LLM+latency) sequentially, passing `tier1_graph` dict into the LLM judge for context
  (L322-328); trace collection is `_adaptive_execute()` L82-154 (idle+timeout+completion-signal
  hybrid) or `_fixed_execute()` L156-204.
- **Server entrypoint**: `src/green/server.py` — `create_app()` L195-345 (FastAPI, port 9009
  default), `_process_evaluation_request()` L132-192, `main()` L389-407.
- **Trace storage**: `src/green/trace_store.py` — thread-safe in-memory `TraceStore`
  (`add_traces`/`get_all_traces`/`get_traces_by_id`), no persistence.
- **Settings pattern**: `src/green/settings.py` (`GreenSettings`, prefix `GREEN_`, docstring
  L22-36) and `src/common/settings.py` (`LLMSettings`, prefix `AGENTBEATS_LLM_`, L12-27;
  `A2ASettings`, prefix `AGENTBEATS_A2A_`, L30-42). **Any new Astra/steering config follows this
  exact `pydantic-settings` + prefixed-env-var pattern** and lands in `src/common/settings.py` (or
  a sibling settings class) — `docs/TODO.md` L38-41 already tracks a "settings consolidation" item,
  don't fight it by inventing a new ad-hoc config path.
- **Purple agent** (baseline test generator, port 9010): `src/purple/{agent,executor,server,
  settings}.py` — mirrors Green's structure; not directly touched by either workstream, but the
  pattern to mirror if a new agent-like component is ever needed.
- **Synthetic-trace pattern-builder** (reuse point for drift-scenario test fixtures):
  `src/green/server.py`'s `_build_traces_from_pattern()` L49-105.
- **Tests**: flat `tests/*.py`, one file per component, naming `test_{green|purple}_<component>_
  <behavior>`; `tests/e2e/*.py` uses `ASGITransport` (no real server/network) — pattern documented
  in `docs/TODO.md` L15-21.
- **CI**: `.github/workflows/pytest.yaml` — `on: workflow_dispatch` only, confirmed zero automatic
  runs. No other workflow runs the suite automatically.
- **Reference blueprints (external)**: [WeightWatcher](https://github.com/calculatedcontent/weightwatcher)
  and [PerforatedAI](https://github.com/PerforatedAI/PerforatedAI) (Phase 2, per README roadmap);
  [DGM paper](https://arxiv.org/abs/2410.04444) (Phase 3).

## Docs & issues audit (per-milestone, not one-time — re-run this section at each ship)

- **CHANGELOG.md**: exists at root, Keep-a-Changelog format, `<!-- scriv-insert-here -->` marker
  present — but **no scriv Makefile target exists** in this repo's `Makefile` (unlike sibling repos).
  Confirm the actual fragment-insertion mechanism before the first real code lands; a docs-only plan
  file does not warrant an entry yet.
- **ADRs**: this repo has **no `docs/decisions/` directory** — no ADR convention to follow here,
  unlike `agentic-job-offer-to-application-kit`. Don't invent one for this arc alone.
- **README Roadmap** (`README.md:56-67`, quoted verbatim for reference):
  `✅ Phase 1: A2A + Graph + Basic eval (current)` / `🔜 Phase 2 (outlook): ART training...` /
  `🔮 Phase 3 (outlook): Self-evolving GreenAgent...` — update these checkboxes only as each phase
  actually ships (C4), not now.
- **`docs/GreenAgent-UserStory.md`**: scoped strictly to the Phase-1 problem statement; silent on
  oversight/self-evolution. Needs a **new section**, not an edit-in-place — part of C4.
- **`docs/PRD.md`**: not yet reviewed for relevance — check before C4, don't assume either way.
- **Open issues relevant to this arc**: **#15** "enable Ruff S (bandit-equivalent) rules" — folded
  into A1 (see table). **#14** "bump CVE-affected deps (11 alerts)" — related but out of scope for
  this arc; don't pull it in without a separate decision. #18 and #13 are unrelated.
- **Env vars / CLI switches**: any new Astra API key or steering-related setting is documented in
  the existing `src/green/settings.py` / `src/common/settings.py` docstring tables (see source map)
  — not a new standalone list in the README.

## Quality gates (every item below)

- **Strict TDD, RED first**: write the failing test modeling the desired behavior before writing
  the implementation. Applies to the drift-detector, the steering-adapter's protocol handling, and
  the ART-harness's reward computation.
- **Only non-trivial tests**: no tests for wiring, settings classes, or doc changes — those are
  covered by `make validate` (lint + types) and e2e, not unit tests.
- **Lint + sec always**: `make validate` on every commit; A1 closes the security-lint gap (#15)
  before any new module lands, so new code is covered by `S`-rules from its first commit, not
  retrofitted.
- **Git discipline**: one branch per topic (`feat/`, `fix/`, `test/`, `docs/`, `chore/`), commits
  scoped to that topic, squash-merge only once CI + `make validate` + tests are all green, delete
  the branch after merge (local + remote) — except the dependabot branches noted above.

## Remaining-work table

| # | Item | Gate | Done-when |
|---|---|---|---|
| A1 | Enable Ruff `S` (bandit-equivalent) rules — closes #15 — before any new module lands | agent | `make validate` runs with `S` rules active, zero new findings on existing code (or findings triaged) |
| A2 | Resolve the cross-repo trace-format question: does `agentic-job-offer-to-application-kit`'s Workflow-tool execution already emit A2A-compatible traces, or is a bridge needed | agent | Answer recorded in this plan (append a note); bridge design sketched if needed |
| A3 | Design drift signals (claimed-done-without-evidence, graph-centrality anomaly, LLM-judge flag) as a Tier 4 `BaseEvaluator` subclass | agent | Design note added to `docs/architecture.md`-equivalent (this repo has none yet — add `docs/architecture.md`) |
| A4 | TDD the Tier 4 drift-detector against synthetic traces (extend `_build_traces_from_pattern()`) | agent | `make validate` green; new tier tested in isolation, RED commit precedes GREEN commit |
| A5 | TDD the Astra steering-adapter (mock the Responses API/WebSocket calls in tests — no real spend) | agent | Unit tests pass fully mocked |
| A6 | TDD the ART-training harness skeleton (trace-loader, graph-score-as-reward, training-loop wrapper; mock the actual training call) | agent | Tests pass; no real GPU/compute touched |
| A7 | Investigate the 26-minute local `make test_all` runtime | agent | Root cause identified; fix or documented tradeoff |
| B1 | Provide/confirm an OpenAI API key + spend cap for real Astra calls | owner | Key + budget set |
| B2 | Decide the Phase 2 compute path — local GPU, a small paid cloud instance, or explicit defer | owner | Decision recorded in this plan |
| B3 | Review the drift signals from A3 | owner | Sign-off, or redirected |
| B4 | Approve merging the Workstream 1 PR(s) | owner | Merge (squash, CI green, never agent auto-merge) |
| C1 | Wire real Astra calls with the approved key; run the oversight layer live against the resolved (A2) driven task end-to-end | agent | Live run produces a real drift-detection + steering-correction trace |
| C2 | Run the small-scale ART experiment on whatever B2 decided; capture before/after coordination-quality scores | agent | Numbers recorded, not just claimed |
| C3 | Record the demo video (README's "Coming Soon" placeholder) covering Phase 1 baseline + the new oversight layer | owner/agent | Video linked from README |
| C4 | Update README roadmap checkboxes, add the new `docs/GreenAgent-UserStory.md` section, check `docs/PRD.md` relevance, add a real CHANGELOG entry | agent | All four docs reflect shipped state |

## Approach notes

- Both workstreams share Tier 4's evaluator scaffolding (A3/A4) as their common dependency — build
  that once, reuse for both, per this repo's own KISS/DRY rule.
- Workstream 2 (A6, C2) is explicitly the "outlook" tier the README already flags as time-boxed —
  keep it a bounded spike, not an open-ended research program; Phase 3 (DGM-style self-evolution)
  stays unscoped beyond a design note until Phase 2's actual results justify committing further.
