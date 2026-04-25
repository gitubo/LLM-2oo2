# Semantic Validator

## Purpose

The Semantic Validator is the most domain-knowledge-rich component in the entire pipeline. It operates after the Full Schema Validator, on a document that is already formally valid. Its job is to verify that the plan is correct not only in form but in meaning: that operations make sense in their context, that dependencies reflect the semantic requirements of the domain, that no mandatory tasks are missing, and that the plan as a whole is coherent with the intent that originated it.

The distinction between syntactic and semantic validation is fundamental: a plan can be perfectly valid according to the JSON schema and completely wrong with respect to the problem it was meant to solve.

---

## The Capability Registry as the source of truth for validation

The Semantic Validator uses the Capability Registry as a reference for a specific category of checks: the correctness of transversal nodes relative to the resource they operate on.

When the validator encounters a `filter` node, it walks up the linked list to the first contextual node and reads the corresponding resource's schema from the registry. It verifies that the field the filter operates on exists in the schema, that the type of the value used is compatible with the declared type, and that the value conforms to any enum constraints. The same applies to `sort` nodes, which must operate on fields that exist in the schema.

This type of check is structurally different from absolute invariants and intent-dependent rules: it requires no knowledge of the application domain — only the ability to compare the plan against the resource's formal definition. The registry makes this comparison possible in a deterministic way.

---

## The categories of checks

The Semantic Validator performs checks that belong to two categories with different natures, which are important to keep distinct.

The first category comprises **absolute invariants**: rules that are true regardless of intent, because they derive from the semantics of the operations themselves. A Limit without a Sort produces non-deterministic results — always. A write node without a preceding lock acquisition is always a problem. A fetch that comes after a transformation that was supposed to receive that fetch's output as input is always circular. These rules can be deterministically encoded and are stable over time.

The second category comprises **intent-dependent rules**: rules that are only true in the presence of certain intent types, because they derive from what the user wants to achieve. A semantic Limit (the 10 most recent) requires a Sort; an arbitrary Limit (give me any 10) does not. These rules require access to the structured intent produced by the Intent Parser in order to be applied correctly.

---

## Absolute invariants

These checks are applied always, regardless of the intent's content. They are common-sense rules of the domain that have no known exceptions.

**Acyclicity verification of the dependency graph** guarantees that no circular dependencies exist. This can be done with a standard cycle detection algorithm.

**Connectivity verification** guarantees that every node is reachable from the graph, and that the graph has a well-defined entry point and exit point. Isolated nodes or disconnected subgraphs are structural errors.

**Operation preconditions** encode the mandatory semantic dependencies between task types: fetch before transform, sort before semantic limit, lock before write. These rules are documented as (prerequisite, dependent) pairs with the condition under which they apply.

**The mandatory task checklist per operation type** is the primary defense against correlated omissions: for certain categories of intent, certain tasks must be present in the plan. If a plan of type `fetch_ordered_subset` contains no sort node, the Semantic Validator detects it.

---

## Intent-dependent rules

These checks receive both the plan and the structured intent produced by the Intent Parser as input. This is why the structured intent is propagated as immutable context throughout the entire pipeline: the Semantic Validator needs it.

A concrete example: the `requires_realtime` field in the structured intent, if true, excludes the presence of caching tasks in the plan. The Semantic Validator verifies this coherence. If the plan includes a caching task and the intent specifies real-time data, there is an inconsistency.

Another example: the `ordering.required` field in the structured intent, if true, makes the presence of a sort node in the plan mandatory. If `ordering.required` is false, a sort is not required even if the plan includes a limit.

The complete list of these rules is a domain document, not a technical one. It must be built and maintained by domain experts in the specific domain where the system operates.

---

## The dependency on the Intent Parser

The Semantic Validator depends on the correctness of the structured intent to apply the second category of rules. If the Intent Parser has produced a wrong intent, the Semantic Validator will apply the correct rules for the wrong intent — approving or rejecting the plan in ways that do not correspond to the user's actual intentions.

This is one of the reasons why the quality of the Intent Parser is so critical: its errors propagate not only to the Plan Generator but also to the Semantic Validator, which is the last component before the Comparator to have access to domain knowledge.

---

## Behavior on failure

A Semantic Validator failure is more serious than a Full Schema Validator failure. It means the plan was formally correct but semantically wrong. This type of error cannot be corrected automatically: it would require understanding why the model produced a semantically wrong plan, which implies either a prompt problem or a case not anticipated by the taxonomy.

The failure is logged with detail, categorized by the type of rule violated, and typically triggers escalation to human review rather than automatic retry. A retry on the same input with the same prompt has a high probability of producing the same semantic error.

---

## Practical example

A plan arrives at the Semantic Validator with the following sequence: fetch of orders, filter by status, limit to 10. The structured intent contains `ordering.required: true` with field `created_at` and direction `DESC`.

The Semantic Validator checks the intent-dependent rules and finds the violation: `ordering.required` is true but the plan contains no sort node. The plan is rejected with a precise description: "the plan does not satisfy the ordering requirement expressed in the intent — a sort node on field `created_at` with direction `DESC` must precede the limit node."

This failure is logged and categorized as a mandatory task omission for an intent with ordering. Analyzing this pattern over time may reveal that the model systematically omits the sort when a limit is explicit in the input, suggesting that a specific example should be added to the prompt.

---

## Advantages of this approach

The Semantic Validator is the component that introduces domain knowledge into the pipeline in an explicit, testable, and maintainable way. Rules are documented, exceptions are manageable, and behavior is predictable.

The separation between absolute invariants and intent-dependent rules makes the component easier to reason about: invariants are stable and rarely require maintenance; intent-dependent rules evolve along with the intent taxonomy.

---

## Drawbacks, open questions, and known issues

The quality of the Semantic Validator depends entirely on the completeness of the checklist and domain rules. Missing rules do not produce visible errors: they produce validated plans that then fail during execution or produce wrong results. Discovering missing rules requires observing production behavior, which means the system will learn some of them only after making a mistake.

The boundary between the Semantic Validator's responsibilities and the Optimizer's is not always sharp. Some transformations — such as collapsing two sequential filters into one with AND — could be considered optimizations or could be considered corrections to a poorly structured plan. The choice of where to place a given piece of logic has implications for execution order and error visibility.

An open question concerns the management of domain rules that change over time. If a mandatory precondition is removed or added, plans generated before the change may no longer be valid. A rule versioning mechanism is needed that allows this transition to be managed without invalidating plans that were correctly generated under the previous rules.
