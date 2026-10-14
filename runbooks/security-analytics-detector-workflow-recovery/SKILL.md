---
name: security-analytics-detector-workflow-recovery
description: >-
  Recover a Security Analytics detector whose underlying alerting workflow has been orphaned.
  Saving a detector with an empty rule list deletes the workflow that orchestrates its monitors
  and clears the reference to it; a later save that re-adds rules creates monitors but never
  recreates the workflow, so the detector reports a successful save while producing no alerts.
  Use this when a detector saves cleanly but no alerts appear, when stopping or starting a
  detector reports that its underlying workflow was not found, or after any save that emptied a
  detector's rule list. Recovery requires deleting and recreating the detector, so read the
  impact section before starting.
metadata:
  agent_types:
    - INCIDENT_TRIAGE
    - INCIDENT_MITIGATION
  skill_type: USER
  service: OpenSearch Security Analytics / Alerting
  applies_to_versions: Reported as affecting every line with workflow support, 2.12 and later. No fix exists. See "What differs by version".
  risk_level: high
  read_only: false
  reversible: no
  automatable: partial
  verification: documented
  last_verified: 2026-10-14
  last_edited: 2026-10-14
  references:
    - title: Clearing a detector's rules causes the underlying workflow to be orphaned
      url: https://github.com/opensearch-project/security-analytics/issues/1720
      kind: upstream-change
    - title: Security Analytics Detector APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/detector-api/
      kind: api-reference
    - title: Creating detectors — the detector and workflow relationship
      url: https://docs.opensearch.org/latest/security-analytics/sec-analytics-config/detectors-config/
      kind: product-doc
    - title: Working with detectors — stopping and starting a detector
      url: https://docs.opensearch.org/latest/security-analytics/usage/detectors/
      kind: product-doc
    - title: Composite monitors — workflow APIs
      url: https://docs.opensearch.org/latest/observing-your-data/alerting/composite-monitors/
      kind: product-doc
    - title: Security Analytics Alert and Finding APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/alert-finding-api/
      kind: api-reference
---

# Runbook — A detector saves successfully but produces no alerts

## Before you use this

- **Confirm the behaviour against current OpenSearch documentation, or with your support channel, before you execute this.** Plugin behaviour moves with versions, and this runbook states the range it was written for in `metadata.applies_to_versions`. Where this document and the documentation disagree, the documentation is current and this is not.
- **Exercise the procedure in a non-production environment first.** This is not optional advice here. The recovery deletes a detector, and no public source states whether its findings and alert history survive that. Run it once on a detector you can afford to lose before running it on one you cannot.
- **You remain responsible for the outcome in your environment.** This document is offered as-is, with no warranty and no liability for any effect of its use. Step 4 is irreversible, and the decision to run it is yours.
- **Send corrections back.** A step that failed, output that differed, a version where the behaviour changed, or a clearer command are worth more than a clean document that is quietly wrong. Open an issue or a pull request on the repository this runbook was shared from.

## When to use this

**Trigger.** Any of:

- A detector saves without error but no alerts appear for it, while findings may still be arriving.
- Stopping or starting a detector fails with a message that the underlying workflow associated with that detector was not found, advising deletion and recreation.
- Any save that left a detector's rule list empty, whether or not rules were added back afterwards.

**Applies when all of these hold.**

| Condition | How to check | Expected |
|---|---|---|
| The detector has monitors | `GET ${CONFIG_INDEX}/_doc/${DETECTOR_ID}` → `_source` → `monitor_id` | Non-empty |
| The detector has no workflow | Same document → `workflow_ids` | Empty array |
| The detector currently has rules | `GET /_plugins/_security_analytics/detectors/${DETECTOR_ID}` → the rule arrays | At least one rule |

That combination — monitors present, workflow absent, rules present — is the signature. A detector with no rules and no workflow is simply unconfigured, not orphaned.

> ⚠️ **The two fields that identify this state are not returned by the detector API.** `GET /_plugins/_security_analytics/detectors/${DETECTOR_ID}` returns `{_id, _version, detector:{…}}`, and the `detector` object **omits `monitor_id` and `workflow_ids` entirely — on healthy detectors as well as broken ones.** This has been checked against both on two separate clusters. Two consequences:
> - **Reading those fields from the detector API and finding them absent proves nothing.** It is the normal result. Treating absence there as the signature produces a false positive on every detector in the fleet.
> - **The fields live in the plugin's configuration document**, which is where the checks above point. That is a system-index read, so the caveat in *Before you start* applies to it.

**What differs by version.** There is no fix to upgrade to, so the version question is only about whether the defect can occur at all.

