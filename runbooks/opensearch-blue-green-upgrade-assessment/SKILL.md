---
name: opensearch-blue-green-upgrade-assessment
description: >-
  Assess a managed OpenSearch domain before, during and after a blue/green version upgrade, and
  tell a stalled upgrade apart from a slow one. A blue/green provisions a parallel fleet, copies
  every shard to it and switches over; the copy phase stops making progress when a shard cannot
  move, and the upgrade cannot be paused or cancelled once started. Use this to run pre-flight
  readiness checks, to report progress while an upgrade is running, to decide whether a stalled
  copy phase needs reaching out about, and to interpret a yellow cluster status during or after
  change. Read-only throughout - it performs no writes and needs no change window.
metadata:
  agent_types:          # GENERIC cannot be combined with other values, so it is omitted here
    - INCIDENT_TRIAGE
    - PREVENTION
    - CHAT
  skill_type: USER
  service: Amazon OpenSearch Service / OpenSearch
  applies_to_versions: All managed OpenSearch and legacy Elasticsearch domains. Several thresholds and capabilities change at 2.17 and 2.19 - see "What differs by version".
  risk_level: low
  read_only: false   # steps 2-5 are pure reads; step 1's dry run overwrites the previous dry-run record
  reversible: partial  # the domain is untouched, but a prior dry-run record cannot be restored
  automatable: partial # step 1 needs the intended change composed by a person; steps 2-5 are fully automatable
  verification: documented
  last_verified: 2026-10-14
  last_edited: 2026-10-14
  references:
    - title: Making configuration changes in Amazon OpenSearch Service — blue/green stages and validation codes
      url: https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-configuration-changes.html
      kind: product-doc
    - title: Upgrading Amazon OpenSearch Service domains
      url: https://docs.aws.amazon.com/opensearch-service/latest/developerguide/version-migration.html
      kind: product-doc
    - title: Operational best practices — shard count and shard size
      url: https://docs.aws.amazon.com/opensearch-service/latest/developerguide/bp.html
      kind: product-doc
    - title: Supported operations in Amazon OpenSearch Service
      url: https://docs.aws.amazon.com/opensearch-service/latest/developerguide/supported-operations.html
      kind: product-doc
    - title: DescribeDomainChangeProgress
      url: https://docs.aws.amazon.com/opensearch-service/latest/APIReference/API_DescribeDomainChangeProgress.html
      kind: api-reference
    - title: GetUpgradeStatus
      url: https://docs.aws.amazon.com/opensearch-service/latest/APIReference/API_GetUpgradeStatus.html
      kind: api-reference
    - title: How do I troubleshoot stuck or failed upgrades in OpenSearch Service?
      url: https://repost.aws/knowledge-center/opensearch-stuck-failed-upgrade
      kind: product-doc
    - title: Why is my OpenSearch Service domain stuck in the Modifying state?
      url: https://repost.aws/knowledge-center/opensearch-domain-stuck-processing
      kind: product-doc
    - title: Why is my OpenSearch Service cluster in a red or yellow status?
      url: https://repost.aws/knowledge-center/opensearch-red-yellow-status
      kind: product-doc
    - title: Cluster Health API
      url: https://docs.opensearch.org/latest/api-reference/cluster-api/cluster-health/
      kind: api-reference
    - title: CAT Recovery API
      url: https://docs.opensearch.org/latest/api-reference/cat/cat-recovery/
      kind: api-reference
    - title: Cluster Allocation Explain API
      url: https://docs.opensearch.org/latest/api-reference/cluster-api/cluster-allocation/
      kind: api-reference
    - title: Shard allocation filtering
      url: https://docs.opensearch.org/latest/api-reference/index-apis/shard-allocation/
      kind: api-reference
---

# Runbook — Assess a blue/green upgrade: readiness, progress, and whether it is stuck

## Before you use this

