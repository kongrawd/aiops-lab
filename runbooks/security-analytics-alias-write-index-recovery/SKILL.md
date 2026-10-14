---
name: security-analytics-alias-write-index-recovery
description: >-
  Restore detection for a Security Analytics detector whose data source is an index alias that
  has no explicit write index. A detector reading an alias resolves it to the alias's write index;
  where
  no member carries that designation the resolution yields nothing, no documents are scanned,
  and the detector reports healthy while producing no findings. The same state also makes the
  detector fail to update on engines without the empty-run-context fix. Use this when a detector
  on an alias stops producing findings with no error, when a detector update fails after an
  alias gained a second member, or as the first check whenever a detector's data source is an
  alias rather than a concrete index.
metadata:
  agent_types:
    - INCIDENT_TRIAGE
    - INCIDENT_MITIGATION
  skill_type: USER
  service: OpenSearch Security Analytics / Alerting
  applies_to_versions: All lines, for the detection failure - it follows from documented alias behaviour rather than from a defect. The update failure depends on the empty-run-context change; see "What differs by version".
  risk_level: medium
  read_only: false
  reversible: yes
  automatable: partial
  verification: documented
  last_verified: 2026-10-14
  last_edited: 2026-10-14
  references:
    - title: Creating detectors — the alias data source requirement
      url: https://docs.opensearch.org/latest/security-analytics/sec-analytics-config/detectors-config/
      kind: product-doc
    - title: Index aliases — designating a write index
      url: https://docs.opensearch.org/latest/im-plugin/index-alias/
      kind: product-doc
    - title: Manage Aliases API
      url: https://docs.opensearch.org/latest/api-reference/alias/aliases-api/
      kind: api-reference
    - title: CAT aliases API
      url: https://docs.opensearch.org/latest/api-reference/cat/cat-aliases/
      kind: api-reference
    - title: Security Analytics Detector APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/detector-api/
      kind: api-reference
    - title: Fix ClassCastException when updating a doc-level monitor with an empty run context
      url: https://github.com/opensearch-project/alerting/pull/2271
      kind: upstream-change
    - title: ClassCastException when updating a doc-level monitor with absent last_run_context
      url: https://github.com/opensearch-project/alerting/issues/2220
      kind: upstream-change
---

# Runbook — A detector on an alias produces no findings, or cannot be updated

## Before you use this

- **Confirm the behaviour against current OpenSearch documentation, or with your support channel, before you execute this.** Plugin behaviour moves with versions, and this runbook states the range it was written for in `metadata.applies_to_versions`. Where this document and the documentation disagree, the documentation is current and this is not.
- **Exercise the procedure in a non-production environment first.** Step 3 changes which index an alias writes to. Run it end to end on an alias you can afford to disrupt, confirm each step's output, and only then run it where it matters.
- **You remain responsible for the outcome in your environment.** This document is offered as-is, with no warranty and no liability for any effect of its use. Designating a write index changes where ingestion lands, and the decision to run it is yours.
- **Send corrections back.** A step that failed, output that differed, a version where the behaviour changed, or a clearer command are worth more than a clean document that is quietly wrong. Open an issue or a pull request on the repository this runbook was shared from.

## When to use this

**Trigger.** Either of:

- **Silent non-detection** — a detector whose data source is an alias produces no findings, reports enabled, and raises no error.
- **Update failure** — a detector cannot be saved or re-enabled, and its data source is an alias that has gained a second member.

**Applies when all of these hold.**

| Condition | How to check | Expected |
|---|---|---|
| The detector's data source is an alias | `POST /_plugins/_security_analytics/detectors/_search` → `inputs[0].detector_input.indices[0]`, then `GET /_resolve/index/${NAME}` | The name appears under `aliases` |
| The alias has two or more member indices | `GET /_cat/aliases/${ALIAS}?v&h=alias,index,is_write_index` | More than one row |
| No member carries the write-index designation | Same output | No row reads `true` |

**Does not apply when:**

- **The alias has exactly one member and no designation.** This is normal and needs no fix. The documentation states that a single referenced index behaves as the write index until a second is referenced — so a freshly rolled single-member alias is healthy. Acting here would be a false positive.
- **The data source is a concrete index or a data stream.** `GET /_resolve/index/${NAME}` distinguishes them. A detector on a concrete index does not resolve a write index at all.
- **You are matching on an error string alone.** The cast error that accompanies the update failure has at least one other published precondition — a monitor created but not yet executed — so the string does not identify this situation. The alias state in step 1 is what identifies it.

