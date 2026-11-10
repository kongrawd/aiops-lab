# Layer 2 — Alias and ingestion

This layer decides whether records land where the detector is looking. It is the layer that most often fails, and the one whose failure is most completely silent.

## The recommended state

Four properties, each independently checkable:

1. **One alias per log source**, read by one detector.
2. **An explicit write index** — exactly one member carrying `"is_write_index": true`.
3. **Shippers write to the alias**, not to a dated index name they construct themselves.
4. **The next index is created by an ISM `rollover` action**, which creates it already on the alias as the write index — not by the first record written after midnight.

**Why each matters.** A detector on an alias tracks its read position on that alias's **write index**. Where no member carries the designation, the resolution yields nothing and the detector scans no part of a new index. Where the shipper names the dated index itself, rollover does not move it — rollover moves the alias, not the shipper's destination — so a shipper left pointing at dated names creates indices outside the rollover series that the detector never reads.

## The reads

```
GET /_cat/aliases/${ALIAS}?v&h=alias,index,is_write_index
GET /_plugins/_ism/explain/${INDEX}
GET /_plugins/_ism/policies/${POLICY_ID}
GET /${INDEX}/_settings
```

> ⚠️ **An unset `is_write_index` renders as a literal `-`.** Not `false`, not an empty column. A check that greps for `false`, or tests for an empty field, matches nothing.

**Four alias states, distinguishable only by reading the flag *and* counting members:**

| Members | `is_write_index` | State |
|---|---|---|
| One | `-` | The single member is the **implicit** write index. Conformant for a single-member alias, and the reference records it as such |
| Two or more | every row `-` | **No write index.** Writes through the alias fail with `no write index is defined for alias`, and the detector resolves nothing |
| Exactly one | explicit `false` | **Also a failure** — an explicit `false` overrides the implicit behaviour |
| Any | one row `true` | Designated. This is the target state, and an explicit `true` wins over other members' unset values |

## Bootstrapping the series

A rollover series cannot start from an index whose name is a literal dated string. Rollover requires a name ending in a numeric counter, so the first index of the series is created explicitly, with date math, and with the alias as its write index:

```
PUT <${INDEX_PREFIX}-{now/d{yyyy-MM-dd}}-000001>
{"aliases":{"${ALIAS}":{"is_write_index":true}}}
```

The name is sent URL-encoded. Three properties of this that decide whether the series continues:

- **The index retains the date-math expression as its `provided_name`.** Each rollover resolves it again and increments the counter, so names keep a date and the counter never resets.
- **An existing literal dated index cannot be rolled.** Its `provided_name` does not end in a counter, so rollover fails against it — which is why a bootstrap index is needed rather than simply enabling rollover.
- **Omitting `is_write_index: true` on the bootstrap is a failure, not a default.** The alias then has two members and no write index.

Do not set `is_write_index: true` on the pre-existing dated index, and do not alter its alias entry. The explicit `true` on the bootstrap makes it the only write index while the older member stays readable.

## The rollover action, and where it must sit in the policy

The reference adds `{"rollover": {"min_index_age": "1d"}}` as the action of the policy's **first** state, ahead of any state that manipulates the alias. That ordering is load-bearing:

- An ISM `alias` action that *adds* the alias writes a fresh alias entry, which **clears `is_write_index`** on that index.
- If that ran against the current write index, the alias would be left with members and no write index. Writes through it would fail and the next rollover would fail its pre-check.
- With rollover in the first state, an index reaches the alias-manipulating state only *after* it has been rolled — by which point it is no longer the write index, and clearing an already-`false` flag is harmless.

**Index age is counted from creation and rollover does not reset it**, so retention transitions expressed as `min_index_age` keep their meaning after rollover is introduced.

## Two alternatives for introducing rollover

Both work. The trade-off, not a verdict:

**Alternative A — add the rollover action to the existing policy, same id.**
- One policy owns the prefix. The delete and alias logic exist once and cannot drift.
- Rollback is a single `PUT` of the body you fetched.
- Use optimistic concurrency on the write (`if_seq_no` and `if_primary_term` from the read) so a policy that changed since you read it fails rather than being overwritten.

