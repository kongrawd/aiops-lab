---
name: security-analytics-detector-disable-verification
description: >-
  Confirm that disabling a Security Analytics detector actually stopped execution, and repair
  it when it did not. A detector and the composite monitor (workflow) that executes it are two
  separate records that can disagree: the detector can report disabled while its workflow stays
  enabled and scheduled, so rules keep running and findings keep arriving. Use this after
  disabling any detector, and use it when a detector shows as disabled but findings, CPU or the
  condition it was disabled to stop are continuing. Covers reading the workflow flag back
  through the Alerting plugin API, repairing a stale workflow, and the system-index query that
  returns zero hits regardless of what is actually stored.
metadata:
  agent_types:
    - INCIDENT_TRIAGE
    - INCIDENT_MITIGATION
    - PREVENTION
  skill_type: USER
  service: OpenSearch Security Analytics / Alerting
  applies_to_versions: OpenSearch 2.9 and later, where composite monitors and the workflow APIs exist. The timeout behaviour in Impact was reported on 3.7 and is not established for earlier versions.
  risk_level: medium
  read_only: false   # step 3 replaces a live workflow document
  reversible: yes
  automatable: yes
  verification: documented
  last_verified: 2026-10-14
  last_edited: 2026-10-14
  references:
    - title: Composite monitors — workflow APIs and the enabled field
      url: https://docs.opensearch.org/latest/observing-your-data/alerting/composite-monitors/
      kind: product-doc
    - title: Security Analytics Detector APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/detector-api/
      kind: api-reference
    - title: Security Analytics Alert and Finding APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/alert-finding-api/
      kind: api-reference
    - title: System indexes — Security plugin protection and scrubbed results
      url: https://docs.opensearch.org/latest/security/configuration/system-indices/
      kind: product-doc
    - title: OpenSearch Alerting plugin
      url: https://github.com/opensearch-project/alerting
      kind: upstream-change
---

# Runbook — A detector reports disabled; confirm it actually stopped

## Before you use this

- **Confirm the behaviour against current OpenSearch documentation, or with your support channel, before you execute this.** Plugin behaviour moves with versions, and this runbook states the range it was written for in `metadata.applies_to_versions`. Where this document and the documentation disagree, the documentation is current and this is not.
- **Exercise the procedure in a non-production environment first.** Run it end to end against a detector you can afford to break, confirm each step's output matches what is stated here, and only then run it where it matters.
- **You remain responsible for the outcome in your environment.** This document is offered as-is, with no warranty and no liability for any effect of its use. Step 3 replaces a live workflow document, and the decision to run it is yours.
- **Send corrections back.** A step that failed, output that differed, a version where the behaviour changed, or a clearer command are worth more than a clean document that is quietly wrong. Open an issue or a pull request on the repository this runbook was shared from.

## When to use this

**Trigger.** Either of:

- **Routine** — run this immediately after disabling any detector. A disable that has not been read back on the workflow is not a confirmed disable.
- **Reactive** — a detector shows as disabled, and findings are still arriving for it, CPU has not fallen, or the condition it was disabled to stop is continuing.

**Applies when all of these hold.**

| Condition | How to check | Expected |
|---|---|---|
| The detector reports disabled | `POST /_plugins/_security_analytics/detectors/_search` → its `enabled` field | `false` |
| The detector has an associated workflow | `GET ${CONFIG_INDEX}/_doc/${DETECTOR_ID}` → `_source` → `workflow_ids` | At least one id |
| The Alerting workflow API is reachable | `GET _plugins/_alerting/workflows/${WORKFLOW_ID}` | `200` with the workflow document |

**Does not apply when:**

- **The detector reports `enabled: true`.** It was never disabled, or something re-enabled it. This runbook verifies a disable; it does not perform one.
- **The detector has no `workflow_ids`.** With no workflow there is nothing scheduled to go stale, so continuing findings have a different cause. Investigate the detector's configuration instead.
- **Findings continue while the workflow already reads `enabled: false`.** The flag is not the fault here. Treat it as an investigation into what else is producing findings for that detector.

