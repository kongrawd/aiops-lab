# Routing fixtures

The cases this skill was checked against. A wayfinder runs no commands of its own, so the usual evidence — an observed output matching a documented one — is unavailable to it. These fixtures are the substitute: a signal, the cluster state that produces it, and the component a correct route reaches.

**Status.** Not yet run against a cluster. Running them is what raises this skill to `verification: replicated`.

**Expected results name components, never documents.** A fixture that expected a particular runbook would rot the first time one was renamed — which is the failure this skill is built to avoid.

## How to use these

1. **Against a cluster.** Produce the state, give the signal to an operator or agent holding only this skill, and record which component they reach. The only form that establishes the skill works.
2. **Against the documents.** Walk steps 1–3 on paper for each row. Catches ordering faults and renderings that make a check unable to fire.
3. **As a regression check.** Re-run form 2 when a component is added or a confirming read changes.

Record expected against actual. **A row that reaches a plausible but different component is a failure, not a near miss.**

## Positive cases — one correct component

| # | Signal as stated | State | Expected component |
|---|---|---|---|
| P1 | "No alerts at all since we saved the detector." | The detector has no resolvable composite monitor | Composite monitor |
| P2 | "Stopping it says the underlying workflow wasn't found." | Same as P1 | Composite monitor |
| P3 | "Error alerts every minute since the upgrade." | The detector's query index does not exist; the error names it as not found | Query index |
| P4 | "It's never produced a finding." | The configured data source name does not match any existing alias or index | Data source |
| P5 | "NoOp triggers, and the document count in the error keeps climbing." | Clause-limit breach; `Failed to run percolate search ... for <N> document(s)` with `<N>` climbing across alerts | Rule set — clause limit |
| P6 | "Findings stopped. No error anywhere." | Alias with three members, every row showing `-` | Data source — no write index |
| P7 | "We disabled it yesterday and findings are still arriving." | Detector `enabled: false`, composite monitor `enabled: true` | Composite monitor |
| P8 | "Field mappings vanish every Monday." | Alias members declaring one field with different types; the detector reads mappings only from the write index | Field mappings |
| P9 | "Some rules fire, others never have." | A rule set larger than what one evaluation retrieves on this build | Rule set |
| P10 | "Nothing has ever alerted, and the detector looks fine." | The detector carries no alert condition | Detector record |
| P11 | "CPU pinned since we enabled the detectors." | Search queue rejections, detectors producing findings normally | Rule set — cost |

## Ordering cases — where the first-looking read is wrong

| # | Signal | State | Expected | Why |
|---|---|---|---|---|
| O1 | "Error alerts, and the source index only has one shard." | Clause limit breached; source index single-shard | **Rule set** | One shard limits parallelism and is not necessarily the cause. A fresh duplicate detector is not a sound test — a per document monitor skips pre-existing documents on its first run |
| O2 | "Intermittent transport errors, no cause in the message." | An alert whose `error_message` opens with a transport wrapper | **Whatever the chain names** | `error_message` carries the full nested cause chain. Read to the end of the string before treating it as cause-free |
| O3 | "No findings." | Error alerts present **and** the alias has no write index | **Rule set or query index first** — whatever the error names | A rejected evaluation never reaches the data source, so the alias state is not what is stopping findings |
| O4 | "No findings." | No resolvable composite monitor **and** a rule set past the clause limit | **Both**, composite monitor first | Independent faults; repairing one leaves the other. Order the read-only procedure first |
| O5 | "Detectors broke after the upgrade." | Query indexes absent after a prior change to the domain | **Query index**, reached by asking what changed | An access log is the wrong instrument — transport-layer writes do not appear in it, so an empty log is not evidence |
| O6 | "No findings." | A link identifier that lives only in a plugin-owned index, and a caller without access to it | **No component, that read recorded as not evaluated** | The state is real and invisible to this caller. Recording it as "did not fire" is the dangerous failure — it reads as health |

## Negative cases — the correct answer is no component

