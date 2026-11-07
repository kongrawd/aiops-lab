---
name: security-analytics-rule-coverage-audit
description: >-
  Determine whether every detection rule attached to a Security Analytics detector is actually
  being evaluated. On engines without the percolate page-size fix, a doc-level monitor retrieves
  only a bounded page of matching rules per evaluation, so a detector carrying more rules than
  that bound silently produces no findings for the remainder. Nothing is logged and the
  configuration still lists every rule as enabled. Use this to audit detectors proactively, when
  a rule that should match is producing no findings, after adding rules to an existing detector,
  or before relying on a detector for compliance coverage. Read-only throughout.
metadata:
  agent_types:
    - INCIDENT_TRIAGE
    - PREVENTION
  skill_type: USER
  service: OpenSearch Security Analytics / Alerting
  applies_to_versions: All lines. The defect is absent where the percolate page-size change is present and active elsewhere - see "What differs by version" rather than relying on this range.
  risk_level: low
  read_only: true
  automatable: yes
  verification: documented
  last_verified: 2026-10-14
  last_edited: 2026-10-14
  references:
    - title: Doc-level monitor produces false negatives when a detector has more than ten rules
      url: https://github.com/opensearch-project/security-analytics/issues/1656
      kind: upstream-change
    - title: Update number of search hits to return for percolate queries in doc level monitor
      url: https://github.com/opensearch-project/alerting/pull/2064
      kind: upstream-change
    - title: Security Analytics Detector APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/detector-api/
      kind: api-reference
    - title: Security Analytics Alert and Finding APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/alert-finding-api/
      kind: api-reference
    - title: Cluster Information API
      url: https://docs.opensearch.org/latest/api-reference/cluster-api/info/
      kind: api-reference
    - title: CAT plugins API
      url: https://docs.opensearch.org/latest/api-reference/cat/cat-plugins/
      kind: api-reference
---

# Runbook — Confirm every rule on a detector is actually being evaluated

## Before you use this

- **Confirm the behaviour against current OpenSearch documentation, or with your support channel, before you execute this.** Plugin behaviour moves with versions, and this runbook states the range it was written for in `metadata.applies_to_versions`. Where this document and the documentation disagree, the documentation is current and this is not.
- **Exercise the procedure in a non-production environment first.** Every step here is a read, so the risk is low — but run it once somewhere you can check the output against a detector whose rule set you already know.
- **You remain responsible for the outcome in your environment.** This document is offered as-is, with no warranty and no liability for any effect of its use.
- **Send corrections back.** A step that failed, output that differed, a version where the behaviour changed, or a clearer command are worth more than a clean document that is quietly wrong. Open an issue or a pull request on the repository this runbook was shared from.

## When to use this

**Trigger.** Any of:

- **Proactive** — auditing detector coverage, or before citing a detector as the control for a rule set.
- **Reactive** — a rule that should match is producing no findings, while other rules on the same detector are producing them normally.
- **After a change** — rules were added to an existing detector, taking its total past the page bound.

**Applies when all of these hold.**

| Condition | How to check | Expected |
|---|---|---|
| Security Analytics detectors exist | `POST /_plugins/_security_analytics/detectors/_search` | At least one hit |
| A detector carries more rules than the page bound | Rule count from step 2 | Greater than 10 |
| The engine lacks the percolate page-size change | See **What differs by version** | Change absent |

**What differs by version.** The bound is not a product limit — it is the default page size of a search the plugin runs internally, and a published change sets it explicitly.

| Behaviour | Where the change is absent | Where it is present | How to tell which you have |
|---|---|---|---|
| Rules retrieved per evaluation | At most 10 matching rules; the rest are dropped with no error, warning or log entry | Up to the engine's search ceiling, which the implementation sets to 10,000 | `GET /_cat/plugins?v&h=name,component,version` → read the `opensearch-alerting` row, then check that version against the change linked in `metadata.references` |

**Read your own build rather than trusting a version written here.** The change is merged on the development line and present on some recent release lines, and at the time of writing every backport to the older lines remains open. That state moves. The two behaviours above do not, so use the behavioural check in step 4 as your answer and the version read as corroboration.

**Does not apply when:**

- **The detector carries ten rules or fewer.** It cannot hit this bound. A rule not matching has a different cause.
- **The rule's own query does not match the data.** This runbook establishes that a rule is never *evaluated*; it does not establish that an evaluated rule should have matched.
- **Findings stopped for the whole detector rather than for some rules.** Four situations look alike from the outside — a detector that is not producing what you expect — and they have different instruments. Discriminate before proceeding:

| What you also see | What it is | Where to go |
|---|---|---|
| Some rules fire, others never do, **no error** | silent truncation — this runbook | continue |
| Nothing fires, **and error-state alerts exist** | the evaluation is being rejected, not truncated | `security-analytics-percolate-failure-investigation` |
| Nothing fires, and the detector **reports disabled** | it may not have stopped at all | `security-analytics-detector-disable-verification` |
| Nothing fires, and the data source is an **alias with no explicit write index** | nothing is being read | `security-analytics-alias-write-index-recovery` |

The distinction that matters most is the first against the second: **truncation is silent and rejection raises an error.** This runbook's whole premise is that nothing is logged, so an error-state alert means you are in the wrong document.

## Values to obtain

| Name | What it is | How to obtain the current value |
|---|---|---|
| `${DETECTOR_ID}` | The detector being audited | `POST /_plugins/_security_analytics/detectors/_search` → `hits.hits[]._id` |
| `${WINDOW_START}`, `${WINDOW_END}` | The epoch-millisecond bounds of the period you are auditing | Your own choice. Make it long enough that every rule you expect to fire has had the chance |

## Impact

**Mechanism.** A doc-level monitor evaluates a document by percolating it against the detector's rules and reading back the rules that matched. That read-back is a search, and a search returns a bounded page of hits. Each hit is one matched rule. Where the page bound is lower than the number of rules that matched, the surplus matches are discarded — and because a short page is a valid search result rather than an error, nothing is logged, no alert fires, and the detector's configuration continues to list every rule as enabled. The rules are present, enabled, and never able to produce a finding.

**Size it for this system.** The magnitude is the number of rules that cannot produce findings. For each detector, take the rule count:

```
POST /_plugins/_security_analytics/detectors/_search
{"query":{"match_all":{}},"size":100}
```

Rule count is the number of entries across the detector's rule arrays — see step 2 for the key shapes to sum. Any count above the page bound means the difference is the number of rules carrying no coverage. Summed across detectors, that is the size of the gap in this system.

**Blast radius.** Every rule beyond the bound on every affected detector. Which specific rules are dropped is not something this runbook can establish — the discarded hits leave no record — so treat the whole rule set of an affected detector as unverified coverage rather than assuming the first ten are safe.

**Recoverability.** Nothing is recoverable. Documents that passed through while a rule was unevaluated are not re-examined, because a doc-level monitor advances through documents as they arrive rather than re-scanning a past window. The gap is closed going forward only.

**Severity.** Urgent where an affected detector is the control relied on for a rule set that matters — a compliance obligation, or threat coverage someone has attested to. Routine where this is a proactive audit. Note that this cannot be deferred on the grounds that nothing has alerted: the absence of alerts is the symptom.

## Before you start

- [ ] Permission to call the Security Analytics detector and findings APIs, and to read `GET /` and `GET /_cat/plugins` (the latter needs `cluster:monitor/nodes/info`)
- [ ] Awareness that `plugins.security_analytics.filter_by_backend_roles`, where enabled, limits the detectors you can see to those created by users sharing a backend role — so your enumeration may be a subset. Read it with `GET /_cluster/settings?include_defaults=true` and record the value with your results

> ⚠️ **Do not use the detector `_execute` route to check coverage.** The public report states that it runs the same code path and carries the same limitation, so it cannot verify the thing you would be invoking it to verify. It also executes rather than inspects, which would make this procedure something other than read-only.

## Procedure

### Step 1 — Confirm there is a detector that can hit the bound

```
POST /_plugins/_security_analytics/detectors/_search
{"query":{"match_all":{}},"size":100}
```

**Expect:** one hit per detector. Record `_id`, `_source.enabled`, and the rule arrays.

**If every detector carries ten rules or fewer:** none can hit this bound. **Stop here.**

**If any detector carries more than ten:** record it as a candidate and continue to step 2.

### Step 2 — Count the rules per candidate detector

Sum the entries across the rule arrays. **Sum all of these key shapes** — the published API documentation uses three different shapes across its own examples, and counting only one of them undercounts:

- `inputs[0].detector_input.custom_rules[]`
- `inputs[0].detector_input.pre_packaged_rules[]`
- `inputs[0].input.custom_rules[]` and `inputs[0].input.pre_packaged_rules[]`
- `inputs[0].rules[]`

**Expect:** a single integer per detector. A count above 10 is the condition this runbook addresses.

### Step 3 — Read the engine and plugin versions

```
GET /
GET /_cat/plugins?v&h=name,component,version
```

**Expect:** `version.number` from the first; an `opensearch-alerting` row from the second, whose version carries a fourth digit the engine version does not.

