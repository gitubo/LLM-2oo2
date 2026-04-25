# Pre-Validator (First-Level Syntactic Validation)

## Purpose

The Pre-Validator is the first component to encounter the raw output of the Plan Generator. Its sole job is to determine whether the incoming material is sufficiently structured to be processed by subsequent components — in particular, the Sanitizer.

It does not evaluate the correctness of the plan. It does not check values, verify types, or apply domain rules. It answers a single question: is there enough structure here to work with?

The principle is fail fast: it is better to immediately discard unrecoverable output than to pass it to the Sanitizer, where it would fail anyway after consuming resources and producing less informative error messages.

---

## What it checks

The Pre-Validator applies a minimal set of checks that reflect the fundamental structural requirements of the expected format.

First, it verifies that the output is valid, parseable JSON. If the model has produced free text, markdown with code blocks, partial JSON, or truncated JSON, the pre-validator detects this immediately and discards the material.

Next, it verifies the presence of the mandatory macro-fields defined in advance: typically a `metadata` field containing information about the plan itself, and a `plan` field containing the list of tasks. It does not check the contents of these fields — only their existence.

It verifies that the `plan` field is an array and that it contains at least one element. An empty plan is not a plan.

This is the complete list of checks. Any additional verification belongs to the Full Schema Validator.

---

## Behavior on failure

If the Pre-Validator fails, the plan is considered unrecoverable and a retry of generation is immediately triggered on the same channel. Nothing is passed to the Sanitizer.

The retry uses the same prompt, unless the implementation includes a mechanism for prompt variation on retry (different temperature, additional instructions about the expected structure). The choice depends on the most likely cause of failure: if failures are rare and random, a simple retry is sufficient; if they are systematic, a different prompt strategy is needed.

There is a maximum number of retries beyond which the channel is considered failed and the system behaves accordingly — which may mean proceeding with a single channel or escalating to the operator.

---

## Practical examples

A Plan Generator might produce output like the following when something goes wrong:

    Here is the plan you requested:
    
    1. Retrieve orders
    2. Filter by confirmed status
    3. Sort by date

This is free text, not JSON. The Pre-Validator discards it immediately without attempting to interpret it.

Another typical case is partial JSON due to output truncated by a token limit:

    {"metadata": {"version": "1.0"}, "plan": [{"id": "t1", "task_type": "fetch"

This fails JSON parsing. The Pre-Validator discards it.

A less obvious case is valid JSON but without the expected fields:

    {"result": "ok", "tasks": [...]}

This is parseable JSON, but the `metadata` and `plan` fields are absent. The Pre-Validator discards it because the Sanitizer has no anchor to work from.

---

## Logging

The Pre-Validator emits a structured event for every execution, regardless of outcome. The event includes the `trace_id`, the channel, the outcome, and in the event of failure the error category (not JSON, missing field, empty plan, etc.).

These logs have significant diagnostic value. A high failure rate at the Pre-Validator is a signal that the model is systematically producing malformed output, which may indicate a problem in the prompt, a model update that changed behavior, or an input type the model was not adequately trained to handle.

---

## Advantages of this approach

Simplicity is the main advantage. A component with few responsibilities is easy to test, understand, and maintain. There are no gray areas: either the output passes the checks or it is discarded.

The separation from the Sanitizer is architecturally correct. The Sanitizer works on material that already has a basic structure, and can therefore make reasonable assumptions about what to correct. If the Sanitizer also had to handle completely malformed output, it would become more complex and less predictable.

The speed of failing fast reduces latency in error cases: instead of passing unrecoverable material through all pipeline stages only to see it fail at the last one, the problem is identified at the first stage.

---

## Drawbacks, open questions, and known issues

The Pre-Validator introduces a dependency on the macro-field format that must be kept aligned with the rest of the system. If the schema evolves and the names of the fundamental fields change, the Pre-Validator must be updated at the same time. This is not a serious technical problem, but it is a misalignment risk in environments with frequent deployments.

The boundary between "unrecoverable" and "recoverable by the Sanitizer" is not always sharp. There are gray areas where material has some structure but not enough to make automatic correction safe. The decision of where to draw this line is subjective and depends on the risk tolerance of the specific use case.

An open question concerns the handling of the case where both channels fail the Pre-Validator simultaneously. The probability is low if the models are different, but it is not zero. The system must have a defined behavior for this case, which is typically immediate escalation with a clear message to the operator.
