# Layer 7 — Detection health signals

A conformant setup still drifts. This layer checks that drift would be noticed.

## The recommended state

**Error-state alerts routed to a channel someone reads.** A failed detector run raises an error-state alert, and that alert can be routed like any other. It is the one well-covered case, and the reference calls it the best available indicator of detector health.

## The reads

```
GET /_plugins/_security_analytics/alerts?alertState=ERROR&sortString=start_time&sortOrder=desc
```

**Expect:** `alerts[]` with `state`, `error_message`, `start_time` and `trigger_name`, and `total_alerts` at the response root.

Three properties of this call, each of which causes a wrong conclusion when missed:

- **Omit `detectorType` to sweep every detector visible to you.** Supplying it narrows to one log type, so a scoped call returns a complete-looking list that omits everything else.
- **Read `total_alerts`, not the length of `alerts`.** The array is page-bounded at `size`, default 20. An array *shorter* than the page size is trustworthy.
- **`trigger_name` of `"NoOp trigger"` is the literal marker for a failed detector execution**, as distinct from any trigger someone named.

Then confirm the alerts have somewhere to go. The channel wiring is configured per trigger, so read the detector's triggers from layer 4's enumeration and confirm each carries a notification channel — and that a test detector's trigger deliberately does not.

## What error-state alerts do not catch

**They catch a detector that fails. They do not catch a detector that quietly stopped matching.** Nothing off the shelf covers that, and it is the failure mode every layer below this one produces:

| Condition | Caught by an error alert? |
|---|---|
| Detector failing outright | **Yes** — this is the covered case |
| Field aliases dropped at a rollover | No. Affected rules are skipped silently |
| A rule enabled before its field existed | No |
| A field typed `text` where rules match terms | No. Zero findings reads as "nothing matched" |
| An alias with no write index | No. Nothing is scanned and nothing errors |
| Records written before an index joined the alias | No |
| A rule past the evaluation page bound | No. Nothing is logged |

So the honest posture is: error alerts are necessary and insufficient, and the gap is covered by **running the layers above on a schedule** rather than by a signal.

## What to monitor beyond the alerts

Findings arriving at all, per detector, is the cheapest proxy for "still matching":

```
GET /_plugins/_security_analytics/findings/_search?detector_id=${DETECTOR_ID}&sortOrder=desc&size=1
```

> ⚠️ **Read `total_findings`, not the length of the `findings` array.** A request with `size=0` returns an empty array beside a non-zero total.

**A non-advancing finding timestamp does not distinguish "nothing matched, correctly" from "nothing could match."** It is a prompt to check layers 1 to 3, not a verdict. And detectors evaluate documents by arrival rather than by a timestamp window, so a quiet detector over back-dated data is not thereby faulty.

**Check the mapping state on a schedule rather than on suspicion.** A field-alias drop is only visible as an absence, and the absence is created at a rollover — so a check run at an arbitrary hour can pass while the condition recurs daily. The reference's own verification list is per-rollover for this reason.

## Cluster-level signals worth having

Not detector health, but the conditions under which detector health degrades: per-node CPU, JVM memory pressure, garbage-collection counts, thread-pool rejections, and the cluster's shard count. Read per node where the metric supports it — a domain-level average hides a single-node fault.

**Cluster health is not a detector-health signal.** It stays green through a detection outage, and green with every data node saturated.

## What non-conformance costs

| Finding | Cost | Route |
|---|---|---|
| Error-state alerts not routed anywhere | The one covered failure mode goes unnoticed | Operator change: wire a channel per trigger |
| Error-state alerts present | The setup is not merely non-conformant — evaluation is being rejected | `security-analytics-percolate-failure-investigation` |
| No scheduled mapping check | Every silent condition in the table above accumulates undetected | Run layers 1 to 3 on a cadence tied to the rollover |
| A production trigger with no channel, or a test trigger with one | Alerts go nowhere, or test alerts reach production | Layer 6 |

## Sources

- **Published documentation** for the alerts route and its `alertState` filter, for `total_alerts` at the response root and the array's page bound, for the findings search route with `detector_id` and `total_findings`, and for notification channels being configured per trigger.
- **Published documentation** for the cluster metrics named, and for their per-node dimensions.
- **A reference setup produced during a support engagement** for the verification list being per-rollover rather than point-in-time, and for the statement that error alerts are the best current indicator of detector health. **Engagement-sourced.**
- **Engagement observation**, carried from `security-analytics-wayfinder`, for: `"NoOp trigger"` being the literal marker of a failed execution; cluster health staying green through a multi-week detection failure; and the statement that a failed run raises an error alert but that nothing off the shelf catches a detector that stopped matching.

**Not established.** What a non-advancing finding timestamp means on its own — it cannot separate a correct quiet period from an inability to match, which is why this layer treats it as a prompt rather than a signal. Detector progress metrics and mapping watchers were described during the engagement as being explored for a future release; nothing here depends on them existing.
