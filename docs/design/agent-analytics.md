# Agent Analytics — the telemetry corpus as a pillar

**Status:** Exploration — **not a committed direction**. This record
names a thesis, an inventory, and a gap list. It commits warren to
nothing. No roadmap item, no seam, and no schema follows from it until
an owner go. Every mechanism sketched below is provisional and yields
to what the paying feature teaches. Modeled on
[`extensions.md`](./extensions.md), which is the house form for a
direction record rather than a locked contract.
**Date:** 2026-08-07, from a survey of HEAD plus PHILOSOPHY, ROADMAP,
`extensions.md`, and the audit-log `FRICTION.md`.
**Amended:** 2026-08-11 — the Forge campaign got its owner go
([`forge-contract.md`](./forge-contract.md), plan pl-d1c9), which is
the ordering precondition §11 names.
**Grounds:** [`PHILOSOPHY.md`](../PHILOSOPHY.md) operating rules 1, 2,
5, and 6; [`ROADMAP.md`](../../ROADMAP.md) Next items 1, 2, and 6;
[`tier1-observation-bus.md`](./tier1-observation-bus.md);
`extensions/audit-log/FRICTION.md`.

---

## 0. What this cuts

Warren ships an analytics layer today, and it is treated as a feature
among features. This record asks a different question: is the telemetry
that dispatch produces the durable asset, and is the dispatch loop the
commodity around it?

The record exists to fix the vocabulary and the gap inventory before
anyone writes a seed. It deliberately does **not** design a schema, a
store, or an extension package. ROADMAP rule 2 puts a schema in a seed
at implementation time or in a design record after a design lock. There
is no lock here, so §5.1 and §11 name *candidates*, not columns to
build.

## 1. The thesis

Warren is built and operated as an orchestration system: dispatch a
run, sandbox it, stream it, reap it, push a branch. The reframe: the
durable value of a control plane is not the dispatch loop. It is the
**telemetry corpus the dispatch loop produces**, and the analytics that
corpus supports.

Insights of the kind this direction targets:

- "Agents struggle with this section of the codebase."
- "This model fails in this specific way."
- "This tool call produces context that is 90% wasteful."
- "Runs steered by a human in the first 10 minutes merge at twice the
  rate of unsteered runs."

The compounding property: every run a team dispatches makes the corpus
more valuable. Orchestration is a commodity. A year of outcome-joined
run telemetry about *your* codebase is not.

## 2. Why warren's position is unique

Two structural advantages, both already true of the code.

**The vantage point.** Harness-level observability tools see the
transcript. Warren sees the transcript **and the real-world
consequence**: branch pushed, commits ahead, PR opened, PR merged
(plan-runs today), seed closed, CI checks, salvage, empty push with
dirty paths. The join `transcript × outcome` is the thing only a
control plane can compute. It turns "the agent called grep 40 times"
into "the agent called grep 40 times *and the PR did not merge*."

**The contract lock.** Every harness lands in one `NormalizedEvent` row
shape (`src/runtime/contract.ts:114`). Every runtime goes through
`RuntimeProvider`. Trackers and forges move behind `IssueTracker` and
`Forge`. Anything a team bolts on must speak warren's contracts, so
analytics computed against those contracts works whatever the harness,
model, tracker, or forge. The analytics layer inherits
stack-agnosticism from the seams instead of building N integrations.

A third advantage is specific to os-eco and is covered in §8: warren
detects the struggle, and mulch is already the delivery channel for the
remedy.

## 3. What exists today (this is not greenfield)

Warren already has a first-party analytics layer.

- **Endpoints.** `GET /analytics/runs` (totals, per-agent, per-model
  and per-provider buckets, failure-reason breakdown, duration
  percentiles, token daily series, top seeds by context),
  `GET /analytics/cost` (eight breakdowns), `GET /analytics/behavior`
  (command mining), `GET /metrics` (Prometheus). Handlers in
  `src/server/handlers/runs/analytics.ts`, aggregators in
  `src/runs/analytics/`.
- **Insight mining exists in embryo.** `src/runs/analytics/insights.ts`
  ships six severity-coded callouts: worst-success agent, most-retried
  command, model cost outlier, steering anomaly, and two more. Command
  mining computes a `stuckScore` from retry-after-failure. That is the
  "agents struggle with X" genre at v0 depth.
- **UI.** The RunAnalytics and CostAnalytics pages, with Recharts
  already a dependency.
- **Public-projection discipline.** `PUBLIC_RUN_ANALYTICS_FIELDS` and
  the redaction allowlists classify every aggregate field for spectator
  visibility. That pattern stays.

So the direction is not "add analytics". It is "promote analytics from
a feature to a pillar, and fix the data problems that cap its depth."

## 4. The flagship insights, mapped to data requirements

