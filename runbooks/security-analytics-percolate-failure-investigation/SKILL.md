---
name: security-analytics-percolate-failure-investigation
description: >-
  Find why a Security Analytics detector's evaluation fails rather than runs slowly. A detector
  converts its rules into a percolate query; broad wildcard or regular-expression rules expand
  into query clauses, and past the engine's clause limit the search is rejected outright, so the
  evaluation produces nothing. The stored error is often generic, naming only that the percolate
  search failed. Use this when a detector reports a percolate-search failure, emits error-state
  alerts, produces NoOp triggers, or stops producing findings after rules were added. Read-only
  throughout - it takes the cause from the error, measures the detector's rule count and compiled
  query length against the documented limit, and leaves the decision to simplify or split with you.
metadata:
  agent_types:
    - INCIDENT_TRIAGE
    - INCIDENT_RCA
  skill_type: USER
  service: OpenSearch Security Analytics / Alerting
  applies_to_versions: All lines. What a failed detector's error record tells you is version-dependent - see "What differs by version".
  risk_level: low
  read_only: true
  automatable: partial
  verification: documented
  last_verified: 2026-10-14
  last_edited: 2026-10-14
  references:
    - title: Rewrite parameter — clause expansion and the max_clause_count limit
      url: https://docs.opensearch.org/latest/query-dsl/rewrite-parameter/
      kind: api-reference
    - title: Percolate query — term extraction and performance
      url: https://docs.opensearch.org/latest/query-dsl/specialized/percolate/
      kind: api-reference
    - title: Regexp query — max_determinized_states and index.max_regex_length
      url: https://docs.opensearch.org/latest/query-dsl/term/regexp/
      kind: api-reference
    - title: Search settings
      url: https://docs.opensearch.org/latest/install-and-configure/configuring-opensearch/search-settings/
      kind: product-doc
    - title: Security Analytics Rule APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/rule-api/
      kind: api-reference
    - title: Security Analytics Alert and Finding APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/alert-finding-api/
      kind: api-reference
    - title: Warning system when a detector query becomes too large
      url: https://github.com/opensearch-project/security-analytics/issues/1655
      kind: upstream-change
    - title: Alerts created from NoOp trigger with Error status
      url: https://github.com/opensearch-project/security-analytics/issues/1476
      kind: upstream-change
---

# Runbook — A detector's evaluation fails, not just slows

## Before you use this

- **Confirm the behaviour against current OpenSearch documentation, or with your support channel, before you execute this.** Plugin behaviour moves with versions, and this runbook states the range it was written for in `metadata.applies_to_versions`. Where this document and the documentation disagree, the documentation is current and this is not.
- **Exercise the procedure in a non-production environment first.** Every step is a read, so the risk is low — but run it once where you know the rule set, so you recognise a normal result.
- **You remain responsible for the outcome in your environment.** This document is offered as-is, with no warranty and no liability for any effect of its use.
- **Send corrections back.** A step that failed, output that differed, a version where the behaviour changed, or a clearer command are worth more than a clean document that is quietly wrong. Open an issue or a pull request on the repository this runbook was shared from.

## When to use this

**Trigger.** Any of:

- A detector's error record or an error-state alert reports a failure to run the percolate search.
- A detector emits **NoOp triggers** — it ran, produced nothing, and raised no useful reason.
- A detector stopped producing findings shortly after rules were added or broadened.

**Applies when all of these hold.**

| Condition | How to check | Expected |
|---|---|---|
| The detector is enabled | `GET /_plugins/_security_analytics/detectors/${DETECTOR_ID}` → `enabled` | `true` |
| It is failing rather than merely quiet | `GET /_plugins/_security_analytics/alerts?detector_id=${DETECTOR_ID}&alertState=ERROR&size=20` | At least one error-state alert |
| Its data source is still receiving documents | `GET /<data index or alias>/_count` | A rising count across two samples |

**What differs by version.** What the stored error tells you is the thing that changes.

| Behaviour | Where the correction is absent | Where it is present | How to tell which you have |
|---|---|---|---|
| Detail in a failed detector's error record | The record carries a generic failure — a percolate search failed, all shards failed — while the real cause exists only in server-side logs. An error-handling step in the alerting layer replaces the detailed cause with a shortened message before it is stored | The error names the cause, for example a query exceeding the clause limit, so you can act without escalating | Read the error record for a failed detector and see whether it names a cause |

**Read your own error record rather than a version number.** This correction shipped as a managed-service patch rather than an upstream change, so there is no public issue to match a version against. Step 2 branches on what the record actually says.

**Does not apply when:**

- **The detector is producing findings normally but slowly, or driving load without failing.** That is the per-document cost problem, not a rejection — use `security-analytics-rule-cost-audit`.
- **Findings stopped for the whole detector and it reports disabled.** Use `security-analytics-detector-disable-verification`.
- **Findings arrive for some rules and never for others, with no error.** That is silent truncation, not rejection — use `security-analytics-rule-coverage-audit`.
- **The detector's data source is an alias with no write index.** Nothing is being read at all. Use `security-analytics-alias-write-index-recovery`.

