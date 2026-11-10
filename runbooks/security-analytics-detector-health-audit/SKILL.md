---
name: security-analytics-detector-health-audit
description: >-
  Sweep every Security Analytics detector for the silent failure states that produce no error and
  no alert, and route each finding to the procedure that repairs it. A detector can be orphaned
  from its workflow, disabled in name only, carrying more rules than an evaluation can return,
  pointed at an alias with no write index, or pointed at an index that no longer exists - and in
  every case the console reports it as healthy. Use this first, before investigating any single
  detector, when a detector is misbehaving and you do not know which one, after any bulk detector
  change, or as a periodic hygiene pass. This runbook flags and counts; it diagnoses nothing and
  repairs nothing. Read-only throughout.
metadata:
  agent_types:
    - INCIDENT_TRIAGE
    - PREVENTION
  skill_type: USER
  service: OpenSearch Security Analytics / Alerting
  applies_to_versions: All lines. One flag - the rule-count bound - is version-dependent; see "What differs by version". The others are configuration states and are not.
  risk_level: low
  read_only: true
  automatable: yes
  verification: documented
  last_verified: 2026-10-15
  last_edited: 2026-10-15
  references:
    - title: Security Analytics Detector APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/detector-api/
      kind: api-reference
    - title: Security Analytics Alert and Finding APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/alert-finding-api/
      kind: api-reference
    - title: Composite monitors — workflow APIs and the enabled field
      url: https://docs.opensearch.org/latest/observing-your-data/alerting/composite-monitors/
      kind: product-doc
    - title: CAT aliases API — the is_write_index column
      url: https://docs.opensearch.org/latest/api-reference/cat/cat-aliases/
      kind: api-reference
    - title: System indexes and the Security plugin's handling of unauthorized reads
      url: https://docs.opensearch.org/latest/security/configuration/system-indices/
      kind: product-doc
    - title: Doc-level monitor produces false negatives when a detector has more than ten rules
      url: https://github.com/opensearch-project/security-analytics/issues/1656
      kind: upstream-change
    - title: Detector cannot be updated when its alias has no resolvable write index
      url: https://github.com/opensearch-project/alerting/issues/2271
      kind: upstream-change
---

# Runbook — Find which detectors need attention, across the whole fleet

## Before you use this

- **Confirm the behaviour against current OpenSearch documentation, or with your support channel, before you execute this.** Plugin behaviour moves with versions, and this runbook states the range it was written for in `metadata.applies_to_versions`. Where this document and the documentation disagree, the documentation is current and this is not.
- **Exercise the procedure in a non-production environment first.** Every step here is a read, so the risk is low — but run it once where you know the detector set, so you recognise a normal result.
- **You remain responsible for the outcome in your environment.** This document is offered as-is, with no warranty and no liability for any effect of its use.
- **Send corrections back.** A step that failed, output that differed, a version where the behaviour changed, or a clearer command are worth more than a clean document that is quietly wrong. Open an issue or a pull request on the repository this runbook was shared from.

## When to use this

**Trigger.** Any of:

- **Before investigating one detector** — you have a symptom and do not yet know which detector owns it.
- **Proactive** — a periodic hygiene pass, or before citing the detector set as the control for a rule set.
- **After a change** — a bulk detector edit, a rule-set import, an engine upgrade, or a rollover change to any index the detectors read.
- **After an incident** — to establish whether the state that caused it exists anywhere else.

**Applies when all of these hold.**

| Condition | How to check | Expected |
|---|---|---|
| Security Analytics detectors exist | `POST /_plugins/_security_analytics/detectors/_search` | At least one hit |
| You can read the plugin configuration index | Step 2 | Non-zero `docs.count` |
| You can read the Alerting workflow route | `GET /_plugins/_alerting/stats` | `200` |

**What differs by version.** One flag in step 4 is only meaningful on a build that lacks the percolate page-size change. The others are configuration states that no version alters.

| Behaviour | Where the change is absent | Where it is present | How to tell which you have |
|---|---|---|---|
| Rules retrieved per evaluation | At most 10 matching rules; the rest are dropped with no error, warning or log entry | Up to the engine's search ceiling, which the implementation sets to 10,000 | `GET /_cat/plugins?v&h=name,component,version` → read the `opensearch-alerting` row, then check that version against the change linked in `metadata.references` |

