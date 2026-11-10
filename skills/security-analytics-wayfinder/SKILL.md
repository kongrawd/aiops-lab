---
# Latest version:
#   SKILL_BASE_URL=https://github.com/kongrawd/aiops-lab/tree/main npx skill skills/security-analytics-wayfinder
# Re-run that to pick up changes rather than editing a local copy in place.
name: security-analytics-wayfinder
description: >-
  Work out what is wrong with an OpenSearch Security Analytics detector, starting from one read
  that almost always has the answer in it — the detector's error-state alerts. Maps the error
  message to the component that produced it, gives the read that confirms that component, then
  finds whatever procedure currently covers it. Use this when a detector produces no findings,
  when NoOp-trigger or error-state alerts appear, when findings continue after a detector was
  disabled, when a detector update or enable fails, when field mappings disappear after an index
  rollover, when detector evaluation is driving CPU or search rejections, or after a blue/green deployment,
  upgrade or any prior remediation on detectors. Also use it to decide which signals to monitor so
  a silently broken detector is caught before someone notices. It identifies the component and the
  confirming read; it diagnoses nothing and changes nothing, and is read-only throughout.
metadata:
  agent_types:
    - INCIDENT_TRIAGE
    - PREVENTION
    - CHAT
  skill_type: USER
  service: OpenSearch Security Analytics / Alerting
  applies_to_versions: All lines. Behaviours that differ by version say so where they appear, with the read that settles it.
  risk_level: low
  read_only: true
  automatable: yes
  verification: documented
  last_verified: 2026-11-10
  last_edited: 2026-11-10
  references:
    - title: Security Analytics Alert and Finding APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/alert-finding-api/
      kind: api-reference
    - title: Security Analytics Detector APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/detector-api/
      kind: api-reference
    - title: Composite monitors — the workflow ID and delegate monitors
      url: https://docs.opensearch.org/latest/observing-your-data/alerting/composite-monitors/
      kind: product-doc
    - title: Composite Monitor APIs — the workflow route and the enabled field
      url: https://docs.opensearch.org/latest/observing-your-data/alerting/composite-monitors-api/
      kind: api-reference
    - title: Security Analytics mappings API
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/mappings-api/
      kind: api-reference
    - title: CAT aliases API — the is_write_index column
      url: https://docs.opensearch.org/latest/api-reference/cat/cat-aliases/
      kind: api-reference
    - title: Cluster settings for indexes — indices.query.bool.max_clause_count
      url: https://docs.opensearch.org/latest/install-and-configure/configuring-opensearch/cluster-settings-for-indexes/
      kind: product-doc
    - title: Monitoring OpenSearch cluster metrics with Amazon CloudWatch
      url: https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-cloudwatchmetrics.html
      kind: product-doc
---

# Security Analytics wayfinder

**Signal → component → confirming read → whatever procedure exists now.**

This skill carries product facts: which component produces which behaviour, and which read proves it. Those stay true as the runbook collection grows and gets rewritten. **It carries no list of documents** — step 4 searches the collection when you need it, so a renamed or newly added runbook is found without editing anything here.

**The component and its confirming read are the deliverable.** A document name is a convenience. If step 4 finds nothing you still have a complete result: the component, the read, and evidence to hand over.

## Step 1 — Read the error alerts first

Most of the time the answer is already in here, and this is the cheapest read available.

```
GET _plugins/_security_analytics/alerts?alertState=ERROR&sortString=start_time&sortOrder=desc
```

Three things about this call, each of which causes a wrong conclusion when missed:

- **Omit `detectorType` to sweep the whole cluster.** It is optional, and leaving it out returns the error alerts of every detector visible to you in one search. Supply it only to narrow to one log type.
- **Read `total_alerts` at the response root, not the length of the `alerts` array.** The array is page-bounded at `size`, default 20. An array *shorter* than the page size is trustworthy. **`total_alerts` on the all-detectors path is tracked only up to 10,000**, so exactly `10000` is a floor rather than a count.
- **Parameter casing is inconsistent.** `detectorType` is camelCase; the single-detector form uses `detector_id` in snake_case.

Then read these fields per alert:

