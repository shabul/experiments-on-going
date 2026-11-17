# Experiments grid

I would propose **9 experiments**, covering prompt understanding, model selection, quality prediction, output-token estimation, cost and latency prediction.

Our objective is to identify which approach gives the best **quality-to-cost trade-off while keeping routing overhead within 20–30 ms**.

The important architectural decision is that we should not necessarily use one ML model for everything. Quality, cost and latency may have different underlying relationships, so we should test them separately before combining them into a routing engine.

## 1. Our problem setup

**Only input at inference: Stateless user prompt**

**Prompt feature extraction**

Length · task type · complexity · constraints · semantic embedding · estimated output length

| Target | Prediction |
| --- | --- |
| Quality | Success probability |
| Cost | Expected token usage |
| Latency | Estimated completion time |

**Choose optimal model + reasoning effort**

We'll use the historical dataset of approximately 10,000 unique prompts and 15–20 model-effort configurations, with binary success labels, observed cost and latency.

At inference, there is no conversation history, tool state, previous response or customer-provided metadata. The model catalogue and default routing policies are maintained internally.

## 2. Proposed experiments

### 01 — Prompt metadata extraction and complexity estimation

Develop a lightweight feature extractor that derives task family, complexity, approximate reasoning requirements, code presence, numerical complexity, formatting constraints, prompt length and explicit output-length instructions.

Compare deterministic rules, TF-IDF/linear models and a lightweight text classifier. Evaluate feature usefulness, inference overhead and whether complexity features improve downstream model selection.

### 02 — Statistical and rule-based routing baseline

Without any advanced ML, calculate the historical success rate, average cost and latency for each model-effort configuration, globally and by task family.

Choose the cheapest configuration that satisfies the required quality threshold. This establishes whether sophisticated prompt understanding genuinely outperforms straightforward statistical routing.

### 03 — Semantic similarity-based routing

Generate embeddings for historical prompts and store them in an approximate nearest-neighbor index. For every new prompt, retrieve the most similar historical prompts and use their observed outcomes to estimate which model-effort combination is most suitable.

Experiment with different embedding dimensions, nearest-neighbor counts and similarity weighting. Main question: can retrieval alone deliver competitive routing without training a dedicated predictor?

### 04 — Gradient-boosted quality prediction — LightGBM / XGBoost

Train a candidate-conditioned classifier using prompt features, model identity and reasoning-effort level to predict the probability of successful completion.

Compare handcrafted metadata alone against metadata combined with compressed embeddings. This would be one of my highest-priority experiments because gradient-boosted trees are efficient and can work well on datasets of this size.

### 05 — Small neural network using prompt embeddings

Use a frozen lightweight encoder, such as MiniLM, followed by an MLP that combines the prompt embedding with learned representations of the model and reasoning effort.

Compare frozen embeddings, partially fine-tuned encoders and different MLP sizes. The goal is to understand whether semantic representations capture differences in model suitability that tabular features miss.

### 06 — Matrix factorization and pairwise ranking

Treat prompts as users and model-effort configurations as items in a recommendation system. Train a model to learn which configurations succeed on which types of prompts.