| Insight | Data needed | Status today |
|---|---|---|
| Agents struggle with this codebase section | File paths per tool call, joined to failure, retry, and steering, aggregated by directory | Paths sit inside `tool_use` payloads (Read, Edit, Write inputs) and nothing extracts them. Only shell commands are mined. File paths as structured data appear **only** on the failure path (`reap.empty_push.dirtyPaths`) |
| This model fails in this way | Model as a queryable dimension × failure taxonomy × behavioral failure classes | Model and provider are **not columns**. Both are read out of `rendered_agent_json` frontmatter at query time, unindexed, and both describe the *declared* model, not the model the run used (mid-run fallback is invisible). The failure taxonomy (`RUN_FAILURE_REASONS`, 13 values) is infrastructure-heavy — `oom_killed`, `burrow_unreachable` — with no behavioral vocabulary for wrong approach, gate flunk, or spin loop |
| This tool call is 90% wasteful context | Per-tool-call token and byte attribution, context-window accounting | Tokens are run-level totals, nullable, and only pi and claude-code report them (sapling: null). No per-turn attribution, no context-window size, no compaction events. A cheap proxy is available now: `tool_result` payload byte sizes correlated against run context tokens |
| Steering changes outcomes | Steering events × outcome | `run_inbox` holds the full steering text and delivery latency, `steer.sent` events exist, and a steering-anomaly insight already ships. The best-supported insight today, blocked mainly by outcome quality (§5.3) |
| This run pattern lands PRs | PR merged or closed, per run | **Only plan-run children track merge** (`pr_merged_at`). For standalone runs `pr_url` is written once and never revisited, and `successRate` carries exit-code semantics |

## 5. The gaps, grouped by layer

These are the load-bearing gaps, not the full inventory.

### 5.1 Capture — core records too little

- No `queued_at` or `created_at` on runs, so queue wait is
  unmeasurable, and run ids are random rather than time-sortable.
- No model or provider columns, and no signal for the model a run
  actually used.
- `events.origin` (agent versus warren authorship) is dropped at write
  time by design, so provenance is unrecoverable afterward.
- No diff stats (files changed, insertions, deletions). No per-run CI
  check persistence — `pr-checks.ts` polls and discards.
  `commitsAhead` lives in event payloads, not in a column.
- No tool-call table. Every behavioral query re-parses raw
  `payload_json`, capped at 20k rows
  (`DEFAULT_TOOL_EVENT_CAP`, `src/db/repos/events.ts:19`) with silent
  truncation.
- No retry lineage past `parent_run_id`: no chain depth, no root id, no
  attempts counter. The healer counts attempts by scanning event JSON.
- A retention hazard: `ON DELETE CASCADE` runs from projects to runs to
  events. Deleting a project destroys its telemetry history, and no
  rollup survives the cascade.

### 5.2 Normalization — semantics are per-harness and incomplete

The envelope is normalized. The meaning is not. Usage reading, terminal
detection, and tool-shape parsing are keyed per runtime across three
modules, and coverage is uneven.

- `USAGE_SHAPES` covers pi and claude-code only
  (`src/core/usage-shape.ts:150`). **Sapling runs produce null cost,
  null tokens, and null terminal detection.** Per-provider analytics is
  structurally blind to one of the three shipped harnesses.
- Command mining assumes the Anthropic tool-call shape
  (`payload.input.command`, `tool_use_id` joins). `/analytics/behavior`
  is therefore not harness-agnostic, and its response cannot separate
  "the harness emitted no commands" from "the commands did not parse."

This is the `AgentRuntimeAdapter` seam, ROADMAP Next item 2.
**Analytics is a second, larger payer for that seam.** Add a
`toolShape` (and a `fileShape` for path extraction) beside `usageShape`
in the adapter registry, and every insight in §4 becomes
harness-agnostic by construction.

### 5.3 Outcome — success means "exited 0"

The highest-value single fix. Generalize the plan-runs `PrMergeChecker`
out of `src/plan-runs/`, drive it from a `post_reap` bus subscriber
(whose payload already carries `outcome`, `branchPushed`,
`commitsAhead`, and `prUrl`), and record PR state and merged-at for
standalone runs. The `Forge` contract is the natural home for the
polling. This turns every percentage in the analytics layer from
exit-code semantics into landed-work semantics.

## 6. Architecture — the philosophy-compliant split

PHILOSOPHY's litmus test — a feature that observes or reacts to the run
lifecycle is an extension — sorts this into three layers.

1. **Capture fidelity is core.** Rule 5 says data formats are API.
   Adding a queue timestamp, model columns, persisted origin, diff
   stats, merge state, or a durable global lifecycle stream enriches
   the data format, which is kernel work. Core records facts. It does
   not interpret them.
