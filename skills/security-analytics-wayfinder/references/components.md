# The components

A Security Analytics detector is not one object. It is a detector, a composite monitor and its delegate monitors, an index of compiled saved queries, a data source, a set of field mappings, and output indexes — each stored separately. **Most detector problems are two of these disagreeing**, which is why a detector can report healthy and produce nothing.

This page carries documented structure and documented behaviour: what each component is, what it owns, and the read that tells you its state. **It does not carry defect signatures.** A defect gets fixed and backported, so a document that routes on one tells a reader to look for something that no longer happens — that belongs in a runbook, which states its own affected versions and carries the fix status. This page gets you to the component; step 4 of `SKILL.md` finds the runbook.

Before interpreting any read here as clean, check it against `blind-checks.md`.

---

## Detector record

**What it is.** The configuration: a name, a detector type, a data source, detection rules, alert triggers, a schedule, and an `enabled` flag.

**What it owns.** Intent. `enabled: false` states that someone asked for it to stop, not that it stopped.

**Read it:**

```
POST /_plugins/_security_analytics/detectors/_search
{"query":{"match_all":{}},"size":100}
```

Page past `size`.

**Two documented requirements decide whether this component can work at all:**

- **A detector needs at least one alert condition before it generates findings.** The creating-detectors documentation states this directly. A detector with no trigger is not broken; it is incomplete by design, and it produces nothing.
- **All data sources on one detector must hold logs of the same log type.** The documentation recommends separate detectors per log type.

**Check which fields your response carries.** On the builds behind this collection the detector API returns `name`, `detector_type`, `enabled`, `schedule`, `inputs`, `last_update_time`, `enabled_time`, `threat_intel_enabled` and `triggers` — and not the identifiers that link it to its monitor and query index. Where those are absent, this read cannot tell a detector linked to a live composite monitor from one that is not, so the link has to be read elsewhere.

**Rule-array shapes.** When counting rules, sum all of these — the API documentation uses different shapes across its own examples, so counting one undercounts:

- `inputs[0].detector_input.rules[]` — the documented response shape
- `inputs[0].detector_input.custom_rules[]` and `...pre_packaged_rules[]`
- `inputs[0].input.custom_rules[]` and `inputs[0].input.pre_packaged_rules[]` — these appear in a documented *request* example rather than a response

---

## Composite monitor

**What it is.** The scheduled job that executes the detector's work. The alerting documentation defines it as a monitor that runs several delegate monitors in sequence as a single execution, and states that **the workflow ID is synonymous with the composite monitor's monitor ID**. The object is the composite monitor; `workflow_id` identifies the sequence of its delegate monitors and equals its monitor ID.

**What it owns.** Execution. This is the record that decides whether anything runs.

**Read it:**

```
GET _plugins/_alerting/workflows/${WORKFLOW_ID}
```

Read `enabled` and `last_update_time`.

**Why this read matters.** The detector record and the composite monitor are two separate documents with independent `enabled` flags. A detector reporting disabled is not evidence that its composite monitor stopped — only this read is. Compare the two, and compare `last_update_time` against your own change record: a timestamp later than your change means something other than your change wrote it, and the writer is unidentified at that point.

**Read the flag back after any write.** A write that returns success is not a change that took effect.

**Delegate alerts are audit alerts.** The documentation states that delegate monitors generate alerts in an audit state, that users are not notified about them, and that they exist to evaluate the composite monitor's chained alert trigger condition. So an absence of user-facing notifications is not evidence that delegates did not run.

---

## The task list

**What it is.** The work in flight right now.

**Read it:**

```
GET _cat/tasks
```

Look for the action strings `alerting/workflow/execute` and `alerting/monitor/doclevel/fanout`.

**What it owns.** Proof of execution, independent of any configuration. Configuration states intent; this states what is happening. Pair it with any `enabled: false` before concluding a detector stopped.

