# LLM Prompt Builder

## Purpose

The LLM Prompt Builder is the component that sits between the Capability Registry and the Plan Generator. Its job is to read the registry and construct the context to pass to the LLM: not the full registry, but exactly the information needed to generate the plan for the current request.

It is a deterministic component: given the same structured intent and the same registry, it always produces the same prompt. There is no probabilistic logic, no ambiguous decisions. This makes it testable in isolation and its output statically verifiable.

---

## Why it exists as a separate component

The obvious temptation is to include the registry in the system prompt directly, passing the complete definition of all resources to the LLM. This solution has two problems.

The first is token cost. A registry with dozens of resources — each with a schema, actions, and transversal operation support — can be very large. Passing it in full for every request is expensive and fills the context window with information irrelevant to the specific request.

The second is the quality of the generated plan. An LLM that receives more information than necessary tends to produce noisier plans: tasks referencing irrelevant resources, parameters copied from wrong examples, ambiguity in interpreting fields with similar names across different resources. Less relevant context means less unwanted variability.

The LLM Prompt Builder solves both problems by selecting only the resources relevant to the request from the registry and constructing a dense, precise context.

---

## What it produces

The Prompt Builder produces two distinct elements that are passed to the Plan Generator.

The **system prompt** contains structural instructions: the expected output format, the rules the Planner must follow (use only declared resources and fields, do not infer, do not optimize), and the complete vocabulary of relevant resources extracted from the registry.

The **vocabulary for each resource** includes: the human-readable description, the schema fields with their types and permitted values, the available actions with their parameters, and for each action the fields on which filtering, sorting, or limiting are possible along with their respective operators. This information delimits the LLM's decision space: it cannot use an undeclared operator, filter on a field not present in the schema, or reference a resource not included in the context.

---

## Selecting relevant resources

The selection of resources to include in the context is the Prompt Builder's most important decision. Including too much degrades plan quality. Excluding something necessary makes it impossible to generate the correct plan.

Selection can happen in different ways depending on domain complexity. In domains with few resources, always including all of them is acceptable. In domains with many resources, the structured intent produced by the Intent Parser is used as a key: entities mentioned in the intent identify the primary resources, and correlated resources (if the registry declares them) are included automatically.

When the Semantic Context Cache is available, the Prompt Builder uses its output as an additional input: the cache provides a ranked suggestion of which resources and fields have been relevant for similar intents in the past, allowing the Prompt Builder to construct a tighter, more precise context. The cache output is advisory — the Prompt Builder can override it if the registry has changed or if the cache entry's confidence is below threshold. When no cache entry exists, the Prompt Builder falls back to full intent-driven selection from the registry.

This dependency on the structured intent further reinforces the importance of the Intent Parser: a misclassified intent can lead the Prompt Builder to include the wrong resources in the context, producing a plan generated on incorrect foundations.

---

## The fundamental constraint on Planner behavior

The system prompt produced by the Prompt Builder must include an explicit and unambiguous constraint: the Planner produces the most explicit plan possible, without optimizations. It never collapses transversal nodes into the preceding contextual node, never infers undeclared parameters, never abbreviates the linked list.

This constraint is essential because optimization is the Optimizer's responsibility, not the Planner's. If the Planner optimizes, the output of the two channels will be influenced by different optimization choices that the Optimizer will not always be able to normalize, producing false disagreements at the Comparator.

The separation of responsibilities between Planner and Optimizer depends entirely on the quality of this constraint in the prompt.

---

## The relationship with the two channels

The Prompt Builder produces a single prompt that is passed identically to both channels of the 2oo2. This is correct: variability between the two channels must come from the diversity of the models, not from different prompts. A different prompt for each channel would introduce an uncontrolled variable that would make disagreements at the Comparator less interpretable.

---

## Conceptual example

For a request like "give me the first 10 active admin users," the Intent Parser identifies the `users` resource. The Prompt Builder reads the `users` definition from the registry and constructs a context that includes:

The resource description. The schema fields with their types: `id` (string), `role` (string with enum admin/viewer/editor), `active` (boolean), `created_at` (date-time), `last_login_at` (date-time). The fetch action with its direct parameters and filterable fields: `role` with operators `eq` and `neq`, `active` with operator `eq`. Sort support on fields `email`, `role`, `created_at`. The absence of native limit support.

The LLM receives this context and produces a plan using only these elements. It cannot invent a `status` field that does not exist in the schema. It cannot use a `gt` operator on `role` because it is not declared as supported for that field.

---

## Advantages of this approach

The Prompt Builder transforms the registry from a configuration document into an active constraint tool on LLM behavior. The limits of what plan can be generated are not imposed only by post-hoc validation but by the construction of context upfront.

This reduces the frequency of failures at the Semantic Validator: an LLM that has never seen an invalid operator in the context is less likely to use one.

The component's deterministic nature makes it verifiable: for every structured intent and every registry version, the prompt produced is predictable and inspectable. This greatly simplifies the debugging of unexpected Planner behavior.

---

## Drawbacks, open questions, and known issues

The Prompt Builder is the only point where it is decided what the LLM can see. This centrality makes it critical: a bug in the selection of relevant resources is invisible to the Semantic Validator, because the validator checks plan correctness against the registry but does not verify that the Planner received the right context.

Automatic selection of relevant resources from the structured intent works well for simple requests but may fail on requests that span multiple resources in a non-explicit way. If the intent mentions an attribute of a correlated resource without naming the resource, the Prompt Builder may not include it in the context.

An open question concerns context management when the registry evolves during the lifecycle of a request. In scenarios with asynchronous processing or with retries, it is possible that the prompt is built on a different registry version than the one used for downstream validation. An explicit strategy is needed to guarantee registry version consistency throughout the entire pipeline for a single request.
