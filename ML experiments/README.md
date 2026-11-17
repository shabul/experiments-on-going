# Predicting success probabilities for intelligent model routing

## Objective

Build a model that estimates the probability that a given **prompt, model, and reasoning-effort configuration** will succeed. Use those estimates to select a configuration that balances quality and cost.

This is **conditional success-probability prediction**, formulated as supervised binary classification:

$$
\hat P(Y=1\mid P,M,E)
$$

Here, $P$ is the prompt, $M$ is the model identity, $E$ is the reasoning-effort configuration, and $Y$ is the observed success label (1 for success, 0 for failure).

The predictor estimates quality. A separate routing policy decides how much predicted quality to trade for lower cost.

## 1. Training data and prediction targets

An illustrative dataset:

| Prompt | Model | Effort | Success | Cost | Latency |
| --- | --- | --- | --- | --- | --- |
| Write a Python sorting function | GPT-A | Low | 1 | $0.002 | 0.8 s |
| Write a Python sorting function | GPT-A | High | 1 | $0.009 | 2.1 s |
| Write a Python sorting function | Claude-B | Low | 0 | $0.003 | 1.0 s |
| Summarize this document | GPT-A | Low | 1 | $0.001 | 0.6 s |
| Summarize this document | Claude-B | Low | 1 | $0.002 | 0.9 s |

These names and numbers are examples, not measured model performance. Reasoning-effort settings must be supported by the chosen model; they need not have equivalent meanings across providers.

For a new prompt, the predictor might return:

| Model | Effort | Predicted success |
| --- | --- | --- |
| GPT-A | Low | 82% |
| GPT-A | High | 95% |
| Claude-B | Low | 89% |
| Claude-B | High | 96% |

If Claude-B High costs twice as much as GPT-A High, a routing policy might choose GPT-A High for its lower cost and one-percentage-point difference in predicted success.

## 2. Representing the prompt

The customer supplies the prompt. Task type, complexity, and other features can be derived internally:

1. Encode the prompt with a pretrained text embedding model.
2. Extract inexpensive metadata, such as token count, presence of code or mathematics, and structural constraints.
3. Combine those features with model identity and reasoning-effort configuration.
4. Estimate success probability for each eligible configuration.

Keep outcome information, observed response cost, and observed latency out of success-predictor inputs when they are unavailable before routing. Cost estimates used by the routing policy must also be available before the selected response is generated.

## 3. Candidate predictors

### A. Regularized logistic regression and IRT-style models

Start with an interpretable baseline. A simple capability–difficulty formulation is:

$$
P(\text{success}\mid p,m,e)=\sigma(a_{m,e}-d(p))
$$

Here, $a_{m,e}$ represents the capability of a model–effort configuration, $d(p)$ estimates prompt difficulty, and $\sigma$ is the sigmoid function.

For unseen prompts, difficulty must be estimated from prompt features rather than stored as a free parameter for each training prompt. A multidimensional extension can represent different task demands, such as coding, mathematics, and extraction, and learn corresponding model strengths.

This approach is related to **Item Response Theory (IRT)**, which models the probability of answering assessment questions correctly.

### B. Embeddings plus LightGBM

Use this as the primary nonlinear candidate:

1. Embed each prompt.
2. Optionally reduce embeddings to a compact representation, initially testing 32–128 dimensions.
3. Add prompt metadata, model identity, and effort configuration.
4. Train LightGBM on binary success labels.

This offers efficient training and nonlinear interactions without fine-tuning a large language model. Fit dimensionality reduction only on training data. Compare compressed embeddings with alternatives rather than assuming compression improves performance.

### C. Embeddings plus a small neural predictor

Train a small multilayer perceptron using:

$$
[\text{prompt embedding};\text{model embedding};\text{effort embedding}]
$$

A sigmoid output estimates success probability. The network learns interactions between prompt semantics and configuration capabilities.

Initially freeze the text encoder. With a small dataset, end-to-end fine-tuning may overfit. Learned model embeddings also require training evidence for each configuration; adding a new model needs a separate cold-start strategy or new evaluations.

## 4. Is 10,000 rows enough?

Count **unique prompts**, not just evaluation rows:

| Scenario | Unique prompts | Evaluations | Implication |
| --- | --- | --- | --- |
| A | 10,000 | Each prompt evaluated across multiple configurations | A more promising foundation for learning prompt semantics |
| B | 500 | 20 configurations per prompt, totaling 10,000 rows | Less prompt diversity and a higher risk of poor generalization |

