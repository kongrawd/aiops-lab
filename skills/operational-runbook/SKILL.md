---
# Latest version:
#   SKILL_BASE_URL=https://github.com/kongrawd/aiops-lab/tree/main npx skill skills/operational-runbook
# Re-run that to pick up changes rather than editing a local copy in place.
name: operational-runbook
description: Turn established knowledge or past incidents into operational runbooks any operator can execute and that are safe to publish — one short procedure per situation, with quantified impact, a symptom check before anything else, exact commands, explicit branches, validation, rollback, and escalation on a clock. Use this skill whenever someone asks for a runbook, procedure, SOP, playbook, operator guide, "steps to fix X", "what should they do when Y happens", or wants to convert a case, ticket, RCA, postmortem or knowledge-base entry into something an operator can follow under pressure. Also use it when the same failure keeps being re-raised and the real gap is that no documented procedure exists, or when asked to write recovery, remediation, failover, rollback or validation steps for a recurring issue.
---

# Operational runbook

A runbook is one short, ordered procedure for one recurring situation, written so a tired operator can execute it correctly without re-reading the whole document. It answers *"this is happening, what do I do"* — not *"how does this system work."* That second question is a different document, and conflating them is the most common way a runbook becomes unusable.

AWS Well-Architected **OPS07-BP03** defines it as a documented process to achieve a specific outcome, and requires it to carry error handling, tools, permissions, exceptions and escalations. **OPS07-BP04** draws the line this skill depends on: a *playbook* investigates an unknown cause; a *runbook* executes a known one. When you find yourself writing "investigate why…", you have found a branch that belongs in a different document — see **Branching** below.

## The workflow

### 1. Get the starting points, then fan out

Ask what to start from. Accept anything: case or ticket IDs, a knowledge-base entry, a cluster or service name, a symptom that keeps being reported, or just "whatever we have on X."

Then dispatch research agents in parallel to gather the material and, critically, to **find the discrete runbooks hiding in it**. One incident usually contains several — a detection procedure, a recovery procedure, a prevention change, a validation routine. Each is its own runbook. Keeping them separate is what makes each one usable under pressure.

Have each agent return, per candidate: the situation in one line, the observable trigger, what the fix was, whether a rollback exists, and the source it came from. Ask them to flag anything they could not verify rather than filling the gap.

### 2. Propose the set before writing

Show the user the candidate list with a one-line description each, and say which you would write first and why. Let them cut, merge or add. This step is cheap and it prevents the expensive mistake of writing six runbooks when the user wanted two.

**Apply the verification gate here, before any writing.** For each candidate, say whether its steps are replicated, source-backed, or resting on inference — and list the inference-backed ones as *skipped*, with what evidence would admit them. See **Verify before you write** below. Sorting this at proposal time costs a sentence per candidate; discovering it after drafting costs the draft.

The test for whether two candidates are really one runbook: same trigger, same owner, same safety profile, and the combined procedure still reads in under two minutes. If any of those fails, keep them separate and link them.

### 3. Write each one

**Scale the document to the procedure.** The template shows the full shape, which suits a procedure with branches, a destructive step and several environment-specific values. Most situations need less than that, and a short runbook that matches its situation beats a long one padded to fill a template. Carry these always:

- **Before you use this**, unedited — it travels with every published copy.
- **When to use this**, with the trigger and the checkable conditions.
- **Impact**, as mechanism plus a command that sizes it.
- **Procedure**, step 1 confirming the problem is real.
- **Validation**, with exit criteria.
- **If this does not work**, with a route and the evidence to attach.
- **Verification status**.

Include the rest only where the situation produces them: *Values to obtain* where more than one value varies by environment; *Before you start* where access, tooling or timing can block someone mid-procedure; a whole-procedure *Rollback* where the sequence may need undoing as a unit, as opposed to the per-step rollbacks that always sit beside a destructive step; *Rollback of the rollback* for high-risk procedures only; *Related runbooks* where a branch actually left the document.

