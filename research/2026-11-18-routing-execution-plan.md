# Intelligent model routing: execution and experiment plan

Date: 2026-11-18
Status: implementation-ready research plan; architecture selection is gated on evidence.
This is the current plan. Earlier dated discussion notes remain history; where they differ, this plan governs the proposed experiments.
No dataset has been inspected, API experiment run, or router trained yet.

## 1. Objective and fixed decisions

Build a router for individual requests across approximately 15–20 model/reasoning configurations, using approximately 10,000 fully evaluated prompts. Input columns: prompt, task type, model used, reasoning effort, cost, latency, success. Accept supplied judge labels as ground truth for this iteration. Runtime must not require task type.

Separate eligibility, outcome prediction, and customer policy. Do not select the final predictor in advance. Evaluate joint model–effort actions; provider/deployment becomes part of action identity when available. Models without reasoning controls have an explicit not-applicable value. Do not equate effort levels across models.

Primary quality target: at most 1 percentage point average success loss against a frozen fixed-configuration baseline on comparable eligible traffic. Evaluate 0 and 2 pp policies too. This is an aggregate noninferiority objective, not a per-request guarantee.

Preserve the originally stated quality > latency > cost preference as the default profile. Also evaluate a cost-saving profile because cheaper routing is an explicit use case:
- latency_first: quality gate, then minimize measured latency, cost as tie-breaker.
- cost_first: quality gate plus latency limit, then minimize cost.
Policies and quality tolerance are configuration, not training labels or prose prompts.

Proposed engineering screening targets (not established customer SLAs): router p95 <=100 ms for local-only candidates; <=250 ms for API-assisted candidates. Measure over realistic prompt lengths, warm/cold conditions, and concurrency. An API candidate that misses this screen may still qualify for an explicitly relaxed customer policy; it does not qualify for the default low-latency profile. Customer SLA overrides these trial targets.

## 2. Working structure and contracts

Implement an offline harness first, using Python, tabular files, a conventional ML library, and versioned configuration. These are suggested implementation choices, not library-version claims. Pin package versions in the implementation environment. Avoid distributed infrastructure initially.

Proposed layout:
- configs/: dataset schema, splits, experiment definitions, customer policy, action catalog.
- routing/data/: import, validation, deduplication, split construction.
- routing/features/: cheap features, embeddings, Decisions feature extraction.
- routing/predictors/: fixed, neighbors, independent classifiers, pooled model.
- routing/policy/: eligibility, selection, fallback.
- routing/evaluation/: replay, confidence intervals, calibration, reporting.
- routing/providers/: adapters added after offline selection.
- tests/: leakage, schema, eligibility, policy, replay, timeout behavior.
- artifacts/<run_id>/: manifests, scores, reports, model artifacts; large/sensitive artifacts outside Git.
- docs/: design decisions, onboarding checklist, experiment report.

Logical interfaces:
RequestFeatures = derive(prompt, optional_trusted_request_fields)
Predictions = predictor.predict(features, eligible_action_ids)
Prediction row = action_id, success_probability, estimated_cost, estimated_latency, uncertainty_or_support
RouteDecision = policy.select(predictions, policy_config, runtime_health)

Training targets remain separate from input features. Task type can support offline slices but must not enter the main deployed feature pipeline. Customer constraints come from trusted configuration, not user-text inference.

Every artifact records dataset hash, split hash, code revision, feature version, predictor version, action catalog version, policy version, random seed, and measurement environment. Cached semantic features key on exact prompt hash plus feature instructions/model version. Cache hits in experiments must not hide the cost/latency of generating features for novel production prompts.

## Phase 0 — Data audit and benchmark contract

Estimated effort: 1–2 engineer-days once data is available.