- **Confirm the behaviour against current vendor documentation, or with your support channel, before you execute this.** Managed-service behaviour and thresholds move with versions, and this runbook states the range it was written for in `metadata.applies_to_versions`. Where this document and the documentation disagree, the documentation is current and this is not.
- **Exercise the procedure in a non-production environment first.** Every step here is a read, so the risk is low — but run it once on a non-production domain so you know what healthy output looks like before you need to recognise unhealthy output.
- **You remain responsible for the outcome in your environment.** This document is offered as-is, with no warranty and no liability for any effect of its use.
- **Send corrections back.** A step that failed, output that differed, a version where the behaviour changed, or a clearer command are worth more than a clean document that is quietly wrong. Open an issue or a pull request on the repository this runbook was shared from.

## When to use this

**Trigger.** Any of:

- **Pre-flight** — before starting a version upgrade, to find blockers while they are still cheap to fix.
- **In-flight** — to report which phase an upgrade is in and whether it is progressing.
- **Stalled** — the copy phase appears to have stopped, or the domain has been in a modifying state longer than expected.
- **Yellow status** — the cluster went yellow during or after the change and someone needs to know whether that is self-resolving.

**Applies when all of these hold.** Each is checkable.

| Condition | How to check | Expected |
|---|---|---|
| The domain is managed rather than self-hosted | `aws opensearch describe-domain --domain-name <name>` | A domain configuration is returned |
| You can reach the data plane | `GET /_cluster/health` | `200` with a `status` field |
| A configuration change is in progress, or you are assessing before one | `aws opensearch describe-domain-change-progress --domain-name <name>` | A change status, or none if idle |

**What differs by version.** Several thresholds this runbook compares against are version-dependent, and one capability changes outright.

| Behaviour | Earlier lines | Later lines | How to tell which you have |
|---|---|---|---|
| Maximum shards per data node | 1,000 for Elasticsearch and OpenSearch 1.1 through 2.15 | 4,000 from 2.17, expressed as 1,000 shards per 16 GiB of JVM heap | `GET /` → `version.number` |
| `cluster.max_shards_per_node` writable | Modifiable on 2.15 and earlier | Cannot be modified from 2.17 | Same read |
| Cluster Insights surfaces | Console tab from legacy 6.8 and all OpenSearch 1.0 onward | Dashboards interface needs 2.17; the query view needs 2.19 | Same read |
| Upgrading to a 3.x line | From 1.3 or 2.x you must reach 2.19 first | Deprecated index settings must be removed or validation fails: `index.knn.algo_param.ef_construction`, `index.knn.algo_param.m`, `index.knn.space_type`, `index.store.hybrid.mmap.extensions`. Indexes created on 1.3 or legacy 7.10 and earlier must be reindexed first | Same read, plus `GET /_all/_settings?flat_settings=true` filtered for those keys |

The **25 shards per GiB of heap** guideline and the **50 GiB maximum shard size** are published as general guidance and are not version-gated, so they apply across all of the above.

**Does not apply when:**

- **The cluster status is red.** Resolve that first. Reconfiguring a red domain is documented to risk leaving it stuck in a modifying state, so a red cluster is a stop condition rather than something to assess past.
- **The domain is self-managed.** The control-plane steps have no equivalent, and the published phase vocabulary and validation codes do not exist.
- **You intend to change something.** This runbook only reads. Clearing an allocation filter, lifting a block or retrying allocation are writes and live elsewhere — see **Related runbooks**.

## Values to obtain

| Name | What it is | How to obtain the current value |
|---|---|---|
| `${DOMAIN}` | The managed domain name | `aws opensearch list-domain-names` |
| `${HEAP_GIB}` | JVM heap per data node, which sets the shard ceiling | `GET /_nodes/stats/jvm` → heap maximum per node |
| `${NODE_COUNT}` | Data node count, for the shard-count-as-a-multiple check | `GET /_cat/nodes?v` |

## Impact

**Mechanism.** A blue/green provisions a second fleet, copies every shard onto it, routes traffic across, then removes the old nodes. The data plane stays available for search and indexing throughout, and the switchover is documented to occur without data loss. Two properties make assessment worth doing rather than waiting:

- **The copy phase blocks rather than fails when a shard cannot move.** Anything that makes a shard unmovable — an index-level allocation filter, insufficient space on the new nodes, an unassigned primary — leaves the phase in place indefinitely rather than erroring.
- **The upgrade cannot be paused or cancelled once started.** Where it does fail outright, the service is documented to restore automatically from the snapshot taken beforehand, and no further action is needed. But a *stalled* upgrade is not a failed one, and there is no abort.