Neither scenario guarantees sufficient data. Coverage of tasks and configurations, label quality, and the deployment distribution matter.

Missing evaluations are **unobserved outcomes**, not failures. If historical configuration choices were selective, the observed data may be biased toward particular prompt–configuration combinations. Use balanced evaluations on a representative prompt subset where feasible.

A single pass/fail result is one observation, not a measured probability. Repeat selected prompt–configuration evaluations to estimate stochastic variability and judge consistency.

## 5. Evaluation and probability calibration

A calibrated predictor assigning 80% success to many comparable unseen examples should see roughly 80% of those examples succeed over time.

Evaluate more than classification accuracy:

| Measure | Question |
| --- | --- |
| Brier score and log loss | How accurate are the predicted probabilities? |
| Reliability diagrams and calibration error | Do observed success rates match predicted probabilities? |
| Within-prompt ranking | Does the predictor distinguish stronger and weaker eligible configurations? |
| Realized routing success and cost | Does routing reduce cost while preserving the required success rate? |
| Latency | What response-time tradeoffs does the selected policy create? |

Split by **unique prompt**, keeping all configuration evaluations for a prompt in the same partition. Group near-duplicate prompts or shared templates when needed to prevent leakage. Consider a time-based holdout if production data changes over time.

Use separate training, validation/calibration, and final test partitions. Fit preprocessing on training data; tune predictors, calibration, and routing thresholds without using the final test set. Report uncertainty, with resampling grouped by prompt where appropriate.

Verify that binary labels consistently represent task success. For subjective tasks, document the evaluation rubric and judge agreement. Inspect calibration by configuration and task group where sample counts permit.

## 6. Routing policy

For prompt $p$, let $\mathcal A(p)$ contain the configurations that meet eligibility requirements, such as supported effort settings and context limits.

Define the best predicted quality:

$$
Q_{\max}(p)=\max_{(m,e)\in\mathcal A(p)}\hat P(Y=1\mid p,m,e)
$$

Given an allowed predicted quality reduction $\delta$, select:

$$
(m^*,e^*)=\arg\min_{(m,e)\in\mathcal A(p)}\hat C(p,m,e)
$$

subject to:

$$
\hat P(Y=1\mid p,m,e)\ge Q_{\max}(p)-\delta
$$

Here, $\hat C$ is expected cost estimated before execution. For example, $\delta=0.02$ permits a two-percentage-point reduction in **predicted** success probability.

This does not guarantee the same bound on realized quality loss. Validate the policy on held-out prompts and report cost–success tradeoffs. Relative tolerance also does not establish an absolute quality floor: every available configuration may have a low predicted success rate.

## 7. First experiment

Train these three predictors on the same data and prompt-grouped splits:

| Experiment | Features | Algorithm |
| --- | --- | --- |
| Baseline | Prompt features, model identity, and effort | Regularized logistic regression / IRT-style model |
| Primary | Prompt metadata, compressed embedding, model identity, and effort | LightGBM |
| Challenger | Prompt embedding and learned model/effort embeddings | Small MLP |

For all three, target **calibrated probability of success**, rather than an arbitrary quality score.

Compare the learned routing policies against simple fixed-configuration baselines, including an inexpensive configuration and a high-success configuration selected using validation data. Measure predictor overhead as part of routing cost and latency.

Deliver a reproducible comparison containing probability metrics, calibration plots, held-out cost–success curves, and performance by configuration. Use the results to decide whether a more complex router is justified.

**First dataset question:** Do the 10,000 rows represent 10,000 distinct prompts, or roughly 500 prompts evaluated across 20 configurations?

## 8. Research references

The supplied proposal identifies two relevant starting points for a literature review:

- [RouteLLM: Learning to Route LLMs with Preference Data](https://arxiv.org/abs/2406.18665) — prompt-dependent routing using preference data; distinguish those labels from absolute pass/fail outcomes when adapting the approach.
- [EmbedLLM: Learning Compact Representations of Large Language Models](https://proceedings.iclr.cc/paper_files/paper/2025/file/bf5e4b85d203481d6e37bd32d9600162-Paper-Conference.pdf) — representations of models and prompts for predicting model correctness.

These references provide research context. The experiments above must establish what works for this dataset and its success definition.
