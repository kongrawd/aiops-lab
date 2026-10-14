---
name: security-analytics-rule-cost-audit
description: >-
  Measure what a Security Analytics detector's rules cost to evaluate, and identify rules whose
  query terms cannot be extracted. A doc-level monitor percolates every incoming document against
  its rules; a saved query whose terms cannot be extracted - a wildcard query, for example - is
  evaluated against every percolated document rather than being pre-filtered, so its cost scales
  with ingest volume. Use this when detector evaluation is driving sustained CPU, when search
  queues back up after enabling detectors, or when auditing a rule set before expanding coverage.
  Rules out two other causes with known fixes before pointing at rule content. Where a detector
  is failing outright rather than running slowly, use the percolate-failure runbook instead.
  Read-only throughout.
metadata:
  agent_types:
    - INCIDENT_TRIAGE
    - INCIDENT_RCA
    - PREVENTION
  skill_type: USER
  service: OpenSearch Security Analytics / Alerting
  applies_to_versions: All lines. Two competing causes are version-dependent and must be excluded first - see "What differs by version".
  risk_level: low
  read_only: true
  automatable: partial
  verification: documented
  last_verified: 2026-10-14
  last_edited: 2026-10-14
  references:
    - title: Percolate query — performance considerations and term extraction
      url: https://docs.opensearch.org/latest/query-dsl/specialized/percolate/
      kind: api-reference
    - title: Wildcard query
      url: https://docs.opensearch.org/latest/query-dsl/term/wildcard/
      kind: api-reference
    - title: Query string query — allow_leading_wildcard
      url: https://docs.opensearch.org/latest/query-dsl/full-text/query-string/
      kind: api-reference
    - title: Security Analytics Rule APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/rule-api/
      kind: api-reference
    - title: Security Analytics Detector APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/detector-api/
      kind: api-reference
    - title: Index Stats API
      url: https://docs.opensearch.org/latest/api-reference/index-apis/stats/
      kind: api-reference
    - title: Unbounded memory growth and CPU degradation from findings accumulation
      url: https://github.com/opensearch-project/alerting/issues/2243
      kind: upstream-change
    - title: Periodic cleanup job to delete doc-level monitor percolate query indices
      url: https://github.com/opensearch-project/alerting/pull/2024
      kind: upstream-change
    - title: Warning system when a detector query becomes too large
      url: https://github.com/opensearch-project/security-analytics/issues/1655
      kind: upstream-change
    - title: System indexes — Security plugin protection and scrubbed results
      url: https://docs.opensearch.org/latest/security/configuration/system-indices/
      kind: product-doc
---

# Runbook — Detector evaluation is expensive; find out what is driving it

## Before you use this

- **Confirm the behaviour against current OpenSearch documentation, or with your support channel, before you execute this.** Plugin behaviour moves with versions, and this runbook states the range it was written for in `metadata.applies_to_versions`. Where this document and the documentation disagree, the documentation is current and this is not.
- **Exercise the procedure in a non-production environment first.** Every step is a read, so the risk is low — but run it once where you know the rule set, so you recognise a normal result.
- **You remain responsible for the outcome in your environment.** This document is offered as-is, with no warranty and no liability for any effect of its use.
- **Send corrections back.** A step that failed, output that differed, a version where the behaviour changed, or a clearer command are worth more than a clean document that is quietly wrong. Open an issue or a pull request on the repository this runbook was shared from.

## When to use this

**Trigger.** Any of:

- Sustained CPU or search-queue pressure that began or worsened after detectors were enabled or rules were added.
- `GET /_plugins/_alerting/stats` reporting `running_on_time: false` for a detector's job — the monitor is falling behind its schedule.
- A rule-set review before expanding coverage, where you want the cost profile before committing.

**Applies when all of these hold.**

| Condition | How to check | Expected |
|---|---|---|
| Security Analytics detectors are enabled | `POST /_plugins/_security_analytics/detectors/_search` | At least one with `enabled: true` |
| You can read the detector's query index name | Same response → `rule_topic_index` | A concrete index name |
| Thread-pool pressure or schedule slippage is observable | `GET /_cat/thread_pool?v`, `GET /_plugins/_alerting/stats` | Rejections, or `running_on_time: false` |

**What differs by version.** Two causes with published fixes must be excluded before rule content is a reasonable suspect. Both are version-dependent; rule cost is not.