**What differs by version.** Two separate things fail here, and only one of them is a defect with a fix.

| Behaviour | Where the change is absent | Where it is present | How to tell which you have |
|---|---|---|---|
| Detection, when the alias has no explicit write index | No documents scanned, no findings, no error | **Unchanged — still no documents scanned.** This is documented alias behaviour, not a defect | Not version-dependent. Step 1 is the check |
| Updating the detector while in that state | The update fails with a cast error; the detector cannot be updated or re-enabled | The update succeeds | `GET /_cat/plugins?v&h=name,component,version` → read the `opensearch-alerting` row, then check it against the change linked in `metadata.references` |

**The write-index fix in step 3 is what restores detection, on every version.** The upstream change only removes the update failure. At the time of writing that change is merged on the development line and every backport to a release line remains open, so read your own build rather than trusting a version written here.

## Values to obtain

| Name | What it is | How to obtain the current value |
|---|---|---|
| `${DETECTOR_ID}` | The detector that stopped producing findings | `POST /_plugins/_security_analytics/detectors/_search` → `hits.hits[]._id` |
| `${ALIAS}` | Its configured data source | Same response → `inputs[0].detector_input.indices[0]` |
| `${TARGET_INDEX}` | The alias member that should receive writes — normally the newest, and the one your shipper is writing to | `GET /_cat/indices/<pattern>?v&h=index,creation.date.string&s=creation.date` |
| `${OTHER_MEMBERS}` | Every other member of the alias | `GET /_cat/aliases/${ALIAS}?v&h=alias,index,is_write_index` |

`${TARGET_INDEX}` changes at every rollover, so obtain it from the alias listing rather than reusing one from a previous run.

## Impact

**Mechanism.** A doc-level monitor tracks its position per concrete index. For an alias input it asks the cluster for the alias's write index, and that lookup returns the designated member or nothing at all — there is no fallback to the newest or largest member. With no designation, the lookup yields nothing, so no position is tracked and no documents are scanned. The detector's own state is unaffected: it remains enabled, its schedule continues, and it reports no error, because an empty set of indices to scan is not a failure condition.

The same empty state is what breaks the update path on affected builds: the empty position record is omitted when the monitor's metadata is written, read back as an immutable empty map, and then cast to a mutable one, which throws.

Note also that this alias state rejects writes, and does so loudly: a write through such an alias has been observed failing with `no write index is defined for alias`. The documentation states the same — an alias pointing at several indices rejects indexing requests until one is designated. Pinning the write index is therefore a prerequisite for alias ingestion rather than an optimisation. So a shipper writing through the alias is failing too, while a shipper writing directly to a concrete index is succeeding and bypassing detection entirely.

**Size it for this system.** The magnitude is how long detection has been absent and across how many detectors. Enumerate every detector whose data source is an alias, and check each alias:

```
POST /_plugins/_security_analytics/detectors/_search
{"query":{"match_all":{}},"size":100}

GET /_cat/aliases/${ALIAS}?v&h=alias,index,is_write_index
```

Every detector whose alias has two or more members and no designated write index is one unit of this impact. For the duration, take the creation time of the member that joined second — detection has been absent at least since then:

```
GET /_cat/indices/<pattern>?v&h=index,creation.date.string&s=creation.date
```

**Blast radius.** Every detector reading an affected alias, and therefore every rule on those detectors. Where several detectors share one alias, all of them are affected together.

**Recoverability.** The alias state is fully recoverable and the fix is reversible. The detection gap is not: a doc-level monitor advances through documents as they arrive rather than re-scanning a past window, so documents that passed through while no index was designated are not examined after the fix. Designating a write index closes the gap going forward only.

**Severity.** Urgent whenever it is confirmed, because the detector is reporting healthy while providing no coverage, and nothing else in the system will surface that. Treat the absence of findings as the alarm rather than waiting for one.

## Before you start

- [ ] Permission to call `POST _aliases`, plus read access to the Security Analytics detector API and the CAT routes
- [ ] Confirmation of which index your shipper is currently writing to, so the designation lands on the index actually receiving data
- [ ] Awareness of any index lifecycle policy managing this alias — if a policy performs rollover against it, coordinate, because the policy will also change the designation

**Do not proceed if** you cannot establish which member should receive writes. Designating the wrong index sends ingestion to an index your pipeline is not expecting, which is a worse state than the one you are fixing. Determine it first, or escalate.

## Procedure

### Step 1 — Confirm the alias has members but no explicit write index