**Scope.** Composite monitors are cluster-scheduled. Per document monitor work is fanned out one assignment per shard, but the assignments are distributed round-robin over a **shuffled list of eligible data nodes** and the node then reads its shard remotely. The percolating node is therefore not the node holding the shard, and it is re-chosen on every execution. A fault that stays on one node is evidence about that node, not about shard placement.

---

## Query index

**What it is.** The index of saved queries that the detection rules compile into, matched by `.opensearch-sap-*-detectors-queries*`. The detector API names it in the `rule_topic_index` field. Rules are stored there as `percolator` fields, and every incoming document is percolated against them.

**What it owns.** Whether the rules are evaluable at all.

**Read it:**

```
GET /_cat/indices/.opensearch-sap-*-detectors-queries*?v
```

Compare the result against the detector inventory. **A detector with no corresponding query index cannot evaluate anything.** `POST /detectors/_search` returns each detector's `rule_topic_index`, so compare that against this listing. `GET /detectors/{id}` omits the field, as do the create and update responses.

**Run it after a service software update, a version upgrade, or a prior detector repair.** A blue/green deployment migrates data to the new environment and switches over with no data loss, so a missing query index after one is itself the anomaly rather than an expected outcome. A missing query index produces a detector that fails on every schedule while its configuration still reads correctly.

**This read also bounds a repair.** Where one detector's query index is missing and the rest are intact, the fix narrows to that detector rather than to every detector of the type.

**The clause limit.** Rules containing wildcards or regular expressions expand into boolean clauses, and a query past the cluster's clause limit is rejected. The limit is a cluster setting, and the Cluster Settings API omits defaults unless you ask for them:

```
GET _cluster/settings?include_defaults=true
```

Read `indices.query.bool.max_clause_count`, default 1024. `include_defaults` returns defaults **from the local node**, which matters on a multi-node domain.

Two things make this harder to confirm than it looks:

- **The setting has only been dynamic since 2.16**, and existing search thread pools can keep using the previous value after a change, so **a later error can echo the old limit even though the raise applied.** Do not read that as the change failing.
- **The percolator degrades rather than failing.** Where a saved query's extracted terms would exceed the limit, it builds a query form that is not clause-limited instead of raising. So a rule past the limit does not reliably produce a clause-limit error.

Raise it one step at a time: a larger boolean query consumes more heap, so a high ceiling on a cluster already under memory pressure trades one failure for another.

**The reader-runnable test for an unextractable rule** is a search against the query index: the percolator field carries an `extraction_result` subfield, and querying it for the `failed` value returns the saved queries whose terms could not be extracted. Those are the rules evaluated against every document rather than pre-filtered.

**A single rule can breach the limit on its own.** Measure pattern breadth per rule, not in aggregate: a leading wildcard is the most expensive form, and the documented guard against leading wildcards in query-string queries does not apply to regular expressions.

**A per-execution rebuild of the query index is a fault signature, not normal cost.** An execution upserts its saved queries by deterministic id, so the index is not recreated in the normal path. The delete-and-recreate sits in an exception handler: where a mapping update on the query index throws, and the monitor's `delete_query_index_in_every_run` field is true, the index is deleted and rebuilt before retrying. Seeing that repeatedly means mapping updates are failing.

**Two properties worth knowing:** the query index carries `index.hidden: "true"`, so it does not appear in a default `_cat/indices` listing, and it is single-shard.

---

## Data source

**What it is.** The indexes, alias, or data stream the detector reads.

**What it owns.** What gets scanned.

**Read it:**

```
GET _cat/aliases/${ALIAS}?v&h=alias,index,is_write_index
GET ${ALIAS}/_mapping
```

**The documented requirement.** The creating-detectors documentation states that an alias used as a detector data source **must be attached to a write index alias**, and that documents must be ingested through the alias rather than through the index the alias was created for. The alias documentation gives the mechanism: an alias pointing at more than one index rejects indexing requests until one of them is designated the write index.

So an alias with several members and no designation is a documented misconfiguration, not a defect. Four states, distinguishable only by reading the flag **and** counting members:

