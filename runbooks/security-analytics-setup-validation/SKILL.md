---
name: security-analytics-setup-validation
description: >-
  Check an existing Security Analytics log source against the reference setup, layer by layer, from
  the index template and its shards through the alias and ingestion path, the field aliases, the
  detector, and the rules. Answers "is this built the way it needs to be" rather than "is something
  broken" - a conformance review, not an incident procedure. Use it when onboarding a log source,
  when reviewing a setup someone else built, after migrating ingestion to an alias, or before
  relying on a detector for coverage someone has attested to. Each layer states the recommended
  state, the read that shows conformance, what non-conformance costs, and the sibling that repairs
  it. Read-only throughout - it reports conformance and routes, and changes nothing.
metadata:
  agent_types:
    - PREVENTION
    - CHAT
  skill_type: USER
  service: OpenSearch Security Analytics / Alerting
  applies_to_versions: All lines. Where a layer's recommendation depends on plugin behaviour that differs by version, the layer file says so and names the read that settles it.
  risk_level: low
  read_only: true
  automatable: partial
  verification: documented
  last_verified: 2026-11-10
  last_edited: 2026-11-10
  references:
    - title: Setting up ingestion for a Security Analytics detector
      url: https://docs.aws.amazon.com/opensearch-service/latest/developerguide/sa-ingestion-setup.html
      kind: product-doc
    - title: Creating detectors — the alias and write-index requirement
      url: https://docs.opensearch.org/latest/security-analytics/sec-analytics-config/detectors-config/
      kind: product-doc
    - title: Index templates — composed_of merge order, template precedence and priority
      url: https://docs.opensearch.org/latest/im-plugin/index-templates/
      kind: product-doc
    - title: Index State Management — the rollover action and rollover_alias
      url: https://docs.opensearch.org/latest/im-plugin/ism/policies/
      kind: product-doc
    - title: Index alias API — the is_write_index designation
      url: https://docs.opensearch.org/latest/im-plugin/index-alias/
      kind: api-reference
    - title: Security Analytics mappings API
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/mappings-api/
      kind: api-reference
    - title: Security Analytics detector APIs
      url: https://docs.opensearch.org/latest/security-analytics/api-tools/detector-api/
      kind: api-reference
---

# Runbook — Check a log source against the reference Security Analytics setup

## Before you use this

- **Confirm the behaviour against current OpenSearch documentation, or with your support channel, before you execute this.** Plugin behaviour moves with versions, and this runbook states the range it was written for in `metadata.applies_to_versions`. Where this document and the documentation disagree, the documentation is current and this is not.
- **Exercise the procedure in a non-production environment first.** Every step here is a read, so the risk is low — but run it once where you know the setup, so you recognise a conformant result.
- **You remain responsible for the outcome in your environment.** This document is offered as-is, with no warranty and no liability for any effect of its use.
- **Send corrections back.** A step that failed, output that differed, a version where the behaviour changed, or a clearer command are worth more than a clean document that is quietly wrong. Open an issue or a pull request on the repository this runbook was shared from.

## When to use this

**Trigger.** Any of:

- **Onboarding a log source** — before the detector is relied on for anything.
- **Reviewing a setup someone else built**, including one inherited with a cluster.
- **After migrating ingestion** from writing to dated indices to writing through an alias.
- **Before attesting coverage** — before citing a detector as the control for a rule set.

**Applies when all of these hold.**

| Condition | How to check | Expected |
|---|---|---|
| A Security Analytics detector exists for the log source | `POST /_plugins/_security_analytics/detectors/_search` with `{"query":{"match_all":{}},"size":100}` | At least one hit |
| You can read index templates and component templates | `GET /_index_template`, `GET /_component_template` | A template list from each |
| The detector's data source resolves | `GET /_resolve/index/${ALIAS}` | The name appears under `aliases` or `indices` |

**Does not apply when:**

- **Something is already broken and you need it working.** This runbook reports conformance; it does not diagnose or repair. Go to `security-analytics-detector-health-audit` for a fleet sweep of silent failure states, or `security-analytics-wayfinder` to identify a component from a symptom.
- **You are measuring cost rather than correctness.** Use `security-analytics-rule-cost-audit`.
- **The data source is a wildcard index pattern rather than an alias.** The reference setup is built on an alias, so most layers do not apply as written. Layer 4 says what a pattern costs and routes you onward.

## Values to obtain

Declare these once and refer to the names throughout. **Every one of them changes per log source**, and the index names change at every rollover, so read them at execution time rather than carrying them between runs.

