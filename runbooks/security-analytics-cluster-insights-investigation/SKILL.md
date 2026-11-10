---
name: security-analytics-cluster-insights-investigation
description: >-
  Establish who owns the shards behind a Cluster Insights recommendation on a managed OpenSearch
  domain before anyone acts on it. On a domain running Security Analytics the plugin owns index
  families the operator never created, all of them hidden, so a shard count can be composed mostly
  of indexes nobody intended to keep while the recommendation points at the operator's own data -
  and re-sharding or scaling in response need not move the number. Use this when a shard-count or
  sharding recommendation is open on a domain running detectors, or when a recommendation has
  already been applied and the condition did not change. Read-only throughout - it classifies the
  shard population and reports both figures, and changes nothing.
metadata:
  agent_types:
    - INCIDENT_TRIAGE
    - PREVENTION
  skill_type: USER
  service: Amazon OpenSearch Service Cluster Insights / Security Analytics
  applies_to_versions: Insights are surfaced through the console and the Insights APIs for Elasticsearch 6.8 and later and all OpenSearch lines. Two other surfaces have higher floors, and regional availability is a stated limit; see "When to use this".
  risk_level: low
  read_only: true
  automatable: yes
  verification: documented
  last_verified: 2026-11-10
  last_edited: 2026-11-10
  references:
    - title: Unified operational monitoring with Cluster Insights
      url: https://docs.aws.amazon.com/opensearch-service/latest/developerguide/cluster-insights.html
      kind: product-doc
    - title: Insights Catalog — the insights and the figures each is generated against
      url: https://docs.aws.amazon.com/opensearch-service/latest/developerguide/insights-catalog.html
      kind: product-doc
    - title: ListInsights API
      url: https://docs.aws.amazon.com/opensearch-service/latest/APIReference/API_ListInsights.html
      kind: api-reference
    - title: Insight — the object's fields and their valid values
      url: https://docs.aws.amazon.com/opensearch-service/latest/APIReference/API_Insight.html
      kind: api-reference
    - title: DescribeInsightDetails API
      url: https://docs.aws.amazon.com/opensearch-service/latest/APIReference/API_DescribeInsightDetails.html
      kind: api-reference
    - title: CAT indices API — the expand_wildcards parameter
      url: https://docs.opensearch.org/latest/api-reference/cat/cat-indices/
      kind: api-reference
    - title: Index name syntax — how expand_wildcards resolves hidden indexes
      url: https://docs.opensearch.org/latest/api-reference/index-name-syntax/
      kind: api-reference
---

# Runbook — Who owns the shards behind this recommendation?

## Before you use this

- **Confirm the behaviour against current vendor documentation, or with your support channel, before you execute this.** Product behaviour moves with versions, and this runbook states the version range it was written for in `metadata.applies_to_versions`. Where this document and the documentation disagree, the documentation is current and this is not.
- **Exercise the procedure in a non-production environment first.** Every step here is a read, so the risk is low — but run it once where you know the index set, so you recognise a normal result.
- **You remain responsible for the outcome in your environment.** This document is offered as-is, with no warranty and no liability for any effect of its use.
- **Send corrections back.** A step that failed, output that differed, a version where the behaviour changed, or a clearer command are worth more than a clean document that is quietly wrong. Open an issue or a pull request on the repository this runbook was shared from.

## When to use this

**Trigger.** Either of:

- A **shard-count or sharding recommendation** is open on a domain running Security Analytics detectors.
- A recommendation was **applied and the condition did not change** — shards reduced, an index re-sharded, the domain scaled, and the insight still open.

**Applies when all of these hold.**

| Condition | How to check | Expected |
|---|---|---|
| Detectors exist on the domain | `POST /_plugins/_security_analytics/detectors/_search` with `{"query":{"match_all":{}},"size":100}` | At least one hit |
| An insight is open | Step 1 | At least one entry in `Insights[]` |
| You can list indexes including hidden ones | Step 2 | Rows whose index names begin with a dot |

**What differs by surface and region.** The Insights APIs and the console view are available for Elasticsearch 6.8 and later and all OpenSearch lines. The Dashboards overview requires OpenSearch 2.17 or later, and the Query View tab 2.19 or later with fine-grained access control and a role mapping for query insights. **The catalog also states that the feature is supported in a listed set of regions and that the list is expanding** — so read the catalog for your region rather than assuming availability.