```
GET /_cat/aliases/${ALIAS}?v&h=alias,index,is_write_index
```

**Expect:** one row per member index. The documented columns for this route are `alias`, `index`, `filter`, `routing.index`, `routing.search` and `is_write_index`.

> ⚠️ **An unset `is_write_index` prints as a literal `-`.** Not `false`, not an empty column. A check that greps for `false`, or tests for an empty field, misreads every row. This has been observed on multiple live clusters.

**Branch. Three states matter, and they are distinguishable only by reading the flag _and_ counting members.**

- **Two or more rows, every one showing `-`** → this is the situation. Continue to step 2.
- **Exactly one row showing `-`** → healthy. A single referenced member behaves as the implicit write index, so no explicit designation is needed. **Stop here.**
- **Exactly one row explicitly reading `false`** → **also a failure.** An explicit `false` overrides the implicit behaviour, leaving the alias with no write index at all even with one member. Continue to step 2.
- **One row reading `true`** → the designation is present, and a detector producing no findings has a different cause. **Stop here** and see **Related runbooks**.

### Step 2 — Identify which member should receive writes

```
GET /_cat/indices/<pattern>?v&h=index,creation.date.string,docs.count&s=creation.date
```

**Expect:** the alias members in creation order with their document counts.

Cross-check against your shipper's configuration. The member your pipeline is writing to is `${TARGET_INDEX}`; it is normally the newest, but confirm rather than assume — the whole failure mode is a mismatch between what the pipeline writes to and what the detector reads.

### Step 3 — Designate the write index

> ⚠️ **This changes state.** It changes where writes through the alias land. Combining the actions in one request makes the change atomic, which the documentation states is the point: an alias never points at an unintended set of indices, even momentarily.

```
POST _aliases
{
  "actions": [
    { "add": { "index": "${TARGET_INDEX}", "alias": "${ALIAS}", "is_write_index": true } },
    { "add": { "index": "${OTHER_MEMBERS}", "alias": "${ALIAS}", "is_write_index": false } }
  ]
}
```

Repeat the second action once per other member. Keep the current output of step 1 — it is the body for the rollback.

**Expect:** `{"acknowledged": true}`.

**Rollback this step:**
```
POST _aliases
{
  "actions": [
    { "add": { "index": "${TARGET_INDEX}", "alias": "${ALIAS}", "is_write_index": false } }
  ]
}
```
This returns the alias to having no designated write index — that is, to the broken state, which is only the right move if the designation sent ingestion somewhere harmful. It does not undo documents written to `${TARGET_INDEX}` in the meantime, and it re-opens the detection gap.

### Step 4 — Read the designation back

```
GET /_cat/aliases/${ALIAS}?v&h=alias,index,is_write_index
```

**Expect:** exactly one row reading `true`, and it is `${TARGET_INDEX}`.

**If no row reads `true`:** the write did not take effect. Do not retry blindly — re-read step 2's output and confirm `${TARGET_INDEX}` is genuinely a member of this alias, then go to **Reach out**.

### Step 5 — Check whether the detector itself still needs recreating

> ⚠️ **Correcting the alias restores detection for a healthy detector. It does not recover one that has already recorded an empty position.** This has been observed: an update attempted *after* the alias was corrected still raised the same error, and that was confirmed as the expected outcome. A runbook that ends at "set the write index" leaves an already-poisoned detector broken, and the operator reports that the remediation did not work.

```
GET /_plugins/_security_analytics/detectors/${DETECTOR_ID}
```

Then attempt a no-op save of the detector — the same body, unchanged.

**Branch.**

- **The save succeeds** → the detector is intact. Continue to step 6.
- **The save fails with a cast error** → the detector's stored position is already empty and the alias fix cannot clear it. The detector needs recreating. Go to `security-analytics-detector-workflow-recovery`, which carries the backup-delete-recreate procedure and its history caveats. Do not repeat the alias change; it is already correct.

### Step 6 — Confirm the shipper writes through the alias, not around it

The documentation is explicit that documents must be ingested through the alias and not through the concrete index the alias was created for. A pipeline writing directly to a concrete index will keep bypassing detection even with the designation correct.

You cannot read your shipper's configuration from the cluster, but you can observe the symptom: documents arriving in a member index that is **not** the write index.

```
GET /_cat/indices/<the alias member pattern>?v&h=index,docs.count&s=index
```

Take two samples a few minutes apart and compare `docs.count` per member against the designation from step 4.

**Branch.**

