# Full Schema Validator (Complete Syntactic Validation)

## Purpose

The Full Schema Validator is the second level of syntactic validation and operates after the Sanitizer. At this point, the document already has a basic structure verified by the Pre-Validator and has undergone the Sanitizer's recoverable corrections. The Full Schema Validator applies formal, complete verification against the JSON schema defined for the action plan.

Its behavior is binary: the document either conforms to the schema or it does not. It does not correct, infer, or apply defaults. If something is wrong, it produces a precise error message that indicates exactly which field is at fault, which type was expected, and which value was found.

---

## Why the Pre-Validator and Full Schema Validator are separate

Applying the Full Schema Validator directly to raw model output would produce a high failure rate due to trivial deviations that the Sanitizer could have corrected. This unnecessarily lowers system availability and makes error messages less informative, because it becomes impossible to distinguish a genuine structural error from a string where a number was expected.

By applying the Full Schema Validator after the Sanitizer, almost all failures that reach this stage are real errors rather than trivial format deviations. Failures are therefore more meaningful and more informative.

---

## What it verifies

The formal JSON schema covers the complete structure of the document, including all fields, their types, permitted values for enumerated fields, format constraints, cardinality relationships (arrays with at least N elements, strings matching a specific pattern), and mandatory versus optional fields.

For plan nodes specifically, it verifies that every node has a unique identifier, that the operation type belongs to the defined vocabulary, that the fields specific to each operation type are present and correct, and that the dependencies declared in each node reference identifiers that exist in the plan.

This last check — verifying references in the dependency list — is technically outside the scope of a pure JSON Schema, but it is essential for ensuring the graph is internally consistent. It can be implemented as an additional pass after schema validation.

---

## Behavior on failure

A Full Schema Validator failure produces a structured event that includes the `trace_id`, the channel, the list of validation errors with precise JSON paths, and an error categorization.

This event does not automatically trigger a Plan Generator retry. It is first analyzed: is the failure an unanticipated case that suggests updating the Sanitizer? Is it an error that emerges systematically on a specific input type, indicating a prompt problem? Is it a genuinely rare case that requires escalation?

Retrying the Plan Generator is one option, but not the automatic response. A retry using the same prompt on the same input has a high probability of producing the same error.

---

## The diagnostic value of failures

Failures at the Full Schema Validator are valuable information, not mere operational errors. Over time, failure patterns reveal a great deal about system quality.

If the same field fails repeatedly with the same type of error, the Plan Generator's prompt is not adequately instructing the model on that field — or the Sanitizer should cover that correction.

If failures emerge on specific operation types, those tasks may be particularly difficult for the model to generate correctly, which suggests adding specific examples to the prompt or revisiting the definition of that operation type.

If failures are rare and distributed across many different fields, this is likely normal model variability that the Sanitizer cannot fully cover. This may be acceptable if the rate stays below a certain threshold.

---

## Relationship with the domain schema

The JSON schema validated by the Full Schema Validator is the formal artifact that defines the contract between the Plan Generator and the rest of the pipeline. All subsequent components — from the Semantic Validator to the Optimizer to the Binding — can make assumptions about what they have received based on this contract.

Schema versioning is a critical aspect. When the schema evolves, all components that depend on it must be updated in a coordinated manner. An explicit versioning mechanism in the document's `metadata` field is useful for detecting misalignments.

---

## Practical example

A plan with three nodes arrives at the Full Schema Validator after the Sanitizer. The `sort` node has the `direction` field set to `"descending"` instead of the expected value `"DESC"`. The Sanitizer did not correct this because the normalization rule for `direction` enum values was not included (it is a field with specific permitted values, and `"descending"` is understandable but invalid).

The Full Schema Validator fails with a precise message: `task[2].direction` must be one of `["ASC", "DESC"]`, found `"descending"`. This failure warrants analysis: should the Sanitizer normalize this pattern? Is it common enough to justify it?

---

## Advantages of this approach

Formal verification via JSON Schema is deterministic, testable, and independent of probabilistic components. Its correctness can be guaranteed with exhaustive unit tests.

Separating syntactic validation from semantic validation (which comes later) keeps each validator focused on one category of problems. The Full Schema Validator knows nothing about domain semantics and does not need to.

---

## Drawbacks, open questions, and known issues

The JSON schema must be kept aligned with the task vocabulary and the expectations of downstream components. In environments with active development, this synchronization requires discipline and automation.

Standard JSON Schema validators do not support all the types of constraints that might be desired — in particular, constraints that require looking at multiple fields simultaneously (for example, if `task_type` is `"sort"` then the `sort_field` field is mandatory). These constraints can be implemented as additional validations outside the pure schema, but they add complexity.

An open question concerns the depth of reference verification in dependencies. Verifying that every dependency points to an existing node is relatively straightforward. Verifying that the resulting graph is acyclic is an additional step that technically belongs to the Semantic Validator but could be implemented here for simplicity. The choice of where to draw the boundary between the two components is not fixed.
