# Starting points

The signal inventory: what exists, which of it a reader outside the service can obtain, and what to monitor so a silently broken detector is caught before someone notices.

Use this when you do not yet have a signal, or when you need to know whether a signal you are missing is obtainable at all.

## The prerequisite that cannot be met retroactively

**Error logs exist only if publishing to CloudWatch Logs is already enabled.** They cannot be turned on afterwards to recover past lines, so a detector problem that started before publishing was configured has no log evidence at all.

**Error logs carry more than `WARN` and above.** Alongside `WARN`, `ERROR` and `FATAL`, error-log publishing also includes several `DEBUG`-level exceptions — among them `MapperParsingException`, `QueryShardException`, `SearchPhaseExecutionException`, `OpenSearchRejectedExecutionException` and `IllegalArgumentException`. The mapping-conflict signature below is built from the first two, so it is reachable without raising any logger level.

## Why the ordering is not what it looks like

The reads that look most natural are the ones that mislead: cluster health, a domain-level CPU average, and a monitor search all return healthy while a detector produces nothing. Several detector problems are not visible in any log, and are established instead from published plugin source plus a controlled reproduction on a throwaway domain.

So the ordering is not "cheapest first." It is **the read that can distinguish failure classes, before the read that looks most natural.**

## Tier 1 — the error alert, and what it tells you

The cheapest read, and the one that most often carries the answer.

```
GET _plugins/_security_analytics/alerts?alertState=ERROR&detectorType=${DETECTOR_TYPE}&sortString=start_time&sortOrder=desc
```

The field-by-field anatomy and the five `error_message` shapes are in `SKILL.md` steps 1 and 2 — that is the routing path and it belongs there. Two properties of the alert worth recording here because they are retention constraints rather than routing facts:

- **`alert_history` is capped at 10 entries**, and it logs *distinct* error messages rather than executions — it grows only when the message changes.
- **`version` is the durable count.** It survives as a cumulative failure counter when the history has rolled.

One alert identity detail: `id` renders as `error-alert-<monitorId>-<uuid>` — it embeds the **monitor** id, not the detector id.

## Tier 2 — the configuration reads

| Read | Establishes |
|---|---|
| `POST /_plugins/_security_analytics/detectors/_search` with `{"query":{"match_all":{}},"size":100}` | The detector inventory, `enabled` state, rule arrays, data source, and the detector types tier 1 needs |
| `GET ${CONFIG_INDEX}/_doc/${DETECTOR_ID}` | `monitor_id` and `workflow_ids`, which the detector API does not return |
| `GET _plugins/_alerting/workflows/${WORKFLOW_ID}` | Whether the scheduled job agrees with the detector record |
| `GET _cat/tasks` | What is executing now, independent of any configuration |
| `GET _cat/aliases/${ALIAS}?v&h=alias,index,is_write_index` | Whether the data source resolves to anything |
| `GET /_cat/indices/.opensearch-sap-*-detectors-queries*?v` | Whether each detector's query index still exists. Compare against the inventory — a mismatch is a broken detector |
| `GET _cluster/settings?include_defaults=true` | `indices.query.bool.max_clause_count`, and `plugins.security_analytics.filter_by_backend_roles`. Defaults come from the local node |
| `GET /_cat/plugins?v&h=name,component,version` | Which side of a version-dependent behaviour this build sits on |

**`filter_by_backend_roles` changes what your enumeration means.** Where it is enabled, you see only detectors created by users sharing a backend role — so a sweep over a subset can report a clean fleet. Read it and record the value with your results.

## Tier 3 — cluster and cost signals