| Name | What it is | How to obtain the current value |
|---|---|---|
| `${LOG_SOURCE}` | The log source this setup serves. One log source per detector is the reference shape | Your own naming. It is the organising unit for every layer below |
| `${ALIAS}` | The index alias the detector reads. Conventionally `detector-${LOG_SOURCE}` | `POST /_plugins/_security_analytics/detectors/_search` → `inputs[0].detector_input.indices[0]` |
| `${INDEX_PREFIX}` | The prefix of the concrete indices behind `${ALIAS}` | `GET /_cat/aliases/${ALIAS}?v&h=alias,index,is_write_index` → the common prefix of the `index` column |
| `${INDEX}` | One concrete member index. Changes at every rollover | Same read → the `index` column |
| `${TEMPLATE}` | The index template whose `index_patterns` match `${INDEX_PREFIX}-*`. **Resolve it from a member index name, not from the alias name** — layer 3 explains why | `GET /_index_template` → the entry whose `index_patterns` match `${INDEX}` |
| `${POLICY_ID}` | The ISM policy managing those indices | `GET /_plugins/_ism/explain/${INDEX}` → `policy_id` |
| `${COMPONENT}` | The plugin-managed component template holding the detector's field aliases | Discover rather than assume: `GET /_component_template/*alias-mappings*`, then confirm against `${TEMPLATE}`'s `composed_of` |
| `${DETECTOR_ID}` | The detector reading `${ALIAS}` | The detector search above → `hits.hits[]._id` |

## Impact

**Mechanism.** A Security Analytics setup is a stack, and each layer depends on the one below holding. The index template decides what fields a new index carries and how many shards it has. The alias and its write-index designation decide which index the detector reads and whether records land where it is looking. The field aliases decide whether an evaluated rule can match anything. The detector decides what is evaluated and how often. The rules decide coverage and most of the cost.

**A break in a lower layer presents as an absence in a higher one.** No findings is the symptom of a missing write index, a dropped field alias, an unevaluated rule and a correctly-configured detector over data that genuinely did not match — four different layers, one presentation. That is why this runbook checks bottom-up rather than starting where the symptom appears.

**Non-conformance is usually silent.** A rule whose field is missing from the index is skipped without an error being reported. An alias with no write index still resolves. A detector over an alias it shares with another detector still saves cleanly. So conformance has to be read, and cannot be inferred from the absence of complaints.

**Size it for this system.** Count the log sources in scope and the detectors reading each alias — the second is the number that most often turns out to be wrong:

```
POST /_plugins/_security_analytics/detectors/_search
{"query":{"match_all":{}},"size":100}
```

Group the hits by `inputs[0].detector_input.indices[0]`. One detector per alias is the reference shape; any alias with two or more is a layer 6 finding.

**Blast radius.** Per non-conformant log source, every rule attached to its detector, for as long as the condition has held. Where the finding is in layers 1 to 3, the exposure usually began at a rollover rather than at the configuration change, so the start is not the date someone edited something.

**Recoverability.** The configuration is recoverable at every layer. **The missed detection is not** — findings are not generated retroactively, so a detector that evaluated nothing for a week does not produce that week's findings once the setup is corrected. That asymmetry is the argument for running this before an incident rather than after one.

**Severity.** Urgent before an attestation, and before anyone relies on a newly onboarded source. Routine as a review. It cannot be deferred on the grounds that nothing has alerted — a conformance gap in layers 1 to 3 produces no alert by construction.

## Before you start

- [ ] Permission to call `GET /_index_template`, `GET /_component_template`, `GET /_cat/aliases`, `GET /_cat/indices`, `GET /_cat/shards`, `GET /_plugins/_ism/explain`, `GET /_plugins/_ism/policies`, the Security Analytics detector and mappings **GET** routes, and `GET /_cluster/settings`
- [ ] The log source list you intend to review. This runbook checks one `${LOG_SOURCE}` at a time; run it per source rather than attempting a fleet pass
- [ ] Awareness that a layer can be conformant for the index you read and non-conformant for another member of the same alias. Where a layer says to repeat per member index, repeat it

> ⚠️ **`GET` on the Security Analytics mappings route is the read. `POST` to the same path is a write** — it applies mappings and rewrites `${COMPONENT}`. Use the `GET` form only. The Dashboards *Edit field mappings* page has been reported to issue the `POST` form when it opens; treat opening that page as a change, not an inspection.

> ⚠️ **This runbook reports; it does not repair.** Every layer routes its findings to a sibling or names the operator decision. Resist fixing as you go — a conformance review whose findings were partly repaired mid-pass cannot be reported as a baseline, and several of the repairs need their own approval.

## Procedure

Work the layers in order. Each layer file carries the recommended state, the read, the cost of non-conformance, and where to go. **Record a verdict per layer even where it is conformant** — the value of this runbook is the complete picture, not the first finding.

