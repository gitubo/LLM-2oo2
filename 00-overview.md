# Validated Inference Architecture for Action Plan Generation

## The problem this architecture addresses

Large Language Models are powerful tools, but they are inherently probabilistic. This characteristic — the very thing that makes them flexible and capable of complex reasoning — is also their primary weakness: they produce correct output with high probability, but not with certainty. In contexts where model output drives concrete actions on real systems, that residual error probability is not negligible.

The problem compounds when you consider the nature of LLM errors. These are not random, uniformly distributed errors, like noise in an electronic signal. Language model errors are systematic and correlated: they tend to surface on the same types of input, follow the same training biases, and omit the same categories of information. This makes classical redundancy mechanisms — designed for independent errors — less effective in practice than theory would suggest.

The specific case this architecture addresses is the automated generation of structured action plans: ordered sequences of atomic tasks, represented as directed acyclic graphs serialized as JSON, executed by a downstream system. The fundamental constraint is that a wrong plan is not merely low-quality output — it is a wrong instruction that will be executed.

---

## The 2oo2 pattern and why we do not follow it as a normative reference

This architecture draws inspiration from the 2oo2 (two-out-of-two) pattern used in safety-critical industrial systems, such as avionics, railway control, and nuclear plant management. In that context, two independent channels process the same input, and the system proceeds only if both agree. Disagreement is treated as a danger signal, regardless of which channel is correct.

The principle is elegant: you do not need to know who is wrong — you only need to know that something does not add up in order to stop.

However, the industrial 2oo2 pattern was designed in a radically different context. Redundant hardware systems have genuinely independent failures: the probability that two transistors malfunction in a way that produces the same wrong output is negligible. Two LLMs — even ones that differ in architecture and provider — share training biases rooted in the same corpus of data that the entire field has been trained on. Error correlation is never zero, and for certain types of input it can be quite high.

This architecture therefore uses the 2oo2 pattern as a conceptual starting point, adapting it substantially to account for the specific characteristics of language models: correlated errors, the difficulty of defining "agreement" on structured output, the need to normalize output before comparison, and the presence of deterministic components that break the symmetry between the two channels.

---

## What this architecture aims to solve

The primary objective is to reduce the probability that a wrong action plan reaches execution, while maintaining acceptable system availability. These two goals are in tension: adding more checks reduces the probability of errors but increases the probability of blocks, delays, and false alarms.

This architecture does not aim to eliminate errors. It aims to make them visible, categorized, and manageable — distinguishing between errors that can be corrected automatically, errors that require human intervention, and errors that are signals of a systemic design problem.

The secondary objectives, equally important, are observability and continuous improvement. A system that silently validates without emitting metrics makes it impossible to tell whether it is working correctly or slowly degrading over time. A system that does not use its own production data to improve itself stays frozen at its initial state.

---

## The foundational principles

**Separation of responsibilities per component.** Every component in the pipeline has a precise responsibility and does not concern itself with problems belonging to other components. The syntactic validator does not reason about semantics; the optimizer does not validate; the comparator does not correct. This separation makes every component testable in isolation and the system's behavior predictable.

**Progressive validation.** Model output is validated in successive stages, each of which assumes that prior stages have already done their work. This allows progressively more expensive checks to be applied only to material that has already passed earlier ones, and produces precise error messages that identify exactly where and why something failed.

**Normalization before comparison.** Comparison between the two channels happens on canonical plans, not on raw output. Two semantically equivalent plans expressed in different forms are reduced to the same representation before comparison, reducing false disagreements that would unnecessarily degrade system availability.

**Separation of semantic and infrastructure decisions.** Decisions about what to do — which tasks, in which order, with which constraints — are made during the planning phase of the pipeline. Decisions about how to do it physically — which endpoint, which connection, which instance — are made after validation, once the plan has already been approved. This prevents transient runtime conditions from contaminating the validation logic.

**Observability as a design property, not an afterthought.** Every component emits structured events with a common trace_id that allows the path of a single input through the entire pipeline to be reconstructed. Aggregated metrics make it possible to detect degradation before it becomes visible to users.