## Values to obtain

| Name | What it is | How to obtain the current value |
|---|---|---|
| `${DETECTOR_ID}` | The detector whose disable you are verifying | `POST /_plugins/_security_analytics/detectors/_search` with `{"query":{"match_all":{}},"size":100}` → `_id` |
| `${WORKFLOW_ID}` | The composite monitor that actually executes the detector. The documentation defines a workflow ID as synonymous with a composite monitor's monitor ID | **Not returned by the detector API** — read `workflow_ids` from the plugin configuration document, `GET ${CONFIG_INDEX}/_doc/${DETECTOR_ID}` |
| `${DISABLE_TIME}` | When the disable was attempted | Your own change record. Needed to read the workflow's `last_update_time` against something |
| `${CONFIG_INDEX}` | The plugin's detector configuration index, which holds the fields the detector API does not return | Discover it rather than assuming the name — it is plugin-internal and can change:<br>`GET /_cat/indices/.opensearch-sap-*?expand_wildcards=all&v&h=index,docs.count`<br>Pick the one holding detector configuration documents; confirm with `GET /${CONFIG_INDEX}/_mapping` that it carries `workflow_ids` |

`${WORKFLOW_ID}` changes if the detector is recreated, so obtain it from step 1 rather than reusing one from a previous run.

## Impact

**Mechanism.** Detector state and workflow state are two separate records, and nothing reconciles them. A disable writes `enabled: false` to the detector and to the workflow in sequence. Where the write to the workflow does not land, the detector reports disabled and the console agrees, while the workflow stays `enabled: true` and stays scheduled. Rules keep running, findings keep arriving, and fan-out keeps consuming capacity across data nodes — the workflow is cluster-scheduled, so this is not confined to one node.

The reported failure path is a timeout: under cluster load the disable sequence can end before reaching the workflow write. That makes the fault self-reinforcing where the reason for disabling was load, because the load that prompted the disable is also what prevents the disable from completing.

**Size it for this system.** The magnitude is the number of detectors believed to be off that are still running. Enumerate, then read each workflow:

```
POST /_plugins/_security_analytics/detectors/_search
{"query":{"match_all":{}},"size":100}

GET _plugins/_alerting/workflows/${WORKFLOW_ID}
```

Every detector reporting `enabled: false` whose workflow reports `enabled: true` is one unit of this impact, and that count is the figure to carry into a status update. Pair it with the interval: the workflow's `last_update_time` against `${DISABLE_TIME}` gives how long the records have disagreed.

Findings volume is not a usable proxy for the magnitude. A stale workflow consumes execution capacity whether or not its rules match anything, so a detector producing nothing can be fully affected.

**Blast radius.** Every stale workflow continues to execute and to consume fan-out capacity across data nodes. Where detectors were disabled to relieve load, none of the intended relief is in place. Where one was disabled to stop a noisy or incorrect rule, that rule is still firing.

**Recoverability.** The flag is fully recoverable and the repair is reversible. The execution that took place while the two records disagreed is not — it has already consumed capacity and already written whatever it wrote.

**Severity.** Urgent where the detector was disabled to contain an active incident, because the containment is absent and the records say otherwise. Urgent where it was disabled to shed load during saturation. Routine where this is post-disable verification with no incident attached — and still worth running every time, because the disagreement produces no signal of its own.

## Before you start

- [ ] Access to the Security Analytics and Alerting plugin REST APIs, through Dev Tools or an equivalent client, with permission to `PUT` an Alerting workflow
- [ ] Your change record for the disable, which supplies `${DISABLE_TIME}`
- [ ] Cluster health. Where the cluster is saturated, **expect the repair write to fail the way the original disable did.** Read back after every write

**Do not proceed if** the cluster is rejecting writes. The repair is itself a write, and retrying it into a saturated cluster adds load to the condition you are trying to clear. Create headroom first, or reach out.

