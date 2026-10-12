# Runbook shape — section spec, failure modes, worked example

Read this before writing your first runbook in a session. It expands each section of `assets/runbook-template.md` with what the section is for, how it fails, and what good looks like.

## Contents

1. [Length and the cold-read test](#length-and-the-cold-read-test)
2. [Section spec](#section-spec)
3. [The standing disclaimer](#the-standing-disclaimer)
4. [Worked example](#worked-example)
5. [Splitting and linking](#splitting-and-linking)
6. [Quantifying impact](#quantifying-impact)
7. [Before you hand it over](#before-you-hand-it-over)
8. [Maturity — when a runbook should become a script](#maturity--when-a-runbook-should-become-a-script)
9. [Sources](#sources)

---

## Length and the cold-read test

Aim for a document the operator can read in about two minutes — roughly 200–600 words of actual instruction, not counting command blocks. The limit is not aesthetic. A runbook long enough to skim is a runbook where a step gets missed, and the most common cause of excess length is two runbooks pretending to be one.

The real test is a cold read: **could someone who has never performed this procedure complete it from the document alone, and recognise whether it worked?** If they would need to ask a question, the answer to that question belongs in the runbook.

Two signs the scope is wrong:

- **One step long.** "Restart the service" does not need a runbook; the alert message can carry the command. A runbook earns its keep when there are pre-flight checks, branches, or a non-obvious rollback.
- **No branches and no rollback.** If the procedure is a flat list of safe commands, it is a checklist. That is fine, but do not dress it as a runbook — and consider scripting it instead.

## Section spec

### Metadata header

Carries relevance, scope and staleness. `name` and `description` are what an agent reads to decide whether this runbook applies; everything else sits under `metadata`. The fields most often wrong:

- **`last_edited` and `last_verified` are different fields,** and this is the distinction that matters most. Editing a sentence establishes nothing about whether the recovery path still works. `last_verified` records when the procedure was last confirmed against a running system. Treat a procedure unverified for two quarters in an actively changing system as presumed stale. These two are the only dates the document carries.
- **`applies_to_versions` is the scope claim the whole document rests on.** A reader on a version outside the range has no runbook here, and is better served knowing that than following it.
- **`verification` is `replicated` or `documented`** — there is no third value, because a procedure resting on inference does not get written. See the gate in `SKILL.md`.
- **`read_only` is the field a reader checks first,** because it answers whether running this can make anything worse. `true` means no step changes state, so there is nothing to roll back and `reversible` does not apply. It is the most consequential claim in the header: an agent may run a read-only procedure unattended, and a person may run it before they hold change approval. One write anywhere — including an API call that records an acknowledgement as a side effect — makes it `false`.
- **`risk_level` and `reversible` set expectations before the reader starts.** `reversible: partial` is a signal to read the rollback section first rather than last. Both are read in light of `read_only`.
- **Ownership stays out of the file.** A published runbook is read by people in other organisations, so naming an owner inside it is both unhelpful to them and a detail that decays. Assign ownership where the runbook is published.

### Trigger / when to use this

The unmistakable signal that this is the right runbook. Word it as the alert words it, so the two match at a glance.

*Fails when* it names a category rather than a condition. "Service degradation" matches everything and therefore selects nothing. "Replica lag above 60s for 10 minutes, writes still succeeding" selects one runbook.

**Include a does-not-apply line.** The near-miss case is where the wrong runbook gets run, and naming it with a link costs one sentence. This is especially valuable where several failures share an error string — a symptom with multiple causes needs a discriminator, not a guess.

### Impact

See [Quantifying impact](#quantifying-impact). This section exists so the operator can judge urgency and write the first status update without escalating to ask.

*Fails when* written as a possibility rather than a measurement. Also fails when it omits **recoverability** — whether the damage can be undone afterwards changes how hard the operator should push and whether a slower, safer path is acceptable.

### Before you start

Access, tools, blast radius, timing. The failure mode this prevents is specific: an operator three steps into a rollback discovering they lack the permission to finish. That is a defect in the runbook, not in the person.

Keep it to three to six checkboxes. If the pre-flight is longer than the procedure, the procedure is probably two runbooks.

### Procedure

Numbered, imperative, second person. One action per step — a step needing two sentences is two steps.

Each step carries: the **exact command** in a code block, the **literal expected output**, and where relevant the **branch** on that output. The operator should not have to interpret the result; the runbook interprets it for them. Where a value must be substituted, mark it unmistakably (`<cluster-name>`) and say where to get it.

**Step 1 is always a confirmation that the problem is real,** and it must say what to do when the check comes back clean. Recovery procedures run against healthy systems are a reliable way to escalate a minor alert into an outage.

Mark any state-changing step visibly and put its rollback immediately beneath it.

*Fails when* steps describe rather than instruct — "check the database" instead of the query and the number that means trouble. Also fails when it explains mechanism inside the procedure. Why the queue grows is a different document; link it at the bottom.

### Validation

Two or more independent observations, each with a command and an expected value, plus an **observation window**.

The subtle failure here is a single metric that moves for the wrong reason. Connection count falling could mean recovery, or could mean the application stopped serving traffic altogether. Pair a resource check with a user-facing check so the two cannot both be satisfied by the wrong outcome.

**Exit criteria** state the specific condition that means done. Without it the operator stops too early, or keeps intervening long after the system recovered.

### Rollback

Beside every destructive step, and as a section for the procedure as a whole where the whole thing may need undoing.

State what the rollback **does not** restore. Where recreating a resource clears its history, the rollback restores the resource but not the history — and the operator needs to know that before they start, not after.

For high-risk procedures, write the **rollback of the rollback**. The alternative is inventing it during an incident.

### If this does not work

Escalation with a clock, a route, and the evidence to attach. The clock matters because an operator out of steps with no instruction will improvise at the worst possible hour. The evidence list matters because it stops the next responder starting from zero.

Tie this to the existing escalation policy rather than inventing a competing one inside the document.

### Related runbooks

Where the branches went. Name them, with a line on when to use that one instead of this one. A reader who picked the wrong runbook should be able to tell from this section.

*Fails when* the entries describe a procedure instead of naming it — a reader cannot search for "the one about mapping drift". Name it `<service>-<subject>-<action>` so the reference resolves to a directory.

*Also fails when* an entry carries a note on its own status: *not yet written*, *coming soon*, *this collection has no runbook for X*. Give the name and the triggering condition and nothing else. Status commentary makes a published document read as unfinished, goes stale the moment the sibling lands, and leaves the reader holding an absence rather than a destination. A forward reference to a runbook someone has yet to write is still the right pointer, and it resolves on the day it is added. Gaps are reported to the collection's owner, not published inside the artifact.

### Verification status

The class of source this came from, and which steps were replicated versus constructed from documentation. An operator deciding whether to trust step 4 at 03:00 needs to know whether anyone has run it.

*Fails when* it is used to disclose an inference. A step that no source supports is a reason to withdraw the runbook, not a line in this section — see the gate in `SKILL.md`. The two admissible values are `replicated` and `documented`.

**Not established** is the one place a negative result belongs: what the sources could not settle, and any circulated method that turns out not to work.

## The standing disclaimer

The template carries a **Before you use this** block, and it stays in the published document unedited. A runbook shared publicly is read by operators whose environment, version and risk tolerance you have no visibility into, so the block does the work you cannot do for them:

- **Confirm against current documentation or a support channel first.** The runbook declares `applies_to_versions`; behaviour moves with versions, and the documentation is the current statement where the two disagree.
- **Exercise it outside production first.** On a target the reader can afford to break, confirming each step's stated output before the procedure matters.
- **As-is, with no warranty and no liability.** Each step changes a real system, and the decision to run it belongs to the reader.
- **Send corrections back.** A step that failed, output that differed, a version where the behaviour changed, or a clearer command are the most valuable contributions a reader can make — more valuable than a clean document that is quietly wrong. Name the channel so the next operator starts from what the last one learned.

The last point is what keeps a published runbook alive. A document with no return path decays in private; one with a return path accumulates the corrections that keep it true.

## Worked example

Condensed, to show the shape rather than to be copied verbatim.

> **Trigger.** Alert `cert-expiry-www`, threshold 7 days remaining.
>
> **Impact.** None yet. At expiry, every visitor sees a full-page browser security warning and the site is effectively down. Severity 3 while days remain; severity 1 the moment it expires. Urgency: business hours above 72 hours remaining, immediate under 24.
>
> **Step 1 — confirm the alert is real.**
> ```
> echo | openssl s_client -servername www.example.com -connect www.example.com:443 2>/dev/null \
>   | openssl x509 -noout -enddate
> ```
> **Expect:** `notAfter=` within 7 days. If more than 7 days out, the alert is stale — close it and check the monitor's clock. **Stop here.**
>
> **Step 2 — identify the issuer.**
> ```
> kubectl -n prod get certificate www-tls -o jsonpath='{.spec.issuerRef.name}{"\n"}'
> ```
> `letsencrypt-prod` → step 3. Anything else → this certificate is manual, go to **Escalate**.
>
> **Step 3 — force renewal.**
> ```
> kubectl -n prod delete certificaterequest -l cert-manager.io/certificate-name=www-tls
> ```
> **Rollback:** none needed. The existing certificate keeps serving until a new one is issued successfully.
>
> **Validation.** Within 5 minutes, `kubectl -n prod get certificate www-tls` shows `READY=True` and `NOT AFTER` ~90 days out. Then re-run the step 1 command against the live endpoint and confirm the new date is being served. Close the alert.
>
> **Escalate.** If `READY` is not `True` 20 minutes after step 3, or the certificate is manually issued, or under 24 hours remain at any point: contact the platform owner and post a status notice. Do not wait for expiry to communicate.

Note what the example does: branches on an observed value, states expected output literally, declares the rollback as "none needed" rather than omitting the heading, validates from two angles (resource then live endpoint), and escalates on a timer with permission to stop.

## Splitting and linking

A branch becomes its own runbook when any of these is true:

| Condition | Why it has to leave |
|---|---|
| Different owner | The reader may not have access or authority for that path |
| Different safety profile | A destructive branch needs its own pre-flight and rollback |
| Needs its own pre-flight | The main pre-flight would have to cover conditions most readers never hit |
| Cause is unknown | That is a playbook — investigation, not execution (OPS07-BP04) |
| Only the vendor can act | That is an escalation step, with evidence to attach |

Keep the main path short and linear, with branches hanging off it by name. Nesting a long branch inline is what turns a two-minute document into a ten-minute one.

## Quantifying impact

**Mechanism as an invariant, magnitude as a command the operator runs.** That split is what lets a runbook quantify impact without carrying a figure that decays. `durable-content.md` has the full treatment: the four classes of number, what to do about something nobody measured, and the before/after pairs.

One thing worth adding here, because it is a choice made while drafting: the dimensions worth measuring, in rough order of usefulness. Volume within a bounded window. Rate against a baseline the operator establishes. Scope counted at execution time. Whether the condition recurs, and on what cycle. Whether the affected data can be replayed or is lost. Cost or objective exposure, where a figure with an owner exists.

## Before you hand it over

- [ ] **Every step traces to a replication or a reputable source.** Anything resting on inference is removed, and the runbook is withdrawn if that leaves a gap in the path.
- [ ] **The "Before you use this" disclaimer is present and unedited** — verify against documentation, test outside production, as-is with no liability, send corrections back.
- [ ] **Name is `<service>-<subject>-<action>`,** lowercase and hyphenated, matching the directory name and the frontmatter `name` exactly, and consistent with the names already in the collection.
- [ ] **Siblings are cross-referenced by name,** not by an invented identifier, and with no note on whether they exist yet.
- [ ] **Cold-read test.** Could someone who has never done this complete it from the document alone?
- [ ] **Step 1 confirms the problem is real,** and says what to do if it is not.
- [ ] **Impact is quantified,** or explicitly labelled unmeasured with the measurement named.
- [ ] **Every command is exact,** in a code block, with literal expected output.
- [ ] **Every state-changing step has its rollback beside it,** and states what the rollback does not restore.
- [ ] **Exit criteria are an observation,** not "should recover."
- [ ] **Validation checks two independent things,** so one metric moving for the wrong reason cannot fake success.
- [ ] **Escalation has a clock, a route, and an evidence list.**
- [ ] **Branches are explicit,** and the ones that left are linked by name.
- [ ] **Every command is executable by any reader of the published document** — no internal tooling, no internal dashboards.
- [ ] **Permissions appear in the pre-flight,** not as an assumption.
- [ ] **`read_only` matches the procedure.** If `true`, no step writes and there is no rollback section; if `false`, every state-changing step is marked and has its rollback beside it.
- [ ] **`metadata.verification` is `replicated` or `documented`,** and matches what the Verification status section says.
- [ ] **Verification status names the source class** and says which steps are replicated and which are documented.
- [ ] **Nothing internal leaked** — no unconfirmed causes stated as fact, no internal ticket IDs as instructions, no service-team disagreements.

## Maturity — when a runbook should become a script

AWS Well-Architected OPS07-BP03 recommends automating runbooks over time, starting with the short and frequently used. The useful progression:

- **Manual** — human reads and executes. Correct for anything with real decision points.
- **Semi-automated** — a script does the mechanical parts, the human reviews output and decides at the branches. This is the right target for most high-frequency runbooks: it removes error-prone retyping while keeping judgement where it belongs.
- **Fully automated** — event triggers the procedure end to end. Only earned after watching the semi-automated version run cleanly and repeatedly.

A runbook that has been executed many times with no remaining decision points is a script waiting to be written. Keep the human-readable document either way — a button that runs the procedure is excellent until the button stops working.

## Sources

Practitioner consensus on the section shape, the cold-read test, confirming the symptom first, rollback adjacency, exit criteria and timed escalation is drawn from current runbook-writing guidance (Raccoon Page, *How to write a runbook your on-call will actually use*, 2026-05; On Call Journal, *Runbook Template for Production Support Teams*, 2026-09; Pulsetic, *Runbooks: How to Write One That Works at 3am*; Webalert, *Incident Runbook Template*, 2026-03; AIOps SRE, *SRE Runbook Template*).

Authoritative anchors:

- **AWS Well-Architected OPS07-BP03 — Use runbooks to perform procedures.** A runbook is a documented process to achieve a specific outcome; it must include guidance on error handling, tools, permissions, exceptions and escalations; it is published centrally and updated as part of change management; automate the short and frequent first.
- **AWS Well-Architected OPS07-BP04 — Use playbooks to investigate issues.** Playbooks establish an unknown root cause; known root causes get runbooks. This is the basis for the branch-out rule.
- **AWS Well-Architected OPS10-BP02 — Have a process per alert.** An alert without a matching procedure is a noise generator.
- **AWS Well-Architected OPS02-BP02 — Processes and procedures have identified owners.**
- **Google SRE Workbook, *Being On-Call*.** Every alert should have a corresponding playbook entry; playbooks reduce stress, MTTR and the risk of human error; a playbook that is a deterministic list of commands is a candidate for automation.