---

## The high-level flow

A natural language input enters the system and is processed sequentially through the following macro-stages:

**Intent understanding.** The input is transformed from natural language into a structured representation of the user's objective. This representation is not a plan — it is a specification: it describes what the user wants to achieve, under which constraints, and with what degree of certainty. If the input is genuinely ambiguous, the system can request clarification rather than proceeding with an arbitrary interpretation.

**Planner context construction.** The Capability Registry is read to extract the resources relevant to the request. From these, the Planner's system prompt is constructed: field schemas, available actions, permitted operators for each field. The Planner does not receive the full registry — only the information necessary for the specific request. This constrains the model's decision space before validation even intervenes.

**Parallel plan generation.** Two distinct language models, selected to minimize error correlation, independently generate an action plan from the same structured intent and the same context. The produced plan is explicit and unoptimized: every transversal node is separate, and no optimization is performed by the Planner.

**Per-channel validation pipeline.** Each generated plan passes through a validation pipeline that proceeds in stages: a fast syntactic pre-validation that discards incomprehensible output; a sanitizer that corrects small, recoverable deviations; a complete syntactic validation via JSON Schema; and a semantic validation that verifies the plan's coherence with the intent, domain rules, and the resource schemas defined in the registry.

**Optimization.** The validated plan is brought into canonical form. Transversal nodes that the external API supports natively — as declared in the Capability Registry — are collapsed into the preceding contextual node. Remaining nodes are normalized to eliminate equivalent variants that would produce false disagreements during comparison.

**Logical binding.** Before comparison, each contextual node is enriched with implementation information derived from the registry: HTTP method, endpoint, and parameter translation convention. In cases where multiple implementations are available, the choice is guided by the intent's binding constraints.

**Channel comparison.** The two canonical plans are compared. If they agree, the system proceeds. If they diverge, the system categorizes the disagreement and decides the consequent behavior.

**Physical binding and execution.** The agreed plan is translated into concrete, executable instructions by resolving the infrastructure resources available at that moment. This phase occurs after validation, so that transient runtime conditions do not influence planning decisions.

---

## Flow diagram

```
User input (natural language)
           |
           v
   [ Intent Parser ]
     Confidence < threshold?  -->  Clarification  -->  user
           |
           v
   [ LLM Prompt Builder ]  <--  Capability Registry
           |
     (same context)
     |             |
     v             v
 [ LLM A ]     [ LLM B ]        <- different models, parallel
     |             |
     v             v
[ Pre-Validator ] [ Pre-Validator ]
  fail? retry       fail? retry
     |             |
     v             v
[ Sanitizer ]  [ Sanitizer ]
     |             |
     v             v
[ Full Schema  [ Full Schema
  Validator ]   Validator ]
  fail? escalate   fail? escalate
     |             |
     v             v
[ Semantic     [ Semantic
  Validator ]   Validator ]
  fail? escalate   fail? escalate
     |             |
     v             v
[ Optimizer ]  [ Optimizer ]    <- node collapsing via registry
     |             |
     v             v
[ Logical      [ Logical
  Binding ]     Binding ]       <- HTTP contract from registry
     |             |
     +------+------+
            |
            v
      [ Comparator ]
      disagreement?  -->  retry / escalate / human review
            |
            v
   [ Physical Binding ]         <-- runtime resource resolution
     resource unavailable?  -->  logged failover / explicit error
            |
            v
      [ Dispatcher ]
            |
            v
         Execution
```

The Capability Registry does not appear in the vertical sequence because it does not process the flow: it is read transversally by the LLM Prompt Builder, Semantic Validator, Optimizer, and Logical Binding. It is a configuration dependency, not a processing component.

---

## Error handling

Errors in the pipeline are treated as structured events, not generic exceptions. Every error type has a defined behavior.

Pre-validation fails on unrecoverable material: if the model has not produced valid JSON with the fundamental fields present, the plan is discarded and a new generation is requested. There is no point in passing incomprehensible material to subsequent components.

