# Logical Binding

## Purpose

Logical Binding is the component that enriches each node of the canonical plan with information about the implementation category to use for executing that task. It does not resolve physical resources (which server, which connection, which specific endpoint) — it decides which type of implementation is semantically appropriate for that task given the context of the intent.

The distinction between Logical Binding and Physical Binding is the distinction between a semantic decision and an infrastructure decision. Which type of data source to use (database, cache, API) is a semantic decision that depends on the intent and the domain. Which specific instance of that type to use at this moment is an infrastructure decision that depends on the system's current state.

This separation is necessary because Logical Binding occurs before the Comparator and must be deterministic and replicable on both channels, whereas Physical Binding occurs afterward and may depend on runtime conditions.

---

## The relationship with the Capability Registry

In the specific context of this architecture — where every contextual node translates into an HTTP call to an external API defined in the registry — Logical Binding and the registry partially overlap.

The registry declares for each action the HTTP contract: method, endpoint, and parameter translation convention. This is precisely the decision that Logical Binding must make for contextual nodes. In this sense, the registry is already a form of declarative Logical Binding for the HTTP implementation category.

The distinction remains relevant for cases where multiple implementations are possible for the same resource — the registry describes the canonical external API, but alternatives such as caches, replicas, or legacy systems may exist, and Logical Binding can select among them based on the intent's binding constraints. In simpler cases, where only one implementation is available for each resource, Logical Binding reduces to reading the HTTP contract from the registry.

---

## The problem it solves

A plan node with type `fetch` and entity `orders` can be executed in different ways: a direct query to the primary database, a cache read, a call to a business logic service, a query to a legacy system's API. These implementations are not equivalent: they differ in latency, data freshness, side effects, and cost.

The choice among these implementations is neither arbitrary nor purely infrastructural. It depends on the intent: if the user has asked for real-time data, the cache is not appropriate. If the user has asked for historical data beyond a certain date, only the legacy system has it. This is domain knowledge, not infrastructure configuration.

Logical Binding materializes this knowledge in a deterministic way, producing a choice that can be verified and compared.

---

## How it works

Logical Binding receives, for each plan node, the operation type, the parameters, and the structured intent with its binding constraints. It applies a series of deterministic rules that select the appropriate implementation category.

The rules follow an elimination logic: starting from the complete set of available implementations for that operation type, the rules eliminate those that are incompatible with the intent's constraints until the appropriate category is identified. If more than one category remains after elimination, a default preference rule is applied — configurable for the domain.

The binding constraints produced by the LLM during plan generation (`requires_realtime`, `side_effects_acceptable`, etc.) are inputs to Logical Binding but not its only source of decisions. They are integrated with domain rules encoded in the component, which may take higher priority.

---

## The relationship with the structured intent

Logical Binding needs the structured intent for two distinct reasons.

The first is the direct use of binding constraints: if the intent specifies `requires_realtime: true`, Logical Binding uses this information to exclude cache-based implementations.

The second is coherence verification: the binding constraints produced by the LLM are checked against the intent. If the LLM produced `requires_realtime: false` but the original intent required real-time data, this inconsistency is detected here and flagged before it produces a wrong binding.

---

## Logical Binding as an enrichment of the canonical plan

The output of Logical Binding is the canonical plan enriched with a `binding` field for each node, containing the selected implementation category and the reasoning behind the choice. This field becomes part of the plan that is then compared by the Comparator.

The fact that logical binding is part of the comparison is an important architectural characteristic: if the two channels have produced identical canonical plans but Logical Binding chose different implementation categories for the same node, the Comparator detects this disagreement. This can happen if the binding constraints of the two plans are inconsistent, and the disagreement is an informative signal.

---

## Practical example

A canonical plan contains a `fetch` node for the `orders` entity. The structured intent contains `requires_realtime: true` and `side_effects_acceptable: false`.

Logical Binding has the following categories available for fetching orders: `database_primary` (real-time, no side effect), `cache_layer` (potentially stale, no side effect), `order_service` (real-time, with audit log side effect), `legacy_api` (historical beyond 90 days, no side effect).

The rule `requires_realtime: true` eliminates `cache_layer`. The rule `side_effects_acceptable: false` eliminates `order_service`. The `legacy_api` is eliminated because the `created_at` field in the intent is not in the historical range. `database_primary` remains.

The node is enriched with `binding: { category: "database_primary", reason: "realtime_required, no_side_effects" }`.

---

## Advantages of this approach

Separating responsibilities between Logical and Physical Binding prevents transient runtime conditions from contaminating plan validation. Logical Binding is deterministic and stable, which makes it suitable for replication across two channels and for comparison.

Having the implementation type choice documented as part of the plan greatly enriches observability: it is possible to audit not just what was done but why a particular type of implementation was chosen.

---

## Drawbacks, open questions, and known issues

Logical Binding shares with the Sanitizer the problem of being identical on both channels. A bug in the selection rules manifests on both channels, producing agreement at the Comparator even when the choice is wrong.

The dependency on binding constraints produced by the LLM introduces a risk: if the LLM produces inconsistent or wrong constraints, Logical Binding makes the wrong choice with confidence. The coherence check between constraints and intent mitigates this risk but does not eliminate it entirely.

Managing the addition of new implementation categories requires updating the Logical Binding rules. If a new category is added without updating the rules, it is never selected. If it is added but the elimination rules do not consider it correctly, it may be selected in inappropriate cases. This requires a coordinated deployment process.

A deeper open question concerns the cases where no available category satisfies all the intent's constraints. Logical Binding must have a defined behavior for this scenario: signal the problem and block the plan, choose the least incompatible category with a warning, or escalate to a human decision. The choice depends on the domain and the criticality of the operation.