**An absent insight is not an absent condition.** Where the feature has not rolled out to your region, the condition can be at full strength with nothing generated. This runbook attributes an insight that exists; it is not a substitute for reading the shard population directly, which step 2 does anyway.

**Does not apply when:**

- **No detectors exist on the domain.** There is no plugin-owned population to separate, so act on the recommendation.
- **The recommendation is not about shards.** A disk, write-block, KMS-key, cluster-manager or cluster-status insight names a condition a detector fleet does not create. Act on it directly.
- **The insight is not open.** Step 1 branches on this.

## Values to obtain

| Name | What it is | How to obtain the current value |
|---|---|---|
| `${DOMAIN_NAME}` | The domain the insight is raised against | The console's domain list, or `aws opensearch list-domain-names` |
| `${INSIGHT_ID}` | One open insight. A 36-character hyphenated identifier | Step 1 → `Insights[].InsightId` |

The Insights operations take an entity as a type and a value. The documented types are an account and a domain name; **scope to the domain** unless you are sweeping an account, in which case substitute accordingly.

## Impact

**Mechanism.** An insight states a condition, a figure it was generated against, and a remediation. **It does not say which component produced the condition.** On a domain running Security Analytics that distinction decides whether the remediation helps, because the plugin owns index families the operator never created — a findings and an alerts-history family per log type, a saved-query index per detector source, and its own configuration indexes. All of them are created with `index.hidden`, so they are absent from a default index listing and from the operator's picture of the cluster.

A shard count can therefore be composed mostly of indexes nobody intended to keep, while the remediation attached to it points at the operator's own data. **Applying it changes something real and need not move the number.**

**Size it for this system.** Step 2's single listing gives both populations. Report them as two counts, not as a ratio alone — the absolute figures are what a capacity decision needs.

**Blast radius.** The decision rather than the cluster: whichever change the recommendation would prompt — a re-shard, a scale-out, an instance-type change — each costing a deployment and none addressing a plugin-owned cause.

**Recoverability.** This procedure changes nothing. A misattributed remediation already applied is usually reversible at the cost of another deployment; a re-shard of the operator's own indexes is not reversible in place, because shard count is fixed at index creation and takes effect at the next rollover.

**Severity.** Urgent where a remediation is about to be applied, because the window to attribute correctly closes when the change ships. Routine otherwise.

## Before you start

- [ ] IAM permission for the `ListInsights` and `DescribeInsightDetails` operations on `${DOMAIN_NAME}`. Confirm the action names against the service-authorization reference for OpenSearch Service rather than assuming them
- [ ] Permission to call `GET /_cat/indices` and the Security Analytics detector search route on the domain

> ⚠️ **Both Insights operations are reads that use `POST`.** Each is documented as listing or describing, and neither changes anything. Do not infer from the method that they are writes — and do not infer the converse elsewhere.

## Procedure

### Step 1 — Read the open insight and its recommendation

```
POST /2021-01-01/opensearch/insights
{"Entity":{"Type":"DomainName","Value":"${DOMAIN_NAME}"},"MaxResults":500,"SortOrder":"DESC"}
```

**Expect:** `Insights[]`, each entry carrying `InsightId`, `DisplayName`, `Type`, `Priority`, `Status`, `IsExperimental`, `CreationTime` and `UpdateTime`, plus `NextToken` when more results exist.

The documented values are: `Priority` one of `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`; `Status` one of `ACTIVE`, `RESOLVED`, `DISMISSED`; `Type` either `EVENT` or `RECOMMENDATION`.

> ⚠️ **The insight's family name is in `DisplayName`, not in `Type`.** `Type` distinguishes an event from a recommendation; it does not name which insight fired. Match on `DisplayName`.

**Branch.**

- **`Status` is `RESOLVED` or `DISMISSED`** → the signal is stale. Close it and correct whatever view it was read from. **Stop here.**
- **`NextToken` is returned** → page until it is absent. A full page is not the end of the list.
- **`DisplayName` does not name a shard or sharding condition** → out of scope. Act on the insight directly. **Stop here.**
- **`Status` is `ACTIVE` and `DisplayName` names a shard or sharding condition** → continue.