| Behaviour | Where the change is absent | Where it is present | How to tell which you have |
|---|---|---|---|
| Per-node findings state across monitor runs | A map accumulates across executions on each node and is copied on every findings-producing run, so CPU and heap degrade progressively over weeks and recover only on node restart | The map is scoped to one run and reset each execution | `GET /_cat/plugins?v&h=name,component,version` → `opensearch-alerting` row, checked against the fix linked in `metadata.references` |
| Orphaned percolate query indexes | Query indexes accumulate as source indexes roll over and are deleted, reportedly to a hundred or more per detector, raising storage and CPU | A periodic cleanup job removes them | `GET /_cat/indices/<rule_topic_index pattern>?v` → count them. The cleanup change was unmerged at the time of writing |

**The symptom shape tells these apart, and that is more durable than any version number.** Degradation that builds over weeks and clears on a node restart is the accumulation defect. A high and growing *count* of query indexes is the sprawl problem. Cost that tracks ingest volume and rule content, steady rather than creeping, is what this runbook measures.

**Does not apply when:**

- **Detectors are disabled.** There is nothing being evaluated.
- **The pressure predates detectors.** Establish that detector evaluation is implicated before auditing rules; otherwise this audit will find expensive rules that were never the cause, because broad rules are common and usually harmless at low volume.
- **You are looking for missing detections rather than cost.** A different situation — see **Related runbooks**.

## Values to obtain

| Name | What it is | How to obtain the current value |
|---|---|---|
| `${DETECTOR_ID}` | The detector under review | `POST /_plugins/_security_analytics/detectors/_search` → `hits.hits[]._id` |
| `${QUERY_INDEX}` | The percolator query index backing the detector's log type | Same response → `rule_topic_index` |
| `${LOG_TYPE}` | The detector's log type, for the rule search | Same response → the detector's log-type field |
| `${MONITOR_ID}` | The detector's monitor, for per-monitor filtering | **Not returned by the detector API** — read `monitor_id` from the plugin's detector configuration document, or omit the per-monitor filter and use the whole-index figure |

`${QUERY_INDEX}` changes when the plugin rolls the query index over, so read it rather than reusing a previous value.

## Impact

**Mechanism.** Percolation works by pre-filtering. When a saved query is indexed, its terms are extracted and indexed alongside it; at evaluation time the incoming document goes into a temporary in-memory index and those extracted terms select a small set of candidate queries, so most saved queries are never run. The documentation states the exception plainly: terms cannot be extracted from some query types, including `wildcard`, and **a saved query whose terms cannot be extracted is evaluated against every percolated document, which slows percolation as the number of such queries grows.**

That is the whole mechanism, and it is multiplicative rather than additive: a doc-level monitor evaluates per document, so each extraction-failed rule costs once per document ingested, not once per run.

Two documented facts make broad rules more likely than they look. Leading wildcards are the most expensive form, because matching them requires examining every term in the index — and the `allow_leading_wildcard` guard does not apply to regular expressions. And the vendor's own published example of a translated pre-packaged rule carries two leading wildcards, because Sigma `contains` modifiers compile to them. So extraction failure is an ordinary consequence of ordinary rules, not an exotic misconfiguration.

**Breadth has a second, harder edge.** Past the engine's clause limit a query is rejected outright rather than running slowly, so the detector fails instead of costing more. That is a different situation with a different instrument — see `security-analytics-percolate-failure-investigation`.

**What the documentation does not establish is magnitude** for the per-document cost. It gives direction only. Whether extraction-failed rules are the dominant cost in your cluster has to be measured here, which is what step 4 does.

**Size it for this system.** The measurement is the proportion of saved queries that cannot be pre-filtered:

```
GET /${QUERY_INDEX}/_count
{"query":{"term":{"query.extraction_result":"failed"}}}

GET /${QUERY_INDEX}/_count
```

The ratio of the first to the second is the share of rules evaluated against every document. Pair it with the throughput they are multiplied by:

```
GET /${QUERY_INDEX}/_stats/search
```

Sample `query_total` and `query_time_in_millis` twice and take the difference.

**Blast radius.** Cost lands on the search path of every data node holding shards of the detector's source indexes, because evaluation fans out per shard. Documented consequences of the pressure are increased search latency and queue rejections, which affect user-facing search on the same cluster — not only detection.

**Recoverability.** Fully recoverable. Nothing here is a data-loss path; the cost is ongoing rather than accumulated, so reducing rule breadth or volume takes effect immediately.