| Read | Establishes |
|---|---|
| `GET _cat/nodes?v&h=name,uptime,heap.percent,cpu,load_1m,load_5m,load_15m` | Whether load is spread or concentrated, and whether it is climbing. `load_1m > load_5m > load_15m` means actively degrading rather than plateaued. The explicit `h=` list matters — a bare `_cat/nodes?v` buries `cpu` and the load columns among a dozen others |
| `GET _nodes/hot_threads?threads=5&interval=1s` | What is burning CPU, by code path. It separates causes that present identically — query-index rebuild and wildcard evaluation both read as high CPU correlating with ingest |
| `GET _cat/indices/.opensearch-sap-*-findings-*?v&h=index,docs.count,store.size&s=docs.count:desc` | Findings volume **per log type** — the index name carries the log type, not the detector. To attribute volume to one detector, query the findings API with its `detector_id` |
| `GET _cat/shards/${PATTERN}*?v` | How much parallelism a detector gets. One shard means one node does all the percolation |
| `GET ${INDEX}/_mapping` per concrete index | Whether two indexes in scope declare a field with different types |
| `GET ${INDEX}/_field_caps?fields=*&include_unmapped=true` | Whether the fields the rules reference exist. `include_unmapped=true` is the parameter that matters — without it an absent field is simply missing from the response, which reads the same as present and fine |

### Reading hot threads

Three discriminators, in order:

1. **Thread name tells you the subsystem.** `DefaultDispatcher-worker-N` is a plugin coroutine pool, not an OpenSearch thread pool — plugin code. `opensearch[...][search][T#N]` is query evaluation. `[write]`, `[refresh]` and `Lucene Merge Thread` are usually benign.
2. **`N/10 snapshots sharing` separates signal from sampling artefact.** `9/10` or `10/10` on an identical stack is steady state.
3. **The allocating frame names what is being copied.** Which frame allocates tells you what is being copied: a copy constructor names the receiver, while a bulk-add method names the argument. One frame can carry the whole answer.

Two signatures recorded on different clusters with the same outward symptom: `DocLevelMonitorQueries.indexDocLevelQueries` → `transformExistsQuery` → `Regex.split` is per-run query-index rebuild; `WildcardQuery.<init>` → `AutomatonQuery.<init>` under `PercolateQuery` is wildcard evaluation cost. Same presentation, unrelated causes, told apart by one read.

## Error log signatures

All public, subject to the publishing prerequisite above. Grep the logger, then the message.

| Logger / string | Means |
|---|---|
| `[ERROR][o.o.a.t.TransportDocLevelMonitorFanOutAction]` with `Failed to run percolate search for sourceIndex [...] and queryIndex [...] for N document(s)` | A per document monitor's fan-out could not percolate a batch. **The document count is the measurement** |
| `[ERROR][o.o.a.DocumentLevelMonitorRunner]` with `IndexNotFoundException: no such index` | The detector points at something that does not exist; every scheduled run fails outright |
| `[ERROR][o.o.a.u.DocLevelMonitorQueries]` | Saved-query writes into the query index failing. The log pattern abbreviates every package element to one letter, so the bare class name never matches |
| `too_many_nested_clauses`, or `maxClauseCount is set to <N>` | Rule expansion exceeded the limit. **The limit is echoed in the exception**, so after raising it you can confirm the change took effect from the next failure |
| `CircuitBreakingException`, `[parent] Data too large` | Percolate batches tripping the parent breaker |
| `MapperParsingException` with `QueryShardException` on a `.opensearch-sap-*-detectors-queries*` index | A rule's query could not be parsed against the mapping, so the detector's field mappings were not resolvable when it wrote its saved queries. Both exception classes are published with error logs |
| `Underlying workflow associated with detector <detector> not found.` | Surfaced in the console on stop or start: the detector's composite monitor is not resolvable. **The message suggests deleting and recreating the detector, which is irreversible and changes the detector id** — the existing findings and alerts survive but become unreachable through any detector-scoped read. Confirm the cause before following it |

The percolate-failure string is also visible **without** log publishing, in the browser network tab on the detector page.

## Metrics

Read each of these per node where the metric supports a node dimension — a domain-level average hides a single-node fault.

