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
- **Workstream 2 — Self-evolution, scoped to a critique-refine loop (not ART/DGM).** Judge → critique
  → refine → re-run over the existing Tier 1/Tier 2 evaluators, in the Self-Refine/Reflexion
  lineage — explicitly **not** claimed as novel (see Approach notes). Phase 3 (DGM-style
  self-evolution) is **dropped from this arc's scope**, not deferred — no training/weights touched.
  Instrumented with W&B Weave (`@weave.op`) and run inside W&B Sandboxes for isolated execution with
  auto-correlated traces; OpenRouter is a fallback LLM provider (trivial `base_url` swap on the
  existing `openai.AsyncOpenAI` client) if the primary provider is rate-limited mid-event.

**Prize-track correction (2026-09-12, from the live event page — full detail in the working
folder's `../2026-09-12-coreweave-hacks-sf/findings.md`, not duplicated here):** there are 7 tracks,
not 2. Most-Production-Ready stays the primary target, but "Best Use of Weave" is likely already
cleared by A6b for free, and "Best Use of ARIA" and "Best Use of marimo" are each a small,
near-free add-on (A6d, A8 below) — worth doing since they're additive, not competing for the same
build hours. "Best Use of TypeSafe AI" is explicitly skipped — re-confirmed no public API exists.

**What's next, in order:** A1 → A1b → A1c → A2 → A3 → A4 → A5 → A6 → A6b → A6c → A6d → A7 → A8 (all
agent-runnable, no blockers) → **B1-B4 (one owner sitting)** → C1-C5. See the remaining-work table —
it is the single source of truth for what's open; don't let this status section drift from it.

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
- **CodeFactor does not honor Ruff's `# noqa` comments** — `tests/test_green_server_endpoints.py:231`
  already carries `# noqa: S104` yet still shows as an open CodeFactor finding. Don't mistake a
  still-open CodeFactor issue for the noqa not working; the two tools track independently.
- Cross-repo dependency: Workstream 1's "driven task" is a real `agentic-job-offer-to-application-kit`
  workflow. **Unverified as of this writing** whether that repo's Workflow-tool execution already
  emits A2A-compatible traces this repo's executor can ingest, or whether a bridge/adapter is
  needed — resolve this as the first step of A3, don't assume either way.
- **`~/.cache/ms-playwright` is a symlink into `/tmp/devcache/...`** in this devcontainer — `/tmp`
  doesn't survive a container restart, so the symlink can go dangling (contradictory `mkdir`
  errors: "File exists" for the symlink itself, "ENOENT" following it) even though the symlink in
  the persistent home dir looks fine. Fix per-session with
  `mkdir -p /tmp/devcache/home/vscode/.cache/ms-playwright` before any Patchright/Playwright
  browser install; this is a devcontainer-config issue (belongs in whatever repo defines this
  devcontainer, not something fixable from inside a single checkout) and will recur after every
  restart until fixed at that level.