Tasks:
1. Import supplied columns into typed records. Validate finite nonnegative latency/cost and binary success. Establish units and latency meaning (time to first token versus full completion).
2. Assign prompt_id and action_id. Preserve original prompt text; compute exact hashes and normalized duplicate indicators separately.
3. Verify the complete prompt-by-action grid, not merely equal row counts. Identify duplicate prompt/action rows; determine whether these are repeated trials or ingestion duplicates. Do not silently average or discard conflicts.
4. Derive an action catalog from observed model/effort pairs. Do not invent model versions or deployment metadata absent from the data.
5. Recover source dataset/item IDs, judge version, evaluation date, and provider/version metadata where possible. Missing provenance is recorded explicitly.
6. Check whether outcome records were generated with the stated effort settings or imported from another source. An effort value without corresponding execution evidence is not trustworthy training metadata.
7. Group exact/near-duplicate prompts and shared templates. Combine source IDs with text similarity when available.
8. Produce coverage, class balance per action, cost/latency distributions, duplicate report, and missing-provenance report.

Output: validated immutable dataset, action catalog, data dictionary, audit report.
Gate: no unexplained coverage gaps/duplicate semantics; units and success meaning documented. If deployment metadata is missing, proceed with benchmark experiments but label live cost/latency portability unverified.

## Phase 1 — Splits, baselines, and evaluation harness

Estimated effort: 2–3 engineer-days.

Split design:
- Approximately 60% training, 20% validation, 20% locked test by prompt/template groups. All action rows and repeated trials for a prompt stay together.
- Within training, use grouped cross-fitting for probability calibration where practical; refit base estimators on full training after generating out-of-fold predictions. Fit calibration without final-test data.
- Tune predictors and selection policies on validation only. Freeze the selected candidate/profile before opening final test.
- Additionally run leave-one-source-dataset-out experiments if provenance can be recovered. Otherwise report this transfer test as unavailable; do not substitute task labels as equivalent evidence.
- Retain task type only for evaluation slices. Never feed the column into the main routing predictor.
- Any learned feature transform, vocabulary, projection, or neighbor index fits on training only. Frozen pretrained embeddings may encode all partitions independently, but retrieval references remain training-only.

Baselines:
B0: strongest fixed action by validation success; deterministic latency/cost tie-breaking. For dynamic eligibility, freeze an ordered fallback list selected without test outcomes.
B1: cheapest fixed action meeting the validation quality/latency policy, if one exists.
B2: static randomized mix of actions optimized using validation mean outcomes under the same quality objective. This tests whether prompt understanding adds value beyond a global allocation. Evaluate the mixture in expectation using the complete grid; report operational randomized latency separately.
B3: task-type lookup using training outcomes as a diagnostic only, explicitly not deployable where type is absent.
Hindsight reference: per-prompt best observed action and cheapest observed successful action. These are optimistic descriptive references, never deployable baselines or evidence of obtainable stochastic success rates.

Replay:
For each held-out prompt, select an action using only prompt-derived inputs and catalog/policy information, then join that action to its held-out observed success/cost/latency. Record routing overhead separately. If repeated trials exist, aggregate within prompt/action using a declared rule and retain uncertainty; do not treat repeats as independent prompts.

Metrics:
- Success rate, absolute quality difference in pp against B0, and failure rate.
- Mean cost/request, percentage savings, and mean cost per successful request (mean cost divided by success rate; not a retry-cost estimate).
- Completion latency p50/p95/p99, router latency p50/p95/p99, time-to-first-token only if available.
- Mean total cost including router features; total latency measured per request including routing. Do not add p95 values as if their sum were end-to-end p95.
- Brier score, log loss with numerical clipping, reliability plots; report per action and selected actions. Calibration alone is not routing success.
- Selection distribution, fallback rate, support/shift indicators, per-task/source/length/language slice regressions.
- Training time, inference memory, onboarding human time, evaluation spend.

Quality gate:
For each prompt define loss_i = success_baseline_i - success_router_i. Estimate mean loss and a one-sided 95% upper confidence bound using paired resampling at prompt/template cluster level (e.g. 5,000 bootstrap replicates). Pass the default quality gate only if the upper bound <=0.01 on the final holdout. If sample size is insufficient, mark inconclusive; do not relax the interval to claim a win.
With 2,000 independent test prompts, a 1 pp loss corresponds to 20 net additional failures. Precision depends on discordant outcomes and clustering; 10,000 prompts does not guarantee power. Use validation discordance to estimate additional evaluation needs before final testing. Repeated test-set tuning requires a fresh holdout, not repeated reuse.

