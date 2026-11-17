# Routing discussion: dataset and customer policies

Date: 2026-11-18
Status: Working design hypotheses; no final architecture selected.
Updates: routing-objective-and-dataset.md questions on dataset size, labeling, and quality tolerance.

## User-confirmed information
- Approximately 10,000 prompts across 5–7 task families, including coding, summarization, extraction, and tool calling.
- Approximately 15–20 model–reasoning-effort configurations including Claude, GPT, and others.
- Each prompt has multiple rows, one per evaluated configuration. Full grid coverage is not confirmed.
- Binary success labels come from an LLM judge; treat them as ground truth for the current exercise.
- Cost and latency are recorded.
- A 1–2% reduction in predicted success may be acceptable for cheaper routing; final trade-offs depend on customer policy. Absolute percentage points versus relative percentage loss remains unconfirmed.

## Design hypotheses
Separate a shared outcome estimator from a customer policy selector. The estimator predicts success, cost, and latency for eligible actions. The selector applies hard constraints and the customer's quality tolerance and latency/cost priorities. Keep all actual selection policy in application logic, rather than asking a semantic model to interpret customer trade-offs anew on every request.

For a candidate request x, let A(x) be the eligible actions and q_hat(x,a) predicted success. A possible per-request policy retains actions with q_hat(x,a) >= max_b q_hat(x,b) - delta. This is an illustrative predicted-quality constraint, not a guarantee of realized quality. Absolute quality floors, no-feasible-action behavior, uncertainty handling, and aggregate traffic constraints remain separate policy decisions.

Do not infer that every customer prefers latency over cost: a customer may minimize cost under latency and quality bounds, or minimize latency under quality and budget bounds. Pareto filtering may precede either policy.

## Dataset implications
At full coverage, 10,000 prompts by 15–20 configurations would yield 150,000–200,000 rows, but independent request diversity remains approximately 10,000 prompts. Split by prompt and preferably source/template clusters before constructing model rows. Keep all configurations for each prompt in the same split. Preserve a locked final test set; fit probability calibration separately from final testing.

For this iteration, success means success according to the supplied judge. No judge replacement project is required. Record judge version/rubric for interpreting future drift. One observed binary result is not a measured success probability; predictive probabilities must be learned across requests and assessed on held-out data.

Do not blindly substitute failure for missing cells. Examine coverage and whether evaluation selection depends on task or configuration. Evaluate learned policies on comparable held-out outcomes where available, and explicitly identify unevaluated selections.

## Candidate experiments
Compare a fixed eligible configuration and task-family lookup with learned outcome predictors. Explore independent predictors per configuration versus a pooled request/action predictor; the latter may share evidence but does not automatically solve cold-start onboarding. Compare request features alone against added Decisions-derived semantic features under identical splits and measurement conditions. These are experiment candidates, not validated recommendations.

Report paired held-out success differences against a named baseline, prompt-cluster uncertainty intervals, cost, latency distributions, routing overhead, and task-family regressions. Sweep customer quality tolerance to show observed trade-offs. A small predicted quality gap may be below the resolution supported by the holdout, especially within a family.

## Next questions
1. Does 1–2% mean absolute percentage points (e.g. 95% to 93–94%) or relative loss, and is the allowance per request or over traffic?
2. Is the grid nearly complete? Are observations repeated per prompt/configuration?
3. Which pre-inference features exist: prompt text, task family, token count, tool schema, language, modality, other metadata? Are family labels known at routing time or only benchmark annotations?

## Open work
No API experiment, predictor training, or dataset inspection has occurred. Onboarding budget, live router latency budget, performance-label versioning, and provider integration boundaries remain unresolved. Revisit prior working notes alongside this update; no final architecture decision has been made.
