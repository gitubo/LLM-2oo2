# LLM Plan Generator

## Purpose

The Plan Generator is the component that produces an action plan from the structured intent. In the 2oo2 architecture, this component is replicated across two parallel, independent channels. The two models receive the same input, work separately, and produce output that is subsequently compared. Their independence is the fundamental prerequisite of the comparison: if the two channels were identical, the comparator would add no value.

The plan produced is abstract: it describes what to do, in what order, and with what dependencies — but not how to do it physically. The translation into concrete, executable instructions happens later in the pipeline.

---

## What it produces

The expected output is a JSON document representing a directed acyclic graph of atomic tasks. Each node in the graph has an identifier, an operation type belonging to a defined vocabulary, a set of dependencies on other nodes, and the parameters specific to the operation.

A conceptual example: a request to retrieve the latest ten confirmed orders from premium customers might produce a graph with four nodes in sequence — a fetch of the `orders` entity, a filter by status and customer category, a sort by creation date descending, and a limit to ten elements. The dependencies between nodes define the execution order.

The plan contains no references to specific implementations, connections, endpoints, or infrastructure resources. That is the responsibility of the binding stage, which comes later.

---

## Choosing the two models

The selection of the two models is not arbitrary. The goal is to minimize error correlation, which requires the two models to have substantial differences in training, architecture, or provider.

Using two instances of the same model with different parameters is not sufficient. Correlation would remain high because both share the same training biases. The optimal choice is typically a model from one provider paired with a model from a different provider, or a generic model paired with a model fine-tuned on the specific domain.

Residual correlation never drops to zero. All modern language models have been trained on overlapping corpora and tend to make errors on the same categories of difficult input. Reducing correlation is an approximation goal, not an elimination goal.

---

## The role of the structured intent

The Plan Generator does not work on the original user text. It receives the structured intent produced by the Intent Parser. This is a deliberate design choice with important consequences.

The model does not need to interpret natural language — that responsibility has already been handled upstream. Instead, it must translate a formal specification into a formal plan. The task is more constrained, the variability space is narrower, and the result is more comparable between the two channels.

The structured intent also includes the binding constraints, which the Plan Generator can use to make decisions about the plan. If the intent specifies that data must be real-time, the model can choose not to include caching tasks that would introduce potentially stale latency.

---

## Parallel generation

The two channels are started in parallel as soon as the Intent Parser has produced its output. There is no primary channel and no secondary channel: both have equal standing until the comparison.

Generation latency is typically the main bottleneck of the entire pipeline. Starting both models in parallel means that overall latency is determined by the slower channel, not by the sum of the two.

---

## Typical error types

Language models generating structured plans tend to fail in characteristic ways that are worth knowing.

Omission is the most insidious: the model generates a plausible, internally coherent plan that is missing a necessary task. A filter that should precede a sort; a lock task that should precede a write. The plan passes all structural checks because it is well-formed, but it is semantically incomplete.

Incorrect ordering occurs when dependencies between tasks are specified in a way that does not reflect the semantic requirements of the operation. The graph may be acyclic and formally valid but topologically wrong with respect to the intent.

Incorrect granularity can go in either direction: the model may collapse operations that should be separate, or expand operations that should be atomic. This creates comparability problems between the two channels even when both are correct.

Hallucinated unnecessary tasks are less common but possible: the model includes tasks that the intent does not require, perhaps because it saw them frequently in similar training examples.

---

## The prompt question

The quality of the Plan Generator's output depends critically on the quality of the prompt. A prompt that precisely describes the vocabulary of permitted tasks, typical dependencies, structural constraints of the plan, and common edge cases produces much more consistent output than a generic prompt.

The prompt should be treated like code: versioned, tested, and revised when requirements change. Prompt changes must be validated against the golden dataset before deployment to production.

A point that often emerges in practice is that the two channels, while using different models, can share the same prompt schema with only model-specific instructions adapted. This simplifies maintenance but introduces a prompt-level correlation that must be accounted for.

---

## Advantages of this approach

Having two independent generators creates a natural diversity of perspective that the comparator can exploit. If one model tends to be conservative — producing simpler plans with fewer tasks — and the other tends to be more elaborate, the disagreement that emerges is often informative: the truth tends to lie somewhere between the two.

Parallel generation, beyond the latency benefit, has a less obvious advantage: if one of the two models produces unrecoverable output (pre-validation failure), the other can still proceed. The system does not fail completely.

---

## Drawbacks, open questions, and known issues

Error correlation, already discussed in the context of model selection, remains the most serious structural limitation. It cannot be eliminated — only managed through careful choices of models and prompts.

Cost is double compared to a single model, both in terms of tokens and latency. In high-volume systems this has non-trivial economic impacts that must be weighed against the validation benefit.

Managing model versions is a significant practical problem. Providers update models periodically, sometimes without meaningful advance notice. An update to the model on one of the two channels can change the comparator's behavior in unpredictable ways. A testing process is needed to verify system behavior every time either model changes version.

An open question not yet fully resolved concerns the case where the two channels produce plans with different granularity but equivalent semantics. The comparator sees a structural disagreement that is not an error — it is a difference in style. The optimizer mitigates this by bringing plans into canonical form, but it cannot eliminate it entirely for all possible variations.
