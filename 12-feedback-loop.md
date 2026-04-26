# Feedback Loop

## Purpose and premise

The feedback loop is the mechanism through which the system uses data produced in production to improve itself over time. Without it, the system would be static: its capabilities would remain those of the deployment moment, regardless of how much information it accumulates about real-world usage.

The fundamental premise is that a poorly designed feedback loop is more dangerous than no feedback loop at all. A system that learns from its own errors without verifying that it is learning the right thing amplifies its biases instead of correcting them. Designing the feedback loop means designing not just the learning mechanism but above all the constraints that prevent error amplification.

---

## The two types of feedback loop and why they are not equivalent

An **improvement loop** uses production data to identify where the system performs poorly and to guide changes that improve that situation. It requires reliable data, review processes, and measurements of the effect of changes.

A **degradation loop** occurs when the system uses its own output as the source of truth for improving itself. If errors are not correctly labeled, the system learns to produce the same errors with greater frequency. This is a concrete risk in LLM-based systems, where the boundary between correct output and plausible-but-wrong output is not always automatically visible.

The operational distinction between the two is the role of human verification: an improvement loop requires that at least a portion of the feedback signal be verified by human beings with domain knowledge. A degradation loop typically bypasses this verification for scalability reasons, and pays the price in quality.

---

## Signal sources

Not all signal sources have the same value for the feedback loop. Ordering them by reliability is important for understanding where to invest.

**Execution ground truth** is the most reliable signal: the plan was executed and the result was correct or wrong. This signal is binary, unambiguous, and directly corresponds to the quality being measured. The problem is that it requires knowing whether the result was correct — which is not always automatically verifiable.

**Explicit feedback from a human operator** who has seen the output and can say whether it was appropriate is highly reliable but rare and not scalable. It is valuable for building the golden dataset but cannot be the sole signal source for continuous improvement.

**Comparator disagreement** is a weak signal: it signals that there was a problem but does not say which channel was right. It should not be used as a direct quality signal but as an indicator that a case warrants human review.

**Sanitizer corrections** are an indirect signal: they do not say whether the final plan was correct, but they say that the model produced output that deviated from the expected schema in a specific way. This is useful for identifying systematic patterns in model behavior.

**Intent Parser confidence** is a predictive signal: it does not say whether the output was wrong, but it says how likely it was to be wrong. Useful for prioritizing cases to review, not for feeding improvement directly.

**End-user feedback** — a rating or explicit approval on the final response — is the most direct signal available at scale. It validates the full chain from intent parsing through execution from the perspective of the person who made the request. It is more scalable than operator review and more direct than execution ground truth, but it carries a specific risk: users may approve plausible-but-subtly-wrong results, or withhold approval for reasons unrelated to plan correctness (slow execution, UI issues). This signal should be weighted heavily but not exclusively, and a lightweight review process should guard against systematic false positives before it is used to update system logic. The Semantic Context Cache is the primary consumer of this signal.

---

## The three feedback loop rhythms

The feedback loop does not operate at a single speed. Different components of the system evolve on different time scales, and attempting to update everything at the same speed leads to instability or stagnation.

**Real-time rhythm**, on the order of seconds, is appropriate for operational thresholds: the Comparator agreement threshold, the maximum number of retries, timeouts. These parameters can be adjusted automatically in response to anomalous conditions without requiring human review, because they are immediately reversible and have limited effects.

**Periodic rhythm**, on the order of days, is appropriate for model prompts, Sanitizer rules, and Semantic Validator checklists. These updates require analysis of aggregated patterns, a human decision, and regression testing on the golden dataset before deployment. They are not urgent but they are important.

**Long rhythm**, on the order of weeks or months, is appropriate for changes to the intent taxonomy, the JSON schema, the LLM models, and Optimizer rules. These changes have cascading impacts across many components and require a formal coordination process, extensive testing, and planned deployment.

---

## The golden dataset

The golden dataset is the system's most important asset. It is a collection of examples verified by domain experts, each containing the original input, the expected structured intent, and the properties of the correct plan. It grows over time with each case reviewed through the human review queue.

It has three distinct and equally important uses. As a basis for regression testing: every change to the system is verified against the golden dataset to ensure it has not degraded known cases. As a benchmark for comparing different model versions: when a model is updated, the golden dataset makes it possible to objectively measure whether behavior improved or worsened. As a clean training signal: if a model is to be fine-tuned, the golden dataset is the only source of verified data available.

The quality of the golden dataset is more important than its size. One hundred carefully selected and verified examples are more useful than one thousand automatically generated and unverified examples.

Level 4 entries from the Semantic Context Cache — intent mappings confirmed by end-user approval — are natural candidates for the golden dataset. Each such entry includes the original query, the structured intent, and evidence of a correct, approved outcome, making the review process significantly lighter than constructing entries from scratch.

---

## The human review queue

Cases requiring human review are inserted into a structured queue. Each case in the queue includes the `trace_id`, the reason for review, the original input, the two plans produced by the channels, the Comparator diff, and a specific question for the reviewer to answer.

The specific question matters: the reviewer is not asked to re-read everything from scratch and independently figure out what happened. They are presented with already-structured context and a precise question such as "is plan A or plan B more coherent with this intent?" This reduces cognitive load and increases response consistency.

Each review produces three outputs: a verdict on the specific case that resolves the escalation, a label for the golden dataset, and an optional note on observed patterns that might suggest system changes.

---

## Feedback contamination

The main risk of the feedback loop is contamination: using unverified data that may be wrong as a quality signal.

The **signal separation principle** establishes which sources may feed which types of updates. Human-verified signal (execution ground truth, explicit feedback, human review queue review, end-user approval after lightweight review) may feed updates to prompts, schemas, rules, and the Semantic Context Cache at its higher confidence levels. Unverified automatic signal (Comparator disagreement, low Intent Parser confidence, raw end-user approval without review) may only open a case in the human review queue, populate the cache at lower confidence levels, or adjust operational thresholds — never directly update the system's logic.

This principle reduces the feedback loop's scalability but guarantees its correctness. It is a deliberate trade-off.

---

## Silent degradation and how to detect it

The most dangerous form of system degradation is not sudden failure but slow, gradual deterioration. Aggregate metrics appear stable because the degrading cases are a minority — but that minority grows over time.

Typical causes of silent degradation include: input distribution drift (users start using the system in unanticipated ways), unannounced LLM model updates by providers, and the aging of domain rules in response to business changes.

The canary system described in the observability document is the primary mechanism for detecting this type of degradation. But analyzing the distribution of cases in the human review queue over time is also useful: if new categories of cases emerge that were not previously being escalated, this signals that something in the domain has changed.

---

## Open questions

The hardest point of the feedback loop is obtaining reliable ground truth at scale. Manually verifying the outcome of every executed plan is impossible. Verifying a statistically representative sample is necessary but requires resources and processes. Automating verification is only possible for categories of operations where an oracle exists — a reference system that can say whether the result was correct — and this oracle does not always exist.

Managing the feedback loop in regulated environments where every system change requires a formal approval process is an open problem. The natural rhythms of the feedback loop — especially the periodic one — may be incompatible with approval processes that take weeks. Finding a balance between the speed of improvement and regulatory requirements is a challenge specific to certain domains and has no universal solution.

A final point concerns managing the feedback loop when the system is new and the golden dataset is small. In the first weeks of production, almost all data is new and unverified. The feedback loop must be very conservative during this phase, risking missing opportunities for rapid improvement. Finding the right balance between caution and learning speed in the early stages is a decision that depends heavily on the domain and risk tolerance.