`CPUUtilization` (per `NodeId`), `JVMMemoryPressure`, `JVMGCYoungCollectionCount`, `JVMGCOldCollectionCount`, `ThreadpoolWriteRejected`, `ThreadpoolSearchRejected`, `ThreadpoolSearchQueue`, `ThreadpoolWriteThreads`, `IndexingRate`, `SearchRate`, `SearchLatency`, `Shards.active`, `Shards.unassigned`, `Shards.activePrimary`, `SearchableDocuments`, `Nodes`, `ClusterStatus.red`, `ClusterStatus.yellow`, `IopsThrottle`, `ThroughputThrottle`.

Three things about using them:

- **Period selection decides whether the fault is visible.** Hourly aggregation smooths away sub-minute events such as circuit-breaker trips.
- **Compare the same hour across days at equal `IndexingRate` to attribute a CPU change.** A step down in CPU at flat ingest attributes to whatever detector changed; a change in both attributes to load.
- **The `Nodes` metric counts dedicated cluster manager nodes and warm nodes as well as data nodes.** Subtract them before reading it as a data-node figure. A change in the metric is still a change of the same size, because those counts are a constant offset.

## Timing: capture before you remediate

A remediation usually destroys the evidence of its own necessity.

| Signal | Destroyed by | Window |
|---|---|---|
| Hot threads, `_cat/tasks` | Any restart | Live only. Captured minutes before a restart they support an RCA; captured after one they do not exist |
| Cumulative rejection counters | Process restart | Before the bounce only |
| Thread age (`elapsed` in a dump) | Restart | Before the bounce. It is what correlates a stuck thread to its trigger |
| The mapping before-state | Re-applying mappings | One-shot, between the conflict and any fix. Without it the cause has to be argued from product behaviour rather than read from data |
| The component template's contents, and which index template carries it | Any field-mappings save — including opening the Dashboards *Edit field mappings* page, which issues one on open | One-shot. The save overwrites the component template from a single index and can move which template composes it, so the state that explains the fault is gone and the post-repair reads look healthy |
| The conflicting index pair | Retention deleting the older member | Only while both exist — bounded by the retention period on those indexes |
| Reachability of findings and alerts | Recreating the detector | The documents survive, keyed to the old detector id. Record that id first, or no detector-scoped read reaches them again |
| An orphaned resource under investigation | The customer cleaning it up | A resource deleted before anyone inspects it cannot be investigated afterwards — ask for the capture before the cleanup |
| `alert_history` entries | Ten more executions | A bounded ring |
| Node logs | Rotation and retention | A first-failure timestamp survives only while the hour that contains it is still retained |
| The credential-churn signal in an auth-path failure | Mitigating | A controlled re-exposure window had to be engineered to measure it after mitigation erased it |

Four more constraints:

- **Sampling outside the daily ingest window returns a clean bill of health.** Where load arrives as a daily wave, the CPU signature exists only inside it.
- **One dump is not enough.** Telling *frozen* from *slow* needed three dumps over about eighteen minutes and the observation that the same thread identities carried 0.0 ms of CPU between them.
- **A detector that is already working breaks only after the next rollover plus one scheduled run.** The break is not observable when the alias goes bad.
- **A point-in-time REST read cannot settle a retrospective question.** Cluster health is instantaneous and unrecoverable, so a disputed health snapshot can only be answered from retained cluster-status and shard metrics.

The mapping-conflict log burst is **offset from index creation by a fixed interval, and it lasts under a second.** A log search anchored on the rollover timestamp misses it — widen the window and search for the signature rather than the time. Thread-pool metrics also rise before anyone declares impact, so a reported start time is the wrong left edge for a metric query.

## What a reader cannot obtain

**No documented read produces these.** That is a statement about the documented surface, not a guarantee about every future release — where a signal below matters to you, check the current API and metric references before concluding you cannot have it. Each row says *why* it is unavailable, because the reason outlives the absence.