2. **Normalization is the adapter registry.** Per-harness semantics —
   `usageShape`, `toolShape`, terminal detection, failure
   classification — live behind `AgentRuntimeAdapter`, already Next
   item 2. ROADMAP's refusal of per-harness UI surfaces states the
   rule: what a runtime's events *mean* belongs behind the adapter, not
   in a page (or a metric) per harness.
3. **Interpretation is an observer extension.** The analytics engine —
   deep aggregation, directory-level struggle maps, context-waste
   scoring, LLM-assisted transcript analysis, long-horizon trends —
   ships as a Tier-1 observer with its own store, its own release
   track, and the lifecycle stream as its input. Its own store also
   answers the retention cascade in §5.1, because the rollups outlive a
   deleted project. [`extensions.md`](./extensions.md) already names
   metrics exporters and spend dashboards as the observer kind.

One tension to resolve deliberately: the in-core `/analytics/*`
endpoints. Rule 2 (a first-party feature must be expressible as an
extension) suggests the deep engine goes out of core while a thin
descriptive layer stays in core as the batteries-included default — the
same posture as seeds-in-core against trackers-as-extensions. Where the
line sits is a decision, not an obvious call.

## 7. Roadmap convergence — analytics as the converging payer

**The roadmap is already building the analytics substrate without
naming analytics as the payer.** No new campaign has to jump the queue.
The direction rides three items already sequenced.

- **`AgentRuntimeAdapter` phase 1** (Next 2) gives harness-agnostic
  semantics. Analytics adds `toolShape` and `fileShape` to the
  adapter's contract while the contract is cut, which is far cheaper
  than a retrofit.
- **The Forge campaign** (Next 1, owner go 2026-08-11) gives the
  credential and REST consolidation that makes generalized PR-merge
  polling honest.