## Values to obtain

| Name | What it is | How to obtain the current value |
|---|---|---|
| `${DETECTOR_ID}` | The failing detector | `POST /_plugins/_security_analytics/detectors/_search` → `hits.hits[]._id` |
| `${DATA_INDEX}` | The index or alias the detector reads, for the ingest check in *Applies when* | `GET /_plugins/_security_analytics/detectors/${DETECTOR_ID}` → `inputs[0].detector_input.indices[0]` |
| `${LOG_TYPE}` | The detector's log type, for the rule search | Same response → the detector's log-type field |

## Impact

**Mechanism.** A rule is not evaluated as written — it is compiled to a query, and a wildcard or regular expression **expands** against the index's term dictionary. The documentation's own example is `error*` expanding to hundreds or thousands of term queries. `indices.query.bool.max_clause_count`, default **1024**, rejects a query that expands past it with `too_many_clauses` as HTTP 400, or `too_many_nested_clauses` where the total across a query and its children breaches it.

For a detector this is not slowness. The percolate search is rejected, so the evaluation produces **nothing** — and because the stored error may name only the search failure, the detector can look merely quiet.

**Where that expansion resolves is not established**, which bounds what this runbook can claim. The stored rule query is converted in the percolate path with a null index reader and executed against an in-memory index holding a single document, so it is not the data index's term dictionary that drives it. The aggregate size of a detector's compiled rule set is measurable and is bounded by a documented limit; that is what step 3 measures, rather than predicting a clause count.

Full treatment of the limits, the post-rewrite counting change, and why absence of the error does not mean absence of clause pressure: `references/mechanism.md`.

**Size it for this system.** The magnitude is how many rules are at or past the limit on this index, and how long the detector has been failing:

```
GET /_plugins/_security_analytics/alerts?detector_id=${DETECTOR_ID}&alertState=ERROR&size=20
GET /_plugins/_security_analytics/findings/_search?detector_id=${DETECTOR_ID}&sortOrder=desc&size=1
```

The oldest error-state alert bounds the start of the gap. The newest finding's `timestamp`, if it predates that, is the last time this detector produced anything. **Read `total_findings`, not the length of the `findings` array** — a `size=0` request returns an empty array beside a non-zero total.

**Blast radius.** Every rule on a failing detector, not only the offending one — the search is rejected as a whole, so a single over-broad rule silences the detector. Where several detectors share a log type, check each one.

**Recoverability.** Nothing is recoverable. A doc-level monitor advances through documents as they arrive rather than re-scanning a past window, so documents that passed through while the search was failing are not examined after a fix.

**Severity.** Urgent whenever confirmed. The detector reports healthy configuration, every rule still lists as enabled, and the only outward sign is an absence — so nothing else will escalate this for you.

## Before you start

- [ ] Permission to read the Security Analytics detector, rule, alert and findings APIs
- [ ] Access to wherever the cluster's server-side logs are delivered, in case step 2 needs them
- [ ] The detector's rule list, so you can tell a rule you added from one that shipped with the product

## Procedure

### Step 1 — Confirm the detector is failing, not idle

```
GET /_plugins/_security_analytics/alerts?detector_id=${DETECTOR_ID}&alertState=ERROR&size=20
```

**Expect:** `alerts[]` with `state`, `error_message`, `start_time` and `trigger_name`, plus `total_alerts`.

**If there are no error-state alerts:** the detector is not failing in the way this runbook addresses. Check whether it is producing findings at all and route from *Does not apply when*. **Stop here.**

**If error-state alerts exist:** record the oldest `start_time` and continue.

> ⚠️ Alert `state` is not detector state. Alerts reading `DELETED` have been observed against live, enabled detectors, so do not read alert state as the detector's lifecycle.

### Step 2 — Get the cause, from the record or from the log

Read `error_message` on the alerts from step 1.

**Branch.**

- **It names a cause** — a clause limit, a parse failure, a named field → you have what you need. Continue to step 3 if it concerns clause expansion; go to **Reach out** with the message otherwise.
- **It names only that the percolate search failed** → your engine revision predates the error-detail correction. The cause exists in the **server-side logs**. Search your log destination for entries from the doc-level monitor query component around the oldest `start_time`.

What to look for in the log, and what each means:

| Log content | What it means | Where to go |
|---|---|---|
| `too_many_clauses`, or a message naming the clause limit | Rule expansion exceeded the limit | Step 3 |
| `No field mapping can be found for the field with name [<field>]` | A rule references a field the index does not carry, so the rule could not be indexed | Step 4 |
| Neither, or no entries | The cause is not established here | **Reach out** |

**If you cannot reach the logs:** say so rather than guessing. An unexplained percolate failure is something to reach out about, and step 3 only helps for the clause-expansion branch.

### Step 3 — Measure the two quantities that are documented and bounded

```
POST /_plugins/_security_analytics/rules/_search?pre_packaged=false
{"from":0,"size":100,"query":{"nested":{"path":"rule",
  "query":{"bool":{"must":[{"match":{"rule.category":"${LOG_TYPE}"}}]}}}}}
```