Node count also temporarily increases, often doubling, which is documented to strain cluster manager nodes. The published performance expectations for the window are increased search and indexing latency while data copies, and increased rejections as load rises — so a degraded-but-moving cluster is the documented normal state, not a fault.

**Size it for this system.** Three measurements decide whether this domain is ready, and all three are reads:

```
GET /_cat/allocation?v
GET /_cat/indices?v&bytes=gb
GET /_nodes/stats/jvm
```

Compare per-node `shards` against 25 × `${HEAP_GIB}`, and against the version ceiling from the table above. Compare per-shard size — `pri.store.size` divided by `pri` — against the published 50 GiB maximum. Shards above that maximum are documented to increase copy time.

**Blast radius.** The whole domain. A blocked copy phase holds the cluster in a modifying state, during which other configuration changes cannot proceed.

**Recoverability.** An outright upgrade failure is recoverable by the service automatically from its pre-upgrade snapshot. A stall is not recoverable by you at all — there is no cancel, which is why the pre-flight in step 1 is worth more than any in-flight step.

**Severity.** Urgent where the copy phase has stopped advancing and the domain has been in a modifying state for a prolonged period — the published threshold for opening a case is **24 hours** in a modifying state, or stuck in the deleting-older-resources stage. Routine where phases are advancing, even slowly, because the published expectation for the upgrade itself is 15 minutes to several hours depending on cluster load and size.

## Before you start

- [ ] Control-plane permission for `describe-domain-change-progress`, `get-upgrade-status`, `get-upgrade-history` and, for the pre-flight, `update-domain-config` with a dry run
- [ ] Data-plane read access. The paths used here are on the published supported-operations list: `/_cluster/health`, `/_cluster/allocation/explain`, `/_cluster/settings`, `/_cluster/state`, the `/_cat` family except `/_cat/nodeattrs`, and all index-path operations
- [ ] A way to record two samples a few minutes apart, because progress is a delta rather than a value

## Procedure

### Step 1 — Pre-flight: dry-run the change and read the validation result

Run this before the upgrade window, not in it. The dry run is the documented gate. It does not mutate the domain — but it is **not free of side effects**, and that is why this runbook is not marked read-only.

> ⚠️ **A dry run overwrites the previous dry-run record.** Dry runs are retained for 90 days or until the next configuration update, and each one replaces the last. If someone is relying on an earlier validation result, capture it before you run this. **Steps 2 to 5 are pure reads** and can be run unattended and repeatedly; this step cannot.

```
aws opensearch update-domain-config \
  --domain-name ${DOMAIN} \
  --dry-run --dry-run-mode Verbose \
  <the change you intend to make>

aws opensearch describe-dry-run-progress --domain-name ${DOMAIN} --dry-run-id <id>
```

**Expect:** `DryRunResults.DeploymentType` of `Blue/Green`, and `DryRunProgressStatus.DryRunStatus` of `succeeded` with no `ValidationFailures`.

> ⚠️ **Never let this fall through to the live call.** The same request without `--dry-run` applies the change.

**Branch on any `ValidationFailures[].Code`.**

- `ShardMovementBlocked` → shards are pinned by an allocation filter. Not overridable. Go to `opensearch-shard-pinning-remediation`.
- `TooManyShards`, `MaxShardCountExceeded`, `HAMaxShardCountExceeded` → shard count or per-node placement exceeds what the target configuration allows. The first two are documented as overridable and the third is not; reduce shard count rather than overriding.
- `RedCluster` → stop. Resolve the red status first.
- Any other code → read it against the published validation-code table before proceeding.

Validation results are classified **Critical**, which cannot be accepted, or **Warning**, which blocks by default until its code is passed in `AcceptedWarnings` for that one request. `AcceptedWarnings` applies only to that request and is not persisted on the domain.

**If no change is in progress and the dry run is clean:** the pre-flight is complete. **Stop here** until the upgrade starts.

### Step 2 — In-flight: read which phase the change is in

