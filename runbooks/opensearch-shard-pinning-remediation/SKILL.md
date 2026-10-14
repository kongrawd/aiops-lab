---
name: opensearch-shard-pinning-remediation
description: >-
  Clear the index-level shard allocation filter that is preventing a shard from moving, so a
  blocked relocation, a stalled blue/green upgrade or an unassignable replica can proceed. Shards
  are allocated only to nodes satisfying every active filter, so an include, exclude or require
  filter that matches no eligible node pins the shard where it is. Use this when the allocation
  explanation names the filter decider, when a pre-upgrade validation reports that shards are
  pinned to specific nodes, or when a shard stays unassigned with a filter set on its index. The
  procedure discovers which attribute is responsible rather than assuming one.
metadata:
  agent_types:
    - INCIDENT_MITIGATION
  skill_type: USER
  service: Amazon OpenSearch Service / OpenSearch
  applies_to_versions: All lines. The filter semantics are stable; one setting behaves differently on a managed domain - see "What differs by deployment and version".
  risk_level: medium
  read_only: false
  reversible: yes
  automatable: partial
  verification: documented
  last_verified: 2026-10-14
  last_edited: 2026-10-14
  references:
    - title: Shard allocation filtering
      url: https://docs.opensearch.org/latest/api-reference/index-apis/shard-allocation/
      kind: api-reference
    - title: Update Index Settings API
      url: https://docs.opensearch.org/latest/api-reference/index-apis/update-settings/
      kind: api-reference
    - title: Get Settings API
      url: https://docs.opensearch.org/latest/api-reference/index-apis/get-settings/
      kind: api-reference
    - title: Cluster Allocation Explain API
      url: https://docs.opensearch.org/latest/api-reference/cluster-api/cluster-allocation/
      kind: api-reference
    - title: Making configuration changes in Amazon OpenSearch Service — ShardMovementBlocked
      url: https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-configuration-changes.html
      kind: product-doc
    - title: Supported operations in Amazon OpenSearch Service — index block divergence
      url: https://docs.aws.amazon.com/opensearch-service/latest/developerguide/supported-operations.html
      kind: product-doc
    - title: Why is my OpenSearch Service domain stuck in the Modifying state?
      url: https://repost.aws/knowledge-center/opensearch-domain-stuck-processing
      kind: product-doc
    - title: Why is my OpenSearch Service cluster in a red or yellow status?
      url: https://repost.aws/knowledge-center/opensearch-red-yellow-status
      kind: product-doc
---

# Runbook — A shard will not move because an allocation filter pins it

## Before you use this

- **Confirm the behaviour against current documentation, or with your support channel, before you execute this.** Behaviour moves with versions, and this runbook states the range it was written for in `metadata.applies_to_versions`. Where this document and the documentation disagree, the documentation is current and this is not.
- **Exercise the procedure in a non-production environment first.** Clearing an allocation filter releases shards to move. Run it end to end on an index you can afford to disrupt, confirm each step's output, and only then run it where it matters.
- **You remain responsible for the outcome in your environment.** This document is offered as-is, with no warranty and no liability for any effect of its use. Step 3 changes index settings on a live cluster and can start substantial shard movement, and the decision to run it is yours.
- **Send corrections back.** A step that failed, output that differed, a version where the behaviour changed, or a clearer command are worth more than a clean document that is quietly wrong. Open an issue or a pull request on the repository this runbook was shared from.

## When to use this

**Trigger.** Any of:

- The allocation explanation names the **`filter`** decider with a `NO` decision for a shard that needs to move or to be assigned.
- A pre-upgrade validation reports **`ShardMovementBlocked`** — shards pinned to specific nodes, which is documented as not overridable.
- A shard stays `UNASSIGNED` or `RELOCATING` while its index carries an `index.routing.allocation.*` setting.

**Applies when all of these hold.**

