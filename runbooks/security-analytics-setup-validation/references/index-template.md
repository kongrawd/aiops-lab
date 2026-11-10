# Layer 1 — Index template and shards

The template decides what every future index carries. Everything above this layer inherits from it, which is why it is checked first.

## The recommended state

**One index template per log source**, matching the concrete indices, carrying:

| Element | Recommended | Why it is this way |
|---|---|---|
| `index_patterns` | Two entries: `${INDEX_PREFIX}-*` **and** `${ALIAS}*` | The second is not decoration. On every detector update the plugin looks up the template matching the **alias name**; this pattern makes it find this template, so it does not append a pattern or create a second template |
| `composed_of` | Retains `${COMPONENT}` | The plugin rewrites this template only when its own component is missing from `composed_of` — and when it does, it copies the existing settings. Keeping the entry is what makes the settings below survive a detector save |
| `priority` | Whatever makes this the only template matching the member indices | Two matching templates resolve by highest `priority` and are **not** merged. Equal priorities do not resolve deterministically |
| `settings` | `index.plugins.index_state_management.rollover_alias` set to `${ALIAS}` | Tells ISM which alias to roll over for indices created from this template. Layer 2's rollover action cannot work without it |
| `settings` | Shard and replica counts, refresh interval | See *Shards*, below |
| `mappings.properties` | Every base field the rules read, with its type | A field the index does not carry cannot be matched, and the plugin skips such rules without reporting an error |
| **no `aliases` block** | Absent deliberately | **Rollover fails if a matching template defines the same index-alias.** The bootstrap request adds the alias and rollover moves it — see layer 2 |

**Base field types are pinned, not left dynamic.** The reference declares each field explicitly — `keyword` where rules match exact terms, `text` with a `.keyword` subfield where both forms are needed, `long` for numeric fields, `date` for timestamps. The reason is in layer 5: term-level rule matching does not hit an analysed `text` field, so a field that arrives as `text` by dynamic mapping produces no findings and no error.

## The reads

Resolve `${TEMPLATE}` **from a member index name, never from the alias name.** The two can differ, and when they do, the template you would have assessed is not the one creating the indices.

```
GET /_cat/aliases/${ALIAS}?v&h=alias,index,is_write_index
GET /_index_template
```

From the first, take a member index name. In the second, find the entry whose `index_patterns` match that name — that is `${TEMPLATE}`.

Then read the resolved template and the shard layout:

```
GET /_index_template/${TEMPLATE}
GET /_cat/shards/${INDEX_PREFIX}-*?v&h=index,shard,prirep,node
GET /_cat/nodes?v&h=name,node.role
```

> ⚠️ **Confirm exactly one template matches.** Where two match the member indices, the higher `priority` wins outright and the templates are not merged — so the lower-priority one's mappings are absent from every index, whatever it says. Where two match at **equal** priority, which wins is not determined by name or age. Record that as a finding rather than assessing one of them.

## Shards

Two separate questions, often conflated. **Both are answered by comparing values you read on this cluster** — no threshold is carried in this document, because a figure written here would decay and would not be yours.

**Enough primaries to spread the work.** A per-document monitor fans out one assignment per shard, so an index with a single primary means one node performs that detector's percolation however large the cluster is.

Read both numbers:

```
GET /_cat/nodes?v&h=name,node.role,heap.max
GET /_cat/indices/${INDEX_PREFIX}-*?v&h=index,pri,rep,docs.count,store.size
```

**Expect:** from the first, one row per node — count the rows whose `node.role` carries the data role, and note `heap.max`. From the second, the `pri` column per index.

**Compare them.** A primary count materially below the data-node count means detectors reading this source get no parallelism from it. Where your platform publishes a ratio it flags at, that published guidance is the authority for the figure — read it there rather than from this document.

**Not so many that the cluster carries the cost.** Derive shards per node rather than estimating it:

```
GET /_cat/shards?h=node,index,prirep
```

Group the rows by `node` and count them. Compare that count, and the `heap.max` you read above, against your platform's published shard guidance. **Both the count and the guidance are values you obtain** — this layer tells you which two numbers to put beside each other, not what they should be.

> ⚠️ **That shard listing includes the plugin's own hidden indexes**, which on a detector cluster can be numerous. So a high per-node count is not by itself a statement about *your* indexing — split the listing by index name before drawing a conclusion about the source you are reviewing.

**Shard count is fixed at index creation.** Changing it in the template takes effect at the next rollover, not on the existing indices — so a correction here is not observable immediately, which is a common reason an operator concludes the change did not work.

## What non-conformance costs

| Finding | Cost |
|---|---|
| `rollover_alias` absent | Layer 2's rollover action cannot identify the alias to roll. The reference diff for an existing template is **this setting and nothing else** — it is the single most common gap |
| A base field the rules read is absent, or typed only by dynamic mapping | Those rules are skipped silently from the next rollover onward. Route to layer 5, then `security-analytics-detector-mapping-audit` |
| An `aliases` block defining `${ALIAS}` | Rollover fails. Route to layer 2 |
| `${ALIAS}*` removed from `index_patterns` | A later detector save may create a second template or append a pattern elsewhere, and the settings above can stop surviving saves |
| Two templates match at equal priority | Which one composes is not determinable. Operator decision: set an explicit priority. Reach out if the setup cannot be changed |
| Single primary on a multi-node cluster | No percolation parallelism for that detector. Route to `security-analytics-rule-cost-audit` |

## Sources

- **Published documentation** for template composition and precedence: `composed_of` entries are applied in listed order with later entries winning, the template's own `template` block is applied last and wins over all of them, and two matching index templates resolve by highest `priority` without being merged, an absent priority counting as `0`.
- **Published managed-service documentation** for the field-alias rationale and for the statement that Security Analytics skips rules whose fields are missing from the index **without reporting an error**.
- **Published documentation** for ISM's `rollover_alias` index setting and its role in the rollover action.
- **A reference setup produced during a support engagement** for the concrete template shape: the two `index_patterns` entries and why the alias-name pattern is retained; the statement that the plugin rewrites the template only when its component is missing from `composed_of` and copies existing settings when it does; the absence of an `aliases` block and the statement that **rollover fails if a matching template defines the same index-alias**; and the observation that adding `rollover_alias` was the only change needed against an otherwise-correct template. **Engagement-sourced, not vendor-documented.**
- **Code analysis of the published alerting plugin** for the fan-out claim — `distributeShards` assigns one unit of work per shard over a shuffled node list, which is why primaries bound a detector's parallelism. This is the reason the primary-count comparison is in this layer; **the figure to compare against is not, and is read from your platform's published guidance.**

**Not established.** Whether the equal-priority tie is stable across calls — the engine compares with a strict greater-than over a map whose iteration order is not sorted, so it need not be. This layer reports the tie rather than predicting the winner.