| Rows | `is_write_index` | State |
|---|---|---|
| Two or more | every row `-` | No write index. This is the misconfiguration |
| Exactly one | `-` | A single member behaves as the write target |
| Exactly one | explicit `false` | Also a failure — an explicit `false` overrides the implicit behaviour |
| Any | one row `true` | Designated |

An unset `is_write_index` renders as a literal `-`.

**Where `GET ${ALIAS}/_mapping` returns `404` with `index_not_found_exception`**, the detector points at something that does not exist and every scheduled run fails outright. **Check that the data source resolves before investigating anything else** — it is one call, and a naming mismatch between the configured alias, the alias that exists, and the physical indexes is a detector misconfiguration rather than a platform fault.

**Wildcard patterns are documented as not recommended** for detector data sources. Two documented alias behaviours explain the cost: a wildcard pattern is resolved when the alias is created, so indexes created later are not added automatically; and alias changes are atomic, so an alias never points at an unintended set even momentarily. A pattern gives up both properties, and it changes which index the detector reads mappings from — the newest by creation date rather than the alias write index.

---

## Field mappings

**What it is.** The detector's field mappings, materialised as `"type": "alias"` entries in the underlying index mapping, each with a `path` to a source field. The documentation describes them as what lets the system pass event data from the log to the detector accurately.

**What it owns.** Whether an evaluated rule can match anything. A rule referencing a field the index does not carry cannot match.

**Read it — five calls:**

```
POST /_plugins/_security_analytics/detectors/_search
GET _cat/aliases/${ALIAS}?v&h=alias,index,is_write_index
GET /_plugins/_security_analytics/mappings?index_name=${INDEX}
GET ${INDEX}/_field_caps?fields=*&include_unmapped=true
GET /_component_template/*alias-mappings*
```

The third and fourth run **per concrete index**. `include_unmapped=true` matters: without it an absent field is simply missing from the response, which reads the same as a field that is present and fine.

The fifth reads the component template surface, which the per-index reads do not cover. Confirm it against the `composed_of` of the template that creates the alias's indices.

> ⚠️ **`GET` on the mappings route is the read. `POST` to the same path is a write.** Use the `GET` form here.

**Then compare mappings across every index in scope — because the detector does not.** It resolves its data source to **one** concrete index and reads mappings only from that: for an alias or a data stream, the write index; for an index pattern, the newest index by creation date. It then assumes every other index shares those mappings.

So a type divergence across alias members is never compared and never reported. **A detector can be correctly mapped against its write index and blind on every other member**, and nothing in its own state reflects that. The comparison is yours to make.

**Reading the detector's mappings after a repair proves nothing.** The informative capture is taken while the fault is live.

**Where a detector's field aliases are stored.** Two surfaces hold them, and they are separate objects from the index mapping itself:

- **A plugin-owned component template**, named `.opensearch-sap-alias-mappings-component-<alias>`, listed in the `composed_of` of an index template. Discover the name rather than assuming it — it is plugin-internal.
- **The operator's own index template**, where the same aliases may also be declared.

**An index template is the remedy**, making every index that rolls over carry identical mappings at creation — which is what makes the detector's one-index assumption safe.

**Where an index template and a plugin-generated component template both declare the same field with different types**, the documented rules resolve it: templates listed in `composed_of` are applied in order, and the index template's own `template` block is applied last, so its values win. Where two *index* templates match a name, the highest priority wins outright and they are not merged. Read `GET _index_template` and apply those rules rather than treating the precedence as open.

---

## Rule set

**What it is.** The detection rules attached to the detector.

**What it owns.** Coverage, and most of the evaluation cost.

**Read it:** the rule-array sum from the detector record, then the engine and plugin versions:

```
GET /_cat/plugins?v&h=name,component,version
```

**Read your own build rather than a version written down.** How many rules one evaluation retrieves is version-dependent, and backports land at different times across release lines. Where you cannot establish which behaviour a build has, treat the rule count as unsettled and let a read-only coverage procedure decide.