> ⚠️ **Do not enumerate this from the raw system index.** `.opendistro-alerting-config` is a registered Alerting system index. For a caller without system index access, the Security plugin can substitute an empty index reader, so **a search returns `200 OK` with zero hits even though matching documents exist.** The documentation states it plainly: do not interpret an empty result as proof of access. Treat a zero-hit result as no information at all. This matters because querying that index directly is a circulated method, and it fails silently rather than with an error. Use the plugin API routes below.

## Procedure

### Step 1 — Confirm the detector reports disabled, and get its workflow

```
POST /_plugins/_security_analytics/detectors/_search
{"query":{"match_all":{}},"size":100}
```

**Expect:** one hit per detector carrying `enabled`. Note the detector API does **not** return `workflow_ids` — it is absent on healthy detectors too, so reading it here and finding nothing proves nothing. Get it from the configuration document:

```
GET ${CONFIG_INDEX}/_doc/${DETECTOR_ID}
```

**Expect:** `_source.workflow_ids` with at least one identifier.

> ⚠️ **Validate an empty result before believing it.** This is a protected system index; a caller without access is served an empty reader, so a get can behave as though the document is absent. Confirm the index has documents first:
> ```
> GET /_cat/indices/${CONFIG_INDEX}?v&h=index,docs.count
> ```

**If the detector reports `enabled: true`:** there is no disable to verify. Disable it, then start this runbook again. **Stop here.**

**If it reports `enabled: false`:** record `${DETECTOR_ID}` and `${WORKFLOW_ID}`, and continue to step 2. Record every other detector reporting `enabled: false` as well — validation needs the full set.

### Step 2 — Read the workflow and compare it with the detector

```
GET _plugins/_alerting/workflows/${WORKFLOW_ID}
```

**Expect:** `200` with the workflow document, including `enabled` and `last_update_time`.

**Branch.**

- `enabled: false` → the disable landed on both records. Nothing to repair. **Stop here** for this detector and go to **Validation**.
- `enabled: true` with `last_update_time` **earlier than** `${DISABLE_TIME}` → the workflow has not been written since before your change, so the disable did not reach it. Continue to step 3.
- `enabled: true` with `last_update_time` **later than** `${DISABLE_TIME}` → the workflow was written after your change, by something other than your change. This runbook does not establish what. Flipping the flag would leave that writer in place to undo the repair, so go to **Reach out** with the timestamp and your change record.

### Step 3 — Replace the workflow with the same document, enabled set to false

> ⚠️ **This changes state.** It replaces the whole workflow document rather than patching one field, so a body missing a field changes that field.
>
> **What it is intended to achieve — a scheduled workflow ceasing to execute — is not something these sources establish.** The repair write and its read-back are the least exercised part of this runbook: no engagement record available to this review shows either being performed. Step 4 and the validation are therefore not a formality confirming a known outcome; they are the only evidence you will have that the write did anything.

Build the body from the step 2 response:

1. Take the workflow object itself. Where the response wraps it under a key such as `workflow`, send the inner object rather than the wrapper — the documented `PUT` body is the workflow object (`owner`, `type`, `schedule`, `inputs`, `enabled_time`, `enabled`, `workflow_type`, `name`, and `triggers` where present).
2. Drop response-only envelope fields such as `_id` and `_version` where they appear alongside it.
3. Change `"enabled": true` to `"enabled": false`.
4. **Change nothing else.** Keep `schedule`, `inputs`, `enabled_time`, `triggers` and `name` exactly as returned.

```
PUT _plugins/_alerting/workflows/${WORKFLOW_ID}
{ <the workflow object from step 2, with enabled set to false> }
```

**Expect:** `200`, with the updated workflow returned.

**Keep the step 2 response until this runbook is finished.** It is the body for both the repair and the rollback.

**Rollback this step:**
```
PUT _plugins/_alerting/workflows/${WORKFLOW_ID}
{ <the same workflow object, with enabled set to true> }
```
This returns the workflow to scheduled execution. It does not remove findings written while it was running. It also restores nothing if the body sent in the repair was incomplete — the only true recovery in that case is the unmodified step 2 response, which is why you keep it.

### Step 4 — Read the flag back

**Omitting this step is what lets the fault go unnoticed. It is not optional.**