**Read your own build rather than trusting a version written here.** Backports to the older lines were open at the time of writing and that state moves. Where the rule-count flag fires, `security-analytics-rule-coverage-audit` carries the behavioural check that settles it for your build; this sweep only tells you which detectors are worth that check.

**Does not apply when:**

- **You already know which detector and which fault.** Go straight to the runbook that owns it — see **Related runbooks**. This sweep costs several calls per detector and tells you nothing new once the fault is identified.
- **The question is whether a detector that evaluates its rules should have matched your data.** This sweep establishes configuration state, not detection correctness. That is `security-analytics-detector-mapping-audit`.

## Values to obtain

| Name | What it is | How to obtain the current value |
|---|---|---|
| `${CONFIG_INDEX}` | The plugin's detector configuration index, which holds the fields the detector API does not return | Discover it rather than assuming the name — it is plugin-internal and can change:<br>`GET /_cat/indices/.opensearch-sap-*?expand_wildcards=all&v&h=index,docs.count`<br>Pick the one holding detector configuration documents; confirm with `GET /${CONFIG_INDEX}/_mapping` that it carries `workflow_ids` |
| `${DETECTOR_ID}` | A detector flagged by this sweep | `POST /_plugins/_security_analytics/detectors/_search` → `hits.hits[]._id` |
| `${ALIAS}` | The data source of a flagged detector, where that data source is an alias | Same response → `inputs[0].detector_input.indices[0]` |

## Impact

**Mechanism.** Every state this sweep looks for is silent by construction. A detector orphaned from its workflow still saves cleanly. A detector whose disable reached only the detector record still reports disabled. A rule set past the evaluation page bound still lists every rule as enabled. An alias with no write index still resolves. In each case the failure is the *absence* of something — a finding, an execution, an evaluated rule — and absence raises no error, writes no log line the console surfaces, and triggers no alert. So the fleet's reported health is not evidence of its actual health, and the only way to know which detectors are affected is to read the states directly.

**Size it for this system.** The magnitude is the number of detectors carrying at least one flag, against the total. Both come from step 1 and step 3:

```
POST /_plugins/_security_analytics/detectors/_search
{"query":{"match_all":{}},"size":100}
```

Record the total from `hits.total`, then the flagged count from step 3's table. Carry both figures, not the ratio alone — "4 of 7" and "40 of 70" call for different responses.

**Blast radius.** Per flagged detector, the whole rule set it carries, for the period the flag has been true. Where a flag means the detector was not evaluating, every event in scope for those rules passed unexamined. Note that this cannot be sized from the detector: the sweep establishes the state now, not when it started.

**Recoverability.** The configuration states are all recoverable, and each owning runbook says how. **The missed detection is not.** Findings are not generated retroactively, so a detector that produced nothing for a week does not produce that week's findings once repaired. This is why the sweep is worth running before anyone asks for it.

**Severity.** Urgent where a flagged detector is the control relied on for a compliance obligation or for coverage someone has attested to, and urgent where the flagged count is a large share of the fleet — that pattern suggests a common cause rather than per-detector drift. Routine as a hygiene pass. It cannot be deferred on the grounds that nothing has alerted: no alert is the symptom.

## Before you start

- [ ] Permission to call the Security Analytics detector, findings and alerts APIs, the Alerting workflow route, and `GET /_cat/*`
- [ ] Permission to read the plugin configuration index. Without it the sweep cannot see the two fields that matter most, and the failure is silent — see the warning below
- [ ] Awareness that `plugins.security_analytics.filter_by_backend_roles`, where enabled, limits the detectors you can see to those created by users sharing a backend role — so your enumeration may be a subset. Read it with `GET /_cluster/settings?include_defaults=true` and **record the value with your results**, because a sweep over a subset that reports "fleet clean" is worse than no sweep

> ⚠️ **Do not use the detector `_execute` route anywhere in this sweep.** It executes rather than inspects, which would make this procedure something other than read-only, and the public report states it runs the same code path as a scheduled evaluation and carries the same page-bound limitation — so it cannot verify the thing you would be invoking it to verify.