**Severity.** Urgent where search rejections are affecting user-facing traffic or where a detector is falling behind its schedule, since a monitor that cannot keep up is a detection gap as well as a cost problem. Routine for a review.

## Before you start

- [ ] Permission to read the Security Analytics detector and rule APIs, the CAT routes, `GET /_plugins/_alerting/stats`, and the index stats and count APIs
- [ ] Permission to read `${QUERY_INDEX}` directly. **This is the one that may silently fail** — see the warning below
- [ ] Two sampling points a few minutes apart, since the cost measurements are deltas

> ⚠️ **An empty result from the query index is not evidence of absence.** The published documentation for system-index protection states that for a caller without access the Security plugin can substitute an empty index reader, so **a search returns `200 OK` with zero hits even though matching documents exist**, and that an empty result must not be read as proof of access. Whether this particular index is protected is itself unsettled publicly: the plugin's registered system-index descriptors do not appear to include it, while the Security Analytics documentation states broadly that its indexes are protected as system indexes. Both statements are public and they disagree. **Step 3 therefore checks that you can read the index at all before step 4 interprets a zero.** A zero you have not validated is the most dangerous output in this procedure, because it reads as a clean bill of health.

## Procedure

### Step 1 — Confirm detector evaluation is actually implicated

```
GET /_cat/thread_pool?v
GET /_plugins/_alerting/stats
```

**Expect:** from the first, `active`, `queue` and `rejected` per pool. From the second, `schedule_status`, and per job `last_execution_time` and `running_on_time`.

**If there are no rejections and every job reports `running_on_time: true`:** detector evaluation is keeping up. The pressure you are chasing is elsewhere. **Stop here.**

**If rejections are present or a job is behind schedule:** continue. Note that neither route attributes cost to a specific detector — `GET /_plugins/_alerting/stats` carries no CPU, latency or duration field at all. They establish *that* there is pressure, not *what* caused it.

### Step 2 — Exclude the two causes that have fixes

Do this before looking at rules. Both have published remedies, and auditing rule content while one of these is in play sends you to rewrite rules when an upgrade or a cleanup was the answer.

```
GET /_cat/plugins?v&h=name,component,version
GET /_cat/indices/<the ${QUERY_INDEX} name pattern>?v&h=index,docs.count,store.size
```

**Branch.**

- **Degradation built up over weeks and a node restart relieves it** → the per-node accumulation defect. Check the alerting plugin version against the fix in `metadata.references`; it is merged and backported across many lines. **Stop here** and plan the upgrade.
- **A large and growing number of query indexes** → the sprawl problem. The cleanup change was unmerged at the time of writing, so treat this as an escalation rather than something you resolve here, and report the count.
- **Neither** → continue to step 3.

### Step 3 — Confirm you can read the query index

```
GET /_cat/indices/${QUERY_INDEX}?expand_wildcards=all&v&h=index,docs.count
GET /${QUERY_INDEX}/_count
```

**Expect:** a non-zero `docs.count` from the first, and a matching non-zero `count` from the second.

**If `docs.count` is non-zero but `_count` returns zero:** you are being served a scrubbed empty reader. Do not proceed to step 4 — its zero would be meaningless. Go to **Reach out** and ask for a read with the required access.

**If both are non-zero and consistent:** continue to step 4.

### Step 4 — Count the rules that cannot be pre-filtered

```
GET /${QUERY_INDEX}/_count
{"query":{"term":{"query.extraction_result":"failed"}}}
```

**Expect:** a count, which against step 3's total gives the share of saved queries evaluated against every document.

To narrow to one detector's monitor:

```
GET /${QUERY_INDEX}/_search
{"size":0,"query":{"bool":{"filter":[
  {"term":{"query.extraction_result":"failed"}},
  {"term":{"monitor_id":"${MONITOR_ID}"}}
]}}}
```

The field names here come from the plugin's own query-index mapping and field mapper rather than from the documentation site — the documentation names the subfield to search for but not this index or its mapping. The `monitor_id` filter in particular is source-derived; if it returns nothing while the unfiltered count is non-zero, fall back to the whole-index figure rather than concluding the detector is clean.

### Step 5 — Identify which rules they are

```
POST /_plugins/_security_analytics/rules/_search?pre_packaged=true
{"from":0,"size":100,"query":{"nested":{"path":"rule",
  "query":{"bool":{"must":[{"match":{"rule.category":"${LOG_TYPE}"}}]}}}}}
```