| Condition | How to check | Expected |
|---|---|---|
| The filter decider is what blocks the shard | `GET /_cluster/allocation/explain?pretty` | A decider named `filter` with `"decision": "NO"` |
| The index carries a routing-allocation setting | `GET /<index>/_settings?flat_settings=true` | An `index.routing.allocation.{include,exclude,require}.*` key |
| The cluster is not red | `GET /_cluster/health` | `status` is `green` or `yellow` |

**Does not apply when:**

- **The stuck shards are `RELOCATING` rather than `UNASSIGNED`.** This is the discriminator that matters most and the easiest one to get wrong. An allocation filter decides *whether* a shard may go somewhere; it does not stall a relocation that has already been decided and is in flight. A shard sitting in `RELOCATING` has already passed the filter. Check which state you have before anything else:
  ```
  GET /_cat/shards?v&h=index,shard,prirep,state,node&s=state
  GET /_cluster/health
  ```
  `UNASSIGNED` shards with `relocating_shards` at zero point here. **`RELOCATING` shards, especially with `number_of_pending_tasks` entries whose `source` is `shard-failed` at `HIGH` priority, are a relocation retry loop and a different fault** — clearing filters will not touch it. Go to `opensearch-blue-green-upgrade-assessment` and reach out from there.
- **A different decider returns `NO`.** `disk_threshold` means capacity, `throttling` with `THROTTLE` means a concurrency limit and is expected during recovery, and `no_valid_shard_copy` is a data-loss path to reach out about immediately. Clearing a filter will not help any of those, and this is the most common way to run the wrong procedure here.
- **The cluster is red.** Resolve that first; reconfiguring a red domain is documented to risk leaving it stuck.
- **The filter is deliberate and currently correct.** A filter placed to hold shards on one node for a shrink operation is doing its job. Clear it after the operation, not during.

**What differs by deployment and version.** One setting you may encounter diverges by deployment type, and getting it wrong wastes an outage.

| Behaviour | Self-managed OpenSearch | Managed domain | How to tell which you have |
|---|---|---|---|
| Making an index read-only | `index.blocks.write: true` is the standard method | **`index.blocks.write` is removed automatically within about two minutes.** Use `index.blocks.read_only: true` instead | `aws opensearch describe-domain --domain-name <name>` succeeds on a managed domain |
| Cluster-level allocation and recovery settings | Settable through `PUT /_cluster/settings` | Restricted to an enumerated allowlist that excludes the recovery and concurrency settings | Attempting the call returns that the payload is not allowed |

The filter semantics themselves — `include`, `exclude`, `require`, and `null` as the reset — are stable across lines, so the rest of this procedure does not change with version.

## Values to obtain

| Name | What it is | How to obtain the current value |
|---|---|---|
| `${INDEX}` | The index whose shard cannot move | `GET /_cluster/allocation/explain?pretty` → `index` |
| `${SETTING_KEY}` | The exact blocking setting, including the attribute name | Step 1's `filter` decider explanation, cross-checked against step 2 |
| `${PRIOR_VALUE}` | The current value of that setting, saved before changing it | `GET /${INDEX}/_settings?flat_settings=true` |

**Do not assume `${SETTING_KEY}`.** Discovering it is the point of steps 1 and 2 — see the superseded claim in **Verification status**.

## Impact

**Mechanism.** The three index-level filter families decide where a shard may live: `include.<attr>` allocates to nodes matching any listed value, `exclude.<attr>` refuses nodes matching any listed value, and `require.<attr>` allocates only to nodes matching all listed values. The documentation states the governing rule: **shards are only allocated to nodes that satisfy all active filters.** So a filter naming nodes that no longer exist, or too few of them, leaves the shard with nowhere legal to go — documented as "the shard can become unassigned because it doesn't have enough filtered nodes", and as "if you pin shards to an existing set of nodes, then shard migration can get stuck."

This is what makes it an upgrade blocker rather than a performance issue. A blue/green copies shards onto a new fleet; a filter pinning them to the old one prevents the copy phase from completing, and the upgrade cannot be cancelled. The managed service reports this before the fact as a dedicated, non-overridable validation issue whose documented remedy is to remove the filters from all affected indexes.

