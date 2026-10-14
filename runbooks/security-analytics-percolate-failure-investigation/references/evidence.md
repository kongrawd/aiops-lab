# Evidence — what this runbook rests on, and what it does not

Every step in `SKILL.md` traces to a replication, an engagement observation, or a reputable source. A step resting on inference is reason to leave a procedure unwritten rather than to label it. This file records which is which, so a reader can judge how far to trust each part.

`metadata.verification` is `documented`: the procedure has not been exercised end to end as written.

## Basis

**Published OpenSearch documentation** for every limit and every route: `indices.query.bool.max_clause_count` with its default and its `too_many_clauses` and `too_many_nested_clauses` rejections, and the `error*` expansion example; `search.query.max_query_string_length`; `max_determinized_states` and `index.max_regex_length`; leading-wildcard cost and the fact that `allow_leading_wildcard` does not apply to regular expressions; percolation's term-extraction and candidate-selection behaviour; the Validate Query route and its documented behaviour of returning the executed Lucene query as a string when `rewrite` is set; the rule search route and the `queries[].value` field; and the alerts route with its `alertState` filter and `ERROR` state.

**Code analysis** of the published implementation for the candidate-query degradation — the clause count is read only to decide whether an optimisation applies, and exceeding it disables that optimisation without raising an error. Read against the development line.

**Public upstream issues** for two things the documentation does not cover: a feature request recording the query-string length limit being reached by a detector in practice, with the maintainer position that auditing and splitting rules is the direction rather than raising limits; and a still-open bug report of NoOp-with-error-status alerts that contains the server-side log lines naming an unresolvable field, including one caused by a **misspelled field name in a shipped rule**.

**Engagement observation** for the error-detail behaviour in *What differs by version*. A vendor root-cause analysis delivered during a support engagement recorded that a detector failure reported only that the percolate search failed, that the real cause — rules expanding past the clause limit — existed only in internal server logs, and that an error-handling step in the alerting layer replaced the detailed cause with a shortened message before it was stored. The same analysis recorded a correction preserving the full reason, and named the remedy as simplifying or splitting the rules in a detector. **This is engagement-recorded, not vendor-documented.**

## Replicated

No step is marked replicated. What has been observed in practice, short of a controlled replication:

- Detectors failing with the generic percolate-search message while the clause limit was the cause, with the detail available only server-side.
- Raising the clause threshold relieving the symptom as an interim measure, with rule optimisation named as the real fix.
- Alert `state` reading `DELETED` on live, enabled detectors — the basis for the warning in step 1.
- A `size=0` findings request returning an empty array beside a non-zero total — the basis for the instruction to read `total_findings`.
- Server-side log lines naming an unresolvable field, one of which was a typo in a shipped rule.

## Not established

- **A public artefact for the error-detail correction.** It shipped as a managed-service patch rather than an upstream change. Searches for a matching issue or pull request naming the clause limit returned nothing in either plugin repository. There is therefore no version boundary to quote, which is why *What differs by version* has you read your own error record rather than compare a version.
- **Whether the clause limit can be raised on your deployment.** A per-index override is documented and cluster-wide change is discouraged; a managed deployment may permit neither. Raising it is in any case a stopgap.
- **Where the clause count that breaches the limit is accumulated.** The percolate match path converts the stored query with a null index reader and runs it against a single-document in-memory index, so the data index's term dictionary is not what drives it. The clause check itself is in the search engine's core rather than the alerting plugin and cannot be traced from the plugin source. An earlier draft of this runbook predicted the breach by validating a rule against the data index; that was removed because it answers a different question and would over-report. The procedure now measures documented, bounded quantities instead.
- **Any threshold at which a rule becomes "too broad".** No published source gives one, and the nearest maintainer position is that it would need benchmarking. This is why step 3 expands each rule against the index instead of applying a rule-of-thumb.
- **Whether the clause limit is the most common cause of NoOp triggers.** At least two NoOp families exist — clause expansion, and rules referencing fields the source index does not carry. Step 2's table discriminates between them rather than assuming one. Earlier guidance that treated NoOp as a single-cause symptom was wrong and misrouted triage.
- **The detector-side code path that stores the error.** The error-handling step that shortens the message is described in engagement notes; it was not confirmed by reading the published implementation, which this review did not cover for the detector-indexing path.

## To raise the level

Exercise the procedure on a non-production cluster: author a rule with a deliberately broad leading wildcard against an index with a large term dictionary, confirm the detector fails, confirm step 3 reports `too_many_clauses` for that rule and compact explanations for the others, then set `verification: replicated` and record the version tested.

One thing that exercise would additionally settle: whether the error record on the tested revision names the cause.
