# Layer 5 — Rule hygiene

Layers 1 to 3 decide whether a rule *can* match. This layer is about whether the rules you have are worth evaluating and affordable to evaluate.

## The recommended state

| Property | Recommended |
|---|---|
| Every enabled rule's fields | Present on the index, as a field alias with a declared target — layer 3 |
| New rules | **The field and its alias are added to the template before the rule is enabled** |
| Pattern breadth | No leading wildcard where a keyword or match condition would do |
| Clause expansion | No single rule expanding past the cluster's boolean clause limit |
| Field types | `keyword` where rules match exact terms — layer 1 |
| Detectors | No near-duplicates carrying substantially the same rule set |

## The sequencing rule that prevents the common case

**Add the field and its field alias to the index template before enabling the rule that needs it.** Enabling first produces a rule that is silently skipped until the next rollover after the template is corrected — and because the skip is silent, nobody learns it happened.

This is the single highest-value habit in this layer, and it is stated directly in the reference setup.

## The reads

```
GET /_cluster/settings?include_defaults=true
GET /_cat/plugins?v&h=name,component,version
```

**Expect:** from the first, `indices.query.bool.max_clause_count` — **`include_defaults=true` is required**, the API omits defaults without it, and it returns them from the local node. From the second, the plugin versions.

For the rules themselves, this layer **does not** attempt to map field names back to rules. That needs the rule set read per rule and compared against the index, which is `security-analytics-detector-mapping-audit`'s job. What this layer checks is the *shape* of the rule inventory and whether the sequencing habit is in place.

> ⚠️ **Do not infer coverage from a count.** A rule listed on a detector may have failed to index into the saved-query index, in which case it is absent at evaluation while the detector still lists it. A rule that fails to *compile* is different — that is rejected at rule creation. Neither is visible from a rule count.

## Why type matters as much as presence

A rule field can be present and still never match. Packaged rules match strings with **term-level** semantics — exact equality, and the `endswith` / `startswith` / `contains` modifiers — and those work against a `keyword` field. An analysed `text` field is tokenised at index time, so an exact term query has no matching token to hit. **The result is zero findings, silently**, which is indistinguishable from a rule that correctly matched nothing.

This is why layer 1 pins types explicitly rather than letting dynamic mapping decide, and why a field arriving as `text` is a layer 1 finding rather than a rule problem.

## Cost, where it comes from

Cost is driven by rule content and schedule, not by data volume alone:

```
executions/hour  =  3600 / schedule_interval_seconds
```

A per-document monitor percolates every incoming document against its rules. Query terms are extracted and indexed alongside each saved query and used to select candidates, so **most saved queries are never evaluated against a given document**. A query whose terms **cannot be extracted** — a leading wildcard being the common case — is the exception: it is evaluated against every percolated document rather than being pre-filtered, so its cost scales with ingest.

**That distinction is the whole of rule-cost triage**, and it is why a wildcard audit is useful for tuning and misleading as a diagnosis: broad rules are common and usually harmless at low volume.

**A single rule can breach the clause limit on its own.** Measure pattern breadth per rule rather than in aggregate. Note also that the percolator *degrades* rather than failing where extracted terms would exceed the limit — it builds a form that is not clause-limited — so a rule past the limit does not reliably produce a clause-limit error.

## What non-conformance costs

| Finding | Cost | Route |
|---|---|---|
| A rule enabled before its field and alias were added | Skipped silently until the next rollover after the template is corrected | Layer 1 and layer 3 |
| Rule fields absent from the index | Those rules never match; no error is reported | `security-analytics-detector-mapping-audit` |
| A field typed `text` where rules match exact terms | Zero findings, silently | Layer 1 |
| Leading-wildcard rules at high ingest | Evaluated against every document; cost scales with volume | `security-analytics-rule-cost-audit` |
| A rule past the clause limit | Rejected outright, or silently degraded to an unbounded form | `security-analytics-percolate-failure-investigation` where error-state alerts exist, otherwise `security-analytics-rule-cost-audit` |
| Near-duplicate detectors | Multiplied evaluation cost for no added coverage | `security-analytics-rule-cost-audit`; consolidation is an operator decision |

## Sources

- **Published managed-service documentation** for the statement that Security Analytics **skips rules whose fields are missing from the index without reporting an error** — the basis for treating absence as silent rather than as a failure.
- **Published documentation** for term-level versus analysed-field matching, for `indices.query.bool.max_clause_count` and `include_defaults` returning defaults from the local node, and for percolator performance considerations.
- **Carried from `security-analytics-rule-cost-audit`, where it is sourced in full**, for the extraction model: a per-document monitor percolates every incoming document against its rules, query terms are extracted and indexed alongside each saved query and used to select candidates so most saved queries are never evaluated, and a query whose terms cannot be extracted is evaluated against every percolated document rather than being pre-filtered.
- **Carried from `security-analytics-rule-cost-audit/references/rule-patterns.md`** for the `text`-instead-of-`keyword` failure and for a rule failing to index into the saved-query index while the detector still lists it.
- **A reference setup produced during a support engagement** for the sequencing rule — add the field and its alias before enabling the rule. **Engagement-sourced.**

**Not established.** Whether a given rule's compiled form breaches the clause limit, from the rule text alone. The compiled form has to be read, because a rule containing no wildcard character can still compile to one — an `endswith` modifier produces a leading wildcard. `rule-patterns.md` carries that and the per-field clause arithmetic.
