# Comparator

## Purpose

The Comparator is the convergence point of the 2oo2 architecture. It receives the two canonical plans enriched with logical binding produced by the two parallel channels and determines whether they agree. If they agree, the system has reached sufficient confidence to proceed. If they diverge, the system must decide how to handle the disagreement.

The Comparator's fundamental premise is that disagreement between two independent channels is an informative signal — not necessarily an error. It indicates that the problem was difficult or ambiguous enough to produce different responses in two independent elaborations.

---

## What the Comparator measures

The Comparator does not measure correctness. It measures consistency between the two channels. This distinction is important and frequently misunderstood.

A highly consistent system is not necessarily a correct one: two channels can agree on a wrong plan. A system with low consistency is not necessarily one that makes errors: it may simply have two channels with very different generation styles that produce different representations of equivalent plans — representations that the Optimizer should have normalized.

Consistency is a proxy for correctness, not a direct measure of it. It is a useful proxy because, in the presence of sufficiently independent channels, high consistency is positively correlated with correctness. But the link is not direct, and this must be kept in mind when interpreting metrics.

---

## Levels of comparison

Comparison happens across multiple levels with increasing strictness. The choice of level depends on the domain and the tolerance for false alarms.

**Structural comparison** verifies that the two plans have the same number of nodes and the same macro-structure. It is the coarsest comparison and the least prone to false disagreements.

**Topological comparison** verifies that the dependency graph is identical: the same nodes, in the same relationships, in the same relative order. Two plans with the same tasks but different dependencies are different plans even if they have the same macro-structure.

**Parameter comparison** verifies that the parameters of each corresponding node are identical. This is the most sensitive level to false disagreements caused by generation style variations.

**Logical binding comparison** verifies that the implementation category chosen for each node is identical. A disagreement here may indicate that the binding constraints of the two plans were different, which is itself a signal of inconsistency in generation.

---

## Categorizing the disagreement

When the Comparator detects a disagreement, it does not merely flag it — it categorizes it. Categorization is essential for deciding how to handle it and for analyzing patterns over time.

A **structural disagreement** (different number of nodes) is typically the most serious: the two models produced plans of different complexity, which suggests substantially different interpretations of the intent.

A **topological disagreement** (same nodes but different dependencies) may indicate an omission in one of the two plans, or it may be a borderline case where both orderings are valid but the Optimizer failed to normalize them. Distinguishing these cases requires analysis.

A **parameter disagreement** may be noise (small phrasing differences) or may be substantive (a filter with a different operator, a different numeric limit). Analyzing the specific diff is necessary.

A **logical binding disagreement** is often a signal of inconsistency in the binding constraints produced by the two models, and warrants specific investigation.

---

## Behavior in case of disagreement

The Comparator does not make autonomous decisions when it detects disagreement. It produces a structured event containing the disagreement type, a detailed diff, and the categorization — and the orchestration system decides the consequent behavior based on configured rules.

Typical behaviors in case of disagreement include: retrying both channels (if the disagreement may be due to random variability), retrying only the channel with more Sanitizer corrections (potentially the less reliable one), escalating to human review, or — in the case of structurally different plans — refusing the entire operation.

There is no universally correct answer: the disagreement handling policy depends on the domain, the criticality of the operation, and the availability of a human operator.

---

## False alarms and system availability

A false alarm is the case where the Comparator detects a disagreement between semantically equivalent plans that the Optimizer failed to normalize. This produces an unnecessary block that degrades system availability without corresponding to a real problem.

The false alarm rate is the primary determinant of system availability: if the Comparator frequently generates disagreements on plans that would have been executed correctly, the system is unusable in production even if it technically works.

Reducing the false alarm rate is the Optimizer's responsibility — it must ensure that equivalent variants produce the same canonical plan. But the Optimizer has limits: it cannot capture all possible equivalences, and some generation style differences are not normalizable without risking lossy transformations.

Monitoring the false alarm rate is therefore critical to system health. An increase in the rate suggests that the models have changed their generation style (a version update) or that new input patterns have emerged that the Optimizer does not cover.

---

## Practical example

Channel A produces: `[fetch(orders), filter(status=confirmed), filter(amount>100), sort(created_at DESC), limit(10)]`. Channel B produces: `[fetch(orders), filter(amount>100 AND status=confirmed), sort(created_at DESC), limit(10)]`.

Channel A's Optimizer normalized the two filters but did not collapse them because the collapse rule was not present. Channel B's Optimizer received a plan that the model had already produced with the filters collapsed.

The Comparator sees a structural disagreement: plan A has 5 nodes, plan B has 4. It categorizes the disagreement as structural with the note "difference in number of filter nodes." Analysis reveals this to be equivalent variants, and the case is used to add the collapse rule to the Optimizer.

---

## Advantages of this approach

The Comparator transforms disagreement into structured information. It is not just a gate that blocks — it is a diagnostic tool that over time reveals where the system has the most variability and where normalization rules are incomplete.

The fact that comparison happens on already-validated canonical plans means the Comparator can perform simple, deterministic comparisons without needing to reason about heterogeneous structures or apply subjective judgment.

---

## Drawbacks, open questions, and known issues

The Comparator measures consistency, not correctness. This structural limitation has no solution within the 2oo2 pattern — it is intrinsic to it. The defense is the Semantic Validator upstream, which catches semantic inconsistencies before they reach the Comparator.

The disagreement handling policy is configurable but not universal. Defining the correct response to each type of disagreement requires domain knowledge and understanding of how the system is used, and changes over time as experience accumulates.

An open question concerns cases where one of the two channels does not reach the Comparator because it was blocked by a failure in its own pipeline. At that point the system has only one validated plan. Should it proceed with one plan or block entirely? The answer depends on the confidence level in a single channel and the criticality of the operation, and must be decided explicitly during system design.
