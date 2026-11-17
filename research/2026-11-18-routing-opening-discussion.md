# Intelligent model routing — opening discussion
Date: 2026-11-18
Status: Problem framing; no architecture selected or implementation authorized.

## User intent
Collaboratively investigate OpenAI Decisions for intelligent model routing. Compare prompt-based, deterministic, statistical, learned, and hybrid approaches before choosing an architecture. Minimize latency, cost, operational complexity, and manual model knowledge engineering. Target approximately 1–2 engineer-days to onboard a model or reasoning configuration. Ask focused questions and challenge assumptions.

## Verified initial API facts
The official guide describes POST /v1/decisions with gpt-6-luna, typed predicate/choice/score answers, and up to 400 questions. Independent questions can share one request; dependent questions require separate requests. The documented speed comparison is not a workload-specific latency guarantee.
Source: https://developers.openai.com/api/docs/guides/decisions
The create reference documents shared input and per-question instructions. It is an interface for classification/scoring, not evidence that its answers predict downstream task success.
Source: https://developers.openai.com/api/reference/resources/decisions/methods/create

## Working decomposition (hypotheses, not a selected architecture)
1. Eligibility: enforce supported modalities, tools, context capacity, residency, availability, and budget policy.
2. Request understanding: infer semantic properties only where useful.
3. Outcome estimation: estimate quality, cost, and latency for eligible model/configuration pairs from measured evidence.
4. Selection: apply an explicit objective and uncertainty policy.
5. Execution and learning: bounded retries/escalation, telemetry, evaluation, and drift checks.

Keep model and effort jointly evaluable even if the implementation has separate stages. Choosing a model first can discard a better model–effort pair. Effort labels should not be assumed comparable across models.

## Assumptions to test
- General prompt complexity may be less useful than predicting relative performance between candidate configurations.
- Router confidence is not automatically a calibrated probability of downstream success.
- Logs reveal the selected model's outcome, not counterfactual outcomes for alternatives; adaptation requires deliberate evaluation or controlled exploration.
- One to two days of engineering is distinct from elapsed evaluation time and evaluation spend.
- Prompt-based routing remains a useful baseline; hybrids must justify their added operational burden.
- Novel provider integration may exceed the onboarding budget even if another configuration on an existing adapter fits it.

## First questions
1. What are the first two or three workloads, and is the decision per request, conversation, or agent step?
2. What defines a successful output, and are representative prompts, tests, human labels, or historical outcomes available?
3. What routing overhead and end-to-end latency are acceptable; which wins when quality, latency, and cost conflict?
4. Which providers/modalities/tool requirements are in scope, and does onboarding mean configuration on an existing provider or a new integration?

## Next discussion
Use the answers to formalize constraints and measurable objectives. Then compare candidate architectures against simple baselines, review empirical routing research, and complete an API capability/limits audit. No final recommendation yet.

## Maintenance
Revalidate dated API claims before detailed design and whenever the provider changes interfaces. Keep superseded conclusions explicitly marked rather than silently replacing evidence. Recurring housekeeping cadence remains to be established.