| # | Layer | Reference file | Checks |
|---|---|---|---|
| 1 | Index template and shards | `references/index-template.md` | One template per log source; base fields declared with their types; shard count against the data-node count; `rollover_alias` set |
| 2 | Alias and ingestion | `references/alias-and-ingestion.md` | One alias per log source; an explicit write index; shippers writing to the alias; rollover by ISM action rather than by first-record-after-midnight |
| 3 | Field aliases | `references/field-aliases.md` | A field alias for every rule field name, declared where it survives a detector save |
| 4 | Detector configuration | `references/detector-config.md` | One detector per alias; the data source is the alias; a trigger exists; schedule and rule count |
| 5 | Rule hygiene | `references/rule-hygiene.md` | Rule fields present in the index; pattern breadth; clause count; near-duplicate detectors |
| 6 | Test and production separation | `references/test-separation.md` | How a second detector is run safely, and the alias naming constraint that makes it safe |
| 7 | Detection health signals | `references/health-signals.md` | Error-state alerts routed somewhere, and what they do not catch |

**Stop early only in one case.** If layer 2 finds the alias has no write index, layers 3 to 5 will report findings that are consequences of it rather than independent faults. Record layer 2, note that the layers above it are unassessed for that reason, and route it first.

## Validation

This runbook produces a judgement about a setup, so validation confirms the judgement is trustworthy rather than that a system recovered.

1. **The template you assessed is the one that creates the indices.** Re-derive `${TEMPLATE}` from a member index name rather than from the alias name, and confirm it matches the entry you used for layers 1 and 3.
   ```
   GET /_index_template
   ```
   **Expect:** exactly one entry whose `index_patterns` match `${INDEX}`. Two matching entries resolve by `priority`, and equal priorities do not resolve deterministically — record that as a layer 1 finding rather than picking one.
2. **The component-template read was not blind.** Confirm at least one alias's component template comes back carrying field-alias entries before treating any other alias's empty one as a finding. Where you cannot produce that positive control, record layers 3 and 5 as **not evaluated** rather than conformant.
3. **Every layer has a verdict.** Conformant, non-conformant with a route, or not evaluated with the reason. A layer silently skipped is the failure mode this validation exists to catch.

**Exit criteria.** You can state, for one `${LOG_SOURCE}`: a verdict for all seven layers; the variables you resolved, including which template creates the indices and which policy manages them; and for each non-conformance, the sibling it routes to or the operator decision it needs. A conformant result is only reportable alongside validation items 1 and 2.

## If this does not work

**Reach out when:** two index templates match the member indices at equal priority, so the composing order is not determinable; or `${TEMPLATE}` cannot be resolved because no template matches the member indices; or a layer's read is refused and the positive control in validation item 2 cannot be produced; or the setup predates the reference shape so completely that the layers do not describe it — in which case the gap is a migration, not a conformance finding.

**Reach out to:** your OpenSearch support channel, through the route your organisation uses for a production cluster.

**Attach:** the resolved variables; the per-layer verdict table; the index template and the component template as read; the alias listing with the `is_write_index` column; the ISM policy id and its explain output; the detector count per alias; and the positive control from validation item 2. State that no write was made and that the mappings `POST` route was not used.

## Related runbooks

Siblings are referred to by name, which resolves to a directory and is greppable.

- `security-analytics-detector-health-audit` — where the question is which detectors across a fleet are in a silent failure state, rather than whether one source is built correctly.
- `security-analytics-alias-write-index-recovery` — where layer 2 finds an alias with no explicit write index.
- `security-analytics-field-alias-template-recovery` — where layer 3 finds field aliases missing from new indices, or a component template on a template that does not create them.
- `security-analytics-rule-coverage-audit` — where layer 4 finds a rule count past the evaluation page bound.
- `security-analytics-rule-cost-audit` — where layer 5 finds pattern breadth or duplication worth measuring.
- `security-analytics-percolate-failure-investigation` — where any layer turns up error-state alerts, which mean evaluation is being rejected rather than merely non-conformant.
- `security-analytics-detector-mapping-audit` — where layer 5 finds rule fields absent from the index and the question becomes whether an evaluated rule could have matched.

`security-analytics-wayfinder` is a skill rather than a runbook. Use it when you have a symptom and need the component; use this when you have a setup and need its conformance.

## Verification status

Every layer traces to published documentation, to code analysis of the published plugin, or to a reference setup produced during a support engagement — and each layer file says which, per claim. A step resting on inference is reason to leave it out rather than to label it. The basis, what the reference documents disagree about, and the open questions are in `references/evidence.md` — read it before relying on this runbook for the first time, or before quoting it to anyone.

**The reference setup exists in two revisions that differ on five points**, including which ISM strategy to use and whether a detector update is part of the cutover. Where they differ, the layer files present both as alternatives with the trade-off each carries, rather than asserting one is correct. That is recorded per layer and summarised in `references/evidence.md`.
