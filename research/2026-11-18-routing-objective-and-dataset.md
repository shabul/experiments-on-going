# Routing discussion: objective and existing evaluation grid

Date: 2026-11-18
Status: Working hypotheses; no final architecture selected.
Supersedes the unanswered scope, data-availability, objective-priority, and provider-scope questions in the opening discussion. Other questions remain open.

## User-confirmed requirements
- Route individual requests.
- Existing task/prompt grid covers multiple model–reasoning-effort combinations and includes binary success/failure, latency, cost, and other features.
- Initially explore Pareto optimization, with quality prioritized over latency and cost.
- Providers include OpenAI, Azure, AWS Bedrock, self-hosted models, and potentially Google later.

## Implications and hypotheses
The evaluation grid can support supervised prediction of success, latency, and cost for a request/configuration pair. Dataset completeness, representativeness, repeated trials, and evaluation consistency remain unverified. It provides observed comparisons where alternatives were actually evaluated; one observation is not a true success probability.

Represent an action as provider/deployment + model version + supported reasoning configuration. Provider/deployment can affect latency, availability, costs, and constraints even when model names overlap. Use explicit not-applicable effort for models without that control; do not assume cross-model effort equivalence.

Keep measurable outcomes separate from request understanding. Decisions could supply stable semantic features, but its incremental benefit must be compared with simpler features or embeddings. This is a candidate experiment, not a selected solution or claim of proven effectiveness.

## Objective distinction
Pareto optimization removes options dominated in quality, latency, and cost; it does not select one remaining option. Strict lexicographic optimization maximizes predicted quality, then breaks ties by latency and cost. It may pay substantial latency/cost for negligible or uncertain quality differences.

An alternative for discussion is a quality tolerance: retain candidates within an agreed delta of the best eligible predicted quality, then optimize latency, then cost. Delta, uncertainty treatment, any absolute quality floor, and per-request versus workload-level constraints require user agreement. Delta=0 is the strict-quality conceptual baseline, subject to estimation uncertainty.

Use request-conditional estimates rather than globally pruning every configuration that looks dominated on aggregate. A configuration can be valuable on a subset of tasks. Include router overhead in measured system latency and cost. Never optimize using realized output features unavailable before inference.

## Candidate comparisons, not recommendations
1. Best fixed eligible configuration: tests whether routing adds value.
2. Simple task-family lookup or rules: interpretable low-overhead baseline.
3. Learned request/configuration outcome predictor: uses existing outcome grid, must generalize to held-out tasks.
4. Same predictor with Decisions-derived features: isolates incremental semantic value and overhead.
5. Direct Decisions choice: prompt-based comparison baseline.

Use held-out prompt families/templates and a temporal holdout if available; avoid near-duplicate leakage. Compare success rate, latency distribution, cost, routing overhead, and per-family regressions. A hindsight oracle using observed outcomes is only an optimistic offline reference, not a deployable policy or reliable stochastic upper bound from single trials.

## Focused next questions
1. Approximate number of prompts, task families, and configurations; missing cells; one or repeated runs per cell?
2. How are success labels generated, and which features exist before executing a candidate?
3. Does quality priority mean strict quality-first selection, or an explicitly allowed quality tolerance for latency/cost savings? Is tolerance per request or over traffic?

## Later questions
Traffic representativeness, router latency budget, provider adapters, self-hosted cost accounting, latency measurement conditions, fallback permissions, and onboarding boundaries remain open. Separate time to integrate a configuration from time and spend to establish performance confidence.

## Maintenance
This is a conceptual design note, not new API research. Revisit hypotheses after inspecting a dataset schema and the user's answers. Keep the opening API findings dated and revalidate them before implementation.