> ⚠️ **A zero or an empty result from a plugin-internal index is not evidence of absence.** The published documentation states that for a caller without system-index access the Security plugin can substitute an empty index reader, so a search returns `200 OK` with zero hits even though documents exist, and that an empty result must not be read as proof of access. **Step 2 therefore proves the read works before step 3 interprets anything.** A zero you have not validated is the most dangerous output in this procedure, because across a whole fleet it reads as a clean bill of health.

## Procedure

### Step 1 — Enumerate the detectors, and confirm the enumeration is complete

```
POST /_plugins/_security_analytics/detectors/_search
{"query":{"match_all":{}},"size":100}
```

**Expect:** one hit per detector. Record `_id`, `_source.enabled`, the rule arrays, `triggers`, and `inputs[0].detector_input.indices[0]`.

Sum the rule arrays per detector. **Sum all of these key shapes** — the published API documentation uses different shapes across its own examples, and counting only one of them undercounts:

- `inputs[0].detector_input.custom_rules[]`
- `inputs[0].detector_input.pre_packaged_rules[]`
- `inputs[0].input.custom_rules[]` and `inputs[0].input.pre_packaged_rules[]`
- `inputs[0].rules[]`

**If `hits.total` is zero:** there are no detectors to audit. Confirm that is expected before concluding anything — a backend-role filter can produce this result for a caller who simply cannot see them. **Stop here.**

**If the number of hits equals the `size` you requested:** there are more detectors than you have read. Page with `from` until the returned count is below `size`. A result at the requested size is not the end of the list.

**Otherwise:** continue to step 2.

> ⚠️ **Two fields that identify two of the flags are not returned by this route.** `monitor_id` and `workflow_ids` are omitted from the detector object on healthy detectors as well as broken ones. Reading them here and finding them absent proves nothing — it is the normal result, and treating absence as a signature flags every detector in the fleet. They live in the configuration document, which is step 2.

### Step 2 — Prove you can read the configuration index, then read it

Discover `${CONFIG_INDEX}` using the row in **Values to obtain**, then establish that the read works before trusting anything it returns:

```
GET /_cat/indices/${CONFIG_INDEX}?v&h=index,docs.count
```

**Expect:** a non-zero `docs.count`.

```
GET ${CONFIG_INDEX}/_search
{"size":100,
 "_source":["detector.name","detector.enabled","detector.workflow_ids","detector.monitor_id"],
 "query":{"match_all":{}}}
```

**Expect:** one hit per detector, each carrying `detector.monitor_id` and `detector.workflow_ids` under `_source`.

**If `docs.count` is non-zero and this search returns zero hits:** you are being served a scrubbed result. **Stop here** and obtain a read with the required access. Do not continue — the two flags that depend on this read would come back clean for the entire fleet.

**If the hit count here disagrees with step 1's:** record both. The configuration index can hold documents for detectors the detector API does not return to you, and the reverse indicates an incomplete read. Either way the sweep is over a subset, and that belongs in your results.

### Step 3 — Cross-reference the two reads and flag each detector

This is the sweep. Work through the table once per detector. **Every row is a flag, not a diagnosis** — the sweep's output is a list of detectors and the runbook each one needs.

| Flag | Signature | Where to go |
|---|---|---|
| **Orphaned workflow** | `monitor_id` populated and `workflow_ids` empty or absent | `security-analytics-detector-workflow-recovery` |
| **Rule set past the page bound** | rule-array sum greater than 10, on a build lacking the change in **What differs by version** | `security-analytics-rule-coverage-audit` |
| **Triggers but no rules** | rule-array sum of zero while `triggers` is non-empty | Operator action — see below |
| **Data source is an alias** | `inputs[0].detector_input.indices[0]` resolves to an alias | Step 5 |
| **State disagreement** | detector `enabled: false` — continue to step 4 | `security-analytics-detector-disable-verification` |