```
aws opensearch describe-domain-change-progress --domain-name ${DOMAIN}
aws opensearch get-upgrade-status --domain-name ${DOMAIN}
```

**Expect:** from the first, `ChangeProgressStages[]` with each stage's `Name`, `Status` and `LastUpdated`, plus `ConfigChangeStatus`, `Status` and `TotalNumberOfStages`. From the second, `UpgradeStep` of `PRE_UPGRADE_CHECK`, `SNAPSHOT` or `UPGRADE`, and `StepStatus` of `IN_PROGRESS`, `SUCCEEDED`, `SUCCEEDED_WITH_ISSUES` or `FAILED`.

The published stage vocabulary is `Validation`, `Creating a new environment`, `Provisioning new nodes`, `Traffic routing on new nodes`, `Traffic routing on old nodes`, `Preparing nodes for removal`, **`Copying shards to new nodes`**, `Terminating nodes`, `Deleting older resources` and `Dynamic update`. Not every stage appears on every change; a version upgrade has been observed reporting five, with `TotalNumberOfStages` matching. Report the stage name verbatim — these are the names the console and notifications use.

**The copy progress counter lives in the stage's `Description` string, not in a numeric field.** There is no count field to read; the observed form is `Copying shards to new nodes - <done> of <total> completed.`, and that string is what operators watch. Quote it or parse it, but do not look for a number elsewhere.

**Branch.**

- `StepStatus: FAILED` → the service is documented to restore automatically from the pre-upgrade snapshot. Confirm with `aws opensearch get-upgrade-history --domain-name ${DOMAIN} --max-results 10` and report; no further action is required of you.
- `StepStatus: SUCCEEDED_WITH_ISSUES` → read the history as above and report the issue text.
- `ConfigChangeStatus: ValidationFailed` → return to step 1's branch table.
- Stage is `Copying shards to new nodes` → continue to step 3. This is the phase that blocks.
- Any other stage advancing → report the stage and sample again later.

### Step 3 — Decide whether the copy phase is progressing or blocked

This is the step that distinguishes slow from stuck, and it needs **two samples**.

```
GET /_cat/recovery?active_only=true&v
```

**Expect:** rows with `stage`, `bytes_percent`, `bytes_recovered`, `bytes_total`, `time`, `type`, `source_node` and `target_node`.

**Read `bytes_percent`, not `files_percent`.** The published guidance names `bytes_percent` as the healthy-relocation indicator, and observation bears it out: on live peer recoveries `files_percent` consistently runs ahead of `bytes_percent`, sometimes by a wide margin, so it reports a recovery as nearly complete while most of the data is still to move. Reading it would tell you a stalled copy is almost done.

> ⚠️ **An empty recovery table does not mean finished.** `active_only=true` shows only recoveries in flight, and a stalled copy phase has been observed with nothing in this table while shards were demonstrably mid-relocation. To an operator under pressure an empty table reads as "nothing wrong". **Always pair it with the shard state:**
> ```
> GET /_cat/shards?v&h=index,shard,prirep,state,node&s=state
> ```
> This is the read that actually exposes the stall — shards sitting in `RELOCATING` that the recovery table does not show.

**Branch.**

- **`bytes_percent` increases between samples** → the copy is progressing. Report the figure and the stage, sample again later. **Stop here.**
- **`bytes_percent` does not increase, or the recovery table is empty while shards read `RELOCATING`** → movement is stalled. Continue to step 4.

Corroborate with cluster health, which distinguishes the same two cases from the other side:

```
GET /_cluster/health
```

Record `status`, `relocating_shards`, `unassigned_shards`, `initializing_shards`, `active_shards_percent_as_number`, `number_of_pending_tasks` and `task_max_waiting_in_queue_millis`. A rising value in that last field is documented to indicate an overloaded cluster manager, consistent with the documented strain from the temporarily doubled node count.

Two readings, needing different instruments:

- **`unassigned_shards` above zero with `relocating_shards` at zero** → allocation is being refused. The allocation explanation in step 4 is the right tool.
- **`relocating_shards` above zero with `unassigned_shards` at zero** → relocation was decided and is failing in flight. Look for `number_of_pending_tasks` entries whose `source` is `shard-failed` at `HIGH` priority, indicating a retry loop. **The allocation explanation will not help here**, because the decision has already been made.

