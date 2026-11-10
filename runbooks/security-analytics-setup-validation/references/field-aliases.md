# Layer 3 — Field aliases

Rules reference field names that do not exist in the log. A field alias is what connects the two, and where it is declared decides whether it survives.

## The recommended state

**A field alias for every rule field name, declared in your own index template**, alongside the base field it points at.

Two surfaces can hold these aliases, and the difference is the whole of this layer:

| Surface | Lifetime |
|---|---|
| `${COMPONENT}` — the plugin-managed component template | **Replaced on every detector create or field-mappings save**, and rebuilt from the aliases resolvable against **one** index at that moment |
| `${TEMPLATE}` — your own index template | Yours. Applied after `composed_of`, so its entries win |

Declaring them yourself is the documented recommendation, and it is what makes a new index independent of which template the component landed on and of what the component happens to contain at the time.

**Every alias needs its target base field declared in the same template.** An alias's path must exist when the index is created, and a field that would arrive by dynamic mapping does not exist at creation. This is why layer 1 pins base fields: the two halves are one change.

## The reads

```
GET /_index_template/${TEMPLATE}
GET /_component_template/*alias-mappings*
GET /_plugins/_security_analytics/mappings?index_name=${INDEX}
```

> ⚠️ **Prove the component-template read before interpreting it.** A component template listing no aliases and a caller who cannot read component templates render identically. Confirm at least one alias's component comes back carrying entries before treating another's empty one as a finding. Without that control, record this layer as **not evaluated**.

> ⚠️ **`GET` on the mappings route reads. `POST` to the same path writes** — it rewrites `${COMPONENT}` from whichever index it resolves. Use `GET` only.

**Then check the alias set against the rule field names**, per member index rather than only the write index. The write index is the one most likely to be correct, because a recent save resolved against it; members that predate the save are where gaps sit.

## What conformance looks like

- Every rule field name the detector's enabled rules reference appears as a `"type": "alias"` entry in `${TEMPLATE}`, with a `path` to a base field also declared there.
- `${COMPONENT}` is still listed in `${TEMPLATE}`'s `composed_of`. **Leaving it there is correct** — removing it is not the fix, and the documentation says not to remove the entry.
- The same alias names are present on each member index.

A conformant setup is one where **the component template's contents no longer matter.** That is the test: if emptying it would change what a new index carries, this layer is non-conformant.

## The save that looks destructive and is not

A detector save rewrites `${COMPONENT}` from the aliases present on one index, and can leave it with fewer entries than before, or empty. Against a conformant setup that is harmless, because your template supplies every alias regardless. Against a non-conformant one it is the mechanism by which coverage disappears at the next rollover.

So the same event is benign or damaging depending only on this layer — which is why the reference treats declaring the aliases yourself as the durable fix rather than as duplication.

## What non-conformance costs

| Finding | Cost | Route |
|---|---|---|
| Aliases exist only in `${COMPONENT}` | Coverage depends on the last save. The next save, or a save by a second detector on the same alias, can shrink the set; the following rollover drops those rules silently | `security-analytics-field-alias-template-recovery` |
| An alias in `${TEMPLATE}` whose target base field is not declared there | The alias is not created, because its path does not exist at index creation | Layer 1 — the two are one change |
| `${COMPONENT}` is empty or short against the rules' field set | A flag, not a diagnosis: an alias legitimately absent because its target is absent renders the same as one dropped by a replacing save. The per-index read separates them | `security-analytics-field-alias-template-recovery` |
| `${COMPONENT}` listed on a template that does not create the member indices | New indices carry none of its aliases | `security-analytics-field-alias-template-recovery` |
| Alias names present on the write index but absent from an older member | Rules match on the write index and not on the others; which members a detector reads is unsettled, so treat it as a gap | `security-analytics-detector-mapping-audit` |

**A rule whose field is missing is skipped, not failed.** No error, no alert, and the detector reports healthy. That is the documented behaviour, and it is why this layer cannot be inferred from the absence of complaints.

## Adding a rule later

The reference states the sequencing plainly: **when a new rule needs a field, add the field and its alias to the template before enabling the rule.** Enabling first produces a rule that is silently skipped until the next rollover after the template is corrected.

## Sources

- **Published managed-service documentation** for the core of this layer: that the plugin creates a component template and adds it to the `composed_of` list of the index template matching your indexes; that the entry should not be removed; that mappings under `template.mappings` are applied **after** `composed_of` so your declarations win; that a field alias should be included for every field name an enabled rule references, so that a new index never depends on the plugin-managed component template; and that **rules whose fields are missing from the index are skipped without an error being reported**.
- **Published documentation** for template merge order and for field aliases requiring an existing target path.
- **Code analysis of the published Security Analytics plugin** for the component template being replaced rather than merged: it is built from the request's mappings resolved against a single concrete index — the write index, or the newest backing index — and written without the existing one being read back. A public change to merge instead of replace was opened and **closed without being merged**, so this behaviour is present on every line.
- **A reference setup produced during a support engagement** for the worked alias set and the sequencing rule about adding a field before enabling a rule, and for the observation that a post-cutover detector save may empty the component template without consequence once the aliases are declared in the operator's own template. **Engagement-sourced.**

**Not reconciled in the source material.** Whether to declare the aliases in your own template was answered three different ways across the engagement that produced the reference — *no, leave them to the component*, then *yes, both*, then *optional*. The customer noticed and asked for a ruling; none was given. **This layer follows the published documentation**, which states the case for declaring them yourself, and treats the disagreement as closed by that rather than by the engagement record.