**Size it for this system.** The magnitude is how many indexes carry such a setting and how many shards are consequently stuck:

```
GET /_all/_settings?flat_settings=true
GET /_cluster/health
GET /_cat/shards?h=index,shard,prirep,state,unassigned.reason
```

Count indexes with any `index.routing.allocation.*` key, and read `unassigned_shards` against `relocating_shards` — unassigned above zero while relocating is zero means movement is blocked rather than slow.

**Blast radius.** Clearing a filter cuts both ways, and this is the part to think about before step 3. While it is set, the affected shards cannot move. Once cleared, they become eligible to move, and the cluster may begin relocating them immediately — on a loaded cluster that is additional work at the moment you least want it. Clearing filters across many indexes at once can therefore turn one blocked shard into broad rebalancing.

**Recoverability.** Fully reversible: the prior value can be written back. What is not reversible is shard movement that occurred in the interval, which costs time and I/O to undo.

**Severity.** Urgent where an upgrade or other configuration change is blocked, because the change cannot be cancelled and the domain stays in a modifying state. Urgent where an unassigned primary is involved, since that is a red cluster and partial data unavailability. Routine where the only consequence is a yellow status with replicas unassigned — still worth fixing, but not at any cost to a busy cluster.

## Before you start

- [ ] Permission to call `PUT /<index>/_settings`, plus read access to the allocation-explain, settings, health and CAT routes
- [ ] The output of steps 1 and 2 saved, since it carries `${PRIOR_VALUE}` and is the rollback body
- [ ] An assessment of current cluster load, because clearing a filter can start relocation immediately
- [ ] Confirmation that the filter is not serving a current purpose, such as holding shards together for a shrink

**Do not proceed if** the blocking decider is anything other than `filter`, or if the cluster is red, or if you cannot establish the exact setting key. Clearing the wrong setting leaves the real blockage in place and adds an unexplained change to the incident.

## Procedure

### Step 1 — Confirm the shard is unassigned, then that a filter is what blocks it

First establish the shard state, because the allocation explanation answers a question that only applies to an unassigned shard:

```
GET /_cat/shards?v&h=index,shard,prirep,state,node&s=state
```

**If the affected shards read `RELOCATING`:** allocation was already decided and a filter is not what is holding them. **Stop here** — see *Does not apply when*.

**If they read `UNASSIGNED`:** continue.

```
GET /_cluster/allocation/explain?pretty
```

With an empty body this finds the first unassigned shard and explains why it cannot be allocated. For a specific shard:

```
POST /_cluster/allocation/explain
{"index":"${INDEX}","shard":<n>,"primary":<true|false>}
```

**Expect:** `current_state`, `can_allocate`, `allocate_explanation`, and `node_allocation_decisions[].deciders[]` with a `decider`, `decision` and `explanation` each.

**Look for a decider named `filter` with `"decision": "NO"`.** Its `explanation` names the offending setting and the filter values — that is `${SETTING_KEY}`. The documented role of this decider is to apply "allocation filters based on index settings like `index.routing.allocation.include`, `exclude`, or `require`".

**If no `filter` decider returns `NO`:** this is not the situation. Read the decider that does and act on that instead. **Stop here.**

**If the `filter` decider returns `NO`:** record the explanation verbatim and continue to step 2.

### Step 2 — Read the setting back and save its current value

```
GET /${INDEX}/_settings?flat_settings=true
```

**Expect:** the `index.routing.allocation.*` key named by step 1, with its current value. Save both — this is `${PRIOR_VALUE}` and the rollback body.

Published guidance calls out `index.routing.allocation.require._name` and `index.blocks.write` as the keys to look for when identifying pinned shards. **Treat every `routing.allocation` key as a candidate** and let step 1's explanation decide which one is biting, rather than clearing whichever key looks familiar.

For a whole-domain audit before an upgrade:

```
GET /_all/_settings?flat_settings=true
```

**Expect:** every index, including hidden ones. Enumerate the affected index names explicitly — you will need them in step 3, and you will not be using a wildcard.

