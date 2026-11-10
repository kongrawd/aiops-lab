# What this skill rests on

## Verification status

`verification: documented`. Every read traces to a documented API; every component boundary traces to documented structure.

**Not run end to end against a cluster.** `routing-fixtures.md` is what changes that.

## Check these on your own build

Four things vary by build or by configuration, so read them rather than taking them from this page.

**Which fields the detector API returns.** Where the response omits the identifiers linking a detector to its composite monitor and query index, those have to be read from the plugin-owned document; where it carries them, read them directly. The routing in `SKILL.md` step 3 depends on which case you are in.

**How many rules one evaluation retrieves.** This is version-dependent, and backports land at different times across release lines. Read `GET /_cat/plugins?v&h=name,component,version` and treat a large rule set as unsettled until a read-only coverage procedure decides.

**Which plugin indexes are protected.** The registered system-index pattern set changes between versions and depends on plugin settings. Read the current list rather than assuming which plugin indexes mask an unauthorized read.

**How much cause an error alert carries.** An alert can arrive carrying only a transport wrapper with no reason in it. This skill tells you what to do with a cause-free message rather than predicting whether your build produces one.

## Open questions

**Whether a detector's one-index mapping read is safe on a single-member alias.** The `-` rendering means the alias metadata carries no `is_write_index` value; what the detector resolves that to on a single member is not settled, so the four-state table in `components.md` treats it as healthy and says so.

## Why this skill carries no catalogue

An earlier draft held a table of every runbook with its risk and reversibility. That table was the problem: it duplicated what each runbook's frontmatter already says, and nothing would have failed when it drifted — a routing table is never executed against a cluster, so a wrong one stays wrong silently.

The division is deliberate:

- **This skill owns product facts** — which component produces which behaviour, and which read confirms it. A component does not stop existing because a document was renamed.
- **The collection owns its own metadata.** Step 4 reads `risk_level`, `read_only` and `reversible` from the file at handover, so the warning stays correct after a procedure changes.
- **A fleet sweep, where the collection has one, owns enumeration.** Overlap with the reads here is intended — both read the same fields, and the sweep is exercised against clusters.
- **Runbooks own defects.** A defect is the most perishable fact available: it gets fixed, backported unevenly across release lines, and then a document that routes on its signature sends a reader looking for something that no longer happens. A runbook can carry one because it states its own `applies_to_versions`, names the behavioural difference between builds, and is revisited when the fix lands. This skill routes on documented structure and documented behaviour instead, and reaches a defect through the component that exhibits it.

Naming a specific runbook anywhere in this skill is a fault in the skill.

## Why there are no confidence scores

Comparable router patterns score numerically — similarity thresholds, embeddings, cosine distance over descriptions. This one does not.

The discriminating reads return definite answers, so a probabilistic match over prose would replace a certainty with an estimate. And a confidence figure would be invented: the standard in this collection is that a figure the operator measures beats a figure the document carries, and a fabricated score reads exactly like a measured one.

## What would raise this to `replicated`

Run the fixtures in `routing-fixtures.md` against a cluster carrying each condition. Confirm each signal reaches its expected component, that the negative cases return no component, and that each blind check reproduces its misleading rendering. Then set `verification: replicated`.

Exercise the blind checks first — they are the part an operator is most likely to trust.
