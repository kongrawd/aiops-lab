# Evidence — what this runbook rests on, and what it does not

Every layer traces to published documentation, to code analysis of the published plugin, or to a reference setup produced during a support engagement. A claim resting on inference is reason to leave it out rather than to label it. **Each layer file carries its own `Sources` section**, naming the class per claim; this file records the overall level, what the sources disagree about, and what they could not settle.

`metadata.verification` is `documented`: no layer has been exercised end to end against a live setup.

## The three source classes, and how far to trust each

| Class | Where it appears | How far it goes |
|---|---|---|
| **Published documentation** | Layer 3's core, layer 1's template precedence, layer 2's alias requirement, layer 4's detector requirements, layer 5's matching semantics, layer 7's API surface | A reader can open it and check. Strongest |
| **Code analysis of the published plugin** | Layer 2's null write-index resolution, layer 3's replace-not-merge, layer 6's alias-naming mechanism, layer 1's fan-out claim | Checkable by a reader against published source. Strong, but a fix can move it — see *Version-dependent* below |
| **Reference setup from a support engagement** | Layer 1's concrete template shape, layer 2's bootstrap and policy ordering, layer 5's sequencing rule, layer 7's per-rollover verification | **Not vendor-documented.** It is a worked configuration that one engineering team recommended for one estate. Treat it as a well-reasoned pattern, not as a specification |

**Where a layer's recommendation is engagement-sourced, the layer file says so in that claim's own bullet.** That is deliberate: a reader deciding whether to restructure their ingestion is entitled to know which parts are vendor guidance and which are one team's design.

## The reference setup exists in two revisions, and they disagree

Both revisions describe the same target shape. They differ on five points of execution. **The layer files present these as alternatives with the trade-off each carries**, rather than asserting one is correct — the trade-off is the durable part, and which is right depends on the estate.

| Point | Earlier revision | Later revision | Where |
|---|---|---|---|
| ISM strategy | A new policy at a higher `ism_template` priority | Add the rollover action to the existing policy, same id | Layer 2, Alternatives A and B |
| Retention states | Two states, transition on time since rollover | Keep the existing four states and their age conditions, adding only the rollover action | Layer 2 |
| Detector save after cutover | Update each detector once | No detector update is part of the change | Layer 4 |
| Index date format | Dotted | Hyphenated, with a counter | Below |
| Base field widths | Several fields widened to `text` with a `keyword` subfield | A strict one-line diff that keeps the existing types | Layer 1 |

**The date format is not cosmetic.** The later revision's check for leftover legacy indices works *because* the format changed — under the earlier format, a pattern matching legacy dated indices also matches rollover indices, so the check cannot distinguish them. A setup that picks one format and then uses the other's verification will report a false result. Layer 2 states the consequence rather than mandating a format.

**The later revision also records that the earlier revision's proposed policy did not exist** on the estate when it was checked. That is a statement about one estate, not about the design, and the layer files do not treat it as a defect in the earlier approach.

## Never reconciled in the source material

- **Whether field aliases belong in the operator's own index template.** Answered three ways across the engagement — *no, leave them to the plugin's component*; then *yes, both*; then *optional*. The customer noticed the reversal and asked for a ruling, and none was recorded. **Layer 3 follows the published documentation**, which states the case for declaring them yourself, and says the engagement did not settle it.
- **Whether rule tagging is a supported way to keep test rules from triggering production alerts.** Asked and never answered. Layer 6 describes the mechanical consequence for untagged rules and makes no claim about support.
- **Which root cause the original incident had.** The index-template type mismatch the case was opened on was later stated to be *not one of* the causes. Nothing in this runbook depends on which was right, because every layer checks a state rather than attributing an incident.

## One correction that matters more than the conclusion

Guidance circulated during the engagement said **not** to give a test alias a write index. It was withdrawn in writing after testing showed the opposite: without one, the test detector checks no data in new indices. Layer 6 states the correction, not just the conclusion, because **anyone working from the earlier advice has a test detector that evaluates nothing and reports healthy** — and they will not find that by reading a document that only states the right answer.

## Version-dependent

Layer 6's alias-naming constraint, and layer 3's wrong-template consequence, rest on how a field-mappings save resolves the index template. On the released lines it matches the input name before the backing index and appends the alias pattern to whichever template matched. An upstream change removes both behaviours for alias inputs; at the time of writing it is merged on the development line and one release branch, **backported nowhere**, and that branch has no release tag — so a reader should expect the original behaviour and check their own build. The change repairs nothing already misattached.

## Not established

- **Whether a detector reads every member of its alias or only the write index.** The plugin exposes both a write-index resolution and a resolve-all path; which governs a scheduled evaluation was not traced. No layer depends on the answer — layer 2 reads the designation rather than inferring scope, and layer 3 instructs reading every member rather than assuming parity.
- **Whether an equal-priority tie between two matching index templates is stable.** The engine compares with a strict greater-than over a map whose iteration order is not sorted, so it need not be. Layer 1 reports the tie rather than predicting a winner, and an automation asserting one would be unsound.
- **Whether the shard thresholds quoted in layer 1 count hidden indices.** The managed-service insight that publishes them does not say, and a plugin's own index families are hidden and numerous. Layer 1 uses the thresholds as orientation for the operator's own indices and does not attempt an attribution.
- **How much of the engagement record was reachable.** The correspondence was read in full and is known complete. The worklog thread was not — roughly a hundred of an estimated two hundred and fifty entries were retrievable, and one unreachable entry is cited elsewhere as containing a raw index template. The customer's annotated objections to the reference setup exist only as a prose summary; that attachment has no text layer.

## To raise the level

Exercise the runbook against two setups on a non-production domain: one built to the reference and one deliberately non-conformant at a known layer. Confirm each layer returns the expected verdict and that validation items 1 and 2 pass — in particular that `${TEMPLATE}` resolved from a member index name matches the template that actually created it, and that the component-template positive control can be produced. Then set `verification: replicated` and record the version tested.

The single most valuable addition would be a **per-rollover** run rather than a point-in-time one. Several conditions these layers describe are created at a rollover and are invisible at an arbitrary hour, and no layer here has been observed across one.
