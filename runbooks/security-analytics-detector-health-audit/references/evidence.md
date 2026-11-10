# Evidence — what this runbook rests on, and what it does not

Every step in `SKILL.md` traces to a replication, an engagement observation, or a reputable source. A step resting on inference is reason to leave a procedure unwritten rather than to label it. This file records which is which, so a reader can judge how far to trust each part.

`metadata.verification` is `documented`: the sweep has not been exercised end to end as written against a fleet with known-seeded faults.

**This runbook is a router, so its evidence has an unusual shape.** The enumeration and validation steps rest on published documentation. Each *flag*, however, rests on whatever established the fault it points at — and that evidence lives with the runbook that owns the repair. The basis below says which, because a flag is only as trustworthy as the finding it routes to, and two of the five flags are materially better supported than the others.

**Basis.**

- **Published OpenSearch documentation** for every route the sweep calls: the detector search route and the detector rule arrays; the findings search route with `detector_id`, `sortOrder` and `size`, and its `timestamp` and `total_findings` fields; the alerts route with its `alertState` filter and `ERROR` state; the Alerting workflow route and its `enabled` and `last_update_time` fields; the CAT aliases route and its `is_write_index` column; the CAT indices and CAT plugins routes; and the Cluster Settings route with `include_defaults`.
- **Published documentation on system-index protection** for the warning that governs step 2: a caller without access can be served an empty index reader, so a search returns `200 OK` with zero hits even though documents exist, and an empty result must not be read as proof of access. This is the source for validation item 2 being mandatory rather than advisory.
- **Engagement observation, carried from the owning runbooks**, for the two signatures the detector API cannot show. That `monitor_id` and `workflow_ids` are omitted from the detector API response on healthy and broken detectors alike was checked against both states on two separate clusters; `security-analytics-detector-workflow-recovery` records it. That an unset `is_write_index` renders as a literal `-` rather than `false` was observed on multiple live clusters; `security-analytics-alias-write-index-recovery` records it, together with the four-state branch this runbook's step 5 reproduces.
- **A public upstream defect report** for the rule-count flag: the ten-rule evaluation bound, the statement that no error, warning or log entry is emitted, and the statement that the `_execute` route shares the limitation — which is why the pre-flight forbids it.
- **A public upstream defect report** for the alias write-index flag, covering the un-editable-detector consequence.
- **Engagement observation** for the triggers-but-no-rules flag: detectors in that state were identified as the cause of a null-reference failure on the shared overview page, the plugin was reported to have created no monitor or workflow for them, and deleting them resolved it. Recorded during a support engagement, not vendor-documented.

**Replicated.** No step is marked replicated. What has been observed in practice, short of a controlled sweep:

- The detector API omitting `monitor_id` and `workflow_ids` on both healthy and broken detectors, on two clusters.
- `is_write_index` rendering as `-`, and the single-member-explicitly-`false` state being a genuine third failure mode rather than a variant of the first.
- A `size=0` findings request returning an empty array beside a non-zero total.
- A plugin-internal index returning zero hits for a caller without access while holding documents.
- Detectors with triggers and no rules breaking the overview page for every detector.

**Not established.**

- **Whether this flag list is complete.** It is the union of the states the shipped sibling runbooks establish, which is a record of what has been found, not a proof that nothing else is silent. A detector can fail in a way nobody has yet characterised, and this sweep would report it clean. Treat a clean result as "none of the known silent states", not as "healthy".
- **Whether the configuration-index hit count should equal the detector API count.** Step 2 instructs you to record a disagreement rather than reconcile it, because no published source states the relationship between the two sets. A difference might mean an incomplete read, a stale configuration document, or a detector the API will not return to you — and nothing available distinguishes these.
- **What a non-advancing findings timestamp means on its own.** It cannot separate "nothing matched, correctly" from "nothing could match". That is precisely why step 4 routes to `security-analytics-detector-mapping-audit` instead of concluding. Note also that detectors evaluate documents by arrival rather than by a timestamp window, so a quiet detector on back-dated data is not thereby faulty.
- **Which specific rules are affected where the rule-count flag fires.** The discarded evaluation results leave no record, so no procedure can name them. The flag means the whole rule set of that detector is unverified coverage, not that the first ten are safe.
- **Two flagged states have no owning runbook** — triggers-but-no-rules, and a data source that returns `index_not_found_exception`. Both resolutions are operator decisions (delete, or repoint) rather than procedures, so this runbook reports them and stops. If either acquires a procedure, the route belongs here.
- **Whether a detector reads every in-scope index behind its alias or only the alias's write index.** Two engineering analyses available to this review disagree, and the choice would change whether step 5 needs to inspect every member. This sweep does not depend on the answer: it reads the designation rather than inferring scope from it. The conflict is recorded in `security-analytics-rule-cost-audit/references/rule-patterns.md` and remains open.
- **The size of the rollover window step 5 warns about.** It is visible only during the rollover, so an arbitrary-hour sweep cannot measure it, and this runbook does not claim to. It names the condition and routes onward.

**To raise the level.** Exercise the sweep on a non-production cluster carrying one deliberately seeded instance of each flag — a detector saved with both rule arrays empty, an alias given a second member with the flag unset, a detector carrying twelve rules, and a detector pointed at a deleted index — then confirm step 3 flags exactly those four and nothing else, and that validation items 1 and 2 pass. That last part is the point: a sweep that finds the seeded faults but cannot demonstrate its own reads were complete has not been verified. Then set `verification: replicated` and record the version tested.