| Behaviour | Below workflow support | From workflow support onward | How to tell which you have |
|---|---|---|---|
| Saving a detector with an empty rule list | No workflow exists to orphan | The workflow is deleted and its reference cleared; a later save that re-adds rules creates monitors but no workflow | `GET /` → `version.number`, and `GET /_cluster/settings?include_defaults=true` → `plugins.security_analytics.enable_workflow_usage` |

The reported scope is every line with workflow support, 2.12 and later. **That scope is the reporter's statement, not a maintainer confirmation**, and no fix is available on any line — so treat your own build as affected unless you can show otherwise, and do not wait for an upgrade to resolve it.

**Does not apply when:**

- **`workflow_ids` is populated.** The workflow exists. A detector producing no alerts then has a different cause — see **Related runbooks**.
- **The detector has no rules.** Add rules first. An unconfigured detector is not an orphaned one, though note that saving rules onto it will not create the missing workflow either.
- **Findings are arriving and alerts are expected but triggers are misconfigured.** This runbook addresses a missing orchestrator, not a trigger condition that never matches.

## Values to obtain

| Name | What it is | How to obtain the current value |
|---|---|---|
| `${DETECTOR_ID}` | The affected detector | `POST /_plugins/_security_analytics/detectors/_search` → `hits.hits[]._id` |
| `${DETECTOR_BODY}` | The full detector definition, saved to a file before anything is deleted | `GET /_plugins/_security_analytics/detectors/${DETECTOR_ID}` |
| `${CONFIG_INDEX}` | The plugin's detector configuration index, which holds the fields the detector API does not return | Discover it rather than assuming the name — it is plugin-internal and can change:<br>`GET /_cat/indices/.opensearch-sap-*?expand_wildcards=all&v&h=index,docs.count`<br>Pick the one holding detector configuration documents; confirm with `GET /${CONFIG_INDEX}/_mapping` that it carries `workflow_ids` |

## Impact

**Mechanism.** A detector's rules are converted into alerting monitors, and a workflow orchestrates those monitors and aggregates their output into alerts. When a save leaves the detector with no rules, there are no monitors to keep, so the plugin deletes the workflow and clears the detector's reference to it. The damage is done by the *next* save: the code path that maintains a workflow is guarded by a test that reads the detector's workflow reference, and that reference is now empty, so re-adding rules creates monitors and takes no branch that recreates a workflow. The detector ends up with monitors that have no orchestrator.

Nothing reports this. The save returns success, the detector lists its rules, and the only outward sign is the absence of alerts. Where the deletion itself failed rather than succeeded, the reported alternative outcome is an orphaned workflow that keeps executing against stale monitor references.

**Size it for this system.** The magnitude is the number of detectors in this state. Enumerate and compare the two fields:

```
POST /_plugins/_security_analytics/detectors/_search
{"query":{"match_all":{}},"size":100}
```

Every detector with a non-empty `monitor_id`, an empty `workflow_ids`, and at least one rule is one unit of this impact. For each, establish how long alerts have been absent by comparing the newest alert against the newest finding:

```
GET /_plugins/_security_analytics/alerts?detector_id=${DETECTOR_ID}&size=1
GET /_plugins/_security_analytics/findings/_search?detector_id=${DETECTOR_ID}&sortOrder=desc&size=1
```

Findings advancing while alerts do not is the shape of this fault, and the gap between the two newest timestamps bounds how long it has been in place.

**Blast radius.** Every rule on the affected detector. Monitors may still be producing findings, so the loss is specifically the alerting path — anyone watching alerts sees nothing, while anyone querying findings directly sees data. That split is what makes this fault survive casual review.

**Recoverability.** The detector is recoverable only by deletion and recreation, and that is not a reversal. The recreated detector receives a new identifier, and **no public source states whether findings and alert history recorded under the previous identifier remain retrievable**. Treat that as unknown rather than assuming either outcome, and verify it in a non-production domain before you need the answer. Alerts not raised while the workflow was missing are not raised retrospectively.

**Severity.** Urgent whenever confirmed, because the detector presents as healthy and the alerting path it exists to provide is absent. The absence of alerts is the symptom, so there is no second signal that will raise this for you.

## Before you start

- [ ] Permission to read, delete and create Security Analytics detectors, and to read the alerting workflow route
- [ ] Somewhere to save `${DETECTOR_BODY}` outside the cluster. This is the only copy of the configuration once the detector is deleted
- [ ] An export of any findings or alerts you need to keep, taken **before** deletion — they are retrievable from a broken detector, and expected to become unreachable under the new identifier afterwards
- [ ] A decision, made in advance, about whether losing the detector's historical findings and alert linkage is acceptable — because that outcome is not publicly documented and you may discover it only afterwards
- [ ] Confirmation of who needs to know that alerts for this detector have been absent, and for how long

