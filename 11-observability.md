# Observability Model

## Fundamental principle

Observability is not a layer added to the system after it has been designed. It is a property that must be designed into every component from the start. This statement, often repeated as an abstract principle, has a precise practical consequence: the information needed to understand system behavior must be emitted by the components themselves at the moment they process the data — not reconstructed after the fact.

Adding observability retroactively means reopening every component to add logging and metrics, often discovering that some critical information is no longer available at the point where it is needed because it was discarded or transformed by earlier processing steps.

---

## The three levels

The system's observability is structured across three distinct levels, each answering a different question.

**Logging** answers the question "what happened." Every component emits structured events describing the operations performed, the decisions made, and the outcomes. Logs are the single source of truth about what the system did on a specific input.

**Metrics** answer the question "how often does this happen and how long does it take." They are statistical aggregations of logs that make it possible to understand system behavior over time, identify trends, and define alerts.

**Tracing** answers the question "how did a single input traverse the system." It makes it possible to reconstruct the complete path of a single request through all components, correlating events that occurred at different times and across different components.

---

## The trace_id as the foundational element

Every input that enters the system receives a `trace_id` at the moment it is created — before the Intent Parser. This identifier is propagated unchanged through all components of the entire pipeline, on both channels.

The `trace_id` must not be generated internally by the system: it must be provided by the calling system. This makes it possible to correlate the behavior of the planning pipeline with the behavior of the system that calls it, reconstructing scenarios that cross system boundaries.

Every event emitted by every component includes the `trace_id` as a primary field. This makes it possible to aggregate all events related to a single request and reconstruct its complete history.

---

## Events per component

Every component emits structured events with a minimum set of common fields: `trace_id`, component name, channel (A or B, where applicable), timestamp, outcome, and duration in milliseconds. To these are added component-specific fields.

**Intent Parser** emits: classification confidence, detected intent type, number of ambiguities found, whether the deterministic or LLM path was used in the hybrid approach, and whether a clarification request was necessary.

**Pre-Validator** emits only the outcome and, in case of failure, the error category.

**Sanitizer** emits the complete list of corrections applied, each with the modified field, the original value, the applied value, the rule used, and an estimated confidence for the correction.

**Full Schema Validator** emits the outcome and, in case of failure, the list of validation errors with precise JSON paths.

**Semantic Validator** emits the outcome, the rules checked, and in case of failure the violated rules with detail.

**Optimizer** emits the transformations applied, each with the rule used and the nodes involved.

**Logical Binding** emits, for each node, the selected category, the eliminated categories, and the reasons for elimination.

**Comparator** emits the verdict, the level at which disagreement was found, the structured diff, and the disagreement categorization.

**Physical Binding** emits, for each node, the nominal resource, the actual resource used, and in case of failover the reason.

---

## The metrics that matter

Not all metrics are equally useful. The selection criterion is actionability: a metric is useful if, when it shows an anomaly, it suggests a specific action.

**The Sanitizer correction rate per rule** is the most diagnostic metric for LLM model quality. A rule that corrects frequently signals a systematic problem in the prompt that has a direct solution.

**The distribution of Intent Parser confidence**, in particular the low percentiles, reveals the frequency of edge cases not covered by the intent taxonomy. A worsening p5 over time indicates that the input domain is evolving beyond the design's expectations.

**The Comparator disagreement rate per category** is the overall health indicator of the pipeline. A stable rate is reassuring. An increase in the topological disagreement rate may signal a model version update that changed behavior. An increase in the binding disagreement rate may signal an inconsistency in the way the two models produce binding constraints.

**Per-stage latency at the p50, p95, and p99 percentiles** makes it possible to identify bottlenecks and plan timeouts realistically. The p99 is more important than the mean for this analysis.

**The escalation rate by reason** shows where the system most frequently requires human intervention. A high escalation rate for `comparator_disagree` suggests that the Optimizer's normalization is insufficient for the current domain.

---

## The difference between operational alerts and quality alerts

**Operational alerts** signal a problem requiring immediate action. The system is not functioning or is functioning in a visibly degraded manner for users. Examples: p99 latency above the timeout threshold, Pre-Validator error rate above 20%, escalation rate above 30%.

**Quality alerts** signal a slow degradation that is not yet visible to users but requires attention in the near-to-medium term. They do not wake anyone up at night but are examined during routine maintenance. Examples: Intent Parser confidence declining 10% over the course of a week, Sanitizer correction rate on a specific field increasing 50% compared to the previous week, appearance of a new disagreement category at the Comparator not previously seen.

The distinction is important because the two types of alert require different response processes and have different thresholds.

---

## The canary system

A small percentage of real traffic — on the order of 5–10% — is continuously evaluated in shadow mode against the golden dataset: the plans produced are compared with the expected plans for that type of input, and the match rate is monitored as a quality metric.

The canary system is the only mechanism that makes it possible to detect slow system degradation before it becomes visible. The distribution of inputs in production changes over time, LLM models are updated by providers, and domain rules age in response to business changes. Without a continuous quality monitoring system, these changes are invisible until they produce a visible problem.

---

## The correlation between observability and the feedback loop

Observability data is the raw material of the feedback loop. Without structured events and aggregated metrics it is not possible to systematically identify where and how to improve the system.

This relationship must be designed explicitly: which metrics feed which improvement decisions, at what frequency, and with what review process. An observability data point that is emitted but never read or used is noise, not information.

---

## Open questions

The choice of tools for collecting, aggregating, and visualizing observability data is orthogonal to system design but has significant practical impacts. Excessively complex observability systems tend not to be used; excessively simple ones do not provide the necessary information.

The retention of structured logs must be balanced against storage costs. High-frequency logs with a lot of detail are necessary for debugging specific cases but expensive to maintain long-term. A differentiated retention strategy by level of detail and by outcome — keeping error logs longer than success logs — is common but requires explicit design.

GDPR and privacy requirements may impose constraints on retention duration and log content. If user inputs contain personal data, the pipeline must ensure they are not logged in plain text, or that they are anonymized before persistence. This must be considered in the observability design, not added afterward.