The sanitizer acts on recoverable problems without interrupting the flow, but logs every correction it applies. A high frequency of corrections on a specific field is a signal that the model's prompt has a systematic problem — not an occasional error.

Full validation fails on material that the sanitizer could not correct. This failure triggers a structured analysis: the plan was formally invalid in a way that was not anticipated by the sanitizer's design, and that warrants investigation.

When the comparator detects disagreement between the two channels, it does not simply produce an error — it produces a categorization of the disagreement: structural, topological, or logical binding. This makes it possible to distinguish cases where disagreement is a strong signal (the two models interpreted the intent differently) from cases where it is noise (minor phrasing differences that the optimizer should have normalized).

---

## Observability

Every component emits structured events that include a common trace_id, the component name, the channel it belongs to, the outcome, the corrections or transformations applied, and the latency. These events allow the path of every individual input through the pipeline to be reconstructed.

The aggregated metrics of interest are: the sanitizer correction rate per rule, the comparator disagreement rate per category, the intent parser confidence distribution, per-stage latency at various percentiles, and the escalation rate by reason.

An alerting system distinguishes between operational alerts — signaling an immediate problem — and quality alerts — signaling a slow degradation that requires attention but not emergency response.

---

## Feedback loop

Data produced by the pipeline in production feeds a continuous improvement process that operates on three distinct time scales.

In real time, operational thresholds can be adjusted automatically in response to anomalous conditions — for example, a spike in the escalation rate that suggests temporarily lowering the comparator's acceptance threshold.

On a periodic basis, aggregated logs are analyzed to identify systemic patterns: recurring sanitizer corrections, frequent disagreement types, input categories with low confidence. These patterns guide updates to model prompts, sanitizer rules, and semantic validator checklists.

On longer cycles, changes to the intent taxonomy, the JSON schema, and the LLM models themselves are managed through a formal process that includes regression testing on the golden dataset.

The golden dataset — composed of cases reviewed by domain experts over time — is the system's most valuable asset. It makes it possible to measure the effect of every change and to detect degradation caused by input distribution drift or behavioral changes in models supplied by third parties.

---

## The limits this architecture does not overcome

This architecture significantly reduces the probability of systematic errors and makes them visible when they occur. It does not eliminate certain categories of problems, which it is honest to acknowledge.

Correlated omission remains the most insidious failure mode: both models can generate a plausible, internally coherent plan that omits the same critical task, and the comparator will see agreement. The defense — the semantic validator checklist — is effective only for omissions that are known in advance to require checking.

The cumulative pipeline latency is on the order of 1–2 seconds before execution begins. For plans acting on mutable state, this interval introduces a window during which the system could change between planning time and execution time.

The quality of the entire chain depends critically on the quality of the intent parser, which is the only component without redundancy. A misinterpreted intent propagates unchanged through the entire pipeline with the system's full confidence.

These limits are documented not because they lack partial solutions, but because knowing them is necessary to correctly evaluate when this architecture is appropriate for the problem at hand and when it is not.

---

## Open areas not documented here

One area this documentation does not address in a structured way is the semantic validation of plans containing actions with side effects — that is, operations that modify state in an irreversible way, such as delete or update.

For these cases there is an approach based on a second LLM pass: one component translates the plan into natural language, and a second component compares this description with the original intent to verify semantic correspondence. The pattern is conceptually close to the LLM-as-Judge approach analyzed in the early design phases, and it shares both its advantages and its limitations: it works when the two LLMs have low correlation and when the translation component is strictly constrained not to infer intent, but it remains vulnerable to correlated errors and to the possibility that the plan description ends up artificially close to the original intent.

The more robust alternative defense for side effects is deterministic: the Semantic Validator verifies that every plan containing a destructive node has at least one explicit, sufficiently specific filter node upstream. This structural guarantee does not evaluate whether the filter is semantically correct, but it ensures that the operation does not act on the entire collection without restrictions.

The choice between these two strategies — or a combination of both — depends on how critical side-effect operations are in the specific domain, and on the willingness to accept the latency and complexity costs of the second LLM pass.