**Set `read_only` honestly, and let it drive the shape.** `read_only: true` asserts that no step changes anything — the runbook observes, measures and decides, and the reader can run it on production without a change window. It earns real trust, so it has to be exact: one write anywhere in the procedure makes it `false`, including a write that only looks like a read, such as an API that records an acknowledgement or resets a counter as a side effect. A genuinely read-only runbook then carries no rollback and no state-change warnings, and `reversible` stops applying. Triage and verification runbooks are often read-only; call that out, because it tells an agent the procedure is safe to run unattended and tells a person they can run it before they have approval to change anything.

Omitting a section that has nothing to say is correct. Leaving the heading in place with a placeholder underneath is the failure — a reader cannot tell an empty section from an unfinished one, and under pressure they will assume the latter and go looking elsewhere.

Use the template in `assets/runbook-template.md`. Two references, both worth reading before your first runbook of a session: `references/runbook-shape.md` for the section-by-section spec and what makes each section fail, and `references/durable-content.md` for writing content that does not decay.

### 4. Quality gate before you hand it over

Run both checklists — "Before you hand it over" in `references/runbook-shape.md`, and the durability checklist in `references/durable-content.md`. The four that catch the most defects: does every step still trace to a replication or a reputable source now that it is written out; can someone who has never done this complete it from the document alone; is there a rollback beside every step that changes state; and is there any hardcoded value a command should have produced instead.

Writing a procedure out often exposes a step that was assumed rather than sourced. When that happens, either source it or cut the runbook — the gate applies to the finished document, not only to the candidate.

### 5. Name it, then place it where it will be found

A runbook that one search finds is a runbook that gets used, which is why OPS07-BP03 asks for a central location and for updates to travel with the change that caused them.

**Name it `<service>-<subject>-<action>`,** lowercase with hyphens, no leading or trailing hyphen, 64 characters at most — the same constraint the frontmatter `name` carries, and the two must match the directory name exactly. Examples of the shape: `rds-replica-promotion-recovery`, `eks-node-drain-recovery`, `s3-replication-backlog-investigation`.

Each part earns its place:

- **Service first**, because a collection is browsed and filtered by service long before anyone remembers a procedure's name. An agent catalogue is filtered the same way.
- **Subject** is the situation, not the mechanism by which it fails. A reader searching the symptom should land on it; naming the internal cause makes it findable only by someone who already knows the answer.
- **Action** is what the runbook does, as a noun: `verification`, `recovery`, `investigation`, `remediation`, `rotation`, `failover`.

Drop any token that repeats another. If the name needs four or more tokens to be unambiguous, that is usually a sign the runbook covers two situations — recheck the merge test in step 2.

**Before you settle on a name, read the existing collection.** Match the pattern that is already there rather than introducing a second one; if what is there is inconsistent, say so and propose which to standardise on rather than quietly picking. Consistency across the collection is worth more than any individual name.

**One directory per runbook**, named for the runbook, with `SKILL.md` at its root — so each one is independently loadable and zippable as an agent skill. Not a single document accumulating procedures.

**Cross-reference siblings by name, not by invented identifier.** `rds-replica-promotion-recovery` is stable, greppable and resolves to a directory; `RB-DB-004` requires a registry someone has to maintain, and a reader cannot act on it.

**Say nothing in the document about whether a sibling exists yet.** Give the name and the condition that sends the reader there, and stop. A published runbook that annotates its own references with *not yet written* reads as unfinished, dates the moment the sibling lands, and hands the reader an absence they can do nothing with. The forward reference is still worth making: it names the right destination, and it resolves the day that runbook is added. Report the gaps to whoever owns the collection instead — that is what step 2's skipped-candidate list and your closing summary are for.

Say what you wrote and where. Report the skipped candidates alongside the written ones, with the evidence each one needs — an operator gap the user can see and close is more useful than a procedure that quietly fills it with a guess. For what was written, name which steps are replicated and which rest on documentation, so the user knows what to exercise first.