Record both. This is corroboration for step 4, not a substitute — a version number cannot tell you whether a particular build carries a particular backport.

### Step 4 — Count how many distinct rules actually produced findings

This is the behavioural check, and it is the one that answers the question.

```
GET /_plugins/_security_analytics/findings/_search?detector_id=${DETECTOR_ID}&startTime=${WINDOW_START}&endTime=${WINDOW_END}&sortOrder=desc&size=100&startIndex=0
```

Page with `startIndex` until you have the window, then **tally the distinct values of `queries[].id` across all findings**. There is no server-side grouping for this — the findings API exposes no aggregation or group-by parameter, so the tally is client-side.

If your build rejects `startTime` or `endTime`, drop them and use `sortOrder=desc` with `size`, then filter on each finding's `timestamp` yourself. Those two parameters appear in the published API specification but not on the documentation page, and **no engagement record available to this review shows either one ever being sent** — so treat them as untried rather than as reliable. The parameters confirmed working in practice are `detector_id` and `findingIds`, optionally with `filter_path` to trim the response.

> ⚠️ **Read `total_findings`, not the length of the `findings` array.** A request with `size=0` returns an empty array alongside a non-zero total, so array length reports no detections when there are many.

**Expect:** the count of distinct firing rule ids.

**Branch.**

- **Distinct firing rules plateau at exactly 10 while the rule count exceeds 10** → this is the signature. The surplus rules are not being evaluated. Go to **Validation**, then to **If this does not work** for the remediation route.
- **Distinct firing rules exceed 10** → the bound is not in force on this build. The rules beyond the tenth are being evaluated, and a rule producing nothing has a different cause. **Stop here.**
- **Distinct firing rules are fewer than 10** → inconclusive, and that is a real result rather than a clean one. The window may be too short, or too few rules match this data. Widen `${WINDOW_START}` and repeat. Do not read this as absence of the defect.

## Validation

This procedure produces a judgement, so validation confirms the judgement is trustworthy rather than that a system recovered.

1. **The enumeration was not truncated.** Compare the `size` you requested against the number of hits returned, for both the detector search and each findings page. A result at the requested size means there is more to page through, not that you reached the end.
2. **The detector was enabled for the whole window.** A disabled detector produces no findings for an unrelated reason.
   ```
   POST /_plugins/_security_analytics/detectors/_search
   {"query":{"match_all":{}},"size":100}
   ```
   **Expect:** `enabled: true` for the audited detector. Where it reads `false`, the audit is void — resolve that first.
3. **The plateau is stable, not coincidental.** Re-run step 4 over a second, non-overlapping window. A distinct-rule count that lands on exactly 10 twice is the signature; a count that moves is not.

**Exit criteria.** For each detector carrying more than ten rules, you can state its rule count, its distinct-firing-rule count over at least two windows, and the plugin version — and you have recorded whether the backend-role filter was limiting what you could see. The audit is complete when every such detector has that record, not when the first one does.

## If this does not work

**Reach out when:** the signature is confirmed; or step 4 stays inconclusive across widened windows; or the detector enumeration is limited by the backend-role filter and you cannot see the full set.

**Reach out to:** your OpenSearch support channel, through the route your organisation uses for a production cluster. There is no operator-side configuration that lifts the page bound — the two publicly supported courses are to move to a build carrying the change, or to hold detectors at or below ten rules by splitting a rule set across several detectors.

**Attach:** per detector, the rule count, the distinct-firing-rule count with each window's bounds, the engine and alerting plugin versions, the backend-role filter value, and the requested-versus-returned sizes that show the enumeration was complete. State explicitly that the `_execute` route was not used and why.

## Related runbooks

Siblings are referred to by name, which resolves to a directory and is greppable.

- `security-analytics-detector-health-audit` — the broader read-only sweep across all detectors; run it first where you do not yet know which detectors need attention. It routes here for this condition.
- `security-analytics-detector-disable-verification` — where a detector produces no findings at all rather than for some rules, and reports as disabled.
- `security-analytics-rule-cost-audit` — where rules are being evaluated but detector evaluation is slow or driving sustained load.
- `security-analytics-percolate-failure-investigation` — where the evaluation is rejected outright rather than truncated. Same symptom at a glance, opposite diagnosis: that one has an error to read, this one has none by design.

## Verification status

Every step traces to a reputable source. A step resting on inference is reason to leave a procedure unwritten rather than to label it. The basis, what has been observed in practice, the open questions and any superseded claims are in `references/evidence.md` — read it before relying on this runbook for the first time, or before quoting it to anyone.
