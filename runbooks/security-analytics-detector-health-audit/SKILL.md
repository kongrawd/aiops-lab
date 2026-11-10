---
name: security-analytics-detector-health-audit
description: >-
  Sweep every Security Analytics detector for the silent failure states that produce no error and
  no alert, and route each finding to the procedure that repairs it. A detector can be orphaned
  from its workflow, disabled in name only, carrying more rules than an evaluation can return,
  pointed at an alias with no write index, pointed at an index that no longer exists, sharing an
  alias with a second detector so each save overwrites the other's field aliases, or reading
  indices created without the field aliases its rules need - and in every case the console reports
  it as healthy. Use this first, before investigating any single detector, when a detector is
  misbehaving and you do not know which one, when field mappings disappear after a rollover, after
  any bulk detector change, or as a periodic hygiene pass. This runbook flags and counts; it
  diagnoses nothing and repairs nothing. Read-only throughout.
metadata:
  agent_types:
    - INCIDENT_TRIAGE
    - PREVENTION
  skill_type: USER
  service: OpenSearch Security Analytics / Alerting
  applies_to_versions: All lines. Two behaviours are version-dependent - the rule-count bound, and how a field-mappings save picks the index template; see "What differs by version". The remaining flags are configuration states that no version alters.
  risk_level: low
  read_only: true
  automatable: partial
  verification: documented
  last_verified: 2026-11-10
  last_edited: 2026-11-10
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
    - title: Index templates — composed_of merge order, template precedence and priority
      url: https://docs.opensearch.org/latest/im-plugin/index-templates/
      kind: product-doc
    - title: Field capabilities API — the include_unmapped parameter
      url: https://docs.opensearch.org/latest/api-reference/search-apis/field-caps/
      kind: api-reference
    - title: Index template for an index-alias mappings save is resolved by alias name before backing index
      url: https://github.com/opensearch-project/security-analytics/pull/1835
      kind: upstream-change
    - title: Field-alias component template is replaced rather than merged on each save
      url: https://github.com/opensearch-project/security-analytics/pull/1834
      kind: upstream-change
    - title: Detector creation rewrites a data-stream template's index patterns
      url: https://github.com/opensearch-project/security-analytics/issues/1102
      kind: upstream-change
    - title: Setting up ingestion for a Security Analytics detector — declaring field aliases in your own index template
      url: https://docs.aws.amazon.com/opensearch-service/latest/developerguide/sa-ingestion-setup.html
      kind: product-doc
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

**What differs by version.** Two flags are version-dependent. The rest are configuration states that no version alters.

| Behaviour | Where the change is absent | Where it is present | How to tell which you have |
|---|---|---|---|
| Rules retrieved per evaluation | At most 10 matching rules; the rest are dropped with no error, warning or log entry | Up to the engine's search ceiling, which the implementation sets to 10,000 | `GET /_cat/plugins?v&h=name,component,version` → read the `opensearch-alerting` row, then check that version against the change linked in `metadata.references` |
| How a field-mappings save picks the index template to carry the alias component | The **alias name** is matched first and the concrete index only as a fallback, and `<alias>*` is appended to whichever template matched. So a save can attach the component to a template that creates none of the alias's indices, and widen that template's patterns | For an alias input the name is not matched at all — the template is resolved from the write index, or the newest backing index. The pattern append is suppressed, and a save that would attach to another input's plugin-owned template fails instead | `GET /_cat/plugins?v&h=name,component,version` → the `opensearch-security-analytics` row, checked against the change linked in `metadata.references`. The change is on the development line and the 3.10 branch and **has not been backported**, so treat 2.x and 3.x through 3.9 as carrying the first behaviour |

**Both step 6 flags that concern template attachment are about the first behaviour** — so on a build carrying the change, no *new* misattachment occurs for an alias. Run them anyway, for two reasons given with the change itself: existing misattachments are not repaired by upgrading, and the pattern append is suppressed only for aliases, so a data-stream input still widens the matched template's patterns.

> ⚠️ **On a build carrying the change, one kind of unrepaired misattachment turns into a failing save.** The new guard fires on existing state, but **only where the matched template is plugin-owned and belongs to a different alias** — a save that resolves to an operator-created template is still accepted with no such check. In the case the guard does cover, the next field-mappings call for the affected alias, including a detector edit from Dashboards, fails until the attachment is corrected by hand; the change's review thread records this as intended. Read step 6 **before** an upgrade if you can, and see `opensearch-blue-green-upgrade-assessment`.

