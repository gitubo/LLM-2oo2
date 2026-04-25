# Validated Inference Architecture for Action Plan Generation

A pipeline architecture for generating and validating structured action plans using Large Language Models, designed to reduce the probability of incorrect plans reaching execution while maintaining acceptable system availability.

---

## Table of Contents

- [Overview](#overview)
- [Documentation](#documentation)
- [Flow](#flow)
- [Key Design Decisions](#key-design-decisions)
- [Relationship to Other Validation Patterns](#relationship-to-other-validation-patterns)
- [Known Limits](#known-limits)
- [Open Areas](#open-areas)
- [Design Philosophy](#design-philosophy)

---

## Overview

Large Language Models produce correct output with high probability, but not with certainty. In contexts where model output drives concrete actions on real systems, that residual error probability is not negligible.

This architecture addresses the specific problem of automated generation of structured action plans: ordered sequences of atomic tasks, represented as directed acyclic graphs serialized as JSON, executed by a downstream system. The fundamental constraint is that a wrong plan is not just low-quality output — it is a wrong instruction that will be executed.

The design draws inspiration from the 2oo2 (two-out-of-two) pattern used in safety-critical industrial systems, adapted substantially to account for the specific characteristics of language models: correlated errors, the difficulty of defining agreement on structured output, and the need to normalize output before comparison.

---

## Documentation

| Document | Description |
|---|---|
| [00 — Overview](docs/00-overview.md) | Problem, principles, full flow, error handling, observability, feedback loop, known limits |
| [01 — Intent Parser](docs/01-intent-parser.md) | Natural language to structured intent |
| [02 — LLM Plan Generator](docs/02-llm-plan-generator.md) | Parallel plan generation on two independent channels |
| [03 — Pre-Validator](docs/03-pre-validator.md) | Fast syntactic check, fail fast on unrecoverable output |
| [04 — Sanitizer](docs/04-sanitizer.md) | Automatic correction of recoverable deviations |
| [05 — Full Schema Validator](docs/05-full-schema-validator.md) | Complete JSON Schema validation |
| [06 — Semantic Validator](docs/06-semantic-validator.md) | Domain rules and intent coherence check |
| [07 — Optimizer](docs/07-optimizer.md) | Canonical form and registry-driven node collapsing |
| [08 — Logical Binding](docs/08-logical-binding.md) | Semantic implementation selection |
| [09 — Comparator](docs/09-comparator.md) | 2oo2 agreement check between the two channels |
| [10 — Physical Binding](docs/10-physical-binding.md) | Runtime resource resolution |
| [11 — Observability](docs/11-observability.md) | Logging, metrics, tracing, alerting model |
| [12 — Feedback Loop](docs/12-feedback-loop.md) | Continuous improvement, golden dataset, drift detection |
| [13 — Capability Registry](docs/13-capability-registry.md) | Declarative source of truth for resources, actions, and HTTP contracts |
| [14 — LLM Prompt Builder](docs/14-llm-prompt-builder.md) | Registry-to-prompt translation for the Plan Generator |

---

## Flow

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
   [ Physical Binding ]
            |
            v
         Execution
```

---

## Key Design Decisions

**Two independent channels.** Two different LLM models generate the plan independently from the same structured intent. Models are selected to minimize error correlation: different providers, different architectures, different training data where possible.

**Validation before comparison.** Each channel runs its plan through a full validation pipeline before the plans reach the comparator. The comparator works on canonical, validated plans — not on raw model output.

**Registry-driven behavior.** The Capability Registry is the single source of truth for domain resources, their fields, available actions, HTTP contracts, and native support for transversal operations. Four components read from it: the Prompt Builder, the Semantic Validator, the Optimizer, and the Logical Binding. Updating the registry propagates automatically to all of them.

**Separation of semantic and infrastructure decisions.** What to do (which tasks, in which order, with which constraints) is decided during planning and validation. How to do it physically (which endpoint, which connection, which instance) is decided after the plan has been approved, so that transient runtime conditions do not contaminate planning logic.

**Explicit observability.** Every component emits structured events with a common trace_id. Every correction, every failure, every disagreement is logged with enough detail to reconstruct the full history of a single request. Observability is a design property, not an afterthought.

---

## Relationship to Other Validation Patterns

Several established patterns address the same problem — reducing the probability that an LLM produces wrong output in high-stakes contexts. Each has known limitations that motivated the design choices in this architecture.

**Self-consistency** samples N outputs from the same model and selects the most frequent. It reduces variance but not correlation — all outputs share the same training biases.

**Self-critique / Constitutional AI** has the model review its own output against a set of principles. It inherits the same epistemological problem as LLM-as-Judge: the tool that created the problem is being used to detect it.

**LLM-as-Judge** uses a separate model to validate the output of the generator. It introduces an appearance of verification without adding real independence: a judge trained on similar data tends to approve exactly the plans it should catch. When it fails, it does so silently and with confidence — no explicit failure signal, nothing to feed back into the improvement loop.

**Chain-of-thought with formal verification** forces explicit reasoning and then verifies it with deterministic tools. Promising, but only applicable where a formal verifier exists for the domain.

**Retrieval-augmented validation** checks output against an external knowledge base. Quality depends entirely on the coverage and correctness of that base.

**Multi-model ensemble** — the pattern this architecture builds on — is typically applied without a deterministic validation pipeline upstream of the comparison. Disagreement is measured on raw output, which means normalization problems produce false alarms and correlated errors go undetected.

**Selective human-in-the-loop** uses model confidence to decide when to escalate. Scalable only if high-uncertainty cases remain a small minority.

What distinguishes this architecture is the combination of channel redundancy with a deterministic, multi-stage validation pipeline before comparison. Each channel produces a canonical, fully validated plan before it reaches the comparator. Without that upstream validation, the comparison step is significantly weaker.

---

## Known Limits

This architecture reduces the probability of systematic errors and makes them visible when they occur. It does not eliminate certain categories of problems that are honest to acknowledge.

**Correlated omission.** Both models can generate a plausible, internally coherent plan that omits the same critical task. The comparator will see agreement. The defense — the semantic validator checklist — is effective only for omissions known in advance.

**Intent Parser as single point of failure.** The intent parser has no redundancy. A misinterpreted intent propagates unchanged through the entire pipeline with the system's full confidence.

**Cumulative latency.** The pipeline latency is in the order of 1-2 seconds before execution begins. For plans acting on mutable state, this window introduces a gap between planning time and execution time.

**Consistency vs correctness.** The comparator measures consistency between the two channels, not correctness. Two channels can agree on a wrong plan. High consistency is a proxy for correctness, not a direct measure.

---

## Open Areas

**Side effect validation.** For plans containing destructive actions (delete, update), structural validation is not sufficient. Two strategies are under consideration: a deterministic guard in the Semantic Validator requiring an explicit filter node before any destructive node, and an LLM-based self-consistency check using a Translator and a Judge component. The trade-offs between the two approaches are discussed in the overview document.

**Filterable fields as direct params vs collapsible transversal nodes.** The current registry format does not fully resolve the distinction between fields that are direct parameters of a contextual action and fields that are filterable via native API support. This overlap can create ambiguity in the Optimizer and warrants a more explicit representation in a future registry version.

**Cross-resource filtering.** The registry models resources as flat entities with direct actions. Filtering across relationships — for example, fetching orders filtered by a property of the related user — is not covered by the current format and would require an extension to the resource model.

---

## Design Philosophy

This documentation does not hide weaknesses. Each component document includes a section on drawbacks, doubts, and open points alongside the rationale for design choices. The architecture is presented as a working approach with known trade-offs, not as a definitive solution.

Contributions, objections, and alternative perspectives on any design decision are the intended use of this material.
