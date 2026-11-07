# Evidence — what this runbook rests on, and what it does not

Every step in `SKILL.md` traces to a replication, an engagement observation, or a reputable source. A step resting on inference is reason to leave a procedure unwritten rather than to label it. This file records which is which, so a reader can judge how far to trust each part.

Every step in this document traces to a reputable source. A step resting on inference is reason to leave a procedure unwritten rather than to label it.

**Basis.**

- **Published OpenSearch documentation** for every API call: the detector search route and the detector rule arrays; the findings search route with `detector_id`, `sortOrder`, `size` and `startIndex`, and the `queries[].id` and `timestamp` fields; the Cluster Information route and its `version.number` field; and the CAT plugins route with its `component` and `version` columns. The `startTime` and `endTime` parameters come from the published machine-readable API specification and are absent from the documentation page — step 4 says so and gives the fallback.
- **A public upstream issue** for the defect, the mechanism and the operational consequence, including the statement that no error, warning or log entry is emitted and that the `_execute` route shares the limitation. Reported, labelled as a bug, and open at the time of writing.
- **A public upstream change** for the fix, and **code analysis** of the published implementation on the development line: `alerting/src/main/kotlin/org/opensearch/alerting/transport/TransportDocLevelMonitorFanOutAction.kt` sets the search size explicitly from the constant defined in `alerting/src/main/kotlin/org/opensearch/alerting/util/AlertingUtils.kt`, whose value is 10,000. This is the source for the post-change ceiling being bounded rather than unlimited.

**Replicated.** No step is marked replicated, and `metadata.verification` is `documented` accordingly. The code analysis was performed against a checkout of the development line, which carries the change — so it establishes what a patched build looks like and cannot demonstrate the unpatched behaviour. The published issue is the source for that side.

**Not established.**

- **Which specific rules are dropped.** The discarded hits leave no record, so no procedure can name them. This is why the blast radius is the whole rule set of an affected detector.
- **The mapping from a finding's `queries[].id` to a named rule.** Published examples are inconsistent about the relationship, so step 4 counts distinct ids and does not claim that a given finding came from a given named rule.
- **Whether a particular released build carries the change.** Backports to the older lines were open at the time of writing, and that state moves. This is why step 4 is the answer and step 3 is corroboration.
- **Any recommended maximum rule count.** No published source gives one. The ten in this document is the observed page bound, not guidance.
- **Any engagement evidence of this defect.** Reviewed support records contain no instance of a findings shortfall measured against a configured rule count. Detectors carrying far more than ten rules have been seen running in production, but the failure observed there was the opposite shape — over-production of findings driving load, not silent loss. The public issue and the code change remain this runbook's only support, and step 4's behavioural check is the only thing that will tell you whether a given build is affected. **Do not read this document as case-replicated.**

**To raise the level.** Exercise the procedure against a detector with a known rule set on a build known to lack the change, confirm the distinct-firing-rule count plateaus at the bound, then set `verification: replicated` and record the version tested. The published change carries a regression test that extends a doc-level monitor past the bound and asserts the full count, which is a usable shape for that exercise.