## Non-negotiables

The research is consistent on these. They are what makes the difference between a runbook an operator follows and one they set aside.

**The first step confirms the problem is real.** Alerts fire on transient conditions, on monitoring faults, and on thresholds someone set optimistically two years ago. An operator running a recovery procedure against a healthy system is a reliable way to turn a minor alert into an actual outage. So step 1 is always a check, and it must say what to do when the check comes back clean — close the alert, fix the monitor, stop here.

**Quantify the impact — but as a measurement the operator takes, not a number you inherit.** This is the section used to judge urgency and write the first status update, and it is the one most often written as a vibe. The resolution is to split it: state the **mechanism** as something that does not depend on magnitude (*"messages published before the consumer subscribes are never delivered, because delivery begins at the subscription point"*), then give the operator the **command that sizes it on their system**. A figure copied from one incident starts decaying immediately; a figure the operator measures at execution time is current by construction. Where nothing was ever measured, say so and name the command that would measure it. Never invent one — a fabricated figure reads exactly like a measured one and will be quoted back as fact.

**Write what cannot rot.** If a command can tell the operator the value, the runbook carries the command, not the value. Hostnames, resource and endpoint names, node counts, member lists, current versions: all were true when someone looked, and none of them announce their own change. State invariants, which hold by design; measure observations, which held at a moment. Keep dates out of the body entirely. `references/durable-content.md` has the word list, the four classes of number, how to handle waits and durations, and a before/after set — read it with `runbook-shape.md`.

**Exact commands, literal expected output.** Not "check the queue depth" but the command in a code block, with what correct output looks like underneath. The operator cannot tell success from failure by looking at unfamiliar output, and the runbook interpreting the result for them is most of its value. Where a value must be substituted, mark it unmistakably and say where to get it.

**Rollback sits beside the destructive step, not in a later document.** The moment someone discovers a change made things worse is the worst available moment to start working out how to undo it. Write the inverse next to the action. For high-risk procedures, also write the rollback of the rollback — the alternative is inventing it live.

**Exit criteria are an observation, not a feeling.** Not "the service should recover" but "error rate under 1%, queue below 500, a real request to the endpoint returning 200." Without this the operator either stops too early or keeps intervening long after the system recovered.

**Escalation carries a clock.** "If not recovered 20 minutes after step 4, contact X." An operator who has run out of steps and has no instruction will improvise, at the hour when improvisation is least reliable. Giving explicit permission to stop is often the most valuable sentence in the document.

**Separate `last_edited` from `last_verified`.** A wording change says something about maintenance; it says nothing about whether the recovery path still works. These two dates are the only ones the document carries, and a procedure last confirmed two quarters ago in an actively changing system should be treated as presumed stale rather than as current.

## Branching

Real procedures branch, and a flat list forces the operator to run every step including the destructive ones when the first might have sufficed. Make branches explicit: state what to check, and what each result means.

Split a branch into its own runbook when it has a **different owner**, a **different safety profile**, or a **procedure long enough to need its own pre-flight**. Link to it by name rather than nesting it. Keep the main path short and linear with the branches hanging off it.

Two branch types that always leave the document:

- **Unknown cause.** "Investigate why the queue is growing" is a playbook, not a runbook step. Name it, link it, move on.
- **Anything only the vendor can do.** A step requiring internal tooling or a service-team action is not a procedure step in a runbook written for a public reader (the default — see below). It is an escalation step, with what evidence to attach.

## Visibility: write the publishable version first

Runbooks get shared publicly — into a repo, a documentation site, an agent-skill catalogue, or with anyone operating the same system. Treat the publishable form as the primary artifact and add attribution to a private copy only where it is genuinely needed. Deciding this after the fact means auditing prose for leaks, which is slower and less reliable than never writing them.

**Nothing identifying goes in the runbook.** No person or team name, no case or ticket identifier, no internal URL, no account, domain, cluster or hostname, and no measurement taken from one environment. The template has no slot for any of these on purpose — a slot invites a value, and a committed file is the wrong place for them. Where attribution matters, describe the class of source instead: *"service-team analysis during a support engagement on 3.7"* carries the same evidential weight as a case number and survives publication.

