# Rule patterns — what a Sigma rule compiles to, and which shapes cost

Read this when the audit has found extraction-failed rules and you need to know **what to change**. The procedure in `SKILL.md` identifies cost; this explains where it comes from and which authoring shapes produce it.

Everything here is about the **compiled** query, not the rule as written. That distinction is the single most important thing in this file.

## Contents

1. [Read the compiled query, not the authored rule](#read-the-compiled-query-not-the-authored-rule)
2. [What each construct compiles to](#what-each-construct-compiles-to)
3. [Two traps that are correctness, not cost](#two-traps-that-are-correctness-not-cost)
4. [Rules that never evaluate at all](#rules-that-never-evaluate-at-all)
5. [Anti-patterns, with the reason each one costs](#anti-patterns-with-the-reason-each-one-costs)
6. [Rule-set organisation](#rule-set-organisation)
7. [A disputed remedy, recorded rather than resolved](#a-disputed-remedy-recorded-rather-than-resolved)
8. [Sources](#sources)

---

## Read the compiled query, not the authored rule

**A rule can contain no wildcard character and still compile to a leading wildcard.** Modifiers produce them. So an audit that greps the authored rule for `*` finds the obvious offenders and misses the common ones.

The compiled form is what the engine evaluates, and it is readable: the rule search API returns it as `queries[].value`. Reading it is how modifier expansion, misplaced boolean logic and non-selective clauses all become visible at once — none of which can be seen in the YAML.

Two further reasons the compiled form is the right unit:

- The **clause count** that matters to the engine is the compiled count, not the number of conditions an author wrote.
- Where a log source maps to more than one candidate field, the whole selection is compiled **once per field and OR'd together**, so the compiled clause count can be roughly double what the author would predict.

## What each construct compiles to

Source: analysis of compiled queries read out of the plugin's percolator query index.

| Construct | Compiles to | Cost |
|---|---|---|
| exact value | `(<FIELD>: '<value>')` | cheapest — no wildcard, terms extractable |
| `startswith` | `(<FIELD>: <term>*)` | trailing wildcard only |
| `contains` | `(<FIELD>: *<term>*)` | wildcard both sides |
| **`endswith`** | **`(<FIELD>: *<term>)`** | **leading wildcard — the expensive form, produced by a modifier that looks harmless** |
| a list under an `all`-style modifier | `((<FIELD>: *<t1>*) AND (<FIELD>: *<t2>*) AND … )` | one ANDed wildcard clause **per list element**, all on the same field |
| a selection against a log source with two candidate fields | `((S on <FIELD1>) OR (S on <FIELD2>)) AND ((T on <FIELD1>) OR (T on <FIELD2>))` | clause count roughly doubles |

**The list case has a second failure worth naming.** Where tokenisation produces very short elements, those compile to clauses such as `(<FIELD>: *.*)` or `(<FIELD>: *-*)` — patterns that match nearly every document, contribute no selectivity, and are nonetheless evaluated against every document. A long token list on a free-text field, such as a command line or a script block, is the usual source.

**Where the cost is actually paid.** The query string is re-parsed and the automaton rebuilt **inside the percolate match path**, so the cost scales with documents multiplied by rules rather than being paid once per rule. That is why a broad rule on a high-volume source is expensive in a way the same rule on a quiet source is not.

## Two traps that are correctness, not cost

These do not slow anything down. They make a rule do something other than what it says, which is worse.

**A negation distributes as AND-of-NOTs.** `not <filterblock>` compiles to each condition negated and ANDed together, not to a negation of the block as a whole. An author expressing "exclude if any of these" gets "exclude only if all of these" — which, for mutually exclusive conditions, is never true, so **nothing is excluded and the rule fires on exactly the traffic it was written to suppress**. To express "exclude if any", group the exclusion set with `or:` inside the filter block before negating it.

**A negated clause is guarded by a field-existence check.** Each negated condition compiles with an `_exists_:<FIELD>` guard alongside it, so **a document that lacks the field is not excluded by a negative filter**. If your exclusion depends on absence meaning "not matching", it does not.

## Rules that never evaluate at all

A rule referencing a field that is absent from the detector's field mapping **fails to index into the percolator query index**. The rule does not evaluate — and the failure surfaces only in the cluster log, as a parse failure naming the field:

```
MapperParsingException[failed to parse];
  nested: QueryShardException[No field mapping can be found for the field with name [<FIELD>]]
```

Nothing appears in the interface. The detector keeps running and its other rules keep working, so the detector looks healthy while a subset of its coverage is inert.

The operational consequence is worth stating plainly: **field mapping is the de-facto rule-selection mechanism.** A pre-packaged rule whose fields were never mapped is skipped rather than broken — so adopting a rule set wholesale does not give you that set's coverage, it gives you whatever subset your mapping supports. Where the absent field names look like a generic schema rather than your own, that is the signature of rules bound to a different field vocabulary.

### A field typed `text` instead of `keyword` produces zero findings, silently

This is the same class of failure and arguably more common. Packaged rules match strings with **term-level** semantics — exact equality, and the `endswith` / `startswith` / `contains` modifiers — and those query types work against a `keyword` field. An analysed `text` field is tokenised and lowercased at index time, so an exact term query has **no matching token to hit**.

The detector runs. No error is raised. It produces nothing.

So a rule-matched string field needs a `keyword` mapping, not `text` alone. Note that for an **aggregation** rule's `groupBy` field, a `text` field with a `.keyword` sub-field is not sufficient either — the field itself must be `keyword`.

Two adjacent failures with the same invisibility:

- A rule referencing **both a parent field and a child of it** cannot be mapped; the pair has to be flattened to sibling names.
- A detector pointed at an index or alias that **does not exist** fails the whole monitor for that run, not just one rule.

## Case sensitivity, and a workaround that pulls against cost

Rule matching in this engine is **case-sensitive**. Enumerating every casing permutation as separate rule values does not scale, and the workaround on record is to replace a `contains` modifier with a regular expression using an explicit per-character class — `[Nn][Ee][Tt]` and so on.

**That workaround is in tension with everything else in this file.** A regular expression compiles an automaton just as a wildcard does, and the guard against leading wildcards does not apply to regular expressions at all. No source addresses the interaction, so treat this as a correctness fix with an unmeasured cost, and prefer it on low-volume sources. The underlying limitation is acknowledged upstream with no fix date.

## Anti-patterns, with the reason each one costs

| Anti-pattern | Why |
|---|---|
| Stacking several modifiers on the **same field** in one filter block and expecting OR | Conditions in a block are ANDed. A value cannot simultaneously end with one string, start with another and contain a third |
| An `all`-style modifier over a long token list on a free-text field | One ANDed wildcard clause per token, and short tokens yield clauses matching almost everything |
| Any construct placing a wildcard at the **start** of the compiled string, including `endswith` | Matching a leading wildcard requires examining every term; a modifier rather than an explicit `*` is the usual source |
| Adopting a pre-packaged rule without aligning its field names to the local schema | The rule loads, never matches, and reports nothing |
| Using the correlation builder for boolean logic | It supports field and field-value per log source, AND between queries, a minimum query count and a time window — not OR, not AND NOT, not full query syntax |
| Using finding metadata — severity, technique tags, confidence — as correlation conditions | Correlation joins findings on shared **indexed log field values**; those are rule and finding metadata |
| Reaching for an infrastructure remedy against a cost symptom whose hot threads name the rule-compilation path | The cost does not scale with hardware. A larger instance delays recurrence rather than removing it |

## Rule-set organisation

- **Push boolean complexity down into detection rules**, and keep a correlation rule to a field join plus a time window. Sigma supports OR, AND, AND NOT, wildcards and nested conditions; the correlation builder supports none of them.
- **Decompose one complex requirement into several narrow rules**, one per independent evidence source, then correlate their findings on a shared field. Independent evidence on the same join key is higher-confidence than either source alone, and it is the only shape the correlation builder accepts.
- **Splitting a large rule set across several detectors** keeps the constructed query inside the clause and length limits, and is the recommended answer to a rule set that keeps growing. **But it changes your findings, and that may be unacceptable.** Where several rules match one document, a single detector produces **one** finding citing all of them; separate detectors produce **one finding each** for the same document. An operating team rejected splitting for exactly this reason. Decide which you need before splitting — the trade-off is cost ceiling against per-document finding aggregation, and no source resolves it.
- **Consolidate redundant or overlapping rules.** Fewer rules means a smaller constructed query, and consolidating similar detection logic can hold coverage while reducing the count.
- **Carry technique identifiers and confidence in rule metadata**, not as correlation conditions.
- **Align fields before adopting, then test against real log data.** A rule that loads is not a rule that matches.
- **Keep rules in version control and push them to each cluster through the API.** There is no native cross-cluster rule synchronisation and no native version-control integration; detectors and rules are per-cluster artefacts.

## An unresolved conflict about what a detector actually reads

Two engineering analyses give materially different runtime scopes for a detector whose data source is an alias:

- **One says it percolates across every in-scope index**, used to explain why two mapping shapes behind one alias collided into a merge conflict.
- **One says doc-level monitors track only the alias's write index**, cited to specific lines of the monitor runner, and used to explain coverage gaps for documents written outside the write-index window.

**The choice inverts the remedy**: the first says prune the alias to one mapping shape, the second says abandon the alias for a narrow index pattern. Neither source cites the other.

Code analysis of the published implementation supports the second: the write-index lookup returns the designated member or nothing, and the run-context builder adds only that member. A reconciliation is possible — the all-indexes claim may describe how the query-index mapping is derived at creation time rather than what is scanned at run time — but **no source says so, so do not rely on it.**

## A disputed remedy, recorded rather than resolved

**Does a short detector schedule cause duplicate findings?** Three positions exist and they are not reconcilable as stated:

- Short intervals mean an execution may not finish before the next begins, and consecutive runs contend for the same metadata resources — so raise the interval.
- An operator disputed this: the same interval on their other detectors did not duplicate.
- A root-cause analysis in the same record attributes it to doc-level fan-out not respecting the window boundary, with the two executions landing on different data nodes — citing two public upstream issues, and conceding the operator's objection was valid.

The remedy issued followed the first position; recurrence was reported after applying it. **Treat raising the interval as a cost lever, which it demonstrably is, and not as a duplicate-findings fix, which it is not established to be.**

## Sources

- **Engineering analysis** of compiled queries read from the plugin's percolator query index — the construct table, the negation behaviour and its existence guard, the field-duplication doubling, the non-selective short-token clauses, and the location of the parse-and-rebuild cost inside the percolate match path.
- **Engineering analysis** of cluster logs for the parse failure that leaves a rule unindexed, and for the monitor-level failure when a data source does not exist.
- **Operator guidance** for the organisation patterns, the correlation-builder limits, field alignment before adoption, and version control in the absence of native synchronisation.
- **Published documentation** for leading-wildcard cost, for the non-application of the leading-wildcard guard to regular expressions, and for the clause and query-length limits quoted in `SKILL.md`.

**Two limits on this file.** None of the above is vendor-documented for the construct table specifically: the compiled forms were observed, not published. And **no source anywhere in the material reviewed shows a rule rewritten and its cost re-measured** — not anchoring, not prefix conversion, not leading-wildcard removal. "Avoid wildcards at the start of rule strings" is the full extent of the tested guidance. Every rewrite suggested here is reasoned from the compiled form, not demonstrated. The one attempt on record to produce a rule review was closed without being performed. Treat the table as a reliable guide to what to look for and verify the shape against your own compiled queries before acting on a rule you cannot easily restore.