**Expect:** per hit, `_source.queries[].value` carrying the translated query text. Repeat with `?pre_packaged=true` and page with `from` and `size`. Rule metadata is **nested** — `queries`, `tags` and `references` are arrays of objects carrying a `value` key, not arrays of strings, and a parser written against flat string arrays fails silently.

Then measure, per detector:

1. **Rule count** — the entries across the detector's rule arrays. Sum `custom_rules`, `pre_packaged_rules` and `rules`, under both `detector_input` and `input`, because the published examples use three different shapes.
2. **Total compiled query length** — the summed length of the `queries[].value` strings. `search.query.max_query_string_length` defaults to **32000**, and a public report records a detector breaching it with the resulting parse error naming the actual length.
3. **Pattern breadth** — count the rules whose `queries[].value` carries a `*` immediately after a `:` or `\`, or `/…/` regular-expression delimiters. Leading wildcards are documented as the most expensive form, and the guard against them does not apply to regular expressions.

**Expect:** three figures per detector. There is no published threshold for any of them except the query-string length, so report them beside that one documented limit and against the detector's own history, not against a number from this document.

**Branch.**

- **Total query length at or near 32000** → this is a measurable, documented breach. It is the finding. Continue to step 4.
- **A high rule count with many broad patterns, and the step 2 cause named the clause limit** → the aggregate query is the suspect. Continue to step 4.
- **Low counts, short queries, and step 2 named the clause limit anyway** → the cause is not something this procedure can localise. Go to **Reach out** with the step 2 evidence and these three figures.

> ⚠️ **Do not try to predict the breach by validating a rule against the data index.** It answers a different question. In the percolate path the stored rule query is converted with an index reader of `null` and executed against an in-memory index holding **one document**, so its term expansion has nothing to do with the data index's term dictionary. Validating against the data index over-estimates expansion by a wide margin and will flag rules that never breach. Where the expansion that breaches the limit actually resolves is not established — see `references/mechanism.md`.

### Step 4 — Record what has to change, and stop

This runbook identifies; it does not edit rules, because which rule to narrow and how is a detection-coverage decision rather than an operational one.

Record, per offending rule: its identifier, its `queries[].value`, whether it validated, and whether it is a rule you authored or one that shipped with the product. The documented remedy for a clause breach is to **simplify the rule or split the detector's rules across more detectors** — raising the limit is a stopgap and is documented as discouraged.

Note that a field named in step 2's second branch can be a **typo in a shipped rule** rather than a gap in your mapping; published rule sets have carried misspelled field names. Compare the spelling against the source index mapping before changing your template.

## Validation

This procedure produces a finding, so validation confirms the finding is sound.

1. **The failure is current, not historical.** Re-run step 1 and confirm error-state alerts are still arriving. An old alert with a recovered detector is a different situation.
2. **The figures are per detector, not per log type.** Several detectors can share a log type, so confirm the rule set you measured belongs to the failing detector.
3. **Pagination was complete.** Compare requested `size` against returned hits on every rule-search page; a page at the requested size means more remain.
4. **A low set of figures was not mistaken for a clean result.** Low counts with the clause limit named in step 2 is a real finding — that the cause is not localised here — and belongs in the escalation, not in a conclusion that nothing is wrong.

**Exit criteria.** You can state: the detector is failing, what the error record said, what the log said if the record was generic, the detector's rule count, total compiled query length against the documented limit, and how many rules carry broad patterns. Anything less supports reaching out but not a rule change.

## If this does not work

**Reach out when:** the error record is generic and you cannot reach the server-side logs; or the log names neither a clause limit nor a field; or the step 3 figures are low and the detector still fails; or the cause is a shipped rule you cannot edit.

**Reach out to:** your OpenSearch support channel, through the route your organisation uses for a production cluster.

**Attach:** the error-state alerts with their `error_message` and oldest `start_time`, the relevant server-log entries, the three figures from step 3 with the detector they belong to, the newest finding timestamp, and the identifiers of the rules carrying the broadest patterns. State whether the offending rule is one you authored.

> ⚠️ **Do not raise the clause limit as a first move.** A per-index override exists and cluster-wide change is documented as discouraged; a managed deployment may permit neither. It also hides the problem rather than fixing it — the documented remedy is to simplify or split the rules.

## Related runbooks

Siblings are referred to by name, which resolves to a directory and is greppable.

- `security-analytics-rule-cost-audit` — where the detector works but evaluation is expensive. Same rule material, opposite symptom.
- `security-analytics-rule-coverage-audit` — where findings arrive for some rules and never for others, with no error.
- `security-analytics-detector-health-audit` — the read-only sweep that establishes which detectors need attention before you investigate one.
- `security-analytics-detector-disable-verification` — where the detector reports disabled and the question is whether it stopped.

## Verification status

Every step traces to a reputable source. A step resting on inference is reason to leave a procedure unwritten rather than to label it. The basis, what has been observed in practice, and the open questions are in `references/evidence.md`.