Compare matrix factorization against pairwise ranking models that learn when configuration A is preferable to configuration B. This is inspired by [RouteLLM](https://github.com/lm-sys/RouteLLM), which publishes implementations of matrix factorization, BERT classification and other routing methods.

### 07 — LLM-based routing and knowledge distillation

Provide an LLM with the prompt, available model-effort configurations and a compact description of routing policies. Ask it to select the best configuration in one structured output.

Evaluate its routing accuracy, latency and cost. Separately, use its generated task assessments as offline features or teacher labels for a small student model. This will reveal whether LLM reasoning adds useful information without requiring an expensive LLM call during production routing.

### 08 — Output-token prediction and cost estimation

Predict response length for each prompt, model and reasoning-effort combination. Compare simple statistical distributions by task family, LightGBM regression, and embedding-based regression.

Once output-token usage is estimated, calculate monetary cost mathematically from the provider's pricing schedule. Also compare this approach against directly predicting historical observed cost.

This is supported by research such as [Prompt2Length](https://ieeexplore.ieee.org/document/11290434/), which investigates lightweight prediction of output-length categories directly from prompts.

### 09 — Mathematical versus ML-based latency prediction

Establish a mathematical latency model using input-token count, predicted generated-token count, model-specific processing characteristics and fixed overhead.

Compare this with LightGBM regression using prompt features, embeddings, model ID and reasoning effort. Test both median latency prediction and P90/P95 prediction to account for provider variability, scheduling and queue delays.

Finally test whether introducing predicted cost improves latency estimation, and vice versa, without using future information unavailable when the routing decision is made.

## 3. My thoughts on cost versus latency

I think your idea about their correlation is valid, but there is one important distinction.

**Cost is often mathematically computable. Latency is primarily a prediction problem.**

For token-billed APIs, a simplified cost calculation is:

$$
C=\frac{T_{in}P_{in}+T_{out}P_{out}}{10^6}
$$

Here, input tokens are measurable before calling the model. Output tokens must be estimated. For reasoning models, billed output usage may include hidden reasoning tokens, not just visible response text.

Latency can be approximated as:

$$
L \approx L_{fixed}+L_{prefill}(T_{in})+ \frac{\hat T_{out}}{R_{decode}}
$$

The output count and effective decode speed depend on the model and effort level, while queueing and network overhead introduce additional variability.

My preferred comparison would be:

| Prediction approach | Cost | Latency |
| --- | --- | --- |
| Input token count alone | Weak baseline | Weak baseline |
| Input + predicted output tokens | Strong candidate | Strong candidate |
| Metadata + LightGBM | Useful alternative | Useful alternative |
| Prompt embeddings + regression | Worth testing | Worth testing |
| Cost predicts latency or vice versa | Secondary feature only | Secondary feature only |

A costly model may respond faster than a cheaper model, even for the same number of tokens. Therefore, I would **not use cost as the primary predictor of latency or latency as the primary predictor of cost**.

Also, if the historical records lack actual input/output token counts, that limits our ability to validate the token-based equations. We would need to recover usage records or collect them in subsequent experiments.

## 4. How we'll compare all experiments

We should evaluate every experiment using the same prompt-level train/validation/test split, ensuring that different configuration rows for the same prompt never appear across splits.

| Evaluation dimension | What to measure |
| --- | --- |
| Quality | Actual success rate of selected configuration |
| Cost efficiency | Total cost savings at equal or near-equal quality |
| Quality tolerance | No more than 1–2 percentage-point overall success-rate reduction, initially |
| Routing overhead | P50, P95, P99; target P95 within 20–30 ms |
| Prediction accuracy | Log loss, Brier score and calibration for quality; MAE and quantile coverage for cost/latency |
| Generalization | Performance on unseen prompts and underrepresented task families |
| Robustness | Unexpected prompts, lengthy inputs, model unavailability and prediction uncertainty |

For comparison, use the best fixed model, cheapest fixed model, historical family-based policy and an offline oracle that knows the true outcomes. The oracle represents an upper bound rather than a deployable router.

We should also test **predicted-quality optimization versus realized-quality optimization**. A router can have impressive classifier accuracy and still choose the wrong configuration when the cost of its mistakes is high.

## 5. Recommended execution priority

### Phase 1

**Experiments 01, 02, 04, 08, 09**

Establish metadata, simple baselines, LightGBM quality prediction, and mathematical/ML cost and latency models.

### Phase 2

**Experiments 03, 05, 06**

Test whether embeddings, neural models or preference-learning methods outperform the tabular approach.

### Phase 3

**Experiment 07 + integrated optimization**

Benchmark LLM-based routing, assess distillation, combine the best quality/cost/latency estimators and evaluate the final routing policies.

My current leading candidate is **a frozen lightweight embedding model + LightGBM quality predictor + separate token-based cost and latency estimators + deterministic policy optimization**.

But I would not commit to that architecture before the simpler baselines establish whether embeddings provide measurable gains.

The final decision should be driven by a quality-versus-cost Pareto frontier, routing latency, calibration and performance on genuinely unseen prompts—not by which ML architecture is most sophisticated.
