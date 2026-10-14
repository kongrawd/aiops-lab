# Mechanism — clause expansion, the limits, and why absence of an error proves nothing

Read this when you need to explain the finding to someone else, or when the procedure's branches do not fit what you are seeing. The procedure in `SKILL.md` does not depend on anything here.

## The limits that reject a query

All four are documented, and all four are version-stable enough to write down because they are product defaults rather than observations.

| Limit | Default | What happens when it is breached |
|---|---|---|
| `indices.query.bool.max_clause_count` | 1024 | `too_many_clauses`, HTTP 400. Where the total across a query and its children breaches it, `too_many_nested_clauses` |
| `search.query.max_query_string_length` | 32000 | A parse error, raised before the query runs |
| `max_determinized_states` (regexp) | 10000 | Rejection. Documented as preventing high resource consumption |
| `index.max_regex_length` | 1000 characters | Rejection |

A public feature request in the Security Analytics tracker reports the query-string length limit being hit by a detector in practice, with the maintainer position that a detector "can potentially be configured with thousands of rules" and that large queries harm cluster performance. That issue also records the recommended direction: audit rules to make them more succinct, and create more detectors with fewer rules each. It explicitly discourages raising `max_clause_count` or `max_query_string_length` as the fix.

## Two properties that change how you reason about the clause limit

**The count is taken after rewrite, and the limit applies to every query on the index.** The constraint moved from the boolean query up to the index searcher. The practical consequence stated in the upstream issue: choosing a `rewrite` method no longer avoids the limit, because the clause count *after* rewrite is what is measured. Queries that passed on an older line can breach it after an upgrade, with no change to the rule. The value is also static and node-scoped, which is why changing it cluster-wide is documented as discouraged, and why a per-index override is the documented route where an override is permitted at all.

**The percolator's own candidate query degrades rather than errors.** Code analysis of the published implementation shows the clause count being read only to decide whether an optimisation can apply — the result is a boolean that disables the optimisation when the candidate query would be too large. No exception is raised. So:

- Clause pressure on the **candidate-selection** side costs performance silently.
- `too_many_clauses` comes from the **rule's own query** being rewritten in the match path.

**Therefore absence of the error does not mean absence of clause pressure.** A detector can be paying for breadth without ever failing, which is the situation `security-analytics-rule-cost-audit` measures.

## Where the expansion resolves — not established

This is the open question, and it bounds what the procedure can claim.

What the published implementation shows, read against the development line: in the percolate match path the stored rule query is converted with a query-shard context whose `getIndexReader()` returns `null`, with the in-source comment that the reader which matters is the in-memory index rather than the shard's, and execution is against `MemoryIndex.fromDocument(...)` built from **one** document. So the match path has no shard term dictionary to expand against.

What that rules out: a procedure that predicts the breach by expanding a rule against the **data** index. That answers a different question and would over-report, because a large index's term dictionary is not what the match path sees.

What it does **not** settle: where the clause count that breaches the limit is actually accumulated. The clause check itself lives in the search engine's core rather than in the alerting plugin, so it cannot be traced from the plugin source alone. Three candidates remain open — the parse of a rule's own compiled query, which can carry many clauses by its own structure regardless of any index; the construction of the candidate query over extracted terms, which is bounded but degrades rather than errors; and the aggregate of a detector's whole rule set.

**Consequence for the runbook:** it measures quantities that are documented and bounded — rule count, total compiled query length against the documented maximum, and how many rules carry patterns documented as expensive — and takes the cause itself from the error. It does not predict a clause count.

## Why broad rules are ordinary rather than exotic

Sigma `contains` modifiers compile to leading wildcards. The vendor's own published example of a translated pre-packaged rule carries two of them. So extraction failure and clause expansion are consequences of ordinary rule authoring, not of misconfiguration — which is why the remedy is to split a rule set across detectors rather than to find the one person who wrote a bad rule.

## The related cost mechanism, for contrast

Percolation pre-filters: when a saved query is indexed its terms are extracted and indexed alongside it, and at evaluation time those terms select a small candidate set, so most saved queries are never run. Terms cannot be extracted from some query types including `wildcard`, and the documentation states that such a saved query is evaluated against every percolated document, which slows percolation as the number of such queries grows.

That is a *cost*, not a rejection, and it is measured by counting `extraction_result: failed` in the query index. It belongs to `security-analytics-rule-cost-audit`. The two problems share a cause — rule breadth — and have different symptoms, different instruments and different urgency.

## Sources

- Rewrite parameter documentation — clause expansion, the `error*` worked example, `too_many_clauses` and `too_many_nested_clauses`, and the discouragement of cluster-wide change.
- Percolate query documentation — term extraction, candidate selection, and the statement that a query whose terms cannot be extracted is evaluated against every percolated document.
- Wildcard, query-string and regexp query documentation — leading-wildcard cost, `allow_leading_wildcard` and its non-application to regular expressions, `max_determinized_states`, `index.max_regex_length`.
- Code analysis of the published percolate query builder for the null index reader and the single-document in-memory index, read against the development line.
- Search settings documentation — `search.query.max_query_string_length`.
- An upstream issue in the OpenSearch core tracker describing the move of the clause constraint to the index searcher, post-rewrite counting, and the static node-scoped nature of the value.
- A public Security Analytics feature request reporting the query-string length limit in practice, with the maintainer's recommended direction.
- Code analysis of the published percolator field mapper for the candidate-query degradation, read against the development line.
