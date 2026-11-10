# Blind checks

Reads that return a clean, confident-looking result while the fault is at full strength.

Read this before interpreting any result in this skill as clean. Each row states the mechanism that makes the read misleading, so it applies wherever that mechanism does.

## The rule that governs all of them

**A zero is interpretable only when it sits beside a non-zero from the same read.** A zero with a positive control next to it — sibling nodes in the same response carrying non-zero counts, a known-good object read through the same path — is a true negative. A zero on its own is uninterpretable.

Where you cannot produce a positive control, **record the check as not evaluated rather than as passed.**

## Reads that cannot see what you are asking about

| Read | What it returns | What it means |
|---|---|---|
| `GET /_plugins/_security_analytics/detectors/<id>` | A healthy-looking detector | It returns a hand-built subset and omits `monitor_id`, `workflow_ids` and `rule_topic_index`, so a detector whose composite monitor is deleted, or that points at a deleted saved-query index, is indistinguishable from a healthy one. **Use `POST /detectors/_search`**, which returns the raw `_source` and carries them |
| `_plugins/_alerting/monitors/_search` | `0` document-level monitors | The search restricts results to monitors whose `owner` is absent or equals `alerting`. Security Analytics monitors carry `owner: "security_analytics"`, so **none of them appear.** The filter is skipped entirely, for every owner, whenever the literal string `monitor.owner` occurs anywhere in the serialized query — which is why adding a `monitor.owner` clause makes them visible. `GET /monitors/<id>` is not filtered |
| `GET .opendistro-alerting-config/_search` | `"hits":{"total":{"value":0,"relation":"eq"}}` beside `"_shards":{"successful":5,"failed":0}` and `"timed_out":false` | The security plugin serves an empty reader to a caller without system-index access. **Every integrity marker reads trustworthy and the result is still blind.** Use the plugin REST APIs |
| `GET .opensearch-sap-detectors-config/_doc/<id>` | Empty or absent | The same masking — a get can behave as though the document does not exist. This index is registered as a system index even though the documentation's system-index table omits it, so do not conclude from that table that it is unprotected. Use the plugin API instead |
| `GET _plugins/_security_analytics/alerts?...&detectorType=X` | A complete-looking list | Scoped to that one log type, so detectors of other types are absent. `detectorType` is optional — omit it to cover every detector visible to you in one search |
| `GET _cluster/settings` without `include_defaults=true` | No `max_clause_count` | Defaults are omitted. `include_defaults=true` is required to see them |
| An HTTP access log over a window when something changed | No mutating calls | **Transport-layer writes do not appear in an HTTP access log.** Anything done through internal tooling, or by the platform operator, is invisible there — so an empty access log is not evidence that nothing happened |
| `GET /` | A version far below the engine's actual one | On 1.x and 2.x, `compatibility.override_main_response_version` makes the cluster report `7.10.2`; the setting was removed in 3.0.0. The `version` column of `_cat/nodes` is overridden too, so that is not an escape. Read `GET /_nodes/<node>?filter_path=nodes.*.version` |

## Counts and arrays that under-report

| Read | What it returns | What it means |
|---|---|---|
| `alerts` array from the error-alert API | 20 objects, or exactly your `size` | Page-bounded at `size`, default 20, while `total_alerts` carries the matching total. **Read `total_alerts`** — but the all-detectors path tracks it only up to 10,000, so exactly `10000` is a floor, not a count. An array *shorter* than the page size is trustworthy |
| `findings` array | Empty | A request with `size=0` returns an empty array beside a non-zero total. Read `total_findings` |
| `_plugins/_security_analytics/detectors/_search` with `size:100` | A full page | A result count equal to the `size` you asked for is not evidence of completeness — read `hits.total.value`. `from` plus `size` is itself capped by `index.max_result_window`, default 10,000 |
| Cumulative rejection counters | A low count on one node and a far higher one on another, at identical load | Cumulative since the node process started, and never reset in-process. A recently restarted node reads low because it restarted, not because it is healthy — but it resumes counting, so **a non-zero value after a bounce is real.** Compare against each node's uptime |

## Fields whose rendering is not what you would test for

| Field | Renders as | Not |
|---|---|---|
| `is_write_index`, unset | A literal `-` | `false`, and not an empty column. The `-` means the alias metadata carries no `is_write_index` value at all; `true` and `false` render literally when set. A check that greps for `false` never fires on an unset row. The healthy state for a detector's alias is exactly one index reading `true` |
| `cluster_manager` in `_cat/nodes` | `*` for elected, `-` for non-elected, `x` when no cluster manager is known | `true` / `false` |
| `node.role` in `_cat/nodes` | Compacted, sorted role abbreviations such as `di`, `dimr`; `-` for coordinating-only | Spelled-out words — those are in the separate `node.roles` column, also in the default output |
| `severity` on an error alert | `""` | A severity level. Error alerts carry none |
| `enabled_time` on a disabled detector | An explicit JSON `null` | `0`, and not omitted. It is not preserved across a disable, and it dates the last enable. **Nothing in the detector document dates a run** — `last_update_time` is the last config write |
| `plugin_features.tooltip_support` | The JSON **string** `"false"` | A boolean |
| Any integer cluster setting echoed from a settings read or write | A quoted string, e.g. `"2048"` | An integer. Every cluster-settings value is string-typed on read |
| `ElasticsearchClusterConfig.InstanceCount` in a domain config response | The legacy `es` API's key — you called `DescribeElasticsearchDomainConfig` | `ClusterConfig`, which is what the current `opensearch` API returns. There is no `OpenSearchClusterConfig` |
| A rule set across the detector API's own examples | Several different key shapes | One shape. The documented response shape is `inputs[0].detector_input.rules[]`; `custom_rules[]` and `pre_packaged_rules[]` appear under both `detector_input` and `input` across the request and response examples. Sum them all — counting one undercounts |
| A saved-query index in a listing from a non-dotted wildcard | Absent | Deleted. It carries `index.hidden: true`, and hidden indexes are excluded from wildcard expansion — but a dot-prefixed pattern such as `.opensearch-sap-*-detectors-queries*` does match it, so use one of those rather than reaching for `expand_wildcards` |

