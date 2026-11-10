# Layer 4 — Detector configuration

The layers below decide what the detector *can* see. This layer decides what it actually evaluates.

## The recommended state

| Element | Recommended | Why |
|---|---|---|
| Detectors per alias | **Exactly one** | Detectors sharing an alias share one field-mapping copy, keyed by alias rather than by detector. A save from either replaces it for both, so which aliases survive depends on save order rather than on configuration |
| Data source | `${ALIAS}`, an alias — not a wildcard index pattern | A pattern changes which index the detector reads mappings from, and indices created after the pattern was resolved are not added automatically. Wildcard patterns are documented as not recommended for a detector data source |
| Log types per detector | One | All data sources on one detector must hold logs of the same log type; the documentation recommends separate detectors per log type |
| Triggers | At least one | A detector with no alert condition generates no findings, by documented design. It is not broken; it is incomplete |
| Schedule | As long as the detection objective tolerates | Cost scales with executions per hour. A per-document monitor evaluates documents added since its last run, so a longer interval trades execution count against per-execution size, and adds detection latency |
| Rule count | Below the evaluation page bound on builds that have one | See *What differs by version* in `SKILL.md` |

## The reads

```
POST /_plugins/_security_analytics/detectors/_search
{"query":{"match_all":{}},"size":100}
```

**Expect:** one hit per detector. Read `_source.enabled`, `triggers`, `schedule`, the rule arrays, and `inputs[0].detector_input.indices[0]`.

> **Use `_search`, not the single-detector route.** The single-detector read returns a hand-built subset and omits the identifiers that link a detector to its monitor and query index. A detector whose composite monitor or saved-query index is gone looks healthy through that route.

**Then group every hit by `inputs[0].detector_input.indices[0]`.** Any data source appearing against more than one detector is this layer's most consequential finding, and it costs no extra call.

Resolve what the data source actually is:

```
GET /_resolve/index/${ALIAS}
```

**Expect:** the name under `aliases`. Under `indices` it is a concrete index; if it matches neither and the configured value contains a wildcard, it is a pattern.

**Counting rules.** Sum all of these shapes — the API documentation uses different ones across its own examples, so counting one undercounts:

- `inputs[0].detector_input.rules[]` — the documented response shape
- `inputs[0].detector_input.custom_rules[]` and `...pre_packaged_rules[]`
- `inputs[0].input.custom_rules[]` and `inputs[0].input.pre_packaged_rules[]`

## What non-conformance costs

| Finding | Cost | Route |
|---|---|---|
| Two or more detectors on `${ALIAS}` | Shared field-mapping state; last save wins and the other detector stops matching on the aliases it dropped, with no error | Layer 6 for the supported way to run two detectors |
| Data source is a wildcard pattern | Indices created later are not picked up, and mappings are read from the newest index by creation date rather than the alias write index | Operator change; the reference setup is alias-based throughout |
| Data source resolves to nothing | Every scheduled run fails outright | `security-analytics-wayfinder`, then repoint or delete — an operator decision |
| No triggers | No findings at all, by design | Operator decision: complete or remove the detector |
| Triggers but no rules | The plugin creates no monitor or workflow, and the detector produces nothing. This state has been observed to break the shared overview page for every detector, not only the one at fault | `security-analytics-detector-health-audit` records it; deletion needs its own approval |
| Rule count above the page bound on an affected build | Rules beyond the bound are dropped with no error, warning or log entry | `security-analytics-rule-coverage-audit` |
| Mixed log types on one detector | Field mappings are resolved against the wrong vocabulary for some sources | Split per log type |

## On the detector save after a cutover

The two reference revisions disagree, and both positions are defensible:

- **Alternative A — update each detector once after the cutover.** The save makes the detector re-read field aliases from the current write index, so rules on fields that were previously missing begin producing findings. Do it outside the rollover window.
- **Alternative B — no detector update as part of the change.** If a detector is updated for another reason, do it outside the window from rollover to the retention transition.

**What decides it is layer 3.** Where every rule field alias is declared in your own index template, the save is optional — the aliases are already on every new index regardless of what the plugin's component template holds. Where they are not, the save is what binds them, and skipping it leaves rules skipped until the next save. So this is a consequence of layer 3's verdict rather than an independent choice.

**Either way, a save rewrites the plugin's component template** from the aliases present on one index at that moment, and may leave it shorter or empty. That is harmless against a conformant layer 3 and damaging against a non-conformant one.

## Sources

- **Published documentation** for: an alias data source needing a write index and documents being ingested through the alias; wildcard patterns not being recommended as a detector data source, and a pattern being resolved at creation so later indices are not added; a detector needing at least one alert condition before it generates findings; all data sources on one detector holding one log type, with separate detectors recommended per log type; and the detector search route and its response fields.
- **Published documentation** for the rule-array shapes, which differ across the API documentation's own examples.
- **Tested by the engineering team on a 3.7 line** for the shared-alias behaviour: two detectors on one alias share a single field-mapping copy and a save from either replaces it for both, and the reproduction confirmed the other detector then misses events with no error.
- **A reference setup produced during a support engagement** for both cutover alternatives, including the statement in the later revision that no detector update is part of the change, and the statement in the earlier one that the save makes previously-skipped rules start producing findings. **The two revisions disagree and this file presents both.**
- **Code analysis of the published alerting plugin** for the per-document evaluation model behind the schedule trade-off: a monitor evaluates documents added since its previous run, and an execution that does not complete carries them into the next one.

**Not established.** Whether the page bound on rules retrieved per evaluation applies to a given build. No version number settles it reliably because backports land unevenly; `security-analytics-rule-coverage-audit` carries the behavioural check.