| Field | Reads |
|---|---|
| `trigger_name` | `"NoOp trigger"` is the literal marker for *detector execution failed*, as distinct from any trigger someone named |
| `end_time` | `null` means still firing. A non-null value means it stopped. A point-in-time query returns resolved history beside live failures |
| `version` | A cumulative failure counter — roughly one increment per failed execution. This is your magnitude, and it survives after `alert_history` has rolled |
| `error_message` | The component. Step 2 |
| `finding_ids`, `related_doc_ids` | Always `[]` on an error alert. This is not "nothing matched" — it is "the scan did not complete, so nothing could match" |
| `severity` | Always `""` on an error alert. An empty string, not a level |
| `alert_history` | A log of **distinct** error messages, capped at 10 — appended to only when the message changes. A detector failing the same way 500 times has one entry; `version` carries the count |

**The cleanest discriminator in this whole collection:** no findings **and no error alert** means genuinely nothing matched. No findings **with** a NoOp error alert means you were not looking.

## Step 2 — Error message to component

Read what the message **names**. That is the component, regardless of the exception class wrapped around it.

| The message names | Component | Go to |
|---|---|---|
| A saved-query index — matched by `.opensearch-sap-*-detectors-queries*` — as not found | **Query index** | Step 3, query index |
| The detector's own data source as not found | **Data source** | Step 3, data source |
| A percolate search failure, with a document count | **Rule set** against the clause limit. The document count is the measurement — read it across consecutive alerts | Step 3, rule set |
| A clause limit, or the limit's current value | **Rule set** | Step 3, rule set |
| A node, an address and an action name, with the cause further down the chain | **Read the whole chain** | See below |

**`error_message` carries the full nested cause chain.** It is built by walking every cause and joining them with `; nested: `, so the outermost frame is often a transport wrapper and the component that failed is named further in. Read to the end of the string before concluding the alert has no cause. A redacted address in the message is the service redacting it, not you.

**Where the alerts are empty** — no error alerts and still no findings — the error path cannot help. Go to step 3 and work through detector record, composite monitor, data source and field mappings in that order.

## Step 3 — Confirm the component

Start by enumerating detectors, which also gives you the detector types step 1 needs:

```
POST /_plugins/_security_analytics/detectors/_search
{"query":{"match_all":{}},"size":100}
```

`size` is a hard page bound — page past it.

> **Use `_search`, not `GET /detectors/<id>`.** The single-detector read returns a hand-built subset — `name`, `detector_type`, `enabled`, `schedule`, `inputs`, `last_update_time`, `enabled_time`, `triggers` — and omits `monitor_id`, `workflow_ids` and `rule_topic_index`. `_search` returns the raw `_source`, which carries them. A detector whose composite monitor or saved-query index is gone looks healthy through the single-detector read and has no field to inspect.

| Component | Confirming read |
|---|---|
| **Detector** | The enumeration above — `enabled`, `triggers`, `inputs`, the rule arrays, plus `monitor_id`, `workflow_ids` and `rule_topic_index` from `_source`. A detector with no alert condition generates no findings by documented design |
| **Composite monitor** | `GET _plugins/_alerting/workflows/${WORKFLOW_ID}` → `enabled`, `last_update_time`. Two separate documents with independent flags; compare them |
| **Executing when it should not be** | `GET _cat/tasks` → `alerting/workflow/execute`, `alerting/monitor/doclevel/fanout`. Configuration states intent; the task list states execution |
| **Query index** | `GET /_cat/indices/.opensearch-sap-*-detectors-queries*?v` compared against the detector inventory. A detector with no query index evaluates nothing |
| **Data source** | `GET _cat/aliases/${ALIAS}?v&h=alias,index,is_write_index` then `GET ${ALIAS}/_mapping`. An alias data source must be attached to a write index; a `404` means the detector is misconfigured rather than the platform at fault |
| **Field mappings** | `GET ${INDEX}/_mapping` for **every** concrete index in scope, compared against each other. The fault is two indexes declaring one field with different types |
| **Rule set** | `GET _cluster/settings?include_defaults=true` → `indices.query.bool.max_clause_count`. **`include_defaults=true` is required**; the API omits defaults without it |

`.opensearch-sap-detectors-config` is a registered system index. Reach detector state through the plugin API rather than searching that index directly — see `components.md`, *Reading state out of system indexes*.

### Two reads that look decisive and are not

**One shard is a finding, not a verdict.** `GET _cat/shards/${PATTERN}*?v` showing only shard `0` means one node does all the percolation for that detector — true, worth fixing, and not necessarily the cause, because a clause-limit breach is independent of shard count. **The discriminator: stop the other detectors, create one duplicate detector, and see whether it fails on its first execution.** If it does, shard count is not your problem. The other detectors have to be stopped, or they compete for the same CPU and the test proves nothing.

