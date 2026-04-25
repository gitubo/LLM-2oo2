# Intent Parser

## Purpose

The Intent Parser is the first component in the pipeline and the only one without redundancy. Its job is to transform a natural language input into a structured, unambiguous representation of the user's objective. What it produces is not an action plan — it is a specification: a description of what the user wants to achieve, under which explicit constraints, and with what degree of certainty.

Everything else in the system works from the Intent Parser's output, treating it as ground truth. This position in the pipeline makes it the component with the greatest potential impact on overall correctness: an error here propagates silently through every subsequent stage, with the system's full confidence.

---

## What it produces

The output is a structured document containing at least the following elements.

The intent type, which belongs to a taxonomy defined in advance. This is not a free-form description — it is a classification into a known category, such as `fetch_ordered_subset`, `aggregate_with_grouping`, or `transform_and_filter`. This taxonomy must be built and maintained by domain experts.

The operation parameters, extracted from the input: the entity to operate on, filters, sort criteria, limits, and any other constraint the user has expressed explicitly.

The binding constraints, which are the semantic requirements of the operation that guide the selection of implementations downstream. Typical examples include the need for real-time data, the acceptability or non-acceptability of potentially stale data, and the presence or absence of acceptable side effects.

An `ambiguities` field, which explicitly lists the aspects of the input that could not be determined with sufficient certainty. This field is among the most important in the entire output because it transforms an implicit unknown into an explicit unknown that the system can manage deliberately.

A confidence value, indicating how certain the parser is of its own interpretation. This value is not decorative: below a certain threshold, the system must behave differently — requesting clarification rather than proceeding.

---

## How it can be implemented

There are three main approaches, each with different characteristics.

A grammar-based parser explicitly defines the syntactic structures allowed for each intent type and parses the input against those structures. The result is deterministic and testable, with zero correlation to the LLM models used in later stages. The limitation is rigidity: inputs phrased in unusual ways, or that fall outside the structures the grammar anticipates, produce failures or partial parses.

A dedicated LLM-based parser uses a language model solely to extract the structured intent. It does not plan or reason about execution — it only classifies and extracts parameters. It is robust to diverse phrasings and handles multiple languages and stylistic variations, but it reintroduces non-determinism into the pipeline.

A hybrid approach uses the deterministic parser for common, well-formed intents and the LLM parser for cases the grammar cannot handle. The deterministic parser has priority; the LLM parser intervenes only as a fallback. The confidence value of the result is computed differently in each case, reflecting the different nature of the two approaches.

---

## The problem of genuine ambiguity

Some inputs are ambiguous not because of parser limitations but because of limitations in the input itself. The classic example is a request like "top customers": top by revenue, by order count, by margin, by retention? No parser can resolve this ambiguity without going back to the user.

The `ambiguities` field in the output exists for precisely this reason. When the parser cannot determine an aspect of the intent with sufficient certainty, instead of making a silent choice, it marks that aspect as ambiguous along with its possible interpretations. The system can then decide to request clarification before proceeding.

The confidence threshold below which clarification should be requested is one of the most important design decisions in the entire architecture. A threshold set too high generates too many blocks and degrades the experience. A threshold set too low allows the system to proceed with wrong interpretations with full confidence.

---

## The position in the pipeline and the problem of redundancy

The Intent Parser is upstream of the entire 2oo2 structure. The two parallel channels start after it, both receiving the same structured intent as input. This means that an error by the Intent Parser propagates identically to both channels, which will produce plans consistent with the wrong intent — plans that the comparator will see agreeing.

Replicating the Intent Parser does not fully solve this problem. Two parsers of the same type tend to make errors in the same directions on the same type of input, making the gain from redundancy marginal relative to the cost. There are, however, some strategies that reduce the risk.

One is post-hoc validation: after the intent has been produced, a separate component verifies the coherence between the structured intent and the original natural language input. This is a simpler task than full extraction and can be implemented with a smaller, more specialized model, reducing correlation.

Another is propagating the original input as immutable context throughout the entire pipeline. This way, every component that makes semantic decisions can consult both the structured intent and the original text, and the comparator — in the event of disagreement — has access to both representations.

---

## How it is tested

Testing the Intent Parser cannot be done with automated unit tests alone. It requires building a golden dataset of real examples, each with the expected intent documented by a domain expert.

The golden dataset must cover nominal cases, inputs phrased in unusual ways but semantically clear, genuinely ambiguous cases with the expected response (which is: recognize them as ambiguous), and out-of-domain cases where the parser must signal that it cannot classify the input.

The quality of the parser is measured by its ability to distinguish among these four types — not just on nominal cases. A parser that cannot say "I don't know" is more dangerous than one that fails explicitly.

---

## Advantages of this approach

Separating intent understanding from plan generation is one of the most solid principles of this architecture. It means that the model generating the plan works from a precise, unambiguous specification rather than from raw natural language. This reduces output variability and makes the behavior of the two channels more comparable.

The structured intent representation is also the anchor point for the semantic validator and the logical binding downstream: both can verify the coherence of their decisions against a formal specification rather than against a subjective interpretation of text.

---

## Drawbacks, open questions, and known issues

The fact that the Intent Parser is a single point of failure without redundancy is the architecture's most serious limitation. Mitigating this without making the system excessively complex or costly is an open problem.

The intent taxonomy is a domain artifact that requires active maintenance. When new intent types emerge that the taxonomy does not anticipate, the parser will treat them as ambiguous or misclassify them into the nearest known category. This type of degradation is slow and silent, and is only detected if ambiguity patterns are monitored over time.

Calibrating the confidence value is far from trivial, especially with LLM-based parsers. Language models tend to be poorly calibrated: they express high confidence even on wrong interpretations. Post-calibration techniques exist but add complexity to the system.

The choice of the confidence threshold for requesting clarification interrupts the flow and creates friction in the user experience. In some contexts this is acceptable; in others it is not. The architecture assumes that it is always possible to return to the user for clarification, but this assumption does not hold in all use cases — particularly those where the system is fully automated with no human interaction.
