# Executor

## Purpose

The Executor is the component that receives the executable plan produced by Physical Binding and runs it. Its role in this documentation is intentionally limited: by the time the plan reaches the Executor, every decision has already been made. The Executor does not plan, does not interpret, does not choose. It walks the task graph in dependency order and executes each node mechanically.

This is a deliberate design constraint. Keeping the Executor decision-free is what makes the rest of the pipeline meaningful: if the Executor could override or reinterpret the plan, the guarantees provided by validation and comparison would be weakened.

---

## What it does

The Executor traverses the DAG in topological order, dispatching each node to the appropriate handler based on its type. Contextual nodes become HTTP calls, with the endpoint, method, and parameters already resolved by Physical Binding. Transversal nodes that were not collapsed by the Optimizer — filters, sorts, limits — are applied client-side on the data returned by the preceding contextual node.

Execution is deterministic: given the same executable plan and the same data, the Executor always produces the same result. There is no inference, no fallback logic, no interpretation of ambiguous cases.

---

## The only exception: bounded retry logic

The one area where the Executor exercises limited autonomy is transient failure handling. If an HTTP call times out or returns a retriable error code, the Executor can retry within a pre-configured policy — a fixed number of attempts, with backoff. This is mechanical policy execution, not decision-making: the retry parameters are part of the plan's configuration, not inferred at runtime.

If a node fails beyond the retry limit, the Executor surfaces the error with enough context to correlate it with the plan and the trace — it does not attempt to recover by substituting an alternative resource or reinterpreting the task. Recovery decisions belong to the orchestration layer above.

---

## What it is not

The Executor is not an agent. It does not evaluate whether the plan is achieving the intended goal mid-execution. It does not detect semantic anomalies in intermediate results and adjust course. It does not have access to the original user intent.

Any intelligence applied to the output — detecting that a result set is unexpectedly empty, deciding whether to re-plan, routing the result to the right consumer — lives outside the Executor, in the orchestration layer that called the pipeline in the first place.