Then read the detail, and write down the recommendation in one line:

```
POST /2021-01-01/opensearch/insight-details
{"Entity":{"Type":"DomainName","Value":"${DOMAIN_NAME}"},"InsightId":"${INSIGHT_ID}"}
```

**Expect:** `Fields[]`, each entry a `Name`, `Type` and `Value` triple.

The set of `Name` values is not enumerated in the documentation and varies by insight, so **iterate `Fields[]` and read what is there.** Take the recommendation text from it; take the figures from step 2, which measures them.

### Step 2 — Classify the shard population by owner

One listing, with an explicit pattern so that hidden indexes are included:

```
GET /_cat/indices/*?v&h=index,pri,rep,docs.count,store.size&expand_wildcards=all&s=index
```

**Expect:** one row per index, including indexes whose names begin with a dot.

> ⚠️ **If no row's index name begins with a dot, this listing is not showing hidden indexes** and the classification below cannot be made from it. That is the one result to check before reading anything else — a domain running detectors has plugin-owned indexes by construction. Obtain a listing that includes them before continuing.

**Compute each row's shard count as `pri × (1 + rep)`.** Then split the rows into two populations by name:

- **Plugin and system owned** — index names beginning with a dot.
- **Operator owned** — everything else.

Sum the shard counts per population. **Report both sums and the total.**

### Step 3 — Decide which population the recommendation applies to

Compare each population against the figure the catalog states for this insight, and against your platform's published shard guidance. **Read those figures from the catalog and the guidance, not from this document** — they live there and move there.

**Branch.**

- **Only the plugin-owned population crosses the guidance** → the recommendation's target is not the operator's data. Say so explicitly in whatever decision was pending, and go to `security-analytics-history-retention-investigation`, which establishes whether the plugin's own indexes are accumulating past their retention.
- **Only the operator-owned population crosses it** → the recommendation means what it says. Act on it, and record the split so the question is not reopened.
- **Both cross it independently** → both need action. Order the plugin-owned one first; it is usually the larger and it is addressable without touching the operator's data.
- **Neither crosses it** → the insight was generated against a figure you have not reproduced. Record both sums and the figure you compared against, and reach out — the scoping difference is the finding.

## Validation

1. **The listing included hidden indexes.** Confirm at least one row's index name begins with a dot, and state it with your results. Without it the classification is **not evaluated** rather than clean.
   ```
   GET /_cat/indices/*?v&h=index&expand_wildcards=all&s=index
   ```
2. **The two sums and the total are consistent.** The two population sums must add to the total shard count across the listing. A discrepancy means rows were dropped from the classification.
3. **The insight is still open.** Re-read step 1 and confirm `Status` is still `ACTIVE`. An insight that resolved while you were measuring changes what the measurement is for.

**Exit criteria.** You can state: the insight's `DisplayName`, `Priority` and `Status` as returned; the plugin-owned shard sum, the operator-owned shard sum, and the total; and which population the recommendation applies to, or that neither reproduces the figure the insight was generated against.

## If this does not work

**Reach out when:** the Insights APIs return an error while the console shows insights for the domain; or no listing you can obtain includes hidden indexes; or neither population reproduces the figure the insight states.

**Reach out to:** your OpenSearch Service support channel, through the route your organisation uses for a production domain.

**Attach:** the `Insights[]` entry as returned, including `DisplayName`, `Priority`, `Status` and `IsExperimental`; the `Fields[]` from the detail call; the index listing; the two population sums and the total; and the catalog figure you compared against. State that nothing was changed.

## Related runbooks

- `security-analytics-history-retention-investigation` — where the plugin-owned population is the one crossing the guidance, and the question becomes whether those indexes are accumulating past their retention.
- `security-analytics-rule-cost-audit` — where the concern is what the detectors cost to evaluate rather than what they store.
- `security-analytics-setup-validation` — where the operator-owned population is the one at fault and the question becomes whether the log source is built correctly.
- `opensearch-shard-pinning-remediation` — where a recommendation names routing-allocation attributes on an index.

## Verification status

Every step traces to published documentation. The basis and the open questions are in `references/evidence.md` — read it before relying on this runbook for the first time, or before quoting it to anyone.