**Read your own build rather than trusting a version written here.** Where the rule-count flag fires, `security-analytics-rule-coverage-audit` carries the behavioural check that settles it for your build; this sweep only tells you which detectors are worth that check.

**Does not apply when:**

- **You already know which detector and which fault.** Go straight to the runbook that owns it — see **Related runbooks**. This sweep costs several calls per detector and tells you nothing new once the fault is identified.
- **The question is whether a detector that evaluates its rules should have matched your data.** This sweep establishes configuration state, not detection correctness. That is `security-analytics-detector-mapping-audit`.

## Values to obtain

| Name | What it is | How to obtain the current value |
|---|---|---|
| `${CONFIG_INDEX}` | The plugin's detector configuration index, which holds the fields the detector API does not return | Discover it rather than assuming the name — it is plugin-internal and can change:<br>`GET /_cat/indices/.opensearch-sap-*?expand_wildcards=all&v&h=index,docs.count`<br>Pick the one holding detector configuration documents; confirm with `GET /${CONFIG_INDEX}/_mapping` that it carries `workflow_ids` |
| `${DETECTOR_ID}` | A detector flagged by this sweep | `POST /_plugins/_security_analytics/detectors/_search` → `hits.hits[]._id` |
| `${ALIAS}` | The data source of a flagged detector, where that data source is an alias | Same response → `inputs[0].detector_input.indices[0]` |
| `${INDEX}` | One concrete member index of `${ALIAS}`. Changes at every rollover, so read it at execution time rather than carrying it between steps | `GET /_cat/aliases/${ALIAS}?v&h=alias,index,is_write_index` → the `index` column |
| `${SOURCE_TEMPLATE}` | The index template that creates `${ALIAS}`'s member indices — the one whose `index_patterns` match `${INDEX}` | `GET /_index_template` → find the entry whose `index_patterns` match `${INDEX}`. Do not assume it is the template whose patterns match the alias *name*; step 6 exists because those two can differ |
| `${ALIAS_COMPONENT}` | The plugin-generated component template holding `${ALIAS}`'s field aliases | Discover it rather than assuming the name:<br>`GET /_component_template/*alias-mappings*`<br>then confirm against `${SOURCE_TEMPLATE}`'s `composed_of` list |

## Impact

**Mechanism.** Every state this sweep looks for is silent by construction. A detector orphaned from its workflow still saves cleanly. A detector whose disable reached only the detector record still reports disabled. A rule set past the evaluation page bound still lists every rule as enabled. An alias with no write index still resolves. In each case the failure is the *absence* of something — a finding, an execution, an evaluated rule — and absence raises no error, writes no log line the console surfaces, and triggers no alert. So the fleet's reported health is not evidence of its actual health, and the only way to know which detectors are affected is to read the states directly.

**The field-alias plumbing fails the same way, one layer down.** A detector's field aliases reach each new index through a template, not through the detector: the plugin writes them into a component template and lists that template in the `composed_of` of an index template, so indices created later carry them. Three properties make this silent. The component template is **replaced**, not merged, each time a detector is created or its field mappings are saved. The template chosen to carry it is found by matching the alias *name*, which is not necessarily the template that creates the alias's indices. And a rule whose field is absent from the index it is evaluated against is skipped rather than failed. So a detector can hold a correct rule set, report healthy, and read indices that carry none of the aliases those rules need — and the break appears not when the configuration changes but at the next rollover after it.

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

- [ ] Permission to call the Security Analytics detector, findings, alerts and mappings APIs, the Alerting workflow route, and `GET /_cat/*`
- [ ] Permission to read the plugin configuration index. Without it the sweep cannot see the two fields that matter most, and the failure is silent — see the warning below
- [ ] Permission to call `GET /_index_template`, `GET /_component_template` and `GET ${INDEX}/_field_caps`. Steps 6 and 7 need all three; without them the field-alias flags cannot be evaluated and belong in your results as *not evaluated* rather than as clean
- [ ] Awareness that `plugins.security_analytics.filter_by_backend_roles`, where enabled, limits the detectors you can see to those created by users sharing a backend role — so your enumeration may be a subset. Read it with `GET /_cluster/settings?include_defaults=true` and **record the value with your results**, because a sweep over a subset that reports "fleet clean" is worse than no sweep

