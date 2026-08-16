# The corpus flywheel — dispatch data as an RL asset

**Status:** Draft direction record from an owner conversation on
2026-08-15. No owner go is recorded. This record fixes vocabulary and
sequence only. It designs no schema, no store, and no extension
package, per ROADMAP rule 2. Nothing here is scheduled.
**Date:** 2026-08-15.
**Grounds:** [`agent-analytics.md`](./agent-analytics.md) §1 (the
corpus thesis), §8 (the closed loop), §12 (the judge layer);
[`PHILOSOPHY.md`](../PHILOSOPHY.md) rules 1, 2, and 5;
[ROADMAP.md](../../ROADMAP.md) Next items 1–4 and the Colonies entry
(warren-2fa8).

---

## 0. What this cuts

[`agent-analytics.md`](./agent-analytics.md) argues that the telemetry
corpus is the durable asset and the dispatch loop is the commodity.
This record extends that thesis one step: the corpus is also
**training data for a dispatch policy**, and warren is positioned to
collect it, learn from it, and export it. The record lays out a
six-step direction and names the risks. It decides nothing that needs
an owner go.

## 1. The thesis

Every warren run is an episode with a verifiable reward. The
transcript is the trajectory. The outcome — branch pushed, gates
passed, PR merged — is the reward. The judge verdict is a dense label
over the trajectory. Warren computes the join `trajectory × outcome ×
verdict`, and only a control plane can compute it. Current RL work on
agents wants exactly this shape: rubric-decomposed trajectories with
outcome-grounded rewards.

Three consumers follow from the join, in order of cost:

1. **A dispatch policy.** A learned model that decides, per issue,
   whether to dispatch, at what cost tier, with how many replicas,
   and with which agent. A contextual bandit is enough to start.
2. **A self-improvement loop.** Warren agents change the policy code,
   gated on counterfactual replay against the logged corpus.
3. **A trajectory export.** The corpus as a dataset for post-training
   work outside warren, in the genre of rubric-reward RL.

The one-line sequence: steps 1–2 below make the data clean, steps 3–4
make it big, steps 5–6 make it compound.

## 2. The dispatch decisions that matter

Frontier models converge on capability, so model identity is the
least valuable dimension to route on. It also decays fastest, because
model releases invalidate model comparisons within weeks. The
decisions that hold value are predictions about the **task**, not
about the model:

- **Dispatch or defer.** An issue predicted to fail must not burn a
  run. It must go to decomposition (`sd plan`) or to a human. This is
  the highest-return decision and needs no model comparison.
- **Cost tier.** Whether the issue needs a frontier model or a cheap
  model. At fleet volume this decision is the economics.
- **Replica count.** One frontier run, or several cheap runs with the
  best result kept.
- **Agent and prompt variant.** Which builtin, which prompt, and
  whether the run will need early human steering.

The durable learned asset is a difficulty model over issues and
codebase sections. Difficulty features describe the task, so they
survive model releases. A new model is a cold-start arm that a few
dozen runs re-estimate. A bandit also updates online — every reaped
run is an update — so there is no retrain cadence to schedule.

## 3. The six-step direction

**Step 1 — seam completion.** The `IssueTracker` cut,
`AgentRuntimeAdapter` phases 1–2, and the burrow excision. These are
already ROADMAP Next items 1–4. This direction adds a second payer,
because cheap model, harness, and tracker switching is what makes new
dispatch arms cheap to add. No new work enters the queue.

**Step 2 — record the dispatch context.** For every dispatch, log the
issue features, the chosen action (agent, model, cost cap, replica
count), the queue state, and later the policy's recommendation. The
rule that keeps the pipeline honest: **facts are features, verdicts
are labels.** Mechanical data — outcomes, costs, tool calls, gate
results, merge state — feeds the policy as input. Judge verdicts ride
along as labels for reward shaping and review ranking, never as
trusted input features. This step is additive and does not wait on
step 1.

**Step 3 — the fork fleet.** Run warren hard against three to five
forks of external OSS repos (candidates: an nf-core module repo,
CleanRL, a scverse repo). The fleet is the data engine. It
diversifies the corpus beyond one repo and one language, it
stress-tests the harness-agnostic claims on foreign stacks, and it
teaches bot etiquette on forks before any upstream PR. It is also the
payer for the fleet features warren lacks: parallel dispatch waves,
replica dispatch, and cross-project scheduling — the Colonies entry
(warren-2fa8) finally has a payer.

**Step 4 — the throughput layer.** At fleet volume the binding
constraint is human review, not compute. Two answers: a review queue
ranked by verdict (clean verdict, green gates, small diff first), and
more shepherd automation from the pr-fixer and healer builtins
(branch updates, conflict repair). Step 4 is what keeps step 3's
volume real, and it is the first consumer that proves the verdict
corpus pays rent.

**Step 5 — the router.** An extension that consumes step 2's log at
step 3's volume. Ship it in three stages: a descriptive report, then
shadow mode (log what it would have chosen), then live. Start with
the churn-proof decisions from §2 — dispatch-or-defer and cost tier.
Model identity comes last, if at all.

**Step 6 — close the loop.** Two ends. First, the self-improvement
loop: warren agents change the router policy, and the merge gate is
counterfactual replay — the candidate must beat the incumbent on the
logged corpus. The gate is deterministic and a bad candidate never
touches a live dispatch. Second, the export: `verdicts.jsonl` plus
the audit log plus the events stream as a self-hosted
post-training-grade dataset. The team's corpus never leaves their
deployment.

## 4. Core versus extension

The leaning, deliberately not decided here: everything past step 2 is
extension-tier, except the small dispatch-surface features that only
core can provide (replica dispatch, a defer hook, cross-project
scheduling). The router, the review queue, the replay evaluator, and
the export are readers of published surfaces, in the posture the
judge extension already proved. The decision point arrives when the
first of them becomes a seed.

## 5. Risks

- **Goodhart, twice.** §12.5 of the analytics record already forbids
  raw verdicts in agent context. The router reads verdicts, so the
  rule now guards two doors: verdicts must not reach the judged
  agents, and routed agents must not read the routing features they
  are scored on.
- **Replay needs logged decisions.** Counterfactual evaluation is
  only as good as the decision log. Step 6 is unreachable until
  step 2 has accumulated volume, which is why step 2 starts first.
- **Confounded comparisons.** Merge rate is confounded by issue
  difficulty. Every policy claim ships with denominators and
  confidence, per §9 of the analytics record.
- **OSS etiquette.** Bot PRs to foreign repos burn trust. Forks
  first, human review before anything goes upstream, and the durable
  positioning is warren as a maintainer's own tool, not a bot that
  visits.
- **Low volume.** A bandit on three runs a day learns nothing. The
  router (step 5) is worthless without the fleet (step 3). The
  ordering is the mitigation.

## 6. Open questions

1. What is the minimal feature set for the dispatch-context log, and
   does it live in core columns or in an extension store?
2. Where does replica dispatch sit — a dispatch-time parameter on
   `POST /runs`, or a coordinator like plan-runs?
3. Does cross-project scheduling become the Colonies noun, or policy
   on the event bus? (The warren-2fa8 spike question, now with a
   payer.)
4. Which forks make the first fleet, and at what run cadence and
   budget?
5. Does the trajectory export need a published wire-schema artifact
   beyond what `FRICTION.md` already specifies?