**Cost is driven by rule content and schedule, not by data volume alone.** A percolated query whose terms cannot be extracted — a wildcard is the common case — is evaluated against every document rather than being pre-filtered, so its cost scales with ingest. Fan-out multiplies it:

```
executions/hour  =  3600 / schedule_interval_seconds
work per execution  ∝  concrete target indexes on the node
```

Lengthening a detector's schedule reduces the number of executions proportionally — but a per document monitor evaluates the documents added since its previous run, so each execution then carries more documents. It trades execution count against per-execution size, and adds detection latency.

A per document monitor evaluates the documents added since its previous run, so an execution that does not complete leaves those documents to be carried into the next one.

**One shard is a finding, not a verdict.**

```
GET _cat/shards/${PATTERN}*?v
```

Only shard `0` present means one fan-out assignment for that detector, so no parallelism across shards. Worth fixing, and **not necessarily the cause** — and note the assignment goes to a shuffled data node rather than the one holding the shard, so this is about parallelism, not placement.

**A fresh duplicate detector is not a sound test of it.** A per document monitor does not evaluate documents that already exist in the index on its first run — it evaluates only those added afterwards — so a new detector can pass its first execution regardless of the fault.

Shard count cannot be changed in place; the route is an index template, which takes effect at the next rollover.

---

## Findings and alerts indexes

**What it is.** The output — `.opensearch-sap-<logtype>-findings-YYYY.MM.DD-NNNNNN` and the corresponding alerts indexes.

**What it owns.** Evidence of work done.

**Read it:**

```
GET _cat/indices/.opensearch-sap-*-findings-*?v&h=index,docs.count,store.size&s=docs.count:desc
```

**This ranks log types, not detectors.** The findings index name carries the log type, so every detector of a given log type writes into the same family. To attribute findings to one detector, query the findings API with its `detector_id`.

There are often several rollover generations per day, so a per-day figure needs the whole date prefix rather than one index.

**Findings are not generated retroactively.** A detector that produced nothing for a week does not produce that week's findings once repaired — the missed detection is the unrecoverable part of every problem on this page.

**Deleting a detector does not delete its findings or alerts** — the delete removes the workflow, the monitors and the detector document. But findings are looked up through the detector's *current* monitor ids, so after a recreate the old ones are no longer reachable through the detector-scoped API even though they still exist in the index.

**Recreating also changes the detector id**, which breaks saved dashboards, scripts and notification wiring keyed to the old one. Changing a detector's input does not reset history: the monitor is updated in place with its id preserved.

---

## Reading state out of system indexes

Several identifiers that link these components together live in plugin-owned indexes rather than in the plugin's API responses. The system-indexes documentation is explicit about the constraint: **system indexes are intended to be accessed through their owning component's API**, direct modification can corrupt component state or break across an upgrade, and the Security plugin adds access controls on top.

Two consequences for reading:

- **A direct `_search` against a plugin-owned index is not a substitute for the plugin API.** Where the Security plugin restricts access, the call can succeed and return nothing, which is indistinguishable from the document being absent. Read `docs.count` on the index first, and treat a non-zero count with an empty result as *this caller cannot assess the component*.
- **The protected set is not fixed.** The documentation lists the registered patterns per component and states that the exact set can change between versions and can depend on plugin settings. Check the current list rather than assuming which plugin indexes are protected on your build.

---

## Where no procedure covers a component

Where step 4 of `SKILL.md` finds nothing, the confirming read above is the whole available result. Hand over the read and its output, and say so plainly. Three conditions reliably land here:

- **Field mappings.** The four-call recipe establishes the fault; the repair is not documented here, and the template-precedence question is open.
- **A data source that does not resolve.** The `404` establishes it; repointing or deleting the detector is a change needing its own approval.
- **A detector with no alert condition.** The documented requirement establishes it; completing or removing the detector is a change needing its own approval.