> ⚠️ **Do not use the detector `_execute` route anywhere in this sweep.** It executes rather than inspects, which would make this procedure something other than read-only, and the public report states it runs the same code path as a scheduled evaluation and carries the same page-bound limitation — so it cannot verify the thing you would be invoking it to verify.

> ⚠️ **On the mappings route specifically, `GET` reads and `POST` writes.** `GET /_plugins/_security_analytics/mappings?index_name=…` is a read; `POST` to the same path applies field mappings and rewrites the alias component template from whichever index it resolves, destroying the state steps 6 and 7 measure. **Use the `GET` form only on that route.** This is not a blanket rule about the verb — the detector search this sweep relies on is itself a `POST` and is a read. Check the operation, not the method. The Dashboards *Edit field mappings* page has been reported to issue the `POST` form when it opens, so treat opening it as a change rather than an inspection.

> ⚠️ **A second alias used by a test or secondary detector needs its own write index.** Guidance circulated to the contrary — that a secondary alias should be left without one because the data is written through the primary — and it was withdrawn after testing: a detector resolves *its own* alias, and where that alias has no write index the resolution yields nothing and the detector scans no part of any new index. Treat every alias that any detector reads as in scope for step 5, not only the primary one.

> ⚠️ **A zero or an empty result from a plugin-internal index is not evidence of absence.** The published documentation states that for a caller without system-index access the Security plugin can substitute an empty index reader, so a search returns `200 OK` with zero hits even though documents exist, and that an empty result must not be read as proof of access. **Step 2 therefore proves the read works before step 3 interprets anything.** A zero you have not validated is the most dangerous output in this procedure, because across a whole fleet it reads as a clean bill of health.

## Procedure

### Step 1 — Enumerate the detectors, and confirm the enumeration is complete

```
POST /_plugins/_security_analytics/detectors/_search
{"query":{"match_all":{}},"size":100}
```

**Expect:** one hit per detector. Record `_id`, `_source.enabled`, the rule arrays, `triggers`, and `inputs[0].detector_input.indices[0]`.

**Then group the detectors by that data source value.** Any data source appearing against more than one detector is a flag in step 3, and it costs no extra call to find. Record the groups — step 7 needs them.

Sum the rule arrays per detector. **Sum all of these key shapes** — the published API documentation uses different shapes across its own examples, and counting only one of them undercounts:

- `inputs[0].detector_input.custom_rules[]`
- `inputs[0].detector_input.pre_packaged_rules[]`
- `inputs[0].input.custom_rules[]` and `inputs[0].input.pre_packaged_rules[]`
- `inputs[0].detector_input.rules[]` — the documented response shape

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
| **Shared data source** | one data source value against two or more detectors, from step 1's grouping | Step 7, then `security-analytics-field-alias-template-recovery` |

**On a shared data source.** Detectors reading one alias share a single copy of that alias's field aliases, because the copy is keyed by alias rather than by detector. Saving either detector's field mappings replaces the copy with that detector's set, so which aliases survive depends on which detector was saved last. The detector that did not save it keeps its rule set, reports healthy, and stops matching on the aliases the other one dropped. **This is a flag even when both detectors look correct**, because the state is decided by save order rather than by configuration. The structural fix is one detector per alias, which is a change needing its own approval; step 7 establishes what the current copy actually covers.

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

### Step 6 — Sweep the field-alias plumbing, fleet-wide

Three reads, once each for the whole cluster rather than once per detector.

```
GET /_cat/aliases?v&h=alias,index,is_write_index
GET /_component_template/*alias-mappings*
GET /_index_template
```

**Expect:** from the first, one row per alias-to-index pair. From the second, the plugin's alias component templates, each carrying `template.mappings.properties` entries including `"type": "alias"` entries with a `path`. From the third, every index template with its `index_patterns`, `composed_of` and `priority`.

> ⚠️ **Prove the component-template read before interpreting it.** A component template listing no alias entries and a caller who cannot read component templates look identical. Confirm at least one comes back carrying alias entries before treating an empty one as a finding. Without that positive control, record this step as **not evaluated**.

