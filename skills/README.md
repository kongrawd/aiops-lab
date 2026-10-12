# skills/

Agent skills published from this repository. Each directory is a self-contained skill with `SKILL.md` at its root, loadable on its own.

## Install

```bash
SKILL_BASE_URL=https://github.com/kongrawd/aiops-lab/tree/main npx skill skills/operational-runbook
```

## What a skill here looks like

```bash
<skill-name>/
├── SKILL.md         # frontmatter (name, description) plus the instructions
├── assets/          # templates and files that end up in the output
└── references/      # detail loaded only when needed
```

`name` and `description` are what an agent reads to decide whether the skill applies, so they carry the triggering conditions. `SKILL.md` stays short enough to load on every invocation; anything longer lives under `references/` and is read on demand.

## Contributing

Corrections are more valuable than additions: a step that failed, output that differed, a version where behaviour changed, or a clearer command. Open an issue or a pull request.

Two rules govern anything merged here, and both exist because these documents get executed against production systems:

- **Every step traces to a replication or a reputable source.** Vendor documentation, an API reference, an upstream change record or a published standard. A step resting on inference is a reason to leave a procedure unwritten rather than to ship it with a caveat.
- **Nothing identifying goes in a published file.** No person or team, no case or ticket identifier, no internal URL, account, domain, cluster or hostname, and no measurement taken from one environment. Where a value is needed, the document carries the command that produces it.