- **The flagship extension mechanism** (pl-116e, shipped) gives the
  friction report. `extensions/audit-log/FRICTION.md` already specifies
  what an analytics consumer needs: one durable, sequenced, warren-wide
  lifecycle stream with a cursor per consumer (warren-f566 wants the
  same stream for the UI, so analytics is payer #2), a published
  wire-schema artifact, scoped observer credentials, and actor
  attribution. This direction co-signs every one of those items.

Per rule 1, each capture improvement in §5.1 lands with the analytics
consumer that pays for it, never as a speculative batch.

## 8. The closed loop — analytics to mulch to better runs

The os-eco-specific differentiator. Detection on its own ("agents
struggle with `src/runs/reap/`") is a dashboard. The ecosystem already
owns the remedy channel.

- Analytics detects a struggle pattern with evidence: retry clusters,
  steering hot spots, failed-gate correlation by directory.
- The finding lands as a **mulch record** — a `failure` or a `guide` in
  the project's `.mulch/` — with evidence links.
- `ml prime --files <path>` injects it into the context of the next
  agent that touches those files.
- Analytics then **measures whether the injection moved the metric**,
  which closes the loop and grades its own advice.

The maturity ladder this implies: descriptive (dashboards, exists) →
diagnostic (insight mining, embryonic) → prescriptive (recommended
mulch records, model and agent routing hints) → closed-loop
(auto-recorded expertise, dispatch policy informed by priors). Each
rung is independently valuable, and nothing commits to the top rung.

All of it is self-hosted. The team's behavioral corpus never leaves
their deployment, which is a real difference from SaaS observability.

## 9. Risks and tensions

- **Rule 1 discipline.** "Analytics" is exactly the banner that invites
  speculative core sprawl. Antidote: every schema addition names the
  insight that pays for it.
- **The lossless-payload rule.** The runtime contract passes payloads
  through verbatim by design, so semantic flattening in core breaks it.
  Interpretation stays in adapters and extensions.
- **Measurement validity.** Merged-PR success rate is confounded by
  issue difficulty, and a naive league table ("model X beats model Y")
  misleads. Insights ship with denominators and confidence, and the
  taxonomy prefers *behavioral failure classes* over rankings.
- **Goodhart.** Once an agent's context includes analytics-derived
  guidance, agents are measured by metrics they can read. The stakes
  are low today. Name it before the closed-loop rungs.
- **Sensitivity.** Prompts, steering text, and transcripts are the
  corpus. The public-projection allowlist pattern must extend to
  anything new, and the extension's own surface inherits the audit
  log's auth gap — there is no scoped observer credential yet
  (FRICTION §4).
- **Competitive honesty.** Do not compete with trace viewers on trace
  viewing. The edge is fleet-level, outcome-joined, and cross-harness:
  the control-plane join, not prettier spans.

## 10. Open questions

1. Where is the core/extension line for the analytics engine? Does
   `/analytics/*` stay as the thin in-core default while the deep
   engine goes Tier-1, or does the engine subsume the endpoints?
2. Does the corpus need a derived star schema (runs × tool calls ×
   files × outcomes) in the extension's store, or is on-demand event
   scanning enough at team scale? The 20k-row cap says scanning already
   strains.
3. Behavioral failure taxonomy: mined heuristically (cheap, coarse) or
   LLM-judged over transcripts (expensive, rich)? Probably staged.
4. Context-waste scoring: is `tool_result` byte size against context
   tokens a good-enough v0 proxy, or does it need per-turn usage deltas
   that only some harnesses can emit?
5. Does the sidecar-table amendment (2026-07-29, `(project_id,
   issue_id)` runtime bookkeeping) cover per-run merge-state columns,
   or does outcome tracking want its own decision entry?
6. Naming and positioning: is this a feature of warren, or the second
   headline ("orchestrate and understand") in the README pitch?

## 11. Sequencing sketch

**Non-binding.** A sketch, not a plan, and not a queue. It exists to
show that the direction needs no new campaign. Nothing here is
scheduled.

One ordering decision is recorded (2026-08-07, owner): **the Forge
campaign completes before analytics work begins.** The rationale is
that the biggest gap is outcome truth, and merge polling built before
the `Forge` contract would be a second GitHub client the campaign then
has to consolidate. Waiting means the merge watcher is born a Forge
consumer instead of retrofitted into one. That campaign got its go on
2026-08-11 and runs as plan pl-d1c9.

**Phase 0 — riders on work already sequenced, no new campaign.**

- During the Forge campaign: nothing analytics-specific, but the
  contract review confirms `Forge` exposes PR state
  (`merged | open | closed_unmerged`) and merged-at, because the merge
  watcher becomes its first non-plan-run consumer.
- During `AgentRuntimeAdapter` phase 1: add `toolShape` and
  `fileShape` extractor slots to the adapter contract while it is cut,
  and fill the sapling `usageShape` hole. This is the one item worth
  doing early. Retrofitting harness-agnostic extraction later means
  rebuilding every behavioral insight.

**Phase 1 — capture fidelity, small core changes, after Forge.**

- Candidate columns: `queued_at` on runs; model and provider as real
  columns (declared now, actually-used when a harness can report it);
  `commits_ahead` plus basic diff stats at reap; persisted
  `events.origin`.
- The merge watcher: a `post_reap` bus subscriber driving Forge PR
  polling, writing PR state and merged-at onto runs. This single change
  flips every success metric from exit-code to landed-work semantics.

**Phase 2 — the derived layer.**

- A tool-calls rollup, extracted from event payloads through the
  adapter shapes, so behavioral queries stop re-parsing raw JSON under
  a 20k-row cap. It is also the survival answer to the project-delete
  cascade.

**Phase 3 — insight expansion, in core first.**

- Grow `src/runs/analytics/insights.ts` against the new data:
  per-directory difficulty, cost per merged PR, steering-outcome
  deltas, context-waste proxies, agent-config before-and-after deltas.
  Every insight ships with denominators and confidence. In-core is
  deliberate: it sidesteps the delivery-mechanism friction and proves
  the insights on our own fleet before any packaging work.

**Phase 4 — extension and regulation, later, each with its own payer.**

- Re-platform the deep engine as a Tier-1 observer once the global
  lifecycle stream exists (warren-f566 is payer #1, this is payer #2).
- Only then the decide and regulate rungs: recommendations first, then
  policy gates — the roadmap's named first payer for Tier-2 mutating
  hooks.

## Appendix — pointers into the code

Line numbers are as of 2026-08-11 and drift. Treat them as hints.

- Wire vocabulary: `src/core/wire.ts` (`RUN_FAILURE_REASONS` :219,
  states, trigger kinds, `KNOWN_RUNTIME_IDS` :344)
- Event envelope and usage: `src/core/event-envelope.ts`,
  `src/core/usage-shape.ts` (sapling absent :150)
- Analytics layer: `src/runs/analytics/` (`run-metrics.ts`,
  `command-mining.ts`, `insights.ts`, `run-metrics-token-series.ts`),
  `src/runs/cost-analytics.ts`,
  `src/server/handlers/runs/analytics.ts` (public allowlists :180-247)
- Lifecycle bus: `src/runs/lifecycle-bus.ts` (payloads :69-135),
  wiring `src/server/main/lifecycle-bus-wiring.ts`
- Plan-run merge polling to generalize: `src/plan-runs/pr-merge.ts`,
  `src/plan-runs/in-flight.ts`, `src/plan-runs/merge-gate.ts`
- Event reads and caps: `src/db/repos/events.ts`
  (`DEFAULT_TOOL_EVENT_CAP` :19)
- Extension friction spec: `extensions/audit-log/FRICTION.md`
- Related open issue: warren-f566, the global lifecycle stream for the
  UI
