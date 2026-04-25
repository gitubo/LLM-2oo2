# Sanitizer

## Purpose

The Sanitizer is the component that operates between pre-validation and full validation. It receives material that has passed the Pre-Validator (it is JSON with the macro-fields present) but has not yet faced the Full Schema Validator. Its job is to automatically correct small, recoverable deviations in the raw model output, bringing the document to a state where full validation has a high probability of succeeding.

The most accurate metaphor is that of a normalizer: it does not change the substance of the plan — it fixes the form. It adds optional fields with documented default values, normalizes data types (a number expressed as a string becomes a number), removes extra fields not in the schema, and standardizes formats such as dates and identifiers.

---

## The Sanitizer's fundamental rule

The Sanitizer must correct only what it knows how to correct with certainty and without risk of altering semantics. Whenever there is even the slightest doubt that a correction might change the meaning of the plan, the Sanitizer does not correct. It passes the material through as-is and lets the Full Schema Validator fail with a precise message.

This rule exists because silently wrong corrections are more dangerous than explicit errors. An error that causes the Full Schema Validator to fail is visible and manageable. A correction that introduces a semantically wrong default value is invisible and propagates downstream.

---

## What it can correct

The categories of permitted corrections are limited and well-defined.

**Mandatory fields with unambiguous defaults.** If a required field has a default value that is correct for almost all contexts in the domain, the Sanitizer can add it. The critical qualifier is "almost all contexts": if even one reasonable situation exists where the default might be wrong, that correction does not belong to the Sanitizer — it requires an explicit decision.

**Non-lossy type conversions.** A numeric value represented as a string can be converted to the correct type without ambiguity. A boolean represented as the string `"true"` or `"false"` can be converted. A date in a non-standard format can be normalized to ISO 8601 if the original format is unambiguous.

**Removal of fields not in the schema.** If the model has added extra fields that are not part of the defined schema, the Sanitizer removes them. This is safe because fields not anticipated by the schema are by definition ignored by downstream components.

**Normalization of identifiers.** If task IDs must follow a specific format and the model has produced valid but differently formatted IDs, the Sanitizer can normalize them — guaranteeing it also updates all references (the dependencies pointing to those IDs) consistently.

---

## What it cannot correct

The Sanitizer cannot and must not correct semantic or structural errors. If a plan node is missing a field that has no reasonable default value, the Sanitizer must leave the field absent and allow the Full Schema Validator to flag it. If a field has a value that does not belong to the set of permitted values, the Sanitizer cannot arbitrarily choose an alternative value.

In particular, the Sanitizer must never infer the user's intent in order to make correction decisions. If the direction of a sort (ascending or descending) is not specified, adding a default is already problematic; adding a default based on an interpretation of the query type is entirely outside the Sanitizer's scope.

---

## Logging corrections

Every correction applied must be logged with sufficient detail to be auditable. The log entry for a correction includes: the modified field, the original value (even if it was absent), the applied value, and the sanitization rule that determined the choice.

The Sanitizer's logs are a fundamental diagnostic tool. Aggregated over time, they show which types of corrections are applied most frequently. A high frequency of corrections on a specific field is not normal: it indicates that the Plan Generator's prompt has a systematic problem with that field, and the solution is not to make the Sanitizer more aggressive — it is to improve the prompt upstream.

The confidence level associated with each correction (high if the default is almost universally correct, low if the choice is more arbitrary) helps prioritize which patterns deserve attention.

---

## Practical example

The Plan Generator produces a plan with a `filter` node that has all required fields except `case_sensitive`, which is optional but must be a boolean if present. The model has produced the value `"false"` as a string instead of as a boolean.

The Sanitizer corrects the type conversion: the string `"false"` becomes the boolean `false`. Correction log entry: field `task[2].case_sensitive`, original `"false"` (string), applied `false` (boolean), rule `type_coercion_boolean`, confidence high.

Same scenario, but the `case_sensitive` field is absent. If the domain defines a default of `false` for this field, the Sanitizer adds it. Correction log entry: field `task[2].case_sensitive`, original absent, applied `false` (boolean), rule `default_case_sensitive`, confidence high (if the default is practically always correct in the domain).

Now a case the Sanitizer must not correct: the `filter` node is missing the `operator` field, which can be `"and"` or `"or"`. There is no unambiguous default: `"and"` is more common but `"or"` is semantically different. The Sanitizer leaves the field absent. The Full Schema Validator flags the error. This is the correct behavior.

---

## Advantages of this approach

Separating automatic correction from full validation is architecturally correct because it assigns different responsibilities to different components. The Full Schema Validator can be binary — pass or fail — without worrying about what could be corrected. The Sanitizer can focus on recovery without worrying about what is formally valid.

The Sanitizer reduces the number of Full Schema Validator failures that are not real errors but merely trivial format deviations. This improves system availability and reduces noise in quality metrics.

---

## Drawbacks, open questions, and known issues

The main risk of the Sanitizer is creating a gray area of accountability. When a plan passes the Sanitizer and then the Full Schema Validator fails downstream in production, it can be difficult to determine whether the error was in the original plan or in the application of a wrong default. This requires correlating Sanitizer logs with downstream failures.

The fact that the Sanitizer is the same on both channels makes it a shared single point of failure. A Sanitizer bug that silently converts a wrong value into something formally valid does so on both channels simultaneously. The comparator will see agreement and the plan will pass. The defense is to keep the Sanitizer simple and test it exhaustively, as you would with any critical code.

An open question concerns schema evolution over time. When new mandatory fields are added, it must be decided whether they have sanitizable defaults, and the Sanitizer must be updated accordingly. This requires a coordination process between the teams managing the schema and those managing the Sanitizer.