> ⚠️ **A green cluster does not mean the upgrade is progressing.** A stalled copy phase has been observed reporting `"status": "green"` with zero unassigned shards, full `active_shards_percent_as_number`, and a non-zero relocating count. **Health colour is not a progress signal.** Branch on stage status and shard state, never on colour — any step phrased "if the cluster is yellow" would skip the case that actually occurs.

### Step 4 — Get the reason a shard will not move

```
GET /_cluster/allocation/explain?pretty
```

With an empty body this finds the first unassigned shard and explains why it cannot be allocated. For a specific shard:

```
POST /_cluster/allocation/explain
{"index":"<index>","shard":<n>,"primary":<true|false>}
```

Add `?include_disk_info=true` to bring in free-space figures per node.

**Expect:** `current_state`, `unassigned_info.reason`, `can_allocate`, `allocate_explanation`, and `node_allocation_decisions[].deciders[]` with a `decider`, `decision` and `explanation` each.

**Branch on the decider that returns `NO`.** This is the whole point of the step — act on the named decider rather than on a guess.

- **`filter` with `NO`** → an index-level allocation filter is pinning the shard, and the `explanation` names the offending setting and its values. Go to `opensearch-shard-pinning-remediation`.
- **`disk_threshold` with `NO`**, or low `free_disk_percent` on the new nodes → there is not enough space to copy shards across. Reach out; capacity is not something this runbook changes.
- **`throttling` with `THROTTLE`**, or `can_allocate: throttled` → a concurrency limit, not a blockage. This is expected during recovery. Sample again before concluding anything.
- **`can_allocate: no_valid_shard_copy`** → a data-loss path. Reach out immediately.
- **`max_retry` as the only `NO`** → allocation has exhausted its retry ceiling. The cause must be addressed first; retrying is downstream of that and lives in the remediation runbook.

Then read the settings that could be responsible, so your evidence names them:

```
GET /_all/_settings?flat_settings=true
```

**Expect:** any `index.routing.allocation.include.*`, `.exclude.*` or `.require.*` key, and any `index.blocks.write` or `index.blocks.read_only`. Published guidance calls out `index.routing.allocation.require._name` and `index.blocks.write` as the keys to look for. **Treat every `routing.allocation` key as a candidate** rather than only those two, and let step 4's `filter` explanation tell you which one is actually biting.

### Step 5 — Interpret a yellow status

```
GET /_cluster/health?level=indices
```

**Expect:** `status` for the cluster and per index. An index's status is determined by its worst shard status, and the cluster's by its worst index status, so this read tells you which index is setting the colour.

The published definitions: **green** means all primaries and their replicas are allocated; **yellow** means all primaries are allocated but at least one replica is unassigned, so the cluster is operational but not fully redundant; **red** means at least one primary is unassigned and some data is unavailable.

**Be careful what you tell people here.** Yellow is documented to occur and self-resolve after a node failure, a transient restart, or index creation, as replicas are reallocated — and a blue/green does replace nodes. But **no published source states that a blue/green deployment itself is expected to turn a cluster yellow.** So:

- **Report yellow as what it is** — full primary availability, reduced redundancy, and a documented risk of data loss if a node fails while in that state.
- **Do not wave persistent yellow off as normal for an upgrade.** Run step 4 against an unassigned replica and act on the named decider.

## Validation

This procedure produces a judgement, so validation confirms the judgement is sound rather than that a system recovered.

1. **Two samples exist, taken apart.** A single reading cannot show progress — progress is a delta.
   ```
   GET /_cat/recovery?active_only=true&v&h=index,shard,stage,bytes_percent,bytes_recovered,bytes_total
   ```
   **Expect:** two captures of this output with their wall-clock times recorded. One capture supports no conclusion either way.