### Step 3 — Clear the one setting that blocks the shard

> ⚠️ **This changes state.** It removes an allocation filter from this index's settings. Where that filter was what held the shards, the cluster may begin relocating them immediately, and on a loaded cluster that is extra work at a bad moment.
>
> **Do not treat "the filter is gone" as "the shards will move."** In the one recorded application of this procedure the key was cleared, the write was acknowledged, the key was confirmed absent — and nothing moved. Step 4 tests whether it helped; step 4's exit is what to do when it did not.

```
PUT /${INDEX}/_settings
{
  "${SETTING_KEY}": null
}
```

`null` is the documented way to reset a setting to its default.

**Clear one attribute at a time, on one named index at a time.**

- **No wildcards.** `PUT /*/_settings` has an unbounded blast radius across every index including plugin-owned ones, and the documented semantics work against it: "if the request contains any setting that cannot be updated, the entire request is rejected and no settings are changed." In a support engagement this was attempted and **rejected with `403` and a `security_exception` reading `no permissions for []`** — note the empty brackets, which means the error does not even tell you which privilege was missing. The approach was abandoned for an explicit loop over named indexes. The hazard is asymmetric: it fails closed for an ordinary credential, but a sufficiently privileged wildcard would silently rewrite settings on plugin-owned system indexes. Enumerate from step 2 and name indexes explicitly.
- **Use a credential placeholder in anything you paste.** Reconstructed commands in operational channels have been observed carrying inline basic-auth credentials. Keep secrets out of tickets and chat.
- **Do not copy the clearing example from the engine's shard-allocation page.** That example is defective: its path omits the leading underscore on `_settings`, and it wraps the setting in a `persistent` envelope, which belongs to cluster settings rather than index settings. The form above matches the Update Index Settings API and the managed-service guidance.

**Expect:** `{"acknowledged": true}`.

**Rollback this step:**
```
PUT /${INDEX}/_settings
{
  "${SETTING_KEY}": "${PRIOR_VALUE}"
}
```
This restores the filter exactly as it was. It does not undo shard movement that has already happened — relocated shards stay where they went, and moving them back costs another relocation. This is why step 2's saved output matters and why one index at a time limits what you may have to undo.

### Step 4 — Confirm the decider flipped

```
GET /_cluster/allocation/explain?pretty
```

**Expect:** the `filter` decider now returns `YES`, typically explaining that the node passes the include, exclude and require filters.

```
GET /${INDEX}/_settings?include_defaults=true
```

**Expect:** the cleared key is absent or back at its default. Reading with defaults included is how you confirm a reset actually took rather than being masked.

**If the `filter` decider still returns `NO`:** another attribute in the same family is also set. Return to step 2, pick the next one named in the explanation, and repeat. Do not clear the whole family speculatively.

> ⚠️ **If the key is confirmed gone and the shards have not become movable, the filter was not the cause. Stop clearing settings.** This is the exit that the recorded experience of this procedure demands: in a support engagement the key was cleared, the write was acknowledged, a filtered read confirmed the key absent — and the shards stayed exactly as they were. Roughly the whole incident window went into locating, clearing and re-verifying a routing key that turned out not to be the blocker. Clearing an allocation filter is cheap and reversible, so it is worth trying; believing it must have worked is what costs time. Go to **Reach out** rather than widening the settings hunt.

### Step 5 — Wait for movement to begin, then for it to settle

Wait until relocation starts and then completes — confirm with:

```
GET /_cat/recovery?active_only=true&v
GET /_cluster/health
```

**Expect:** `bytes_percent` increasing between samples while relocation is in progress, and finally `relocating_shards` and `unassigned_shards` both at zero with `status` green, or yellow if replicas remain unassigned for an unrelated reason.

Progress is a delta: take two samples with timestamps. If nothing moves and nothing is unassigned, there was no pending movement to release, which is a valid outcome — the filter was blocking a future change rather than a current one.

Where relocation does not begin at all, note that rebalancing is suppressed by default while any shard is unassigned. That appears in the allocation explanation as a statement about the rebalance setting being set to require all shards active, and it means assignment has to finish before rebalancing starts.

