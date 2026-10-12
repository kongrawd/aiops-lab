---
name: <service>-<subject>-<action>   # lowercase and hyphens only, max 64 chars, no leading
                                     # or trailing hyphen. Must match the directory name
                                     # exactly. Service first so the collection filters by
                                     # it; subject is the situation, not the mechanism;
                                     # action is a noun - verification, recovery,
                                     # investigation, remediation, failover.
description: >-
  <100-1024 characters. The situation, the observable trigger, and when an operator or
  agent should reach for this. Written so relevance can be judged without opening the file.>
metadata:
  agent_types:
    - <GENERIC | CHAT | INCIDENT_TRIAGE | INCIDENT_RCA | INCIDENT_MITIGATION | PREVENTION>
  skill_type: USER
  service: <service / plugin / component>
  applies_to_versions: <range this holds for, e.g. "2.19 and later">
  risk_level: <low | medium | high>
  read_only: <true | false>               # true only if no step changes state. A read-only
                                          # runbook needs no rollback and no state-change
                                          # warnings, and an agent may run it unattended.
                                          # One write anywhere in the procedure makes it false.
  reversible: <yes | no | partial>        # if no or partial, say so in the rollback section.
                                          # Not applicable where read_only is true
  automatable: <yes | no | partial>       # if not fully, state why in "Before you start"
  verification: <replicated | documented>  # only these two. A procedure resting on
                                           # inference does not get written - see SKILL.md
  last_verified: <YYYY-MM-DD>   # when the procedure was last confirmed to still work.
  last_edited: <YYYY-MM-DD>     # when the text last changed. Deliberately separate: editing
                                # a sentence establishes nothing about the recovery path.
                                # These two are the only dates in the document.
  references:                             # public sources only
    - title: <document title>
      url: <public URL>
      kind: <product-doc | api-reference | upstream-change | standard>
---

# Runbook — <situation in one line, as the operator would describe it>

> Authoring note, remove before publishing: no dates in the body. No person, team, case or
> ticket identifier, account, hostname or single-environment measurement anywhere in this file
> - those live only on an internal copy. Where you are tempted to write a value, write the
> command that produces it. Omit an expected duration unless one has been measured - a guess
> is the thing this avoids.

## Before you use this

Keep this section as written. It travels with the document, and the operator who needs it most is the one reading under pressure.

- **Confirm the behaviour against current vendor documentation, or with your support channel, before you execute this.** Product behaviour moves with versions, and this runbook states the version range it was written for in `metadata.applies_to_versions`. Where the document and the documentation disagree, the documentation is current and this is not.
- **Exercise the procedure in a non-production environment first.** Run it end to end on a target you can afford to break, confirm the output matches what each step states, and only then run it where it matters.
- **You remain responsible for the outcome in your environment.** This document is offered as-is, with no warranty and no liability for any effect of its use. Each step changes a real system, and the decision to run it is yours.
- **Send corrections back.** A step that failed, output that differed, a version where the behaviour changed, or a clearer command are all worth more than a clean document that is quietly wrong. Report them through the channel this runbook was shared on, so the next operator starts from what you learned.

## When to use this

**Trigger.** <The exact alert, threshold or observable symptom. Word it the way the signal is worded so the two can be matched at a glance. "P99 above 2s for 5 minutes" is a trigger; "search is slow" is a vibe.>

**Applies when all of these hold.** Each is checkable — if one cannot be checked, label it an assumption.

| Condition | How to check | Expected |
|---|---|---|
| <e.g. data source is an alias, not a concrete index> | `<command>` | `<result>` |
| <e.g. engine version in range> | `<command>` | `<range>` |
| <e.g. configuration this assumes> | `<command>` | `<result>` |

**Does not apply when.** <The near-miss condition that sends the operator elsewhere, with the check that distinguishes it and a link to the right runbook. This is where the wrong runbook gets run.>

## Values to obtain

Declare every environment-specific value once, then refer to the name throughout. Never embed a literal in a step — a hostname or index written into a procedure is a claim that decays silently.

| Name | What it is | How to obtain the current value |
|---|---|---|
| `${<NAME>}` | <what it identifies> | `<discovery command>` |
| `${<NAME>}` | <what it identifies> | `<discovery command, or where the authoritative value lives>` |

<If any of these changes on every deployment or rollover, say so explicitly here — it changes how the operator treats the value.>

## Impact

**Mechanism.** <How the harm arises, stated so it does not depend on magnitude. This is the durable half and the part the operator needs in order to understand what they are seeing.>

**Size it for this system.** <The command or comparison that measures the magnitude at execution time, so the figure is current by construction rather than inherited from someone else's incident.>

```
<measurement command>
```