Output: reproducible baseline report, replay harness, locked split manifest.
Gate: sanity checks reproduce fixed-action metrics; leakage tests pass; baseline and confidence method frozen.

## Phase 2 — Low-complexity empirical routing

Estimated effort: 3–4 engineer-days.

Feature sets:
F0: log character/token estimate length, line count, code-block indicators, punctuation/digit fractions, structured-data cues, language/script signals. Treat heuristics as fallible features.
F1: F0 plus one frozen prompt embedding. Choose one economical embedding setup initially; record provider/model/dimension and truncation. For long prompts, compare bounded head/tail or chunk aggregation on validation; no silent truncation.
Runtime token features must not depend on realized output/reasoning tokens.

Experiments:
E1: independent regularized logistic classifiers per action for success using F0. Use simple regressors for log cost and log latency; assess bias after inverse transformation and calibrate against observed means. A constant target uses an explicit constant estimator.
E2: same with F1. Regularization sweep kept small and declared (e.g. three values); select using routing validation metrics as well as calibration.
E3: embedding nearest-neighbor outcome aggregation. Start with exact cosine search over the training set; test k=16,64,128. Estimate each action using similarity-weighted observed successes with shrinkage toward its training prior:
q_hat(a|x) = (sum_j w_j*y_ja + lambda*p_a)/(sum_j w_j + lambda).
Use nonnegative weights and validation-selected smoothing; report neighborhood support. Estimate cost/latency from the same training neighbors. No held-out outcome enters the index.
E4: pooled predictor with request features and action identity/interactions; compare to independent predictors. Do not assume an unseen action ID provides zero-shot generalization.
Optional E5 only if needed: pairwise preference model or outcome-ranking objective to test whether relative ranking helps; defer neural architectures until these baselines are understood.

Policy mechanism:
After hard eligibility, retain actions with predicted success >= best eligible predicted success - tau. Sweep tau over 0,0.005,0.01,0.02,0.03,0.05 on validation and apply the policy's latency/cost preference. Tau is a tunable decision parameter, NOT the aggregate allowed quality loss delta. Customer policy acceptance is judged by held-out outcomes.
For a latency constraint, distinguish an expected-latency filter from a tail-SLA guarantee. Tail constraints need validated quantile estimates or measured endpoint limits; mean latency predictions cannot certify p95 compliance.
If every action is ineligible, return explicit no-feasible-route; never quietly violate policy.

Output: common-format results and Pareto plots for E1–E4 under both policy profiles.
Gate: nominate the simplest candidate that meets validation quality bounds and improves the primary efficiency metric. Proposed materiality threshold: >=10% reduction in total mean cost for cost_first or measured p95 completion latency for latency_first; retain B0 if none qualifies. This threshold is an engineering proposal, not a customer promise.

## Phase 3 — Decisions capability audit and incremental-value experiment

Estimated effort: 2–3 engineer-days plus API collection time/spend.

Documented foundation: Decisions exposes predicate/choice/score questions over shared inputs. The official guide identifies gpt-6-luna and permits independent questions in a single request. This is a typed judgment API, not documented automatic learning of the routing grid.
Source: https://developers.openai.com/api/docs/guides/decisions

Before coding against it, verify current SDK/schema, input and question limits, supported modalities, pricing/cache accounting, account rate limits, timeout/retry behavior, data controls, version identification, and availability in intended regions. Record facts with source/date; unknown latency remains measured, not assumed from marketing speed comparisons. No requirement to use Decisions remains if it fails this audit or experiment.

