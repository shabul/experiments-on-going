# Routing schema and initial benchmark evidence

Date: 2026-11-18
Status: Research discussion; no dataset inspected or model trained.

## Confirmed schema and provenance
Columns: prompt, task type, model used, reasoning effort, cost, latency, success (0/1).
User says prompts originate from LLMRouterBench and similar datasets; no example currently available. Exact dataset versions, transformations, and whether measured outcomes were rerun or imported remain unverified. Earlier information establishes complete coverage across evaluated configurations.

## Column roles
- prompt: source of runtime request features.
- task type: offline slicing, diagnostics, and optional auxiliary supervision only; exclude from the primary runtime predictor input. Availability is not required from customers.
- model used and reasoning effort: identify candidate actions; they are not the best-action label. Every row contributes success/cost/latency evidence for that candidate.
- success: binary supervised target, accepting user-provided judge labels as ground truth for this exercise.
- cost and latency: observed regression targets and offline evaluation outcomes, never runtime input values for an unexecuted request.

Do not reduce each prompt to one winning model label: multiple candidates can succeed, and customer priorities differ. Estimate outcome vectors or candidate-specific probabilities and apply policy separately.

## Working candidate: empirical neighbor router
Embed each unique training prompt, retrieve similar training prompts for a new request, and aggregate their success/cost/latency per candidate with similarity weights and shrinkage toward candidate-wide statistics. This is a hypothesis, not a proven method. Tune neighbor count, smoothing, and calibration on validation data. Never retrieve held-out prompts or their labels when evaluating. Near-duplicate/template splits are needed.

Advantages to test: directly uses complete outcome vectors, no model capability prose, can add new model outcomes for stored prompts without changing feature prompts. Weaknesses: semantic proximity may not imply shared failure modes, sparse or shifted neighborhoods can be misleading, stored outcome coverage remains necessary, and latency/cost can change with deployment. Similarity alone is not calibrated confidence. Compare with supervised outcome prediction and Decisions-feature augmentation rather than selecting it now.

## Initial literature evidence
LLMRouterBench authors report model complementarity but find several routing methods fail to reliably beat a simple baseline under their unified evaluation. This supports testing simple baselines, not a claim that a particular simple router will win on the user's data.
Source read: https://arxiv.org/abs/2601.07206
Official repository inspected: https://github.com/ynulihao/LLMRouterBench
These sources do not establish how the user's derivative dataset was labeled, priced, timed, or expanded to reasoning settings.

## Provenance improvements for later
Preserve source_dataset, source_item_id, model_version, provider/deployment, evaluation timestamp, run identifier, judge/rubric version, latency definition/unit, and cost currency/pricing basis where recoverable. These are primarily offline audit metadata, not required customer inputs. Stable prompt IDs can be derived locally; cross-source deduplication should also inspect normalized text and near-duplicates. Do not claim irrecoverable fields can be reconstructed from prompt text.

## Evaluation priorities
Use grouped prompt/template splits plus source-dataset holdouts if source provenance can be recovered. Compare fixed eligible baseline, empirical neighbor method, supervised predictor, and Decisions augmentation using identical splits and policy settings. Report paired quality differences and uncertainty alongside total cost and latency. Accepting benchmark labels does not establish transfer to customer traffic. Actual provider latency and contemporary costs require deployment measurement or repricing data; the current schema alone cannot support reliable repricing without token/usage detail.

No further user examples are required to continue conceptual comparisons. Next discussion should compare the empirical neighbor method, supervised outcome prediction, and semantic-feature approaches against onboarding and operating constraints.
