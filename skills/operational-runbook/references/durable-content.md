# Durable content — writing runbooks that do not decay

A runbook is a set of claims about the system, and each claim holds only while the system matches it. Code that drifts raises an exception; a document that drifts stays readable and keeps attracting readers, which is what makes a stale runbook harder to work with than a missing one — it moves the operator confidently at the moment there is least slack to check.

This file is about writing the document so that more of its claims stay true on their own. Durability is a design property you build in once, which holds up better than a review cycle you have to sustain.

## Contents

1. [The governing rule: derive, do not restate](#the-governing-rule-derive-do-not-restate)
2. [Invariants versus observations](#invariants-versus-observations)
3. [The four classes of number](#the-four-classes-of-number)
4. [Quantifying impact without writing a number that decays](#quantifying-impact-without-writing-a-number-that-decays)
5. [Time, durations and waits](#time-durations-and-waits)
6. [Words that anchor a document to a moment](#words-that-anchor-a-document-to-a-moment)
7. [Environment and conditions as declared ground truth](#environment-and-conditions-as-declared-ground-truth)
8. [Before and after](#before-and-after)
9. [Publishable by default](#publishable-by-default)
10. [Durability checklist](#durability-checklist)

---

## The governing rule: derive, do not restate

**If a command can tell the operator the value, the runbook carries the command — not the value.**

This single rule removes most of what rots. A hostname, an index name, a node count, a queue depth, a current version, a shard count, a member list: all of these were true when someone looked, and all of them change without telling the document. Writing them in is how a runbook ends up naming a host that was decommissioned two quarters ago.

The classic failure and its fix: a runbook says `restart payment-server-01.prod` because that was the only host when it was written. There are four now, behind a load balancer. The operator restarts one, the alert does not clear, and the next fifteen minutes go into deciding whether the runbook is wrong or the diagnosis is wrong. The fix is to write the step against an **inventory query that returns the current set, and include the query**.

So: where you are tempted to write a value, write the question that produces it.

```
# Instead of: the alias detector-foo-bar contains 2 indices
# Write:
GET _cat/aliases/<alias>?v&h=alias,index,is_write_index
```

What cannot be derived is **meaning** — why the system is shaped this way, which of three similar fields the rules actually read, what a given state implies. That is the part worth paying a human to write, and it is also the part that does not decay, because it describes intent rather than state.

## Invariants versus observations

Every factual sentence in a runbook is one or the other, and they have opposite lifespans.

An **invariant** is true by design — this one happens to be from a search engine, but the test is the same everywhere. *"A detector resolves its alias to a single index: the write index if one is designated, otherwise the most recently created."* That holds until the product changes, and when it does, the runbook is wrong in a way someone will notice loudly. Invariants are safe to write down.

An **observation** was true at a moment someone looked. *"The target group has two members and no healthy one."* *"Six of the config entries are empty."* *"The replacement joins about twelve minutes after launch."* These are measurements of one instance of the problem, and they begin decaying the moment they are written.

Runbooks should **state invariants and measure observations.** When you catch yourself writing an observation into the procedure, convert it:

| Observation (decays) | Converted (durable) |
|---|---|
| "The target group currently has two healthy targets" | Step: query target health. **Expect:** every registered target healthy. Any unhealthy → branch B |
| "Three of the eight listener rules are missing" | Step: enumerate the rules and count those absent. Any non-zero count → proceed |
| "The replica runs about 12 minutes behind" | Step: compare the primary's position with the replica's. Any non-zero lag is the exposure window |
| "The service runs four replicas" | Step: read the replica count from the deployment. Note it for step 5 |

An observation can still appear in a runbook, in one place only: **Verification status**, as evidence that the problem is real and was seen. It must not appear in the procedure, the impact assessment or the validation criteria, because those are executed against a system that has moved on.

## The four classes of number

Every number in a runbook belongs to exactly one of these, and the class determines whether it may be written down.

**1. Product limit or documented default — write it, cite it.** `max_clause_count` default of 1,024. A queue bound of 1,000 per node. These change only on a version boundary, so pair them with the version they apply to and a link to the documentation. Durable.

**2. Threshold derived from a stated objective — write it, name the source.** "Error rate above 1%" is durable if it comes from an SLO or an agreed target, because then the number has an owner and changing it is a decision someone makes deliberately. Name the objective it came from.

**3. Measured baseline — write the measurement, not the result.** "Normal is 400 to 900 requests per second" is a snapshot of one system at one time. Write the command that establishes the baseline and say what to compare it against. If you must include a figure as orientation, label it as an observed sample with its date, and put it outside the procedure.

**4. Plausible-looking guess — leave it out.** This is the one to watch, because an estimate reads exactly like a measurement and gets quoted back as fact. Where a figure has yet to be measured, say so and give the command that measures it.

The honest-unmeasured statement is more useful than the invented one. *"The size of this window has not been measured on this system; `<command>` establishes it"* tells the operator exactly what to do. A fabricated figure tells them something false with confidence.

## Quantifying impact without writing a number that decays

Impact still has to be quantified — an operator cannot judge urgency or write a status update from a vibe. The resolution is to split the section in two.

**The mechanism is an invariant.** State how the impact arises, in terms that do not depend on magnitude: *"Messages published before the consumer subscribes are never delivered, because delivery begins at the subscription point rather than at the start of the topic."* That sentence stays true as long as the behaviour does, and it is the part the operator needs in order to understand what they are looking at.

**The magnitude is a measurement the operator takes at execution time.** Give them the command. *"To size the exposure: compare the topic creation timestamp with the subscription timestamp, then count messages published in that interval — `<command>`."* Now the impact figure is current by construction, because it was measured on the system in front of them rather than on someone else's system months ago.

It also puts the runbook on the critical path in a useful way: the operator produces a fresh number every time, so a change in the system's behaviour surfaces as a surprising measurement rather than as a quietly out-of-date document.

Keep in the impact section, because none of it decays:

- **Mechanism** — how the harm occurs
- **Blast radius** — what is in scope when this goes wrong, named by role rather than by count ("every consumer reading from the shared topic", not "all fourteen consumers")
- **Recoverability** — whether the damage can be undone afterwards, and by what
- **Severity criteria** — the conditions under which this is urgent, as conditions rather than as a fixed level

## Time, durations and waits

Relative and absolute time are the fastest-decaying content in a runbook, and both have durable replacements.

**Absolute timestamps from one incident do not belong in a procedure.** A sequence like *"00:01 the primary stops accepting writes, 00:08 the old primary is demoted, 00:18 the replica is promoted"* describes one day on one system. The durable form is an **ordered sequence of events with a measurable gap**: writes stop, the old primary is demoted, the replica is promoted — and the window between the first and last is the exposure, measured by `<command>`. Same understanding, no dependence on a clock.

**Replace "wait N minutes" with a condition plus a bound.** A fixed duration is a guess about a system that will get faster or slower. Write the observable the operator is waiting for, and give the timeout as a stopping rule rather than as the expected time:

> Wait until `READY` is `True`. If it is not within 10 minutes, go to **Escalate**.

The condition is ground truth; the bound exists only so the operator knows when to stop waiting. Where the expected duration genuinely helps — it tells the operator at four minutes that fifteen is wrong — record it as a measured figure in the metadata header rather than inside the step, and only once someone has measured it.

**Recurrence is durable; a specific occurrence is not.** "Each daily rollover" is a property of the system. "On the tenth and eleventh" is a log entry. Prefer the former in the procedure and leave the latter to provenance.

**Maintenance windows and safe periods are conditions, not clock times**, unless the clock time is itself the invariant. "Outside the rollover window" is durable and self-explaining; "not between 00:00 and 02:00 UTC" is durable only if the rollover is genuinely pinned to midnight UTC — in which case say both, so the operator knows the reason and can recognise when the reason stops applying.

## Words that anchor a document to a moment

A document is presumed current by virtue of existing, so `currently` and `as of this writing` add nothing and age badly. Avoid these in the body of a runbook:

`currently` · `now` · `recently` · `new`, `newer` · `old`, `older` · `latest` · `soon` · `at present`, `presently` · `as of this writing` · `existing` · `does not yet` · `eventually` · `in the future` · `still` (as in "still broken")

Two specific substitutions:

- **For version-dependent behaviour, name the version instead of the moment.** "On 2.19 and later, the write queue bound is 10,000" rather than "the queue bound is now 10,000."
- **For a pending fix, state the condition rather than the schedule.** "Until the fix carrying `<change>` reaches this domain, use the workaround below; confirm which release carries it before relying on the permanent fix." That stays accurate whether the release lands next week or slips two quarters. A date in the body will be wrong either way.

**No month or year appears in the body at all.** Dates belong in the metadata header (`last_verified`, `last_edited`) and in provenance, where they are doing real work. A date inside a procedure is either decoration or a bug.

## Environment and conditions as declared ground truth

The reason a runbook gets run against the wrong system is almost always that its applicability was described loosely. Make it checkable.

**Declare the environment as conditions the operator can verify, not as prose.** Not "this applies to the production search cluster" but a short list with a check against each: the component, the version range, the configuration this assumes, and the access required. If a condition cannot be checked, it is an assumption — say so.

**Declare every environment-specific value once, with how to obtain it.** A variables block at the top, where each entry is a name, what it is, and the command or location that yields the current value. Then the procedure refers to the name throughout and never embeds a literal. This is the difference between a runbook that works on one cluster and one that works on the estate.

A placeholder with no adjacent explanation is a trap: `<YOUR_NAMESPACE>` tells the operator nothing about which namespace is correct or where to look. Every placeholder needs its discovery command, and if the value changes on every deployment, say that explicitly.

**State the preconditions that make the procedure unsafe.** The one thing a runbook cannot supply is judgement about whether the documented fix is safe right now — a restart that is correct on a normal day can be the wrong move during partial data corruption. Name the conditions under which the operator should stop and escalate instead of proceeding. And where a procedure cannot be automated, the reason is often the most important sentence in the document: it tells the operator why they are being asked to think.

## Before and after

Each pair below is from a different system on purpose. The failure is never about the service — it is about writing down a value the system already knows.

**Hardcoded inventory**

> ✗ Drain node `ip-10-0-3-47`.
> ✓ Identify the nodes currently backing the workload — `<inventory command>`. Drain each one returned, in turn.

**Observation in the procedure**

> ✗ Three of the eight listener rules are missing, so expect traffic to fall through to the default.
> ✓ Count the listener rules present — `<command>`. Compare with the count the routing requires. A shortfall is the condition this runbook addresses.

**Decaying impact figure**

> ✗ Around 9,400 messages per day are never processed.
> ✓ Messages published before the consumer subscribes are never delivered, because delivery begins at the subscription point rather than at the start of the topic. Size it for this system: measure the interval between topic creation and subscription, then count messages published in that interval — `<command>`.

**Relative time**

> ✗ Wait about 12 minutes for the replacement node to join.
> ✓ Wait until the node reports ready — `<command>`. If it has not within 30 minutes, go to **Escalate**. Work scheduled before it joins is in the exposure window.

**Time-anchored language**

> ✗ The fix is currently pending and should ship soon.
> ✓ Until the release carrying `<change>` is applied to this environment, use the workaround in step 4. Confirm the carrying release before removing the workaround.

**Loose applicability**

> ✗ Applies to the production clusters.
> ✓ Applies when all of the following hold: the service sits behind a load balancer rather than taking traffic directly (`<command>`); more than one target is registered (`<command>`); engine version is in range `<range>`. Where the service takes traffic directly, use `<service>-<subject>-<action>` instead.

## Publishable by default

A runbook that cannot be shared has to be rewritten before it can be, and rewriting under time pressure is how identifying detail survives into a public file. Write the shareable version first; add attribution to a private copy only where it earns its place.

**Never in the document:** a person's name, a team name, a case or ticket identifier, an internal URL, an account identifier, a domain, cluster, host or index name, or a measurement taken from one environment.

Each has a durable substitute, and in every case the substitute is the better document:

| Instead of | Write |
|---|---|
| A case or ticket identifier as the basis | The class of source and what it established — "service-team analysis during a support engagement on `<version>`" |
| A named individual who verified it | `verification: replicated` plus which steps were replicated, in the Verification status section |
| A team as owner | Nothing in the file. Ownership is assigned where the runbook is published |
| An account, domain, cluster or host | A declared variable with its discovery command — which the durability rules require anyway |
| A measurement from one incident | The mechanism as an invariant, and the command that sizes it here |
| An internal dashboard or wiki link | The public product or API documentation for the same thing |
| An internal tool invocation | The public API equivalent, or an escalation step if none exists |

The convergence is not a coincidence. Almost everything that makes a runbook unpublishable is also something that decays: a hostname, a case number, a one-off measurement, a named person. **Writing for durability produces a shareable document, and writing for shareability produces a durable one.** Where the two pull apart it is usually evidence — the reader needs to know how much to trust a step — and that is answered by stating verification status rather than by citing a source they cannot open.

## Durability checklist

Run this alongside the main pre-handover checklist in `runbook-shape.md`.

- [ ] **No hostname, index, alias, node, role or endpoint is hardcoded** — each is a declared variable with a discovery command.
- [ ] **Every placeholder has an adjacent explanation** of what the right value is and where to find it.
- [ ] **No observation appears in the procedure, impact or validation** — only invariants, and commands that measure.
- [ ] **Every number is classed** — product limit (cited), objective-derived threshold (sourced), measured baseline (command given), or left out because it has yet to be measured.
- [ ] **No invented figures.** Unmeasured things are labelled unmeasured, with the command that would measure them.
- [ ] **Impact states the mechanism as an invariant** and gives the operator a command to size the magnitude themselves.
- [ ] **No "wait N minutes"** — a condition to wait for, with a bound as a stopping rule.
- [ ] **No absolute timestamps in the procedure** — ordered events with a measurable gap.
- [ ] **No month or year in the body.** Dates confined to the metadata header and provenance.
- [ ] **None of the time-anchored words** from the list above.
- [ ] **Version-dependent behaviour names the version range,** not the moment.
- [ ] **Applicability is a checkable condition list,** with the discriminator for the near-miss case and a link to where it goes instead.
- [ ] **Unsafe preconditions are named,** and if the procedure resists automation, the reason is stated.
- [ ] **Nothing identifying anywhere in the file** — no person, team, case or ticket id, internal URL, account, domain, cluster, host, or single-environment measurement.
- [ ] **Every reference in `metadata.references` is a public URL** the reader can open.
- [ ] **Every command is runnable by the intended reader** — no internal tooling; vendor-only actions are escalation steps.
- [ ] **`metadata.verification` is set honestly,** and the Verification status section says what would raise it.

## Sources

- **Google developer documentation style guide, *Timeless documentation*** — the word list, and the principle that `currently` is implied by the document existing. <https://developers.google.com/style/timeless-documentation>
- **Azion style guide, *Word choice*** — no month or year outside a changelog; machine-generated dates exempt.
- **Max Doubin, *Writing Runbooks That Actually Get Used*** — the hardcoded-hostname failure and the inventory-query fix; the silent no-op command; stating why a procedure cannot be automated.
- **Datapace, *Living documentation*** — derive reference material from the system, diff docs against reality, make documents load-bearing so drift surfaces in calm moments. Also the GitLab 2017 case, where procedures asserted a backup capability that every underlying mechanism had silently stopped providing.
- **Sync-o, *Runbook Documentation Best Practices*** — placeholders need adjacent explanation; `last validated` versus `last updated`; event-driven rather than calendar-driven review.
- **DevOps AI Toolkit, *Runbook Drift*** — cross-checking names in the document against live inventory as the detection method for drift.
- **AWS Well-Architected OPS07-BP03** — runbooks are updated as part of the change management process, and belong in version control.