## Results that are healthy and uninformative

| Read | Why it misleads |
|---|---|
| `GET _cluster/health` | Green survives a detector outage entirely. It stays green through a multi-week detection failure, and green with every data node saturated. **Cluster health is not a detector-health signal.** `"number_of_pending_tasks": 1` also appears inside a cluster reporting green |
| Domain-average CPU | Averaging across data nodes and dedicated cluster manager nodes dilutes a hotspot into a flat line, so a saturated subset reads as fine |
| The `search` thread pool | `0 queue / 0 active` says nothing about the **write** pool, which can carry rejections in the hundreds of thousands at the same moment. Checking only the search pool — the instinct when the symptom is latency — returns a clean bill of health |
| `heap.percent` | Allocate-and-discard cost does not show as retention, so a multi-GB churn leaves heap swinging in its normal band. An operator watching memory sees nothing |
| `ram.percent` near 100% | Page cache. Expected. Not a finding |
| Load-balancer 5XX metrics | Client-side timeouts on requests the cluster never completes are not attributed to the balancer, so real customer-visible 504s can leave the counter at zero. Only request count moves |
| `GET /_plugins/_security_analytics/mappings?index_name=...` returning a full alias set | Meaningful **only while the fault is live.** Read after a reset has been repaired, it returns a healthy set and proves nothing |
| `is_write_index: true` on a repaired alias | A detector binds its field aliases when it is saved, and nothing re-binds them on a schedule. Correcting the alias does not re-bind it. Re-save the detector so it re-reads the field aliases from the current write index; until then the affected rules are skipped with no error |
| An hourly metric period | Sub-minute events average away. Period selection decides whether the fault is visible at all |
| An automated similar-ticket search returning no matches | Returns zero even where a prior record of the same error class exists. Treat it as unknown, not as absence |

## Errors that look like a cause and are not

| Signal | Why |
|---|---|
| `AlertingException[[<nodeId>][<address>:9300][cluster:admin/.../doclevel/fanout]]` as the **start** of `error_message` | Not the whole message. `error_message` is built by walking the entire cause chain and joining it with `; nested: `, so the outermost frame is a transport wrapper and the component that failed is named further along. Read to the end of the string |
| `SearchPhaseExecutionException: all shards failed` | The message string carries no reason; **the response body does.** Read `error.root_cause[]` and `error.failed_shards[].reason` |
| `finding_ids: []` on an error alert | Not "nothing matched" — "the scan did not complete, so nothing could match." **No findings and no error alert** means genuinely nothing matched; **no findings with a NoOp error alert** means you were not looking |
| One shard on the source index | True, worth fixing, and a plausible wrong answer — percolation cost can breach the clause limit independently of shard count. Discriminator: stop the other detectors, create one duplicate detector, and see whether it fails on its first execution. If it does, shard count is not the cause |
| A rule listed under `custom_rules` | A rule that fails to be **indexed** into the saved-query index is absent at percolation while the detector still lists it. Per-rule bulk failures are logged and discarded, the indexing step is skipped silently where the mapping update is not acknowledged, and the detector document is written from the submitted request regardless of what landed. A rule that fails to *compile* is a different thing — it is rejected at rule creation with HTTP 400 |
| High wildcard density in a rule audit | Useful for tuning, misleading as a diagnosis. It coexists with unrelated first-order causes |
| Busy-looking hot threads | A high reported percentage on `MergeRateLimiter.maybePause`, `RateLimiter.pause` or `MergePolicy$MergeSpecification.await` is throttled waiting, not CPU |
| `_nodes/hot_threads` finding nothing | The defaults hide it: the header carries `ignoreIdleThreads=true` and `type` defaults to `cpu`. Re-run with `type=block` or `type=wait`, and `ignore_idle_threads=false`, before concluding anything |

## What no documented read gives you

So nobody builds a check that cannot work. These describe the documented surface; where one matters, confirm against the current API reference before concluding it is unavailable.

- **A per-detector clause-count metric.** The limit applies per query rather than being a consumable cluster resource. A breach shows in the error string.
- **A detector's saved run context.** "I checked and it looked fine" is not available for that component; a controlled reproduction substitutes.
- **Per-node JVM object counts, allocation rate, and thread dumps.** Not reachable for an OpenSearch Service domain. Garbage-collection counts and times *are* published as metrics — only allocation rate is not.