```
GET _plugins/_alerting/workflows/${WORKFLOW_ID}
```

**Expect:** `"enabled": false`, with `last_update_time` later than when you ran step 3.

**If it still reports `enabled: true`:** the write did not land, which is the same failure that defeated the original disable. Do not retry in a loop — each attempt adds load to the condition causing the failure. Go to **Reach out**, stating that the cluster is not accepting the write.

## Validation

1. ```
   GET _plugins/_alerting/workflows/${WORKFLOW_ID}
   ```
   **Expect:** `"enabled": false`, with `last_update_time` later than step 3.
2. **Findings have stopped.** The flag reading `false` describes configuration; findings describe behaviour, and this is a fault where the two disagree.
   ```
   GET /_plugins/_security_analytics/findings/_search?detector_id=${DETECTOR_ID}&sortOrder=desc&size=1
   ```
   **Expect:** the newest `timestamp` is earlier than the `last_update_time` from validation 1, and it does not advance on a repeat call. A `timestamp` that keeps advancing means execution is continuing despite the flag — go to **Reach out**.
3. **No other detector is in the same state**, not only the one you came for. The disagreement emits no signal, so enumeration is the only way to know how many exist.
   ```
   GET ${CONFIG_INDEX}/_search
   {"size":100,
    "_source":["detector.name","detector.enabled","detector.workflow_ids"],
    "query":{"match_all":{}}}
   ```
   Then for each detector reporting `enabled: false`, read its workflow:
   ```
   GET _plugins/_alerting/workflows/<that workflow id>
   ```
   **Expect:** every workflow belonging to a disabled detector reports `enabled: false`. A workflow still reporting `true` is another instance of this fault.

   > ⚠️ Confirm `docs.count` on that index is non-zero before trusting a clean sweep — a scrubbed read returns zero hits and would report the fleet healthy.

**Exit criteria.** Every detector reporting `enabled: false` has a workflow reporting `enabled: false`, and for each one the newest finding predates that workflow's `last_update_time` and is not advancing. A zero-disagreement enumeration is the exit condition; a single repaired detector is not.

## If this does not work

**Reach out when:** the step 3 write does not land after one read-back; or a workflow's `last_update_time` is later than `${DISABLE_TIME}`; or findings keep advancing after the workflow reads `enabled: false`; or the cluster is rejecting writes before you start.

**Reach out to:** your OpenSearch support channel, through the route your organisation uses for a production cluster.

> One route seen used for a fleet-wide sweep of this condition is a direct search of the alerting configuration index for workflow documents reporting `enabled: true`. It is a protected system index, so the warning in *Before you start* applies with full force: a scrubbed read returns `200 OK` with zero hits and would report the fleet clean. Confirm the index has documents with `GET /_cat/indices/<the alerting config index>?v&h=index,docs.count` before trusting a zero, and prefer the plugin API routes in the procedure above.

**Attach:** the step 1 enumeration, the step 2 workflow document for each affected id, `${DISABLE_TIME}`, the step 4 read-back, the newest finding timestamp per detector, and the disagreement count from validation 3. Say explicitly whether the cluster was under load, because that is the reported trigger for the write not landing.

## Related runbooks

Siblings are referred to by name, which resolves to a directory and is greppable.

- `security-analytics-detector-mapping-audit` — for a detector that runs but matches nothing. That fault looks the same from the outside as this one and is a different repair. Shares the step 1 enumeration.
- `security-analytics-detector-state-change-investigation` — for the step 2 branch where `last_update_time` is later than `${DISABLE_TIME}`. The cause is unknown at that point, so the task is investigation rather than execution.

Where the step 3 write will not land because the cluster is rejecting writes, treat it as the reach-out in *If this does not work* rather than as a branch to follow here.

## Verification status

Every step traces to a reputable source. A step resting on inference is reason to leave a procedure unwritten rather than to label it. The basis, what has been observed in practice, the open questions and any superseded claims are in `references/evidence.md` — read it before relying on this runbook for the first time, or before quoting it to anyone.