Repeat with `?pre_packaged=false` for custom rules, and page with `from` and `size` — a single published category example reports well over a thousand rules, so one request will not cover a set.

**Expect:** per hit, `_source.queries[].value` carrying the translated query text, and `_source.rule` carrying the original Sigma definition as a single string.

> ⚠️ **Rule metadata is nested, not flat.** `queries`, `tags`, `references`, `false_positives` and `query_field_names` are each **arrays of objects carrying a `value` key** — not arrays of strings. A parser written against flat string arrays fails silently rather than erroring, which is the worst way for an audit to be wrong. Technique tags, for instance, read as `tags[].value`.

Scan `queries[].value` for a `*` immediately following a `:` or a `\`, and for `/…/` regular-expression delimiters. There is **no documented server-side filter** for this — no parameter selects rules by query content or by rule id — so the scan is client-side on the paged results.

### Step 6 — Measure what the cost actually is

```
GET /${QUERY_INDEX}/_stats/search
```

**Expect:** `query_total` and `query_time_in_millis`. Sample twice, a few minutes apart, and take the difference to get rate and mean latency.

**This figure is unattributed until something confirms it belongs to the detector.** The documentation does not state that percolate cost surfaces in this index's search statistics. Report it as cost observed on the query index, not as the detector's cost.

The comparison that would attribute it — the same two samples with the detector enabled, then disabled — **requires disabling a detector, which is a write and is therefore out of scope for this runbook.** Two ways to get the attribution without breaking that boundary: take the samples across a window in which the detector was already disabled for an unrelated reason, or run the enable-and-disable comparison deliberately under change control, using `security-analytics-detector-disable-verification` to confirm the disable actually took effect before you trust the quiet sample.

## Validation

This procedure produces a judgement, so validation confirms the judgement is sound.

1. **The zero was validated.** If step 4 returned zero, confirm step 3 showed a consistent non-zero total. An unvalidated zero is not a result.
2. **Pagination was complete.** Compare requested `size` against returned hits for every rule-search page. A page at the requested size means more remain.
3. **The cost figure is labelled for what it is.** Either an attributing comparison exists from a window where the detector was already off, or the figure is reported as cost observed on the query index rather than as this detector's cost. Do not disable a detector to obtain it inside this procedure.
4. **The two competing causes were excluded,** with the plugin version and the query-index count recorded, not merely assumed.

**Exit criteria.** You can state: the extraction-failed count against the total, which rules account for it, the measured search rate and latency on the query index with the attribution comparison, the alerting plugin version, and the query-index count. Anything short of that supports an impression but not a change to a rule set.

## If this does not work

**Reach out when:** step 3 shows you cannot read the query index; or the query-index count is high and growing, since the cleanup change is unmerged; or evaluation is falling behind schedule after the two version-dependent causes are excluded.

**Reach out to:** your OpenSearch support channel, through the route your organisation uses for a production cluster.

**Attach:** the thread-pool and alerting-stats output, the plugin version, the query-index count, the extraction-failed count against the total with the step 3 validation that makes the zero meaningful, the search-stats deltas with whatever attribution basis you have, and the rule identifiers whose translated queries carry leading wildcards or regular expressions.

> ⚠️ **Do not set `search.allow_expensive_queries` to `false` as a mitigation.** The percolate documentation states that percolate queries are themselves considered expensive and are **rejected** when that setting is `false`. Setting it would disable every doc-level monitor in the cluster — that is, every detector. It is a hazard to know about, not a tuning knob.

## Related runbooks

Siblings are referred to by name, which resolves to a directory and is greppable.

- `security-analytics-detector-health-audit` — the read-only sweep that establishes which detectors need attention before you audit one in depth.
- `security-analytics-rule-coverage-audit` — where the concern is rules not producing findings rather than rules costing too much. The two have opposite symptoms and a shared enumeration.
- `security-analytics-percolate-failure-investigation` — where the detector is failing rather than slow, because a rule expands past the clause limit. Same rule material, opposite symptom.
- `security-analytics-detector-disable-verification` — where a detector was disabled to shed this load and you need to confirm it actually stopped.

## Verification status

Every step traces to a reputable source. A step resting on inference is reason to leave a procedure unwritten rather than to label it. The basis, what has been observed in practice, the open questions and any superseded claims are in `references/evidence.md` — read it before relying on this runbook for the first time, or before quoting it to anyone.