**On "triggers but no rules":** the recorded resolution is to delete the detector, because the plugin creates no monitor or workflow for it and it produces no findings or alerts. No sibling runbook owns this, and deletion is a change that needs its own approval — so this sweep reports it and stops. Confirm the detector is genuinely unused before anyone deletes it, and note that this state has been observed to break the shared Security Analytics overview page for *every* detector, not only the one at fault.

**If no detector carries any flag:** record that, with the enumeration counts from steps 1 and 2 and the backend-role filter value, and go to **Validation**. A clean sweep is a real result — but only once validation confirms the reads were not scrubbed or truncated.

### Step 4 — For each flagged detector, establish whether it is quiet, failing, or running

A flag is a configuration state. These two reads tell you what that state is costing, and they separate the three outcomes that look identical from the console.

```
GET /_plugins/_security_analytics/findings/_search?detector_id=${DETECTOR_ID}&sortOrder=desc&size=1
GET /_plugins/_security_analytics/alerts?detector_id=${DETECTOR_ID}&alertState=ERROR&size=20
```

**Expect:** from the first, the newest `timestamp`, which should advance across repeated calls; from the second, `alerts[]` with `state`, `error_message`, `start_time` and `trigger_name`, plus `total_alerts`.

> ⚠️ **Read `total_findings`, not the length of the `findings` array.** A request with `size=0` returns an empty array alongside a non-zero total, so array length reports no detections when there are many.

**Branch.**

- **Error-state alerts present** → the evaluation is being rejected, not silently truncated. Go to `security-analytics-percolate-failure-investigation`, which reads the cause from the error record.
- **No error-state alerts and the newest finding timestamp is not advancing** → the detector is quiet. Whether that is correct depends on whether anything should have matched, which this sweep cannot establish. Go to `security-analytics-detector-mapping-audit`.
- **Findings advancing** → the detector is producing. Its flag still stands and still needs the owning runbook, but it is not blind over the window you just read.
- **Detector `enabled: false` while findings are still arriving** → the disable did not take effect. Go to `security-analytics-detector-disable-verification`.

**A quiet detector is not a harmless one.** A detector whose workflow is still scheduled consumes execution capacity whether or not its rules match anything, so "produces nothing" and "costs nothing" are different claims.

### Step 5 — For each detector whose data source is an alias, read the write-index designation

```
GET /_cat/aliases/${ALIAS}?v&h=alias,index,is_write_index
```

**Expect:** one row per member index.

> ⚠️ **An unset `is_write_index` prints as a literal `-`.** Not `false`, not an empty column. A check that greps for `false`, or tests for an empty field, misreads every row.

**Branch. Four states matter, and they are distinguishable only by reading the flag _and_ counting members.**

- **Two or more rows, every one showing `-`** → the alias has no write index. Flag it and go to `security-analytics-alias-write-index-recovery`.
- **Exactly one row showing `-`** → healthy. A single referenced member behaves as the implicit write index, so no explicit designation is needed.
- **Exactly one row explicitly reading `false`** → **also a failure.** An explicit `false` overrides the implicit behaviour, leaving the alias with no write index even with one member. Flag it and go to the same runbook.
- **One row reading `true`** → the designation is present.

Then confirm the data source resolves to something that exists:

```
GET ${ALIAS}/_mapping
```

**Expect:** a mapping per member index.

**If this returns `404` with `index_not_found_exception`:** the detector is pointed at an index or alias that does not exist. The whole monitor run fails on every schedule, not just one rule, so this is both a detection gap and a standing cost. No sibling runbook owns it; the operator action is to repoint the detector at a live data source or delete it, and either is a change needing its own approval.

**A correctly designated write index does not close every gap.** Where an index is created by a shipper and attached to the alias afterwards, records written before the attach are never examined, because the detector reads from the attach point rather than from the index's first record. That window is not visible to this sweep at an arbitrary hour — it is only visible during the rollover. Where a detector reads a daily alias, treat that as an open question this sweep does not answer and see `security-analytics-rollover-blind-window-remediation`.

## Validation

This procedure produces a judgement about a whole fleet, so validation confirms the judgement is trustworthy rather than that a system recovered. Run all four before reporting any result, and especially before reporting a clean one.