2. **The control plane and the data plane tell the same story.**
   ```
   aws opensearch describe-domain-change-progress --domain-name ${DOMAIN}
   GET /_cat/shards?v&h=index,shard,prirep,state,node&s=state
   ```
   **Expect:** a stage of `Copying shards to new nodes` alongside shards in `RELOCATING` or `INITIALIZING`. That stage with **no** relocating shards and **no** unassigned shards is a contradiction — resolve it before reporting either reading, because one of the two is stale.
3. **Each conclusion names the field it rests on.**
   ```
   GET /_cluster/health
   ```
   **Expect:** to be able to point at a value for each claim — "blocked" at a named decider from step 4, "progressing" at a `bytes_percent` delta, "degraded redundancy" at `status` with the index that sets it. A conclusion resting on elapsed time is not an assessment.
4. **The settings enumeration covered every index.**
   ```
   GET /_all/_settings?flat_settings=true
   GET /_cat/indices?expand_wildcards=all&v&h=index,docs.count
   ```
   **Expect:** the settings read to return every index the second call lists, including hidden and system indexes. A pattern-scoped read that silently excluded them would miss exactly the indexes most likely to carry a routing-allocation key.

**Exit criteria.** You can state the current stage by its published name, whether `bytes_percent` advanced between two timestamped samples, the cluster status with the index that sets it, and — where movement is blocked — the specific decider and setting responsible. Anything short of that is an impression rather than an assessment.

## If this does not work

**Reach out when:** `bytes_percent` does not advance across samples and step 4 names a decider you cannot act on; or the domain has been in a modifying state for more than **24 hours**, or stuck in `Deleting older resources` for more than 24 hours, which is the published threshold for opening a case; or `can_allocate` reports `no_valid_shard_copy`; or the copy phase shows no recovery rows at all.

**Reach out to:** your support channel for the managed service. Note that the upgrade cannot be paused or cancelled, so where it is genuinely stuck the published position is that you wait for the underlying issue to be resolved.

**Attach:** the change-progress output with stage names and `LastUpdated` values; both `_cat/recovery` samples with their timestamps; `_cluster/health`; the full `_cluster/allocation/explain` output including the decider list; the `_all/_settings` audit; `_cat/allocation` and `_cat/indices` showing per-node shard counts and per-shard sizes against the thresholds; and the current recovery-throughput settings read with `GET /_cluster/settings?include_defaults=true&flat_settings=true` as a baseline.

> **Two things to reach out about, not steps.** Recovery-throughput tuning — the concurrency settings and the per-node recovery byte rate — is not operator-settable on a managed domain, because `PUT /_cluster/settings` is restricted to an enumerated allowlist that excludes them. The values *are* readable — `GET /_cluster/settings?include_defaults=true&flat_settings=true` — so read them and include them in what you hand over. Whether they can be changed for your domain, by whom, and under what preconditions is not published, so do not plan around it and do not represent it to anyone as available: ask through your support channel and let the answer come from there. Separately, **no published source addresses plugin-owned system indexes blocking an upgrade, or the consequences of deleting them.** Do not delete a system index to unblock a copy phase. The documented action for a non-progressing copy phase is to collect evidence and open a case. Where that path has been taken it did clear the stall, but it destroyed the plugin objects those indexes backed, requiring recreation afterwards — and the recreation was not clean, surfacing a further plugin error that needed another round. Treat it as lossy and not operator-executable, and as something to be agreed explicitly rather than assumed.

## Related runbooks

Siblings are referred to by name, which resolves to a directory and is greppable.

- `opensearch-shard-pinning-remediation` — where step 4 names the `filter` decider. It clears the allocation filter that is blocking the move.
- `opensearch-shard-density-remediation` — where the pre-flight returns `TooManyShards`, `MaxShardCountExceeded` or `HAMaxShardCountExceeded`, or per-node counts exceed the published ceiling.

A published automation document, `AWSSupport-TroubleshootOpenSearchRedYellowCluster`, covers the adjacent red-and-yellow health problem. Prefer it to re-deriving that logic.

## Verification status

Every step traces to a reputable source. A step resting on inference is reason to leave a procedure unwritten rather than to label it. The basis, what has been observed in practice, the open questions and any superseded claims are in `references/evidence.md` — read it before relying on this runbook for the first time, or before quoting it to anyone.