**Do not proceed if** you cannot save the detector body first, or cannot accept an unknown outcome for the detector's history. Reach out instead and ask for the retention behaviour to be confirmed before deleting anything.

## Procedure

### Step 1 — Confirm the signature

Read the plugin's configuration document, which is the only place these two fields appear. For one detector:

```
GET ${CONFIG_INDEX}/_doc/${DETECTOR_ID}
```

For a fleet sweep:

```
GET ${CONFIG_INDEX}/_search
{"size":100,
 "_source":["detector.name","detector.enabled","detector.workflow_ids","detector.monitor_id"],
 "query":{"match_all":{}}}
```

**Expect:** per detector, `monitor_id` and `workflow_ids` under `_source`.

> ⚠️ **Validate a zero or an empty result before believing it.** This is a protected system index, and the published documentation states that for a caller without access the Security plugin can substitute an empty reader, so **a search returns `200 OK` with zero hits even though documents exist**, and a get can behave as though the document is absent. Confirm the index has documents first:
> ```
> GET /_cat/indices/${CONFIG_INDEX}?v&h=index,docs.count
> ```
> A non-zero `docs.count` with a zero-hit search means you are being served a scrubbed result — stop and get a read with the required access rather than concluding the fleet is healthy.

**Branch.**

- **Non-empty `monitor_id`, empty `workflow_ids`, at least one rule** → this is the situation. Continue to step 2.
- **Non-empty `workflow_ids`** → the workflow exists. **Stop here** and see **Related runbooks**.
- **No rules at all** → the detector is unconfigured rather than orphaned. Adding rules is the fix for that, and it will not create a workflow, so return here afterwards and re-check.

Note that `workflow_ids` does not appear in the published Detector API documentation. The field name is established from the published plugin source and from the public defect report, not from the API reference — so treat a missing key and an empty array as the same finding rather than expecting a documented shape.

### Step 2 — Confirm the workflow is genuinely absent, not merely unreferenced

Where any other detector reports a workflow identifier, read one to confirm the route works in this cluster and that you are reading the right field:

```
GET _plugins/_alerting/workflows/<a workflow id from a healthy detector>
```

**Expect:** `200` with a workflow document.

For the affected detector there is no identifier to read, and that absence is the finding. This step exists to rule out the alternative explanation that the route is unavailable or the field is being misread.

### Step 3 — Save the detector definition

```
GET /_plugins/_security_analytics/detectors/${DETECTOR_ID}
```

**Expect:** the full detector definition. Write it to a file outside the cluster.

Keep it until the recreated detector has passed validation. After step 4 this file is the only record of the configuration, including rule selections someone may have spent considerable time assembling.

### Step 4 — Delete the detector

> ⚠️ **This changes state irreversibly.** Deletion cannot be undone. The detector's identifier is gone, and whether its findings and alert history remain retrievable afterwards is not publicly documented.

**Re-adding rules does not repair this state.** The public defect report records exactly that attempt and its outcome: a new monitor is created and the workflow reference stays empty. Do not spend an outage trying it.

```
DELETE /_plugins/_security_analytics/detectors/${DETECTOR_ID}
```

**Expect:** a success response.

**Rollback this step:** there is none. Deletion has no inverse. The only recovery path is step 5, recreating from `${DETECTOR_BODY}`, which produces a new detector with a new identifier rather than restoring the original. This is why step 3 is mandatory and why the pre-flight asks you to accept the unknown history outcome before starting.

### Step 5 — Recreate the detector from the saved definition

```
POST _plugins/_security_analytics/detectors
{ <the detector definition from step 3, with response-only fields such as the identifier removed> }
```

**Expect:** a success response carrying a new detector identifier and a populated `workflow_ids`.

> ⚠️ **Recreating the detector does not stop an orphaned workflow that is still running.** Where the earlier deletion left a workflow behind rather than removing it, that workflow has been observed continuing to execute and to deliver notifications downstream carrying the old workflow identifier — after the original detector was disabled **and** after a replacement was created. At the same time the alerts page showed nothing for it. So a quiet alerts page is not evidence that nothing is firing, and you may now have both a working detector and an orphan producing duplicate notifications. Check your notification destination for messages carrying a workflow identifier that no detector claims, and reach out about those separately; there is no published operator route to remove one.

**If `workflow_ids` is empty on the newly created detector:** creation did not establish a workflow, which is a different problem from the one this runbook addresses. Go to **Reach out** with both the request body and the response.

## Validation