Use 6–10 fixed model-independent questions; start with:
- Requires producing or modifying executable code.
- Requires strict machine-readable output.
- Requires multi-step numerical/logical reasoning.
- Requires combining evidence from separated passages.
- Requires extracting explicitly present fields.
- Requires selecting tools or constructing arguments.
- Important information appears missing or ambiguous.
Each question has an operational definition and examples chosen from training only. Return probabilities rather than forced task labels. Do not send outcome labels, model descriptions, task type, or selected model as evidence.

Run a 300–500 prompt training-only pilot to check parsing, length limits, feature variability, failure rates, latency and cost. Establish a spend cap from measured input usage before scaling. Only then collect features for the full experiment; test prompts use frozen question definitions.
- D1: add these features to the best Phase 2 predictor.
- D2: optional direct-choice baseline, with concise eligibility-safe action descriptions derived from training data only. It is a comparison baseline, not the preferred design.
- D3: optional selective Decisions calls on uncertain requests only after D1 shows value. Gate based on pre-Decision features; tune on validation and evaluate full-path overhead.

Evaluate D1 versus its exact unaugmented parent using identical splits, policies, embeddings, and seeds. Record uncached feature cost/latency for novel prompts, not just replay cache reads. Test timeout, malformed response, long input, and prompt instructions attempting to override feature questions. Treat input prompt as evidence, never as policy authority.

Gate: Decisions stays only if it preserves the quality gate and delivers material total-system benefit over the best local candidate under the relevant policy. A useful default promotion requirement is >=5% incremental primary-efficiency improvement, with uncertainty reported and no SLA breach; a quality gain at comparable efficiency can also justify it. Otherwise retain the local predictor. Values are proposed experimental thresholds.

## Phase 4 — Cold-start onboarding and robustness

Estimated effort: 2–3 engineer-days plus evaluation runtime.

Simulate a new configuration by excluding all its outcomes during initial router training. Reveal its outcomes on 100, 300, 1,000, then all training prompts; never reveal validation/test outcomes to training. Repeat for at least three configurations spanning performance/cost and model families. Also hold out an entire model family where possible, because a new effort setting is easier than a new architecture.

Measure: time and human steps to register, feature changes required, refit time, evaluation spend, quality gap to full-data onboarding, and recovered routing benefit. Sampling uses training-only diversity/uncertainty; do not cherry-pick with hidden held-out labels.

Robustness suite:
- Long prompts, missing optional metadata, new languages, unfamiliar source datasets.
- Action disabled, rate-limited, timed out, or stale catalog.
- Embedding/Decisions unavailable.
- Empty eligible set; incompatible tools/context/format.
- Simulated latency inflation/cost changes, explicitly labeled simulation.
- Feature version mismatch and stale model artifacts.
- New request distribution with poor training support.

Fallback: deterministic validated eligible baseline; all fallbacks reapply customer constraints. Retry only within deadline/budget. Never automatically replay a side-effecting tool operation; distinguish model-call failure from an executed action. Shift scores trigger conservative routing/monitoring, not claims of certainty.

Gate: no hard-constraint bypass in fault tests; missing feature services degrade safely; measured onboarding meets <=16 engineer-hours for an already-supported provider/configuration. New provider adapters and sufficient evaluation evidence are separate work items.

## Phase 5 — Architecture decision and minimal serving integration

Estimated effort: 3–5 engineer-days after experiment selection.

Freeze finalists on validation, then run the locked test once. Architecture decision record includes quality bounds, efficiency, runtime overhead, slices, robustness, onboarding work, and reasons rejected candidates lost. If no candidate passes, deploy no learned router yet; use the baseline and collect targeted evidence.

Deploy only the winning predictor behind:
request -> trusted eligibility -> feature extraction -> outcome predictor -> customer policy -> provider adapter -> telemetry.
This modular contract is fixed; the predictor and whether Decisions is used remain empirical choices.

