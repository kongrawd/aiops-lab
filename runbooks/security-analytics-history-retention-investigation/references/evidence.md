# Evidence — what this runbook rests on, and what it does not

Every step traces to published documentation or to code analysis of the published implementation. A claim resting on inference is reason to leave it out rather than to label it, and this revision removes several that an earlier draft labelled instead.

`metadata.verification` is `documented`: no step has been exercised against a cluster carrying a seeded stall.

## What this runbook deliberately does not attempt

**It does not establish why pruning stopped.** An earlier draft did: it derived the pruning pass's wall-clock time from history-index creation dates, compared that against the automated-snapshot window, and branched on the overlap. Both halves were unsound.

- **The firing time is not derivable from creation dates.** An index is created only on a pass where a rollover condition is met, so creation dates are a subset of passes — and at low findings volume no new index is the normal result, which the draft's branch logic read as the job having stopped entirely.
- **The snapshot listing cannot support the comparison.** Its time columns carry a time of day without a date, so a multi-day overlap check cannot be evaluated from them, and an in-progress snapshot renders an end time rather than leaving it blank.

So the cause is not reachable from the reads available to an operator. The procedure now establishes the condition and sizes it, and the *If this does not work* section states the question to put to support rather than answering it. **Narrower and true.**

## Basis

- **Code analysis of `DetectorIndexManagementService` in the published Security Analytics plugin** for the mechanism in **Impact**: the prune collects every index whose age exceeds the configured retention and sends them in a single delete request, and the retry that follows rebuilds its request from the full list rather than from the index it is iterating, so it fails identically. **The single-request delete and the broken retry are present on every published branch read.**
- **Code analysis of OpenSearch core** for the rejection: index deletion checks whether a running snapshot includes any of the requested indexes and rejects the request **as a whole** before deleting anything.
- **Code analysis of the plugin** for the log levels behind the statement that nothing operator-visible surfaces: the retry's own failure is logged below the default level.
- **Published Security Analytics settings documentation** for the settings step 1 reads — that the `*_rollover_period` settings control how frequently history indexes are rolled over and deleted, and the `*_retention_period` settings how long they are kept.
- **The CAT indices reference** for `expand_wildcards`, for the `creation.date.string`, `pri` and `rep` columns, and the index-name-syntax reference for `all` including hidden indexes.
- **The cluster settings reference** for `include_defaults` being required to see defaults, and for defaults being returned from the local node.
- **Code analysis of the published plugin** for every plugin index family being created with `index.hidden`, which is what makes step 3's dot-prefix classification meaningful.
- **Engagement observation** for the condition having occurred: on a live cluster, no expired history index was pruned over an extended period, and the surviving indexes came to account for the large majority of the cluster's shards while the operator's own indexes were a small minority. **Not vendor-documented**, and no figures from it are reproduced here.

## Removed in this revision rather than labelled

Recorded so they are not re-introduced.

- **The firing-time derivation and the snapshot-window comparison built on it.** See above.
- **A step searching the error log for the engine's rejection string.** The plugin's own error line carries its own message and not the exception text, so the string the draft searched for cannot appear on that path at the default level. The step was removed rather than retargeted, because a log search that depends on publishing having been enabled during the window is not something this procedure can rely on.
- **A "what differs by version" table describing an independent-delete path.** No published branch has one; the behaviour was reasoned by analogy with how the defect might have been fixed. Its discriminator also cost a full pruning interval before the operator could begin.
- **"A single job" framing.** The plugin runs a pass per history family, each with its own schedule and its own delete request, so a collision blocks one family rather than everything expired.
- **"A copy on every data node, growing as the cluster is scaled out."** The replica setting carries a ceiling, so the footprint stops growing. The procedure now reads the setting and its ceiling instead of asserting the behaviour.
- **A two-listing shard diff using `expand_wildcards` on the CAT shards API**, where that parameter is not documented. Step 3 uses a single CAT indices listing, where it is, and classifies by index name.
- **A pre-flight warning that a dot-prefixed pattern without `expand_wildcards=all` returns nothing.** The engine matches a dot-prefixed hidden index against a dot-prefixed wildcard pattern regardless, so the warning did not describe the patterns in use.
- **Two mis-cited references** — an upstream issue about segment merges, titled in the draft as though it were about snapshots, and a delete-index API page supporting no step. Both removed; the claim they were cited for is sourced from the core implementation instead.
- **An illustrative percentage split** inside the procedure, and the claim that expired indexes accumulate without bound, which contradicted the same document's statement that a completed pass prunes them.

## Not established

- **Why a pruning pass does not complete on a given cluster.** This is the subject the runbook declines to decide. The engagement that produced it correlated the condition with a snapshot window but did not demonstrate causation, and the reads an operator has cannot separate that from other reasons a pass might not complete.
- **Whether the condition self-clears over a long window.** Step 2 treats an emptied set as a transient collision, which follows from the mechanism, but the net direction over months was not measured.
- **The relationship between this pass and an operator's own index-lifecycle policy.** Where a policy also manages indexes matching these patterns, which deletion path acts first was not established, and a policy that deletes them would mask the condition entirely.
- **Whether the replica setting can be changed on these families, and what that affects.** The procedure reads it to explain the per-index cost and stops there.

## To raise the level

On a non-production cluster, set a short retention and rollover period so several history indexes expire, and confirm the procedure's three outcomes: that step 1 identifies the expired set, that step 2 distinguishes a transient collision from a persistent one across two intervals, and that step 3's population sums reconcile against the total. Then set `verification: replicated` and record the version tested.