**Every command must be one the reader can run** — console, public API, their own CLI. Internal tooling, internal dashboards and ticket IDs are not instructions. A step that only the vendor can perform is not a procedure step; it is an escalation step, with the evidence to attach.

**Permissions are a prerequisite, not an assumption.** Name the role or group that grants the access. Discovering a missing permission three steps into a rollback is a design failure of the runbook.

**Unconfirmed causes stay unconfirmed.** Where the cause is genuinely open, say what is known and what remains under investigation. Do not resolve it silently in the operator's favour, and do not import the internal debate about it.

**References are public.** `metadata.references` carries product documentation, API references, upstream changes and standards — things the reader can open. An internal link in that list makes the runbook unusable for the person holding it.

## Verify before you write, and skip what you cannot verify

A runbook is executed against production by someone who is trusting it. That sets the bar for what may go in one: **every step traces to something replicated or to a reputable source.** Inference does not qualify, however reasonable it looks.

Check each candidate against this before writing it, not after. Each step must rest on one of:

- **Replication** — the step has been run and the output observed, either by you or by a source that states it ran it.
- **A reputable source** — vendor documentation, vendor engineering analysis of the specific behaviour, an upstream change record, or a published standard.

Where a step rests on inference instead — plausible from how the system appears to work, assembled by analogy with a similar component, or extrapolated from a related behaviour — **skip the runbook and report the gap.** A procedure that looks authoritative and happens to be wrong costs more than no procedure, because the operator stops looking for alternatives. Reporting the gap is a useful result: it names the one thing that would unblock a runbook someone can trust.

When you skip a candidate, say which step could not be sourced, what source would settle it, and who holds it. *"Step 3 assumes the write is idempotent; vendor confirmation or one replication on a test target would settle it"* gives the user something to pursue.

Two values are therefore valid for `metadata.verification`:

- `replicated` — the sequence has been run end to end against a system in this version range, and the observed output matches what the runbook states.
- `documented` — every step traces to a reputable source, and the sequence has yet to be run end to end. Usable, and common for a procedure derived from vendor analysis.

Fill the **Verification status** section to match. Name which steps are replicated and which are documented, because the two carry different weight and the reader is entitled to know which they are holding. Then state what would raise the level: *"Exercise steps 3–4 on one non-production target and set `verification: replicated`"* is an action; *"needs testing"* is not.

**Not established** stays in the document for a different purpose — what the sources could not settle about the situation itself, including any circulated method that turns out not to work. Surface that kind of hazard in the pre-flight as well, because superseded guidance is what sends an operator down a path that silently returns nothing.

## Grounding

Everything in a runbook is an instruction someone will execute against production, so the sourcing bar is higher than for a summary. The gate above sets what is admissible; this is how to hold the line while writing.

Work from what was actually verified. A command that appeared in a case, a ticket or a vendor response is citable — record the class of source it came from. A command assembled from documentation is admissible and is marked as documented rather than replicated. A command assembled from how the system seems to work is not admissible, and recognising that is easiest at the moment you write it, because that is when you know whether you are recalling a source or reasoning from one.

Where the root cause is contested or unconfirmed, do not resolve it silently in the operator's favour. A runbook remains useful while the cause is open, as long as each step is honest about whether it is a mitigation or a fix — an open cause is compatible with well-sourced steps. Record the provenance so the next person can check whether it still holds.

## Reference files

- `assets/runbook-template.md` — the copy-paste shape. Start here when writing.
- `references/runbook-shape.md` — section-by-section spec, worked example, failure modes per section, and the pre-handover checklist.
- `references/durable-content.md` — how to write content that does not decay: derive-do-not-restate, invariants versus observations, the four classes of number, time and waits, the time-anchored word list, environment as checkable conditions, and the durability checklist.
