# Routing discussion: prompt-derived features and provisional policy

Date: 2026-11-18
Status: Proposed experimental choices, not a validated production architecture.
Updates the coverage, label availability, and tolerance questions in routing-data-and-customer-policy.md.

## Confirmed facts
All approximately 10,000 prompts were evaluated against all 15–20 model/reasoning configurations. Quality tolerance means absolute percentage points. The user delegates the initial tolerance decision. Task-family labels must not be required; availability varies across customers. Repeated trials per cell remain unconfirmed.

## Provisional policy decision
Use an average held-out success loss of at most one percentage point against a named fixed baseline as the initial experimental target. Compare against the best fixed eligible configuration selected on validation data, under the same hard constraints. This is not the hindsight oracle and not a guarantee for each request. Where eligibility varies, define and freeze the baseline fallback mapping before final testing. Evaluate two percentage points as an optional customer policy, not the default.

For the first experiment, minimize cost subject to this quality target and a customer-supplied latency constraint. Also retain a latency-first policy for comparison because the user's original preference prioritized quality, latency, then cost; the cost-first experiment is motivated by subsequent requests to select cheaper models. Customer configuration chooses between these policies; do not silently equate them.

Per-request predicted-quality tolerance is a tunable selection mechanism, not necessarily numerically equal to the aggregate realized-quality allowance. Tune the mechanism on validation data and inspect the locked test set once. Estimate paired quality differences with prompt/source-cluster uncertainty; if evidence does not support the allowance, label the candidate unproven and retain the baseline. A future deployment needs monitoring and rollback because offline results do not guarantee future traffic performance.

## Feature candidates available before inference
Cheap text features: character/byte/token length (token estimates depend on tokenizer), line count, code fences and syntax cues, structured-data markers, URLs, language/script signals, and text structure. Heuristics are fallible signals, not reliable task labels or hard constraints.

Semantic representations: embeddings of the request and potentially relevant sections. Evaluate truncation, long-input coverage, embedding overhead, and similarity to training data. Similarity is a shift signal, not a proof of correctness.

Optional semantic judgments: executable-code generation, schema adherence, multi-step quantitative reasoning, cross-passage synthesis, ambiguity or missing information. Decisions is a candidate source of these features; incremental benefit, calibration, latency, and cost require experiments. Do not assume its confidence estimates downstream model success.

Request-envelope metadata where present: attached modality, actual tool schemas, requested response format, allowed providers, region policy, budget, deadline, and context size. These should come from trusted application configuration/structured request fields for hard eligibility checks. Prompt text cannot reliably establish customer policy or operational permissions. Missing fields must be supported explicitly; labels and metadata are optional enrichments unless required to enforce a hard constraint.

Exclude post-response features such as actual output length, judge comments, realized reasoning tokens, realized latency, and success labels from router input. They are training targets or evaluation evidence only.

## Initial experiment ladder
1. Fixed eligible baseline.
2. Cheap prompt features with learned success/cost/latency estimates.
3. Add prompt embeddings.
4. Add a fixed, model-independent set of Decisions features.
5. Test selective semantic calls only if their benefit can be predicted well enough to justify that extra policy.

Use the same held-out prompts and customer policies throughout. Include feature generation, routing, and inference in latency/cost accounting. Report quality loss with uncertainty, mean cost, latency percentiles, and regressions in discovered slices. No requirement for manually labeled task families. The full grid permits paired offline comparisons for evaluated actions, but single-trial noise and transfer to new customer traffic remain limitations.

## Onboarding implication
Stable request features can be reused across configurations, reducing the need for model-description prompts. A newly evaluated configuration adds outcome data and required provider/capability configuration. This does not prove zero-shot support for unseen models or establish that evaluation completes in one to two engineer-days.

## Next useful evidence
An anonymized schema and representative rows can establish available request fields, label semantics, latency definition, cost units, model/deployment identifiers, repeated-trial identifiers, and any post-inference leakage. No dataset inspection or experiments have occurred yet.