- **Issue #14's own diagnosis explains A7 and the CI watch-out above**: `pytest.yaml` being
  `workflow_dispatch`-only is independently named there as "the main reason 110d of staleness
  accumulated without test breakage being caught," with a concrete fix already specified — add
  `pull_request: { branches: [main] }` to its trigger block. Worth folding that exact fix into A7
  rather than re-deriving it.

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
- **Reference blueprints (external, self-evolution lineage — context only, this arc doesn't implement
  DGM/Phase 3)**: [Self-Refine](https://arxiv.org/abs/2303.17651), [Reflexion](https://arxiv.org/abs/2303.11366)
  (the critique-refine loop's actual lineage — not novel, say so if asked); [DGM paper](https://arxiv.org/abs/2505.22954)
  (Zhang, Hu, Lu, Lange, Clune — corrected citation; `2410.04444` is "Gödel Agent" (Yin et al.), a
  different paper previously miscited here).
- **Sponsor tooling (verified first-party, not from memory)**: [W&B Weave](https://weave-docs.wandb.ai/)
  `@weave.op` decorator for OTel-based cross-process tracing; [W&B Sandboxes](https://docs.wandb.ai/guides/sandboxes/)
  for isolated critique-refine iteration execution with auto-correlated Weave traces.

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
- **CodeFactor findings (13 total, identical on `main` and this branch — the plan-doc commit is
  markdown-only, introduced nothing new)**: 8 Security (all Bandit B104/B108, same rule IDs Ruff's
  `S` ruleset catches — see A1's expanded done-when), 4 Maintainability, 1 Duplication. The
  `src/green/executor.py:311` unresolved-FIXME finding is **already tracked** in `docs/TODO.md`'s
  Pydantic-passthrough item — not duplicated as a new item here. The remaining 3 Maintainability +
  1 Duplication findings are folded in as A1b/A1c below.
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
  the branch after merge (local + remote) — except the dependabot branches noted above. Going
  forward, branch names for this arc's items reference the plan number for scan-ability, e.g.
  `feat/0001-tier4-drift-detector` for A4, not just `feat/tier4-drift-detector`.

## Remaining-work table

| # | Item | Gate | Done-when |
|---|---|---|---|
| A1 | Enable Ruff `S` (bandit-equivalent) rules — closes #15 — before any new module lands | agent | `make validate` runs with `S` rules active; the 8 CodeFactor-tracked B104/B108 instances triaged with this exact comment template on `src/green/settings.py:41`, `src/purple/settings.py:37`, and the 4 matching test-file B104 assertions: `# noqa: S104 — binds 0.0.0.0 intentionally for the documented "docker run -p 9009:9009" quickstart`; the 2 B108 `/tmp`-path findings (`tests/test_green_settings.py:112,114`) resolved by changing `GREEN_OUTPUT_FILE`'s default from `/tmp/results.json` to `./output/results.json` (relative, matches the existing `output/` dir already in the repo) rather than noqa'd — no operational reason (unlike the 0.0.0.0 case) to keep a predictable `/tmp` path |
| A1b | Fix 2 ShellCheck bugs found via CodeFactor: `ralph/scripts/setup_project.sh:93` and `:107` self-assign (`PROJECT="$PROJECT"`, `DESCRIPTION="$DESCRIPTION"` — investigate the intended source variable); `ralph/scripts/ralph.sh:85` missing semicolon before `done` (SC1010) | agent | Both fixes verified against ShellCheck; `make validate` still green |
| A1c | Remove the 127-line duplication between `ralph/scripts/init.sh:8-188` and `ralph/scripts/lib/init.sh:8-188` (CodeFactor Critical) — extract shared logic | agent | Duplication resolved; scope stays limited to this duplication, not a broader Ralph-loop refactor |
| A2 | Resolve the cross-repo trace-format question: does `agentic-job-offer-to-application-kit`'s Workflow-tool execution already emit A2A-compatible traces, or is a bridge needed | agent | Answer recorded in this plan (append a note); bridge design sketched if needed |
| A3 | Design drift signals (claimed-done-without-evidence, graph-centrality anomaly, LLM-judge flag) as a Tier 4 `BaseEvaluator` subclass | agent | Design note added to `docs/architecture.md`-equivalent (this repo has none yet — add `docs/architecture.md`) |
| A4 | TDD the Tier 4 drift-detector against synthetic traces (extend `_build_traces_from_pattern()`) | agent | `make validate` green; new tier tested in isolation, RED commit precedes GREEN commit |
| A5 | TDD the Astra steering-adapter (mock the Responses API/WebSocket calls in tests — no real spend) | agent | Unit tests pass fully mocked |
| A6 | TDD the critique-refine loop (judge → critique → refine → re-run, wrapping existing Tier1 `GraphEvaluator`/Tier2 `llm_evaluate()` — no training, no new model) | agent | Unit tests pass; loop demonstrably improves a synthetic before/after score without touching weights |
| A6b | Instrument `GraphEvaluator.evaluate()`, `llm_evaluate()`, and `Executor.evaluate_all()` with W&B Weave (`@weave.op`); run critique-refine iterations inside W&B Sandboxes so traces auto-correlate | agent | A recorded Weave trace shows a full critique-refine iteration running inside a Sandbox |
| A6c | Add OpenRouter as a fallback LLM provider alongside the existing `openai.AsyncOpenAI` client (base_url swap in `src/common/settings.py` / `llm_judge.py`'s `get_llm_client()`) | agent | Fallback provider selectable via settings; unit test covers the swap, no real spend |
| A6d | Log a classic `wandb.log()` Run alongside the `@weave.op` instrumentation from A6b (per-iteration before/after Tier1 graph metrics + Tier2 judge score) — ARIA reads classic Runs, not Weave traces, so A6b alone doesn't clear "Best Use of ARIA" | agent | A classic W&B Run exists with the critique-refine iteration metrics logged; ARIA can answer "what changed between iteration 1 and 3" against it |
| A7 | Investigate the 26-minute local `make test_all` runtime; also add `pull_request: { branches: [main] }` to `.github/workflows/pytest.yaml`'s trigger (issue #14's own fix for why staleness accumulates undetected) | agent | Root cause of runtime identified (fix or documented tradeoff); pytest.yaml runs automatically on PRs going forward |
| A8 | A marimo/molab notebook visualizing Tier-1 graph metrics or the Weave trace timeline (no GPU needed — pure visualization, no risk from the 12hr session cap) — clears "Best Use of marimo" using data A3/A4 already produce | agent | Notebook runs in molab against real A3/A4 output, no new engineering surface beyond the viz |
| B1 | Provide/confirm an OpenAI API key + spend cap for real Astra calls, plus W&B Weave/Sandboxes credentials + quota (Fable's de-risking checklist: verify creds, quota, and cold-start latency for real before the demo, not assumed) | owner | Keys + budgets set; a real (non-mocked) Weave trace + Sandboxes run completed once to confirm latency is demo-safe |
| B2 | Ask on-site whether CoreWeave Hacks' "Most Production-Ready" track judges the Sunday submission snapshot or live repo state at Fully Connected (2 weeks later) — changes how hard to lean on the post-submission-window advantage | owner | Answer recorded in this plan |
| B3 | Review the drift signals from A3 | owner | Sign-off, or redirected |
| B4 | Approve merging the Workstream 1 PR(s) | owner | Merge (squash, CI green, never agent auto-merge) |
| C1 | Wire real Astra calls with the approved key; run the oversight layer live against the resolved (A2) driven task end-to-end | agent | Live run produces a real drift-detection + steering-correction trace |
| C2 | Run the critique-refine loop (A6) end-to-end via Sandboxes (A6b), OpenRouter (A6c) as fallback; capture before/after coordination-quality scores | agent | Numbers recorded, not just claimed; a scripted Sandboxes-down fallback path exercised at least once (Fable's de-risking checklist) |
| C3 | Record the demo video (README's "Coming Soon" placeholder) covering Phase 1 baseline + the new oversight layer; pitch script explicitly names judges Xiangyi Li (BenchFlow) and Jinjing (Stably AI) — frame Tier1-4 as the same agent-self-verification problem they've built companies around, applied structurally instead of per-output | owner/agent | Video linked from README |
| C4 | Update README roadmap checkboxes, add the new `docs/GreenAgent-UserStory.md` section, check `docs/PRD.md` relevance, add a real CHANGELOG entry | agent | All four docs reflect shipped state |
| C5 | Dogfood the arc's own stated goal: run Phase A as an actual unattended session (e.g. via `cc-recursive-team-mode`'s solo/teams harness) and confirm it completes without intervention | agent | A real unattended run log/trace exists showing completion without manual intervention — evidence the "long-running e2e handsoff unattended sessions with minimal supervision" goal holds for this arc's own execution, not just for the oversight feature it built |

## Approach notes

- Both workstreams share Tier 4's evaluator scaffolding (A3/A4) as their common dependency — build
  that once, reuse for both, per this repo's own KISS/DRY rule.
- Workstream 2 (A6/A6b/A6c, C2) is a bounded critique-refine spike, not an open-ended research
  program and not the README's Phase 2 ART-training path — it wraps existing Tier1/Tier2 evaluators
  with judge→critique→refine→re-run, backed by real (2026-current) research: Self-Refine/Reflexion
  lineage, **not novel**, say so if asked. Phase 3 (DGM-style self-evolution) is **out of scope for
  this arc**, not deferred — no training/weights touched anywhere in this plan.
- The narrow, defensible differentiation claim (verified, not assumed): **interpretable post-hoc**
  NetworkX graph metrics (Tier 1) feeding **prompt/behavior** refinement (the critique-refine loop),
  not baked into training or model weights. Graph-structural signal for self-improvement is an
  active 2026 research cluster too (LLM-GNCF, SkillGraph) — don't claim the *idea* is unique, only
  this specific interpretable/post-hoc/no-training combination.
