# Layer 6 — Test and production separation

Running a second detector over the same log source is the most common reason a conformant setup becomes non-conformant. This layer is the supported way to do it.

## The recommended state

**One alias per detector.** Both aliases point at the same indices. A second detector gets its *own* alias rather than sharing the production one.

Three rules make it safe, and all three are required:

| Rule | Why |
|---|---|
| **The test alias needs its own write index** | A detector tracks its read position on *its own* alias's write index. Where the test alias has no write index, the test detector checks no data in new indices. This was tested: it found nothing in a new index until the write index was set |
| **Use the same field mappings in both detectors** | Where both detectors use one rule field name, it must point at the same log field in both. See the hazard below |
| **Never point two detectors at one alias** | This is the condition that caused field-mapping loss before |

Set the test alias's write index the same way the production alias's is set — each new index becomes the write index of both aliases.

## Why this is safe

- **No data duplication.** An alias is another name for the same indices. The write-index designation only matters to something writing *through* that alias; the shipper keeps writing as it does. Each record is stored once and both aliases see it.
- **No cross-channel alerting.** A detector notifies only the channels on its own triggers. A test detector whose trigger carries a test channel, or no channel, sends nothing to production.
- **Each alias gets its own field-mapping copy**, so the two detectors no longer overwrite each other's, and both copies apply to every new index.

## The one hazard that survives this setup

**Mapping the same rule field name to different log fields in the two detectors.** The last save replaces the other — on the current index and on every new index — and the other detector then **stops matching without any error**. This was reproduced: the production detector missed events after the test detector changed the mapping.

So rule 2 is not hygiene. It is the condition that makes two aliases safe, and it is checkable: compare the `path` behind each shared rule field name in both detectors' mappings.

## Alias naming

**A secondary alias name should not begin with the production alias name.** The reason is in layer 3's mechanism: on builds where a field-mappings save resolves the index template by matching the *alias name*, a save on the longer name can match a pattern left behind by the shorter one and attach its component template to the wrong index template.

**What this costs depends on the topology**, which is why layer 3 treats a prefix pair as an observation rather than a fault. Where both aliases draw their indices from the same template, a misattached component still lands somewhere that applies to those indices. Where they draw from different templates, the second alias's new indices receive none of its field aliases.

**A prefix pair has been observed running without detection loss**, in the same-template topology. That is not evidence the mechanism is absent — it means the misattachment was harmless there, and becomes harmful the moment either alias is repointed at indices of its own. Prefer a name that does not collide; where one already exists, record it and check layer 3 per index.

## Alternatives to a second detector

Two options avoid a second alias entirely:

- **Select rules per trigger.** Keep one detector. Put production rules on a trigger carrying the production channel, and test rules on a second trigger with no channel. Test alerts still appear on the alerts page. Each trigger was tested and fired only for its own rules. This needs no extra alias, no write-index change, and carries no field-mapping risk.
- **Separate by severity.** Give test rules one severity and filter triggers on it. **Check first whether any production or prebuilt rule already carries that severity** — if so, it stops alerting the moment the filter is applied.

**A trigger filtered on rule tags drops rules that carry no tags.** Where a rule set includes untagged rules, a tag-based trigger filter silently excludes them. Whether tagging is a supported mechanism for separating test from production rules was raised during the engagement and never answered.

## Monitors over the same indices

**No concern.** Monitors only read; they do not change field mappings, and creating monitors on the same alias was confirmed not to alter index mappings.

One caveat: a monitor searching on *detector* field names rather than the original log field names depends on the detector field mappings, so rule 2 above keeps it working.

## What non-conformance costs

| Finding | Cost | Route |
|---|---|---|
| Two detectors on one alias | Shared field-mapping copy; last save wins and the other goes silent | Give the second its own alias, per this layer |
| Test alias with no write index | The test detector checks no data in new indices — it reports healthy and evaluates nothing | `security-analytics-alias-write-index-recovery` |
| One rule field mapped to different log fields across detectors | The detector that did not save last stops matching on that field, with no error | Align the mappings; `security-analytics-field-alias-template-recovery` for the alias state |
| Secondary alias name prefixed by the production alias name | Topology-dependent; see above | Layer 3 per index |
| Trigger filtered on tags over a rule set containing untagged rules | Those rules stop alerting | Operator decision |

## Sources

- **Tested by the engineering team on a 3.7 line**, and reported as such, for: the three rules; that a test alias without a write index causes the test detector to check no data in new indices; that each alias gets its own mapping template so two detectors no longer overwrite each other; that each trigger fires only for its own rules; that monitors on the same alias do not change index mappings; and the reproduction of the same-rule-field-different-log-field hazard, where the production detector missed events after the test detector changed the mapping.
- **An explicit correction in the engagement record.** Earlier guidance said *not* to make the test alias a write index. It was withdrawn in writing — "that was incorrect" — after testing showed the opposite. **Anyone working from the earlier advice has a test detector that evaluates nothing**, which is why this file states the correction rather than only the conclusion.
- **Code analysis of the published Security Analytics plugin** for the alias-naming constraint: on the released lines a field-mappings save resolves the index template by the input name before the backing index, and appends the alias pattern to whichever template matched. The upstream change that removes this names the shared-prefix case in its own description.
- **Engagement observation** for the prefix pair seen running without loss, and for the untagged-rule consequence of a tag-filtered trigger.

**Not established.** Whether rule tagging is a supported way to keep test rules from triggering production alerts. The question was asked during the engagement and no answer was recorded, so this file describes the mechanical consequence for untagged rules and makes no claim about support.
