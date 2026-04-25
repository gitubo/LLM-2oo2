# Physical Binding

## Purpose

Physical Binding is the component that translates the agreed plan — with its logical binding already defined — into a concretely executable plan. Where Logical Binding established which implementation category to use for each node, Physical Binding establishes which specific physical resource to use right now: which database instance, which API endpoint, which node in the cache cluster.

It operates after the Comparator, at a point where the plan has already received validation from the entire pipeline. This positioning is a deliberate design choice: runtime conditions must not influence planning and validation decisions.

---

## Why it occurs after the Comparator

The main motivation is that runtime conditions are inherently non-deterministic and mutable. Including infrastructure decisions in the validation phase would mean that a network timeout, a failover, or a load variation could produce different resolutions on the two channels, creating disagreements at the Comparator that have nothing to do with the quality of the plan.

Logical Binding ensures that the implementation category was chosen in a deterministic and comparable way. Physical Binding then resolves — once, after agreement — which physical resource of that category to use at this specific moment.

---

## What it does concretely

For each node in the agreed plan, Physical Binding reads the implementation category established by Logical Binding and resolves the appropriate physical resource using current knowledge of the system's state.

For a `database_primary` category, it might resolve to the primary database if available, or to a read replica if the primary is under maintenance, with an explicit log entry for the failover.

For a `cache_layer` category, it resolves to the appropriate cache instance for that entity and verifies that the remaining TTL is compatible with the freshness constraints in the intent (if present).

For a `service` category, it resolves to the available service endpoint, handling any load balancing and circuit breaker mechanisms.

---

## Failover and its visibility

Physical Binding is where infrastructure failovers materialize. When the preferred resource is unavailable and an alternative is used, this event must be logged explicitly and in a structured way — not handled silently.

A silent failover is dangerous because the plan is executed on a different resource than the nominal one, which may have implications for data correctness or freshness, yet no downstream component is aware of it.

The Physical Binding log must include, for each node: the required category, the nominal resource, the resource actually used, and in case of failover the reason. This information makes it possible to correlate any output anomalies with infrastructure conditions at the time of execution.

---

## The time window between validation and execution

Physical Binding operates after a pipeline with a cumulative latency on the order of 1–2 seconds. This means there is always a time window between when the plan was validated and when it is physically resolved and then executed.

During this window, the system's state can change: a resource can become unavailable, a piece of data can be updated, a lock can be acquired by another process. Physical Binding handles infrastructure changes (whether a resource is available or not), but it cannot handle changes in the state of the data that the plan will read or modify.

This is an intrinsic architectural limitation that must be acknowledged and documented. For operations requiring absolute consistency, a strategy at the execution level — such as transactions or explicit locks — is needed, and this is not managed by the planning pipeline.

---

## The relationship with execution

Physical Binding produces as output an executable plan: a structure containing for each node not just what to do and under which implementation category, but the concrete resource details (connection string, endpoint URL, connection parameters) needed to begin execution.

This output is the boundary between the planning pipeline and the execution system. The execution system assumes the plan is correct and complete, and executes it. Errors that emerge during execution — timeouts, business logic errors, data not found — are handled by the execution system with its own retry and error handling logic, not by the planning pipeline.

---

## Practical example

The agreed plan contains a `fetch(orders)` node with logical binding `category: database_primary`. Physical Binding queries service discovery to find the current primary database instance, verifies it is reachable, and resolves the specific connection.

At this moment, the primary database is under maintenance. Physical Binding detects the condition, records in the log: "nominal resource db-primary unavailable, failing over to db-replica-2," and resolves the connection to the read replica.

The log includes a note that data may have a replication delay of a few seconds. If the intent had specified `requires_realtime` with zero tolerance, this failover should produce an error rather than a silent fallback. Physical Binding must be capable of verifying this compatibility.

---

## Advantages of this approach

Separating Physical Binding from the validation phase keeps the pipeline deterministic and reproducible. The same input always produces the same canonical plan and the same logical binding, regardless of the infrastructure conditions at the moment.

Physical Binding as an explicit component makes infrastructure failovers visible and logged, rather than handling them silently at scattered connection points in the execution code.

---

## Drawbacks, open questions, and known issues

Physical Binding is the pipeline component with the least supervision. It operates after the Comparator, which is the last quality checkpoint. Its logs are the only visibility tool for what it does.

A silent failover that produces slightly stale data can be difficult to correlate with an output anomaly, especially if the anomaly manifests long after execution. Investing in structured logging and in automatic correlation between Physical Binding logs and output anomalies is essential.

A significant open question concerns the handling of the case where no physical resource is available for an implementation category. The plan is valid, the logical binding is correct, but the infrastructure cannot serve it at this moment. The system must have a defined behavior for this scenario: typically an explicit error with detail about the unavailable resource, allowing the caller to retry at a later time. Handling this as an unstructured exception rather than a structured error is a pattern to avoid.
