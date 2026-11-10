---
name: security-analytics-history-retention-investigation
description: >-
  Establish whether Security Analytics and Alerting history indexes are accumulating past the
  retention their own settings configure, and size what that costs. The plugin deletes each expired
  family in a single request, and the engine refuses to delete an index a running snapshot holds -
  for the whole request - so a family can stop being pruned with nothing surfacing in the console.
  Use this when findings or alerts-history indexes persist past their configured retention, when a
  shard-count recommendation turns out to be plugin-owned, or as a periodic check on a domain
  running detectors. Read-only throughout - it establishes the condition and sizes it, and neither
  diagnoses the cause nor repairs anything.
metadata:
  agent_types:
    - INCIDENT_TRIAGE
    - PREVENTION
  skill_type: USER
  service: OpenSearch Security Analytics / Alerting
  applies_to_versions: All published lines. The single-request delete and its retry are present on every branch read; see "Verification status".
  risk_level: low
  read_only: true
  automatable: yes
  verification: documented
  last_verified: 2026-11-10
  last_edited: 2026-11-10
  references:
    - title: Security Analytics settings — the rollover and retention periods
      url: https://docs.opensearch.org/latest/security-analytics/settings/
      kind: product-doc
    - title: CAT indices API — expand_wildcards and the creation-date columns
      url: https://docs.opensearch.org/latest/api-reference/cat/cat-indices/
      kind: api-reference
    - title: Index name syntax — how expand_wildcards resolves hidden indexes
      url: https://docs.opensearch.org/latest/api-reference/index-name-syntax/
      kind: api-reference
    - title: Cluster settings API — include_defaults
      url: https://docs.opensearch.org/latest/api-reference/cluster-api/cluster-settings/
      kind: api-reference
    - title: DetectorIndexManagementService — the rollover and retention job
      url: https://github.com/opensearch-project/security-analytics/blob/main/src/main/java/org/opensearch/securityanalytics/indexmanagment/DetectorIndexManagementService.java
      kind: upstream-change
---

# Runbook — Are the plugin's history indexes being pruned?

## Before you use this

- **Confirm the behaviour against current OpenSearch documentation, or with your support channel, before you execute this.** Plugin behaviour moves with versions, and this runbook states the range it was written for in `metadata.applies_to_versions`. Where this document and the documentation disagree, the documentation is current and this is not.
- **Exercise the procedure in a non-production environment first.** Every step here is a read, so the risk is low — but run it once where you know the index set, so you recognise a normal result.
- **You remain responsible for the outcome in your environment.** This document is offered as-is, with no warranty and no liability for any effect of its use.
- **Send corrections back.** A step that failed, output that differed, a version where the behaviour changed, or a clearer command are worth more than a clean document that is quietly wrong. Open an issue or a pull request on the repository this runbook was shared from.

## When to use this

**Trigger.** Any of:

- **Findings or alerts-history indexes persist** with creation dates older than the retention their settings configure.
- **A shard-count recommendation turns out to be plugin-owned** — arriving here from `security-analytics-cluster-insights-investigation`.
- **A periodic check** on a domain running detectors.

**Applies when all of these hold.**

| Condition | How to check | Expected |
|---|---|---|
| Plugin history indexes exist | `GET /_cat/indices/.opensearch-sap-*-findings-*?v&h=index&expand_wildcards=all` | At least one row |
| A retention period is configured | `GET /_cluster/settings?include_defaults=true` | A time value on the retention settings |

**Does not apply when:**

- **No index is older than its configured retention.** Pruning is working. A high shard count then has another cause — see **Related runbooks**.
- **The retention period was shortened within one retention period.** Indexes older than the new setting are expected to survive until the next pruning pass, so compare the setting's own change against the index ages before calling it a stall.

## Values to obtain

| Name | What it is | How to obtain the current value |
|---|---|---|
| `${RETENTION}` | The configured retention for the family you are checking | `GET /_cluster/settings?include_defaults=true` → the `*_retention_period` settings under `plugins.security_analytics` and `plugins.alerting`. **`include_defaults=true` is required** — the API omits defaults without it, and returns them from the local node |
| `${ROLLOVER_PERIOD}` | How often the plugin's rollover-and-prune pass runs | Same read → the `*_rollover_period` settings |

## Impact

**Mechanism.** The plugin runs a rollover-and-prune pass per history family. When it prunes, it collects every index whose age exceeds the configured retention and sends them in **one** delete request. The engine rejects a delete that includes an index a running snapshot holds, **for the whole request** — so a single index caught in a snapshot prevents the deletion of every other expired index in that request. The retry that follows rebuilds its request from the full list rather than from the one index it is iterating, so it fails the same way.

Nothing in that path surfaces an operator-visible failure: the request is internal, and the retry's own failure is logged below the default level.

**Expired indexes then accumulate for as long as the condition holds**, and each one costs shards on every node that carries a copy.

**Size it for this system.** Two measurements.

```
GET /_cat/indices/.opensearch-sap-*-findings-*,.opensearch-sap-*-alerts-history-*,.opendistro-alerting-alert-history-*?v&h=index,creation.date.string,pri,rep,docs.count,store.size&expand_wildcards=all&s=creation.date
```

Count the rows older than `${RETENTION}`, and sum their shards as `pri × (1 + rep)`.

```
GET /${PATTERN}/_settings?expand_wildcards=all&filter_path=**.auto_expand_replicas,**.number_of_replicas
```

**Read the replica setting rather than assuming it.** Where the plugin's families expand replicas automatically, the per-index cost follows the node count up to the ceiling the setting states — so read the value and the ceiling, and compute from them.