**Blast radius.** <What is in scope when this goes wrong, named by role rather than by count — "every consumer reading from the shared topic" rather than "all fourteen consumers". A count dates; a role does not.>

**Recoverability.** <Whether the damage can be undone afterwards, and by what. This changes how hard the operator should push and whether a slower, safer path is acceptable.>

**Severity.** <The conditions under which this is urgent, as conditions rather than a fixed level. Say outright whether it can wait for business hours — the operator cannot infer it under pressure.>

## Before you start

- [ ] <Access or permission needed, named as the exact role or group that grants it — not "prod access". The operator may have to request it and cannot guess what to ask for.>
- [ ] <Tool or CLI, with the version range if it matters.>
- [ ] <Blast radius check: what else this touches.>
- [ ] <Safe-window condition, expressed as the condition and its reason — so the operator can recognise when the reason stops applying.>

**Do not proceed if** <the precondition that makes this procedure unsafe — the state in which the documented fix is the wrong move. Escalate instead.>

<If `automatable: no`, state the reason here. It tells the operator why they are being asked to think.>

## Procedure

### Step 1 — Confirm the problem is real

```
<exact command>
```

**Expect:** `<literal output>`

**If <clean result>:** the signal is stale. <What to do — close it, correct the threshold, raise a ticket against the monitor.> **Stop here.**

**If <abnormal result>:** continue to step 2.

### Step 2 — <action in the imperative>

```
<exact command, using ${NAME} rather than a literal>
```

**Expect:** `<literal output — so a command that exits zero and changes nothing is visible rather than silent>`

**Branch.**
- `<result A>` → continue to step 3.
- `<result B>` → <different step, or another runbook by name>.
- `<result C>` → go to **Escalate**.

### Step 3 — <action>

> ⚠️ **This changes state.** <What it changes and what the consequence is.>

```
<exact command>
```

**Expect:** `<literal output>`

**Rollback this step:**
```
<exact inverse command>
```
<What the rollback restores, and what it does not — e.g. history cleared by the original action. If no clean inverse exists, say that here rather than leaving the heading empty, and name what the only true recovery is.>

### Step 4 — <a step that waits>

Wait until <observable condition> — confirm with:

```
<command>
```

If the condition is not met within <bound>, go to **Escalate**. <The condition is what you are waiting for; the bound exists only so the operator knows when to stop.>

## Validation

Each item is an observation with a command, not a judgement.

1. ```
   <verification command>
   ```
   **Expect:** `<literal output or threshold, with the threshold's source named>`
2. <A second, independent check — ideally one confirming user-facing behaviour, not just the resource. A metric falling because the system stopped serving traffic looks identical to recovery.>
3. **Observation window.** <The condition that confirms it held, and what would indicate a relapse.>

**Exit criteria.** <The specific state that means done. Anything short of this is not done.>

## Rollback

<For procedures where the whole thing may need undoing, not just one step. Ordered, with commands. State what the rollback does not recover.>

**Rollback of the rollback.** <For high-risk procedures only. If omitted, the operator invents it live.>

## If this does not work

**Escalate when:** <condition, with a bound — "the condition in step 4 is not met within its bound", or "any step returns X">

**Escalate to:** <route, and how>

**Attach:** <the evidence to hand over — outputs already collected, the measurements taken in Impact, what was already attempted. This is what stops the next responder starting over.>

## Related runbooks

Refer to siblings by their name, which resolves to a directory and is greppable. Do not invent identifiers - they need a registry someone has to maintain, and a reader cannot act on one. Give the name and the condition that sends the reader there, and nothing further - in particular, no note on whether that runbook exists yet. A forward reference is fine and resolves when the sibling lands; status commentary reads as unfinished and goes stale. Report gaps to the collection's owner instead.

- `<service>-<subject>-<action>` — <the condition that sends the reader there instead of here>
- `<service>-<subject>-investigation` — <the branch where the cause is unknown. Name it, do not inline it>

## Verification status

Every step in this document traces to a replication or to a reputable source. That is the condition for the runbook existing - a step resting on inference is reason to leave the procedure unwritten rather than to label it.

**Basis.** <What class of source each part came from and what it established - "vendor documentation", "vendor engineering analysis of this behaviour on <version>", "upstream change record", "observed in a controlled test". Describe the source; do not identify it.>

**Replicated.** <Which steps have been run with the output observed, and against what version. Name the rest as documented, because an operator deciding how far to trust a step is entitled to know which of the two they are holding.>

**Not established.** <What the sources could not settle about the situation itself - and in particular any circulated method that turns out not to work. Name that here and in "Before you start", because superseded guidance is what sends an operator down a path that silently returns nothing.>

**To raise the level.** <The outstanding verification, as an action. "Exercise steps 3-4 on one non-production target and set `verification: replicated`" is actionable; "needs testing" is not.>