1. **The detector enumeration was not truncated.** Compare the `size` you requested in step 1 against the number of hits returned.
   ```
   POST /_plugins/_security_analytics/detectors/_search
   {"query":{"match_all":{}},"size":100}
   ```
   **Expect:** a hit count strictly below the requested `size`, or a documented paging run that reached the end. A result at the requested size means there is more to read.
2. **The configuration read was not scrubbed.** This is the check that decides whether two of the five flags mean anything.
   ```
   GET /_cat/indices/${CONFIG_INDEX}?v&h=index,docs.count
   ```
   **Expect:** a non-zero `docs.count` alongside the non-zero hit count from step 2. A non-zero `docs.count` beside a zero-hit search means the read was scrubbed and the orphaned-workflow flag is meaningless for every detector.
3. **The visible detector set is the whole set.**
   ```
   GET /_cluster/settings?include_defaults=true
   ```
   **Expect:** the value of `plugins.security_analytics.filter_by_backend_roles`. Where it is enabled, state in your results that the sweep covered only the detectors your backend role can see.
4. **The two enumerations agree.** Compare step 1's hit count against step 2's. **Expect:** the same number. A difference is itself a finding and goes in the results rather than being reconciled silently.

**Exit criteria.** You can state, for every detector the enumeration returned, which flags it carries and which runbook each flag routes to — plus the three figures that bound the sweep: detectors enumerated, detectors flagged, and whether the backend-role filter was limiting what you could see. A single flagged detector is not the exit condition; the audit is complete when every detector has a record, including the ones with no flags.

## If this does not work

**Reach out when:** the configuration index returns a validated zero, so two flags cannot be evaluated at all; or the two enumerations disagree and neither can be shown complete; or a detector carries a flag that none of the routes below fit; or the flagged share of the fleet is large enough to suggest a common cause rather than per-detector drift.

**Reach out to:** your OpenSearch support channel, through the route your organisation uses for a production cluster.

**Attach:** the detector count and the flagged count, the per-detector flag table from step 3, the `docs.count` and hit count that show the configuration read was or was not scrubbed, the backend-role filter value, the engine and alerting plugin versions from `GET /` and `GET /_cat/plugins?v&h=name,component,version`, and the requested-versus-returned sizes that show the enumeration was complete. State explicitly that the `_execute` route was not used and why.

## Related runbooks

Siblings are referred to by name, which resolves to a directory and is greppable.

- `security-analytics-detector-workflow-recovery` — where `monitor_id` is populated and `workflow_ids` is empty. Carries the backup-delete-recreate procedure and its irreversibility caveats.
- `security-analytics-detector-disable-verification` — where a detector reports disabled. Reads the workflow flag back and repairs a stale one.
- `security-analytics-rule-coverage-audit` — where a detector carries more than ten rules. Establishes whether your build truncates the evaluation.
- `security-analytics-alias-write-index-recovery` — where the data source is an alias with no explicit write index.
- `security-analytics-detector-mapping-audit` — where a detector is enabled, not failing, and producing nothing. This sweep establishes configuration state; that one establishes whether an evaluated rule could have matched. Shares the step 1 enumeration.
- `security-analytics-percolate-failure-investigation` — where error-state alerts exist. The evaluation is being rejected rather than silently truncated.
- `security-analytics-rule-cost-audit` — where detectors are producing findings normally but driving sustained load. Same enumeration, a cost question rather than a health one.
- `security-analytics-rollover-blind-window-remediation` — where a detector reads a daily alias and the concern is the records written between index creation and alias attachment.

`security-analytics-wayfinder` is a skill rather than a runbook, and it is the other half of this one. It starts from a single detector's error-state alerts and identifies which component is at fault — detector record, workflow, query index, data source, field mappings, rule set — then searches this collection for whatever procedure currently covers it. This sweep establishes *which detectors* need attention; that identifies *what is wrong* with one of them. Use it when the detector is known and the cause is not; use this when the reverse holds.

## Verification status

Every step traces to a reputable source. A step resting on inference is reason to leave a procedure unwritten rather than to label it. The basis, what has been observed in practice, the open questions and any superseded claims are in `references/evidence.md` — read it before relying on this runbook for the first time, or before quoting it to anyone.