1. **The recreated detector has a workflow.** Read the configuration document, not the detector API — the API does not return either field.
   ```
   GET ${CONFIG_INDEX}/_doc/<the new detector id>
   ```
   **Expect:** `_source` carries a non-empty `workflow_ids` and a non-empty `monitor_id`.
2. **The workflow exists and is reachable.**
   ```
   GET _plugins/_alerting/workflows/<the new workflow id>
   ```
   **Expect:** `200` with a workflow document whose `enabled` matches the detector's intended state.
3. **Alerts resume.** The workflow reference is configuration; alerts are behaviour, and this fault is one where the two disagreed.
   ```
   GET /_plugins/_security_analytics/alerts?detector_id=<the new detector id>&size=1
   ```
   **Expect:** over at least two of the detector's schedule intervals, with data arriving that should match a rule, an alert appears. Read the interval from `schedule.period`.

   Confirm findings are advancing too, so that an absence of alerts is not simply an absence of matching data:
   ```
   GET /_plugins/_security_analytics/findings/_search?detector_id=<the new detector id>&sortOrder=desc&size=1
   ```
   **Expect:** the newest `timestamp` advances across repeated calls. Read `total_findings`, not the length of the `findings` array — a `size=0` request returns an empty array beside a non-zero total.
4. **No other detector is in the same state.** This condition produces no signal of its own, so enumeration is the only way to know how many exist — fixing the one that was reported leaves the others in place.
   ```
   GET ${CONFIG_INDEX}/_search
   {"size":100,
    "_source":["detector.name","detector.enabled","detector.workflow_ids","detector.monitor_id"],
    "query":{"match_all":{}}}
   ```
   **Expect:** no detector with a populated `monitor_id` and an empty `workflow_ids`.

   > ⚠️ Validate the result before believing it. This is a protected system index, and a caller without access is served an empty reader — a search returns `200 OK` with zero hits while documents exist. Confirm the index has documents first:
   > ```
   > GET /_cat/indices/${CONFIG_INDEX}?v&h=index,docs.count
   > ```
   > A non-zero `docs.count` with a zero-hit search means the sweep is scrubbed, not clean.

**Exit criteria.** Every detector has a non-empty `workflow_ids`, each of those workflows returns `200`, and the recreated detector has produced an alert or has been shown to be receiving matching data. A single repaired detector is not the exit condition.

## Prevention

The supported way to take a detector out of service is the status change, not emptying its rule list.

```
PUT /_plugins/_security_analytics/detectors/${DETECTOR_ID}
{ <the full detector body, with "enabled": false> }
```

The documentation describes stopping and starting a detector as the intended action for this, and the `enabled` field as what sets a detector active or inactive. The published documentation also states that at least one rule must be specified for a detector — though that minimum appears in the creation section and is not enforced on update, which is precisely why the empty-rule save goes through and causes this.

**Never save a detector with both rule arrays empty**, even transiently, even as a step towards replacing its rule set. Replace rules in a single save that leaves at least one rule in place throughout.

Note one asymmetry worth knowing before you touch a detector in this state: the workflow-not-found error is raised only when the detector is enabled. A disabled detector proceeds without it, so a disabled detector can be orphaned quietly and reveal it only when someone tries to start it.

## If this does not work

**Reach out when:** the recreated detector has an empty `workflow_ids`; or alerts do not resume after two schedule intervals with findings confirmed to be advancing; or you need the findings-retention behaviour confirmed before deleting anything.

**Reach out to:** your OpenSearch support channel, through the route your organisation uses for a production cluster.

**Attach:** the step 1 enumeration showing `monitor_id` populated against `workflow_ids` empty, the saved detector body, the newest alert and newest finding timestamps before and after, the engine version and the `enable_workflow_usage` setting value, and the count of other detectors found in the same state. Say explicitly that re-adding rules was not attempted as a repair, and why.

## Related runbooks

Siblings are referred to by name, which resolves to a directory and is greppable.

- `security-analytics-detector-health-audit` — the read-only sweep that finds this condition, and others, across every detector. Run it first to size the problem before deleting anything.
- `security-analytics-detector-disable-verification` — where a detector reports disabled and the question is whether it actually stopped.
- `security-analytics-alias-write-index-recovery` — where a detector produces neither findings nor alerts and its data source is an alias.
- `security-analytics-rule-coverage-audit` — where findings arrive for some rules but not others.

## Verification status

Every step traces to a reputable source. A step resting on inference is reason to leave a procedure unwritten rather than to label it. The basis, what has been observed in practice, the open questions and any superseded claims are in `references/evidence.md` — read it before relying on this runbook for the first time, or before quoting it to anyone.