- **Only `${TARGET_INDEX}` is gaining documents** → ingestion is going through the alias, or directly to the write index, and either way the detector reads it. Continue to **Validation**.
- **A member that is not the write index is gaining documents** → something writes to a concrete index directly, bypassing the alias. Those documents are outside what the detector reads. Repoint that writer at `${ALIAS}`; the change lives in your ingestion configuration rather than here, and the fix is incomplete without it.
- **Nothing is gaining documents** → ingestion has stopped for an unrelated reason. Resolve that before judging detection, and go to **Reach out** if the cause is not apparent.

## Validation

1. ```
   GET /_cat/aliases/${ALIAS}?v&h=alias,index,is_write_index
   ```
   **Expect:** one row reading `true`.
2. **Findings resume.** The designation is configuration; findings are behaviour, and this fault is one where the two disagreed.
   ```
   GET /_plugins/_security_analytics/findings/_search?detector_id=${DETECTOR_ID}&sortOrder=desc&size=1
   ```
   **Expect:** the newest `timestamp` advances across repeated calls. A timestamp that stays fixed means documents are still not being scanned — go to **Reach out**.

   > ⚠️ **Read `total_findings`, not the length of the `findings` array.** A request with `size=0` returns `{"findings":[],"total_findings":<N>}` — an empty array alongside a non-zero total. Treating the array as the answer reports "no detections" when there are plenty.

> ⚠️ **Do not substitute a mappings read for the liveness check above.** The Security Analytics mappings endpoint has been observed returning the complete alias set for a detector whose backing query index showed zero search and fetch activity and whose findings API returned nothing. Mappings present is not detection working.

3. **Query each concrete member index separately, never the alias.** An alias-level mappings read reflects only the member it resolves to, so it can report mappings present while another member under the same alias has none — which is the state that silently stops detection at the next rollover.
   ```
   GET /<each member index>/_mapping/field/<comma-separated rule fields>?pretty
   ```
   This field-level read is also the fallback where `_field_caps` is unavailable; in one engagement `_field_caps` was blocked by tooling restrictions and never ran, and the type conflict was established this way instead.
4. **Observation window.** Confirm the newest finding timestamp is still advancing after at least two of the detector's schedule intervals have passed. Read the interval from `schedule.period` on the detector. A single advancing reading is not enough to show that scanning is sustained.
5. **No other alias is in the same state.** This condition produces no signal of its own, so enumeration is the only way to know how many exist.
   ```
   POST /_plugins/_security_analytics/detectors/_search
   {"query":{"match_all":{}},"size":100}
   ```
   For each detector whose `inputs[0].detector_input.indices[0]` resolves to an alias:
   ```
   GET /_cat/aliases/<that alias>?v&h=alias,index,is_write_index
   ```
   **Expect:** for every alias, either a single member, or exactly one row reading `true`.

**Exit criteria.** Every alias backing a detector has exactly one member designated as its write index, the newest finding timestamp for each affected detector is advancing, and every shipper writes through the alias rather than to a concrete index. A single repaired alias is not the exit condition.

## If this does not work

**Reach out when:** the step 3 write does not take effect; or findings do not resume after two schedule intervals with the designation confirmed; or the detector still cannot be updated after the alias is correct.

**Reach out to:** your OpenSearch support channel, through the route your organisation uses for a production cluster.

**Attach:** the step 1 output before and after, `${TARGET_INDEX}` and how you chose it, the step 4 read-back, the newest finding timestamp at two points in time with the detector's schedule interval, and the engine and alerting plugin versions. Where the detector still cannot be updated, say so separately — that is the upstream defect rather than the alias state, and name the cast error without presenting it as the identifying symptom.

## Related runbooks

Siblings are referred to by name, which resolves to a directory and is greppable.

- `security-analytics-detector-health-audit` — the read-only sweep that finds this condition across every detector. Run it to size the problem before fixing one alias.
- `security-analytics-rollover-blind-window-remediation` — where the alias has a correct write index but documents written before an index joined the alias are still unexamined. A different gap with the same shape.
- `security-analytics-detector-disable-verification` — where a detector reports disabled and the question is whether it actually stopped.
- `security-analytics-rule-coverage-audit` — where findings arrive for some rules but not others.

## Verification status

Every step traces to a reputable source. A step resting on inference is reason to leave a procedure unwritten rather than to label it. The basis, what has been observed in practice, the open questions and any superseded claims are in `references/evidence.md` — read it before relying on this runbook for the first time, or before quoting it to anyone.
