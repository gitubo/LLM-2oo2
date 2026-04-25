# Capability Registry

## Purpose

The Capability Registry is the single source of truth for the entire system. It is a declarative document — no logic, no functions — that describes the domain resources, their fields, the actions available on them, and the contract with the external APIs that expose them.

Its position in the architecture is transversal: it is not a component of the sequential pipeline but a configuration artifact read by multiple components at different moments. The LLM Prompt Builder reads it to construct the Planner's context. The Semantic Validator reads it to verify the correctness of fields and operators. The Optimizer reads it to decide which transversal nodes to collapse. Logical Binding reads it to construct the HTTP contract.

This centrality is its primary value: there is only one place to modify the definition of a resource, and that change propagates automatically to all components that depend on it.

---

## The distinction between contextual and transversal nodes

Before describing the registry's structure, it is necessary to understand the fundamental distinction between the two types of nodes that compose a plan.

**Contextual nodes** belong to a specific resource and translate into an HTTP call to an external API. Fetch, delete, and update are contextual nodes. They know the entity they operate on, the HTTP method, and the endpoint.

**Transversal nodes** operate on any collection regardless of the resource. Filter, sort, limit, and map are transversal nodes. They do not make HTTP calls: they are either executed client-side, or — when the external API supports them natively — they are collapsed into the preceding contextual node during the optimization phase.

The registry is the source that defines this distinction for each resource: not in the abstract, but concretely for each field and each operation.

---

## Structure

The registry is organized by resource. Each resource has three main sections.

**Field schema** — describes the fields of the domain object: type, permitted values if enumerable, format if structured. This section is used by the Semantic Validator to verify that transversal nodes operate on fields that actually exist on the resource, and by the LLM to know on which fields it can build filters and sorts.

**Available actions** — describes the operations that can be performed on the resource. For each action: a human-readable description that goes into the prompt, the `side_effect` flag that activates additional validation mechanisms, the direct parameters accepted by the action, and the HTTP contract.

**Native support for transversal operations** — for each action, which transversal operations the external API is capable of handling natively, on which fields, and with which operators. This is the section the Optimizer uses to decide what to collapse.

---

## The format

```json
{
  "registry_version": "1.0",
  "resources": {
    "users": {
      "description": "Registered platform users",
      "schema": {
        "id":           { "type": "string" },
        "role":         { "type": "string", "enum": ["admin", "viewer", "editor"] },
        "active":       { "type": "boolean" },
        "created_at":   { "type": "string", "format": "date-time" },
        "last_login_at":{ "type": "string", "format": "date-time" }
      },
      "actions": {
        "fetch": {
          "description": "Retrieve the list of users",
          "side_effect": false,
          "params": {
            "role":   { "type": "string", "enum": ["admin","viewer","editor"], "optional": true },
            "active": { "type": "boolean", "optional": true }
          },
          "http": {
            "method": "GET",
            "endpoint": "/users",
            "convention": "suffix"
          },
          "supports": {
            "filter": {
              "fields": {
                "role":   { "operators": ["eq", "neq"] },
                "active": { "operators": ["eq"] }
              }
            },
            "sort":  { "fields": ["email", "role", "created_at"] },
            "limit": false
          }
        },
        "delete": {
          "description": "Delete the users in the current collection",
          "side_effect": true,
          "params": {},
          "http": {
            "method": "DELETE",
            "endpoint": "/users",
            "convention": "suffix"
          },
          "supports": {
            "filter": false,
            "sort":   false,
            "limit":  false
          }
        }
      }
    }
  },
  "conventions": {
    "suffix": "field__op=value",
    "json":   "filter={field:{$op:value}}",
    "colon":  "filter=field:op:value",
    "qs":     "where=field op value"
  }
}
```

---

## Design choices

**`registry_version` at the root level.** The registry evolves alongside the domain. The components that read it are updated on different time cycles. The version field allows each component to detect a misalignment instead of behaving unpredictably on a format it does not recognize.

**Conventions as a separate section.** The registry declares for each action the name of the convention used by the external API, not the translation logic. The logic lives in a separate layer with one adapter per convention. This means that adding a new resource never requires writing translation logic, and that adding support for a new external API requires writing only one adapter — reusable by all resources that use that convention.

**`side_effect` as an explicit declaration.** It is not inferred from the HTTP method. A DELETE might have no critical side effects in certain contexts; a POST might have them. The semantics of side effect in the sense that matters to the system — operations that modify state irreversibly — cannot be mechanically deduced from the HTTP method and must be declared deliberately.

**`supports: false` as an explicit value.** When a transversal operation is not natively supported, the value is `false`, not an absent field. The distinction matters for the Optimizer: an absent field might indicate an incomplete registry or an older schema version; `false` is a deliberate declaration that the operation remains a client-side node.

---

## How it is used by downstream components

**The LLM Prompt Builder** reads the registry to build the Planner's context: it extracts the resource description, schema fields, available actions with their parameters, and filterable fields with their permitted operators. It passes to the LLM only the resources relevant to the current request, not the full registry.

**The Semantic Validator** reads the schema to verify that every filter and sort node in the plan operates on fields that exist on the resource, and that the operators used are among those declared as supported.

**The Optimizer** reads the `supports` section of each contextual action to decide which subsequent transversal nodes can be collapsed into it. If filtering on a certain field is natively supported, the filter node is removed from the linked list and its parameters are added to the contextual node's parameters.

**Logical Binding** reads the `http` section to construct the call contract: method, endpoint, and the name of the convention to apply for translating parameters.

---

## Advantages of this approach

The single source of truth eliminates the possibility of misalignments between components. When an external API adds native support for filtering on a new field, only the registry is updated in one place, and the Optimizer automatically starts collapsing that node without any code changes.

The declarative format makes the registry readable and modifiable by anyone with domain knowledge, without requiring technical expertise in the pipeline. It is a configuration document, not a programming one.

The separation between field schema and available actions reflects a real semantic distinction: the schema describes the domain object, and the actions describe what can be done with it. This separation allows the Semantic Validator to perform type checking on transversal nodes by walking up the linked list to the first contextual node and using the schema of the corresponding resource.

---

## Drawbacks, open questions, and known issues

The distinction between filterable fields as direct action parameters and filterable fields as collapsible transversal operations is not fully resolved in the current format. In the proposed format, a field can appear in both `params` (as a direct parameter of the fetch action) and in `supports.filter.fields` (as a natively collapsible filter). These are two filtering mechanisms with different implementations on the external API, and the registry documentation must make this distinction explicit to avoid ambiguity in the Optimizer.

The registry is a single point of knowledge failure: if a resource is defined incorrectly — a missing operator, a field with the wrong type, a wrong endpoint — the error propagates to all components that use it. Registry quality cannot be fully verified automatically: it requires review by domain experts and integration testing with the actual external APIs.

Managing registry versions in environments with frequent deployments is a non-trivial practical problem. A registry update that adds a new mandatory field could invalidate plans generated and cached under the previous version. A compatibility strategy is needed that is not defined in the current format.

An open question concerns resources with actions that operate on sub-resources or on relationships between resources. The current format models well the simple case of a flat resource with direct actions, but does not cover cases such as `fetch(orders).filter(user.role=admin)` where the filter crosses a relationship. This limitation should be explicitly acknowledged in the documentation to avoid discovering it only in production.