> ⚠️ **An absent component template is not a finding on its own.** The plugin skips the template path entirely when a detector's input is already a concrete index, and a detector whose field mappings were never saved has no component template either. Establish that the detector reads an alias and has had mappings saved before reading absence as a fault.

**Resolve `${SOURCE_TEMPLATE}` from a member index name, never from the alias name.** The two can differ, and where they differ the template you would otherwise assess is not the one creating the indices. That single instruction is what this step exists to make you do.

**One flag, and two observations.** Only the first of these is cleanly decidable from the three reads above.

| | Signature | Where to go |
|---|---|---|
| **Flag — component template absent from `${SOURCE_TEMPLATE}`** | `${SOURCE_TEMPLATE}`'s `composed_of` does not list the alias's component template, while some *other* template does | `security-analytics-field-alias-template-recovery` |
| *Observation* — component template empty | it carries no `"type": "alias"` entries at all | Step 7 decides whether it matters |
| *Observation* — alias names share a prefix | one alias name is a leading substring of another, and an `index_patterns` entry of the form `<shorter-alias>*` appears on a template | Step 7, then `security-analytics-field-alias-template-recovery` if a gap is confirmed |

**Why the prefix case is an observation rather than a flag.** It only costs detection where the two aliases' indices come from different templates — and the reads here cannot settle that, because nothing records which template created an index, and the mechanism itself appends `<alias>*` to the wrong template, polluting the name-matching you would use to infer it. Record the prefix pair and let step 7 decide from the index itself.

**A short component template is likewise not decidable here.** An alias legitimately absent because its target base field is absent renders identically to one dropped by a replacing save. Step 7 separates them; this step records the count.

### Step 7 — For each flagged or observed alias, read what is on the index

One read per member index of the alias.

```
GET /_plugins/_security_analytics/mappings?index_name=${INDEX}
```

**Expect:** the field aliases present on that index, each with a `path` to a base field.

**Compare that set against the `"type": "alias"` entries in `${SOURCE_TEMPLATE}`.** That comparison is the whole of this step, and it is decidable from these two reads alone.

**Branch.**

- **An alias entry in `${SOURCE_TEMPLATE}` is missing from `${INDEX}`** → that index was not created from the template you resolved, or was created before the entry was added. Go to `security-analytics-field-alias-template-recovery`.
- **`${SOURCE_TEMPLATE}` declares no alias entries, and the index's aliases come only from the component template** → coverage depends on the last save, which is the condition the documented remedy addresses. Go to `security-analytics-field-alias-template-recovery`.
- **The sets match** → the plumbing is intact for this index. This is a statement about `${INDEX}` only. Repeat per member.

> ⚠️ **Which rules a missing alias affects is out of scope here.** Mapping an absent field name back to the rules that reference it needs the rule set, which this sweep does not read — so this step does not produce a coverage ratio, and you should not infer one. `security-analytics-detector-mapping-audit` owns that question.

**A match on the write index is not a clean result for the alias.** The write index is the one most likely to be correct, because a recent save resolved against it. Members that predate the save are where gaps sit.

## Validation

This procedure produces a judgement about a whole fleet, so validation confirms the judgement is trustworthy rather than that a system recovered. Run all four before reporting any result, and especially before reporting a clean one.

1. **The detector enumeration was not truncated.** Compare the `size` you requested in step 1 against the number of hits returned.
   ```
   POST /_plugins/_security_analytics/detectors/_search
   {"query":{"match_all":{}},"size":100}
   ```
   **Expect:** a hit count strictly below the requested `size`, or a documented paging run that reached the end. A result at the requested size means there is more to read.
2. **The configuration read was not scrubbed.** This is the check that decides whether the orphaned-workflow and disable flags mean anything.
   ```
   GET /_cat/indices/${CONFIG_INDEX}?v&h=index,docs.count
   ```
   **Expect:** a non-zero `docs.count` alongside the non-zero hit count from step 2. A non-zero `docs.count` beside a zero-hit search means the read was scrubbed and the orphaned-workflow flag is meaningless for every detector.
3. **The visible detector set is the whole set.**
   ```
   GET /_cluster/settings?include_defaults=true
   ```
   **Expect:** the value of `plugins.security_analytics.filter_by_backend_roles`. Where it is enabled, state in your results that the sweep covered only the detectors your backend role can see.