| Signal | Why, and the nearest substitute |
|---|---|
| Per-node JVM object counts, or a heap class histogram | Needs service-team access. Support cannot obtain it either, and it is sometimes the only read that quantifies the problem |
| Allocation-rate data | Not published. Garbage-collection counts and times are — `JVMGCYoungCollectionCount`, `JVMGCOldCollectionCount` and their `...Time` counterparts |
| JVM thread dumps | A plugin coroutine dispatcher is not an OpenSearch thread pool, so it does not appear in thread-pool stats, and its threads park, so hot threads excludes them. The public path is recognising the behavioural signature |
| Addressing one specific node | Calls to the domain endpoint reach a load balancer and return a statistical mix across nodes. This is why a per-node fault presents as intermittent |
| A single-node in-place restart | Available as a documented administrative option — restarting the OpenSearch process on a data node, or rebooting the node — subject to its limitations: three or more data nodes, no Multi-AZ with Standby, cluster not red, and every index having replicas configured |
| An aggregated count of detectors in error | `GET _plugins/_security_analytics/alerts?alertState=ERROR` with no `detectorType` returns every visible detector's error alerts in one search, with `total_alerts` |
| A per-detector clause-count metric | The limit is a per-query check rather than a consumable cluster resource, which is the reason given for not exposing it as a metric. A breach shows in the error string |
| The detector's saved run context | No documented read returns it. A controlled reproduction substitutes |
| Managed-service proxy-layer blocked threads | Not published |

Partial substitutes that work: a browser HAR or devtools timing in place of server-side request timing; the `uptime` column of `_cat/nodes` in place of host process uptime; `_nodes/hot_threads` in place of a thread dump, for CPU-active work only; CloudTrail `AssumeRoleWithWebIdentity` counts in place of credential-churn telemetry; the published plugin source in place of a mechanism you cannot observe; and **a controlled reproduction on a throwaway domain in place of a read that does not exist**.

## What to monitor

**A failed detector run raises an error-state alert that can be routed to a notification channel. That alone does not catch a detector that has quietly stopped matching** — nothing off the shelf covers that, so field-mapping loss has to be checked on a schedule.

The posture to aim for:

| Condition | Caught by |
|---|---|
| Detector failing outright | Error-state alerts routed to a channel. This is the one well-covered case |
| Detector stopped matching | Nothing off the shelf. Compare mappings across every index in scope, on a schedule |
| Saved queries missing after a blue/green deployment | `GET /_cat/indices/.opensearch-sap-*-detectors-queries*?v` against the detector inventory. **Run it immediately afterwards**, not later |
| A composite monitor disagreeing with its detector | The composite monitor read, per detector. Not available from the detector API |
| A rule that never compiled into the query index | Nothing public flags it. The rule-count-versus-query-index comparison is a derivation, not a validated procedure |
| Accumulating evaluation cost | Findings volume since the last restart, trended |
| Rising cost at flat ingest | Detector execution duration week over week, and the daily window CPU peak. Neither has a dedicated metric — derive them from execution timing and per-node CPU |

Those last two are the most direct customer-visible signs of accumulating cost, and both have to be derived rather than read.

## A post-remediation checklist, and its status

Run this after a service software update, a version upgrade, or any prior detector repair — all of which use a blue/green deployment. Where the detector API omits the link identifiers, read them from the plugin-owned document instead.

Present it as a proposal, with that step corrected to read the plugin config document:

- List all detectors.
- Check each detector's `enabled` state, and read `monitor_id` / `workflow_ids` **from the config document**.
- Verify query indexes exist and compare the count against the detector count — a mismatch is a broken detector.
- Check for error alerts per detector type.
- Verify findings are being generated, with a recent timestamp filter.

The two unanswered questions are worth carrying as open: why the upgrade process does not validate query-index existence afterwards, and whether the platform can detect and alert when a detector references a non-existent index — which today fails silently with no customer-facing notification.