**A healthy-looking data source proves nothing about the detector.** A detector binds its field aliases when it is saved, and nothing re-binds them on a schedule — so correcting an alias, or rolling over to a new index, leaves the detector still reading the mappings it bound earlier. Re-save the detector so it re-reads the field aliases from the current write index. Until then the affected rules are skipped with no error.

## Step 4 — Find whatever procedure covers it now

Search the collection by component vocabulary — these are the words a runbook on that component would use, not the names of documents:

```bash
grep -rl "${COMPONENT_TERM}" runbooks/*/SKILL.md 2>/dev/null
```

Terms worth trying: `workflow`, `is_write_index`, `field mapping`, `percolate`, `clause`, `rule`, `findings`, `query index`. Then read the matches to choose among them:

```bash
sed -n '/^description:/,/^  references:/p' runbooks/<match>/SKILL.md
```

**Read `risk_level`, `read_only` and `reversible` from the frontmatter before handing a document over.** Some procedures here delete and recreate a detector. That is irreversible, and **the detector id changes** — which breaks saved dashboards, scripts and notification wiring keyed to the old one, and makes the existing findings and alerts unreachable through every read that filters on the new id. The documents themselves survive in the log-type findings and alerts indexes, keyed to the old id. Reading those fields from the file rather than from a table in this skill is what keeps the warning correct when the procedure changes.

**If the search returns nothing**, report the component, the read, and its output, and say no procedure covers it yet. That names the gap precisely. `references/components.md` carries the confirming reads for components that have no procedure.

## Before you call anything clean

Read `references/blind-checks.md`. Most reads in this skill have a rendering that returns a confident, healthy-looking answer while the fault is at full strength — an unset flag printing as `-` rather than `false`, an empty array beside a non-zero total, a monitor search that structurally cannot see Security Analytics monitors, a green cluster through a multi-week outage. Those are the recorded cause of investigations stalling for hours and weeks.

**Three rules:**

- **A zero is interpretable only beside a non-zero from the same read.** Without that positive control, record the check as *not evaluated* rather than as passed.
- **A read you could not run for access is not a read that came back clean.**
- **Capture before you repair.** A remediation usually destroys the evidence of its own necessity. The task list and hot threads are live-only, cumulative counters reset on restart, and a mapping divergence is only visible while both indexes exist. Record the detector id before recreating one — the findings and alerts survive, but only that id reaches them.

## After any prior change to the domain

**Ask what changed through the OpenSearch Service API before investigating through the OpenSearch REST API.** A service software update, a version upgrade, or an earlier detector repair — all of which use a blue/green deployment — can leave a detector referring to infrastructure that no longer exists, most often its saved-query index, so the detector fails on every schedule while its own configuration still reads correctly. Step 3's query-index read settles it in one call.

An access log is not the way to establish what changed. Writes made through the transport layer, or by the platform operator, do not appear in an HTTP access log at all, so an empty log is not evidence that nothing happened.

## When two components fire

Report both. They are not alternatives — a detector can be missing its query index *and* carry a rule set past the clause limit. Order the read-only procedure first.

**Fix the cause before clearing the backlog.** A failed scan carries its unprocessed documents into the next cycle, so the batch grows monotonically across executions until the detector cannot recover on its own. Recreating it clears the backlog, and the growth resumes if the cause is still there.

## Exit criteria

One of these, and nothing else:

- A component, its confirming read and output, and the procedure that covers it.
- A component, its read and output, and an explicit statement that no procedure covers it yet.
- The signal, the reads you ran, the reads you could not run, and a statement that no component here matches.

## Reference files

| File | Read it when |
|---|---|
| `references/blind-checks.md` | Before interpreting any result as clean |
| `references/components.md` | You have the component and need its full model — what it owns, how it fails, and the reads for components with no procedure |
| `references/starting-points.md` | You need the full signal inventory: which logs, metrics and reads exist, which an external reader can obtain, and what to monitor so a silent detector is caught |
| `references/routing-fixtures.md` | You are changing this skill, or checking that it still routes correctly |
| `references/evidence.md` | You need to know what this skill rests on and what it does not |

## Keeping this skill correct

What belongs here is a product fact: a component, a behaviour it produces, a read that confirms it. What does not belong is a document name, a catalogue of procedures, or a claim about which procedures exist — step 4 derives those from the collection, which is why this skill does not go stale when the collection changes.

**When a new runbook lands, nothing here needs editing.** If it covers a component already in step 3, the search finds it. If it covers a component that is not here, add the component, the behaviour and the confirming read — not the document name.

If you find this skill naming a specific runbook, that is a fault in the skill.