## Validation

1. ```
   GET /_cluster/allocation/explain?pretty
   ```
   **Expect:** no `filter` decider returning `NO` for any shard.
2. **The setting is genuinely gone.** Confirm with defaults included rather than by the absence of a key in a plain read — a plain read omits a setting at its default, so absence there proves nothing.
   ```
   GET /${INDEX}/_settings?include_defaults=true&flat_settings=true
   ```
   **Expect:** `${SETTING_KEY}` absent, or present at its default value.
3. **Shard state resolved.**
   ```
   GET /_cluster/health
   ```
   **Expect:** `unassigned_shards` and `relocating_shards` at zero, sustained across two samples a few minutes apart so a transient zero is not mistaken for completion.
4. **Every affected index was treated**, not only the one you came for.
   ```
   GET /_all/_settings?flat_settings=true
   ```
   **Expect:** no `index.routing.allocation.{include,exclude,require}.*` key on any index that should not carry one. Compare against the enumeration you took in step 2 — an index missing from your list was never treated.
5. **Where this was a pre-upgrade fix,** the validation itself is the only proof the upgrade is unblocked, as opposed to one shard being movable in principle.
   ```
   aws opensearch update-domain-config --domain-name <name> \
     --dry-run --dry-run-mode Verbose <the change you intend to make>
   aws opensearch describe-dry-run-progress --domain-name <name> --dry-run-id <id>
   ```
   **Expect:** no `ShardMovementBlocked` entry under `ValidationFailures`. Note the dry run overwrites the previous dry-run record, so capture anything you still need from the earlier one first.

**Exit criteria.** No shard is blocked by the filter decider, the whole-domain audit is clean, health is stable across two samples, and — for the upgrade case — the validation no longer reports shards pinned. A single cleared index is not the exit condition.

## Rollback

Where clearing filters started more relocation than the cluster can absorb, restore the saved values on the indexes you changed, in the order you changed them:

```
PUT /${INDEX}/_settings
{
  "${SETTING_KEY}": "${PRIOR_VALUE}"
}
```

This stops further movement of those shards. It does not return shards that already relocated, and re-pinning them to nodes they have left can leave them unassignable — which is the original fault, inverted. Prefer letting relocation finish over re-pinning mid-flight, unless the cluster is in distress.

## If this does not work

**Reach out when:** the `filter` decider still returns `NO` after every named attribute is cleared; or relocation does not begin and shards remain unassigned with no decider explaining why; or the blocking decider turns out to be `disk_threshold` or `no_valid_shard_copy`; or a configuration change remains blocked after the validation is clean.

**Reach out to:** your support channel for the managed service, or your platform owner for a self-managed cluster.

**Attach:** the full allocation explanation before and after, the saved settings output with the exact key and prior value, the whole-domain settings audit, two timestamped `_cat/recovery` samples, cluster health before and after, and — where relevant — the dry-run validation output before and after.

> ⚠️ **Three things to reach out about, not steps here.** Recovery-throughput tuning is not operator-settable on a managed domain, so read the values and hand them over rather than trying to speed relocation up. Raising the allocation retry ceiling only re-attempts allocation and does not address a cause, so it belongs strictly after the cause is fixed. And **no published source addresses plugin-owned system indexes blocking a configuration change, or the consequences of deleting them** — do not delete a system index to unblock anything.

## Related runbooks

Siblings are referred to by name, which resolves to a directory and is greppable.

- `opensearch-blue-green-upgrade-assessment` — the read-only assessment that identifies this condition and routes here. Run it first; it establishes which decider is responsible.
- `opensearch-shard-density-remediation` — where the pre-upgrade validation reports too many shards rather than pinned shards.

## Verification status

Every step traces to a reputable source. A step resting on inference is reason to leave a procedure unwritten rather than to label it. The basis, what has been observed in practice, the open questions and any superseded claims are in `references/evidence.md` — read it before relying on this runbook for the first time, or before quoting it to anyone.