A route produced for any of these is a false positive, and several components lead to procedures that change state.

| # | Signal | State | Expected |
|---|---|---|---|
| N1 | "No findings from this detector." | Single-member alias showing `-`, no error alerts, 4 rules, composite monitor intact, fields mapped consistently | **No component.** Every read ran, none fired. A single member is the implicit write index, so the data-source read correctly does not fire |
| N2 | "No findings." | Data source returns `404` | **Data source, no procedure.** Repointing or deleting the detector needs its own approval |
| N3 | "The overview page is broken for all our detectors." | One detector with triggers and zero rules | **Detector record, no procedure.** Deletion is the recorded resolution and needs separate approval |
| N4 | "Searches are slow." | No detector implicated; cluster-wide latency | **No component.** Out of scope — search the collection for `opensearch-` instead |
| N5 | "No findings, and the rules should have matched." | Enabled, no errors, 6 rules, composite monitor intact, alias correct, rules reference fields the indexes do not carry | **Field mappings, no repair procedure.** The four-call recipe establishes the fault; the repair is not documented |
| N6 | "It got disabled and nobody did it." | Composite monitor `last_update_time` later than any change on record | **Composite monitor, writer unidentified.** Repairing the state leaves the writer in place to undo it |
| N7 | "We want to alert on detectors that stop matching." | Any | **Nothing off the shelf.** `starting-points.md` says so plainly rather than inventing a check |

## Durability cases — the point of the architecture

These verify that the skill survives changes to the collection. Run them whenever the collection changes.

| # | Change | Expected |
|---|---|---|
| D1 | A runbook is renamed | Step 4's search finds it under the new name. **No edit to this skill is required** |
| D2 | A new runbook covering an existing component lands | The search finds it. No edit required |
| D3 | A runbook's `risk_level` changes from `medium` to `high` | The new value is reported, because step 4 reads it from the file rather than from a table here |
| D4 | A runbook is deleted | The search returns nothing for that component; the component, its read and its output are still reported as a complete result |
| D5 | The collection is not installed at all | Steps 1–3 still work. The component and confirming read are delivered; step 4 reports that no collection is present |
| D6 | A new component is discovered | This is the **only** case needing an edit: add the component, the behaviour, and the confirming read. Never a document name |

**D5 is the load-bearing one.** If the skill cannot produce a useful result with zero runbooks installed, it has drifted back into being a catalogue.

## Rendering traps

Checks that silently match nothing when written against what a value *means* rather than what it *renders as*. They fail invisibly: the reads run, nothing fires, and the result reads as health. The full set is in `blind-checks.md`; these are the ones a fixture run should exercise directly.

| # | Trap | Correct handling |
|---|---|---|
| R1 | An unset `is_write_index` prints as a literal `-` | Test for `-`. A check for `false` or for an empty field can never fire |
| R2 | Rules appear under several different key shapes | Sum every shape, or the rule count undercounts and the page-bound check does not fire |
| R3 | The detector API returns none of `monitor_id`, `workflow_ids`, `rule_topic_index` | Read the config document. Reading the API and finding them absent flags the whole fleet |
| R4 | A protected index returns success with nothing in it | Read `docs.count` first |
| R5 | The alerts array is page-bounded beside a true `total_alerts` | Read `total_alerts`. Otherwise a failing detector routes as silent |
| R6 | `GET _cluster/settings` hides defaults | `include_defaults=true` is mandatory, or `max_clause_count` appears unset |
| R7 | Supplying `detectorType` scopes the error-alert query to one log type | Omit it to cover every visible detector in one search |

## Adding a fixture

When a route is wrong in practice, add the row before changing the skill — the row is what keeps it fixed.

Give the signal in the words it was actually reported in, not a tidied version. Wording is the part of this skill least backed by evidence, for the reason recorded in `evidence.md`.

For a new component, add at least one positive case, one ordering case against whichever existing read it could race, and one negative case where it must not fire.