4. **Both enumeration counts are recorded.** Compare step 1's hit count against step 2's and **record both**. No published source states what the relationship between the two sets should be, so a difference is a finding to report rather than a failure to reconcile — it can mean an incomplete read, a stale configuration document, or a detector the API will not return to you, and nothing available distinguishes them.
5. **The template reads were not blind.** This is the check that decides whether step 6's four flags mean anything.
   ```
   GET /_component_template/*alias-mappings*
   ```
   **Expect:** at least one component template carrying `"type": "alias"` entries. That is the positive control from the step 6 warning. An empty result for every alias means the read cannot see what you are asking about, and every step 6 flag is **not evaluated** rather than clean.

**Exit criteria.** You can state, for every detector the enumeration returned, which flags it carries and which runbook each flag routes to — plus the three figures that bound the sweep: detectors enumerated, detectors flagged, and whether the backend-role filter was limiting what you could see. Where steps 6 and 7 ran, add the alias count covered, and per alias whether the template's declared alias entries matched the index. A single flagged detector is not the exit condition; the audit is complete when every detector has a record, including the ones with no flags.

## If this does not work

**Reach out when:** the configuration index returns a validated zero, so two flags cannot be evaluated at all; or the two enumerations disagree and neither can be shown complete; or a detector carries a flag that none of the routes below fit; or the flagged share of the fleet is large enough to suggest a common cause rather than per-detector drift.

**Reach out to:** your OpenSearch support channel, through the route your organisation uses for a production cluster.

**Attach:** the detector count and the flagged count, the per-detector flag table from step 3, the `docs.count` and hit count that show the configuration read was or was not scrubbed, the backend-role filter value, the engine and alerting plugin versions from `GET /` and `GET /_cat/plugins?v&h=name,component,version`, and the requested-versus-returned sizes that show the enumeration was complete. Where step 6 or 7 ran, add the alias-to-template mapping you derived, the component-template contents with the positive control that proves the read worked, and the unresolved rule fields per flagged detector. State explicitly that the `_execute` route was not used, that no `POST` was made to the mappings route, and why.

## Related runbooks

Siblings are referred to by name, which resolves to a directory and is greppable.

- `security-analytics-detector-workflow-recovery` — where `monitor_id` is populated and `workflow_ids` is empty. Carries the backup-delete-recreate procedure and its irreversibility caveats.
- `security-analytics-detector-disable-verification` — where a detector reports disabled. Reads the workflow flag back and repairs a stale one.
- `security-analytics-rule-coverage-audit` — where a detector carries more than ten rules. Establishes whether your build truncates the evaluation.
- `security-analytics-alias-write-index-recovery` — where the data source is an alias with no explicit write index.
- `security-analytics-field-alias-template-recovery` — where an alias's component template is empty or short, where it is listed on a template that does not create that alias's indices, where two alias names collide by prefix, or where two detectors on one alias map a rule field to different paths.
- `security-analytics-detector-mapping-audit` — where a detector is enabled, not failing, and producing nothing. This sweep establishes configuration state; that one establishes whether an evaluated rule could have matched. Shares the step 1 enumeration.
- `security-analytics-percolate-failure-investigation` — where error-state alerts exist. The evaluation is being rejected rather than silently truncated.
- `security-analytics-rule-cost-audit` — where detectors are producing findings normally but driving sustained load. Same enumeration, a cost question rather than a health one.
- `security-analytics-rollover-blind-window-remediation` — where a detector reads a daily alias and the concern is the records written between index creation and alias attachment.

`security-analytics-wayfinder` is a skill rather than a runbook, and it is the other half of this one. It starts from a single detector's error-state alerts and identifies which component is at fault — detector record, workflow, query index, data source, field mappings, rule set — then searches this collection for whatever procedure currently covers it. This sweep establishes *which detectors* need attention; that identifies *what is wrong* with one of them. Use it when the detector is known and the cause is not; use this when the reverse holds.

## Verification status

Every step traces to a reputable source. A step resting on inference is reason to leave a procedure unwritten rather than to label it. The basis, what has been observed in practice, the open questions and any superseded claims are in `references/evidence.md` — read it before relying on this runbook for the first time, or before quoting it to anyone.