**Blast radius.** Every node carrying a copy, through shard count and the heap each shard holds. Also every decision taken on the cluster's shard count — scaling, re-sharding the operator's own indexes — because the count those decisions rested on included indexes nobody intended to keep.

**Recoverability.** Full, and the data-loss risk runs the other way: every candidate is already past the retention the operator configured, so pruning them loses nothing of value. What is **not** automatically undone is any change already made in response to the shard pressure.

**Severity.** Urgent where shards per node is approaching the cluster's limit, or where a scaling or re-sharding decision is pending on the shard count — in that case the figure is about to be used and it is wrong. Routine otherwise: the accumulation is slow and nothing is being lost.

## Before you start

- [ ] Permission to call `GET /_cat/indices`, `GET /_cluster/settings` and `GET /<index>/_settings` on the domain
- [ ] Awareness that every index this runbook reads is hidden. The patterns below are dot-prefixed and carry `expand_wildcards=all` for that reason

> ⚠️ **Do not delete any index while working through this.** The expired indexes are the evidence, and the measurements above are what justify the repair to whoever approves it. The repair is not a manual delete in any case — the plugin prunes them itself once a pass completes.

## Procedure

### Step 1 — Confirm indexes are present past their configured retention

```
GET /_cluster/settings?include_defaults=true
```

Read `${RETENTION}` for the family you are checking. Then:

```
GET /_cat/indices/.opensearch-sap-*-findings-*,.opensearch-sap-*-alerts-history-*,.opendistro-alerting-alert-history-*?v&h=index,creation.date.string,pri,rep&expand_wildcards=all&s=creation.date
```

**Expect:** rows ordered oldest first, each carrying a date string in `creation.date.string`.

**Branch.**

- **No row is older than `${RETENTION}`** → pruning is working and there is no stall. Close the signal and take a high shard count to **Related runbooks**. **Stop here.**
- **No rows at all** → confirm the read is not blind before concluding. Re-run against the plugin's whole namespace:
  ```
  GET /_cat/indices/.opensearch-sap-*?v&h=index&expand_wildcards=all
  ```
  **Expect:** plugin indexes of some kind. That is your positive control — a domain running detectors has them by construction. An empty result here too means this caller cannot see them, and the check is **not evaluated** rather than clean. **Stop here** and obtain a working read.
- **Rows older than `${RETENTION}` are present** → continue.

### Step 2 — Confirm it persists rather than being a single missed pass

A pruning pass that failed once is not a stall. Re-run step 1's listing after a full `${ROLLOVER_PERIOD}` has elapsed.

**Expect:** the same expired rows still present.

**Branch.**

- **The expired set is unchanged, or has grown** → the condition persists. Continue to step 3.
- **The expired set has emptied, or shrunk to indexes a snapshot currently holds** → a pass completed. This was a transient collision and it resolved itself. Record what you saw and **stop here.**

### Step 3 — Size it, and attribute the cluster's shard population

Take the two measurements in **Impact**: the expired row count with its shard sum, and the replica setting with its ceiling.

Then establish what share of the cluster's shards this accounts for:

```
GET /_cat/indices/*?v&h=index,pri,rep&expand_wildcards=all&s=index
```

**Expect:** one row per index, including names beginning with a dot.

> ⚠️ **If no row's index name begins with a dot, this listing is not showing hidden indexes** and the attribution cannot be made from it. Obtain a listing that includes them before continuing.

Compute each row's shards as `pri × (1 + rep)`, split the rows by whether the name begins with a dot, and sum each population. **Report the expired sum, the plugin-owned sum, the operator-owned sum, and the total.**

**This step exists to stop a correct measurement being misattributed.** A shard-count finding dominated by plugin-owned indexes leads to the wrong remediation if the split is not read, and re-sharding the operator's own indexes in response will not move the number.

## Validation

1. **The reads were not blind.** Confirm the positive control from step 1 — a non-empty plugin-namespace listing — and the dot-prefixed rows from step 3. State both with your results.
2. **The expired set is stable across a pruning interval.** Step 2 is this check; record both observations and the interval between them, not just the second.
3. **The population sums are consistent.** The plugin-owned and operator-owned sums must add to the total from step 3's listing. A discrepancy means rows were dropped from the classification.

**Exit criteria.** You can state: the configured retention and rollover period as read; the count of indexes older than the retention and the shards they hold; that the set was still present after a full rollover period; and the plugin-owned, operator-owned and total shard sums.

## If this does not work

**Reach out when:** the expired set persists across more than one rollover period and the measurements above are in hand; or no listing you can obtain includes hidden indexes; or the retention settings cannot be read with defaults included.

**Reach out to:** your OpenSearch support channel, through the route your organisation uses for a production cluster.

**Attach:** the retention and rollover settings as read with `include_defaults=true`; the expired-index listing with creation dates and shard counts; the two observations from step 2 and the interval between them; the replica setting and its ceiling; the three population sums and the total; and the positive controls from validation item 1. State that no index was deleted and no setting was changed.

**What to ask for.** This runbook establishes that pruning has stopped and what it costs. **It does not establish why**, and the cause is not reachable from the reads available to an operator — so the question to put to support is why the pass is not completing for this family, with the measurements attached as evidence.

## Related runbooks

- `security-analytics-cluster-insights-investigation` — where you arrived from a shard recommendation, or need the population split alone.
- `security-analytics-rule-cost-audit` — where findings volume rather than retention is producing the index count.
- `opensearch-shard-pinning-remediation` — where the shard count is a placement problem rather than a volume problem.
- `security-analytics-setup-validation` — where the operator-owned population is the one at fault.

## Verification status

Every step traces to published documentation or to code analysis of the published implementation. The basis, what this runbook deliberately does not attempt, and the open questions are in `references/evidence.md` — read it before relying on this runbook for the first time, or before quoting it to anyone.
