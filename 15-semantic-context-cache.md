# Semantic Context Cache

## Purpose

The Semantic Context Cache is the component that sits between the Intent Parser and the LLM Prompt Builder. Its job is to enrich the structured intent with a pre-computed mapping of which resources, fields, and operators from the Capability Registry are most relevant for that type of request — avoiding a full registry scan on every call and, over time, incorporating validated evidence about what actually works.

Without this component, the Prompt Builder derives resource relevance from scratch on every request, using only the structured intent and the registry. This is correct but stateless: it does not benefit from the accumulated history of what the system has already seen, validated, and executed successfully.

---

## The problem it solves

In domains with a large registry, determining which resources to include in the LLM context is a non-trivial decision. Including too much degrades plan quality and increases token cost. Including too little makes it impossible to generate the correct plan. The Prompt Builder makes this decision deterministically, but it has no memory: every request is treated as if it were the first.

The Semantic Context Cache introduces memory. It records, for each type of intent and each combination of entities and constraints, which resources and fields have been relevant in the past — and, at higher levels of maturity, which have produced plans that were validated and approved.

---

## The four levels of implementation

The cache can be implemented at four levels of increasing sophistication. Each level is independently useful and builds on the previous one.

### Level 1 — Pre-populated cache on raw user query

The original user query is embedded and indexed. On each new request, the cache retrieves the most semantically similar past queries and returns the resources that were used for those queries.

This is the simplest level to implement and the most suitable as a bootstrap. It can be pre-populated offline from historical query logs before the system goes live, giving the cache useful content from day one. Its limitation is that it reasons on the surface form of the query rather than its logical structure: two queries phrased differently but with the same intent may not retrieve each other.

### Level 2 — Cache on structured intent

Instead of indexing the raw query, this level indexes the output of the Intent Parser: intent type, entities, binding constraints. Similarity is computed on structured representations rather than natural language, which is more precise and more robust to phrasing variation.

Two queries that the Intent Parser has classified identically will always hit the same cache entry, regardless of how they were worded. This level requires the Intent Parser to already be stable, since its output is the index key.

### Level 3 — Cache enriched with execution feedback

Beyond recording which resources were included, this level records which fields and operators from those resources were actually present in the final validated plan. Over time the cache becomes a precision instrument: not just "the `orders` resource is relevant for this intent type" but "for this intent type, the fields `status`, `created_at`, and `amount` are consistently used, and the operators `eq` and `lt` are sufficient."

This allows the Prompt Builder to construct a tighter context, surfacing the most relevant parts of each resource's schema rather than its full definition. The signal source at this level is the Optimizer's output — the canonical plan after all validations — which is a reliable proxy for what the model actually needed.

### Level 4 — Cache enriched with end-user feedback

At this level, the cache incorporates explicit approval signals from the end user: a rating or like on the final response, indicating that the executed plan produced the result they wanted.

This is the most valuable signal in the entire architecture because it is the only one that validates the full chain — intent parsing, plan generation, validation, execution — from the perspective of the person who made the request. An entry in the cache at this level carries a stronger weight: it represents a confirmed mapping between an intent type and a set of resources that produced a correct, user-approved outcome.

The signal must be used carefully. A user approval validates the plan and the execution together. If the plan was correct but the execution introduced an anomaly, the approval may be withheld for reasons unrelated to the planning pipeline. Conversely, a plan that produced a plausible but subtly wrong result may receive approval if the user did not notice the error. The cache should weight this signal highly but not exclusively.

---

## How it integrates with the Prompt Builder

The Semantic Context Cache does not replace the Prompt Builder's resource selection logic — it informs it. The Prompt Builder remains the authoritative component for constructing the prompt; the cache provides a ranked suggestion of which resources and fields to prioritize.

When the cache has a high-confidence entry for the current intent, the Prompt Builder can use it to narrow the context without performing a full registry scan. When the cache has no entry, or the entry has low confidence, the Prompt Builder falls back to its default behavior: full intent-driven selection from the registry.

The cache output is advisory, not binding. The Prompt Builder can override it — for example, if the registry has been updated since the cache entry was written and new resources have become relevant.

---

## Relationship with the feedback loop

The Semantic Context Cache and the feedback loop are deeply connected. The cache is one of the primary beneficiaries of the feedback loop's output: every validated execution, every human review verdict, and every end-user approval is an opportunity to strengthen a cache entry.

The signal separation principle from the feedback loop applies here as well. Level 1 and Level 2 entries can be populated automatically from execution data. Level 3 entries require the plan to have passed full validation before the execution signal is trusted. Level 4 entries require explicit user approval and should go through a lightweight review before being promoted to high-confidence status, to guard against the case where users approve subtly wrong results.

---

## Relationship with the golden dataset

Level 4 cache entries — intent mappings confirmed by user approval — are natural candidates for the golden dataset. Each approved entry includes the original query, the structured intent, and evidence that the resulting resource selection produced a correct outcome. This makes the review process significantly lighter: instead of constructing golden dataset entries from scratch, reviewers can promote cache entries that meet a quality threshold.

---

## Advantages of this approach

The cache reduces token cost and improves prompt precision as the system matures. Early in deployment it adds little value beyond the Prompt Builder's default behavior; over time it becomes an accumulated representation of the domain's query patterns.

The four-level design allows incremental adoption. A team can start with Level 1 pre-population before launch and progressively activate higher levels as the feedback infrastructure matures.

The separation between the cache and the Prompt Builder preserves the deterministic, testable nature of the latter. The cache enriches the input to the Prompt Builder but does not change its internal logic.

---

## Drawbacks, open questions, and known issues

The cache introduces a new failure mode: a stale or incorrect cache entry that causes the Prompt Builder to systematically omit a relevant resource. This is particularly insidious because it is silent — the plan is generated without the missing resource, the validators cannot detect an absence they do not know about, and the error reaches execution. Cache entries must carry a registry version stamp, and entries must be invalidated or re-evaluated whenever the registry changes in ways that affect the indexed resources.

The embedding model used to compute semantic similarity at Levels 1 and 2 is itself a source of potential error. If the embedding space does not capture the domain's relevant distinctions — for example, treating two intents with different binding constraints as similar — the cache will return wrong suggestions. The choice of embedding model and the similarity threshold are design parameters that require domain-specific tuning.

An open question concerns the bootstrapping problem at Level 4. User approval signals accumulate slowly, especially in the early weeks of deployment. A cache that requires approval signals before providing useful suggestions will not be useful when the system most needs it. A pragmatic strategy is to bootstrap Level 4 from expert-curated examples rather than waiting for organic user feedback, and to treat those entries as high-confidence from the start.