Catalog: action ID, model/version, provider/deployment, supported effort, modalities, tool/format support, context/output limits, region eligibility, price version, operational status. Hard fields require explicit verified metadata, not learned guesses.
API output: selected action/configuration, policy version, predicted outcomes, support indicator, fallback/reason code. Avoid presenting predictions as guarantees.
Adapters: common invocation/result/error contract. Implement one actual endpoint first, then add OpenAI/Azure/Bedrock/self-hosted adapters as required; Google can follow. Test per-provider effort mapping and tool/schema differences. Do not assume common names imply equivalent controls.
Metrics: request/decision IDs, artifact versions, chosen action, feature latency/cost, provider latency/cost, failures, fallbacks, eventual judge success. Do not require raw prompt logging; use customer-approved retention and identifiers.

Gate: contract tests, reproducible artifact loading, timeout/budget enforcement, load test at measured target traffic and burst conditions, and tested rollback.

## Phase 6 — Customer validation and controlled rollout

Estimated effort: depends on customer traffic and quality-label volume, not just engineer-days.

Benchmark evidence is not customer evidence. Collect representative customer prompts with approved handling; evaluate baseline and candidate offline/shadow where permitted. Shadow selection alone provides no counterfactual success labels. To measure quality, run a representative paired evaluation or use an appropriately randomized customer trial.
Choose sample size using observed discordance and desired quality bound, not an arbitrary duration. Keep judge protocol consistent with the experiment.
Progress from shadow to small canary to broader traffic only after quality, latency, error, and cost gates hold. Predefine evaluation windows and statistical stopping rules; do not repeatedly peek at fixed-sample intervals to claim significance.
Rollback on hard-constraint violations, operational SLA breaches, or predefined quality regression evidence. Monitor and disable newly uncertain actions as required.

## Onboarding runbook (target: 1–2 engineer-days of hands-on work)

Existing supported provider:
Day 1: register verified capability metadata; run adapter conformance and small smoke set; launch automated representative evaluation; validate units/errors.
Day 2: ingest outcomes; update predictor/index; run regression suite and policy gates; review uncertainty report; release to shadow/canary or retain restricted status.
Engineer time <=16 hours is the target, not guaranteed elapsed evaluation time. If evidence is insufficient, keep the configuration non-default without expanding manual work. Evaluation runtime/spend is separately budgeted. No extensive prose capability profile should be required.
New provider: adapter/authentication/deployment/conformance work is estimated separately. Do not claim the same two-day guarantee before measuring it.

## Immediate backlog: first five working days

1. Create dataset contract, importer, action catalog, audit report.
2. Create grouped split manifest and leakage tests; lock test partition.
3. Implement fixed baselines and paired replay/metrics.
4. Implement cheap-feature classifier plus outcome estimates and policy sweep.
5. Add embeddings and neighbor aggregation; produce first validation report.
No API dependency or provider integration is needed for days 1–3. Full schedule is roughly 16–25 engineer-days for Phases 0–5, with phases partly overlap-capable; paid API collection, new provider work, and rollout observation add elapsed time. These are planning estimates.

## Required tests and artifacts

Tests: row coverage; split disjointness; no task-type/target leakage; no held-out retrieval; unit normalization; unsupported effort rejection; eligibility on every fallback; predicted-vs-realized metric separation; metric replay against hand-calculated fixtures; failure behavior; action-version reproducibility.
Reports: audit, baseline, experiment matrix, calibration, Pareto curves, robustness, onboarding study, final decision record.
Store Markdown and configuration/manifests in the model-routing branch. Keep sensitive prompts, credentials, and large matrices in appropriate storage with hashes/references. Maintain an index with status and superseded links. Revalidate documentation at architecture freeze and each provider/API change. A recurring housekeeping job is not created by this plan; cadence and ownership remain to be configured.

## Research context

LLMRouterBench reports that several methods fail to reliably beat a simple baseline under its unified evaluation. This motivates the comparison ladder, not a prediction of results on this derivative dataset.
Source: https://arxiv.org/abs/2601.07206
Reference implementation: https://github.com/ynulihao/LLMRouterBench

Unresolved but nonblocking for harness work: exact dataset contents, provenance recovery, repeated-trial semantics, latency/cost units, contemporary pricing, customer latency SLA and traffic rate, and existing provider adapter availability.