**Alternative B — a new policy id with a higher `ism_template` priority.**
- New indices pick the higher priority; existing dated indices retire under the old policy.
- **Cost:** two policies match the prefix indefinitely, with priority as the only tie-break, and the delete and alias logic exist twice and can drift apart.

**Either way, a policy update does not reach indices already managed by it** — each keeps its cached copy of the policy until it retires. So the day's existing index behaves as before, and only new indices pick up the change.

## The hazard both alternatives share

If the shipper is still writing dated names when the next day begins, it creates a dated index *after* the policy change. That index picks up the new policy, reaches the rollover step, and **fails its pre-check** because it is not on the alias — reported as a missing-alias-or-not-the-write-index rollover failure. It is then never added to the alias, never read by the detector, and never deleted.

The sequencing that prevents it: switch the shipper to the alias within the same day the policy changes, and wait for the day's existing index to have loaded its policy before changing the policy. Where it has already happened, the recorded remedy is to set `rollover_skip` on that index and retry it, which returns it to the old path.

## Rollover drift

Rollover fires when the index reaches the age condition **plus up to one ISM polling cycle**, so it does not occur at midnight and the firing time moves slightly later each day. Over many months it can cross midnight, at which point one date is skipped in the index names. A manual rollover just after midnight pulls the time back; ISM treats the previous index as already rolled and continues from the new one.

**Consequence for this layer:** a validation that expects a rollover at a fixed clock time will report a false finding. Check that rollover happened within a cycle of the age condition, not that it happened at a particular hour.

## What non-conformance costs

| Finding | Cost | Route |
|---|---|---|
| Two or more members, none designated | Nothing is scanned; writes through the alias fail | `security-analytics-alias-write-index-recovery` |
| Exactly one member, explicit `false` | Same, and easier to miss | `security-analytics-alias-write-index-recovery` |
| Shipper writes dated names | Indices outside the series, unread and undeleted; new indices also miss the template's field aliases if the template does not match them | Operator change, needs its own approval |
| No `rollover` action in the policy | Each day's index is created by the first record after midnight and joins the alias minutes later, without a write index. Records written before it joins are not examined | Alternative A or B above |
| Rollover action placed after an alias-add action | Clears the write-index designation on the live write index | Policy correction |
| More than one detector on `${ALIAS}` | Shared field-mapping state; see layer 6 | Layer 6 |

**The window this layer closes.** Where an index is created by a shipper and attached to the alias afterwards, records written before the attach are not examined, because the detector reads from the attach point rather than from the index's first record. That window is only observable during a rollover, so it cannot be measured at an arbitrary hour — which is why this layer checks the *mechanism* rather than trying to measure the gap. See `security-analytics-rollover-blind-window-remediation`.

## Sources

- **Published documentation** for the alias write-index requirement on a detector data source, for documents needing to be ingested through the alias rather than the underlying index, and for an alias with more than one member rejecting writes until one member is designated.
- **Published documentation** for the ISM `rollover` action, its age condition, and the `rollover_alias` setting.
- **Code analysis of the published alerting plugin** for why a missing designation is fatal: the write-index resolution returns **null** for an alias or data stream with no designation, and the per-index read position is held per shard keyed by index. A detector therefore has nothing to resolve and nothing to track.
- **A reference setup produced during a support engagement** for: the bootstrap request and its date-math form; the statement that an existing literal dated index cannot be rolled because its `provided_name` lacks a counter; that an explicit `true` wins over unset members; that omitting it yields `no write index is defined for alias`; the ordering constraint between the rollover action and any alias-add action, with the reason; that a policy update does not reach already-managed indices; the dated-index-after-policy-change hazard and the `rollover_skip` remedy; the rollover drift and the manual-rollover correction; and the single-member implicit-write-index reading. **Engagement-sourced, not vendor-documented**, and the ordering constraint in particular is reasoning given in that document rather than a published statement.
- **Engagement observation** for the pre-attach window being unexamined, carried from `security-analytics-detector-health-audit`.

**Not established.** Whether a detector reads every member of its alias or only the write index. The plugin exposes both a write-index resolution and a resolve-all-indices path, and which governs a scheduled evaluation was not traced. This layer does not depend on the answer: it checks the designation rather than inferring scope from it.
