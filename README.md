# powerbi-project-editor

A battle-tested [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills)
for **programmatic Power BI work**: PBIP folders (TMDL semantic models + PBIR report
definitions) edited outside Power BI Desktop, verified against the engine, and published.

**Since v2.0.0 it is an overlay.** Microsoft's open-source
[skills for Fabric](https://github.com/microsoft/skills-for-fabric) (the `powerbi-authoring`
plugin: `semantic-model-authoring` and `powerbi-report-cli`) is the core for workflows,
tooling and greenfield guidance. This skill adds what was earned on real production models,
and where the two disagree the overlay wins; every override cites what broke.

> **Requirement:** Microsoft's `powerbi-authoring` plugin must be installed (see Install,
> steps 1 and 2). This skill is an overlay on it, not a replacement. When the plugin is
> missing, Claude stops and asks you to install it, and goes on with the overlay alone only
> if you say so.

Every rule in this skill was earned on real production models: model trims and full
rewrites, a 31→19-table folding campaign, DAX↔M ports verified row-by-row against
92k-row snapshots, silent frontend regressions caught and root-caused. Nothing here is
theory — each Iron Rule broke a real model first.

## What it adds on top of the core

- **Overrides of core defaults** that break existing models or published items: live MCP
  writes to published models, delete-before-deploy, M step renaming, Desktop preview next to
  a Desktop in use, the blanket refusal of role-membership changes (see SKILL.md)
- **Safe TMDL/PBIR surgery** — cache-key semantics of M partitions, transitive
  invalidation, deletion closures, relationship/role/culture bookkeeping
- **Model folding** — a refresh-cost-ordered ladder for shrinking models
  (free folds → one-refresh folds → the calc-table-to-M unlock)
- **DAX↔M translation fidelity** — the comparison-semantics traps (case, trailing
  spaces, blank-key lookups, coercion) and the snapshot acceptance test that catches them
- **Frontend fidelity** — the string-attached PBIR surface (conditional formatting,
  column widths, selectors) that fails silently when renames miss it
- **M performance discipline** — buffer-once evaluation, hash joins vs nested-loop
  degradation, comparer-lambda costs
- **Verification** — engine-backed dependency graphs, numeric fingerprints,
  captured-query perf replay, headless Desktop cycles, offline TMDL validation via TOM

## Install

**1. Prerequisites.** Node.js 20+ and the Azure CLI on PATH: the core's report CLI and its
modeling MCP server run on Node, and its Fabric calls use `az rest` after `az login`.

**2. The core (Microsoft), required,** pinned to the version this overlay was tested against:

```bash
claude plugin marketplace add 'microsoft/skills-for-fabric#v0.3.23'
claude plugin install powerbi-authoring@fabric-collection
npm install -g @microsoft/powerbi-report-authoring-cli
```

(Inside a session, `/plugin marketplace add` and `/plugin install` take the same arguments.)

**3. This overlay.**

Windows (PowerShell):

```powershell
git clone https://github.com/GoCloudToday/powerbi-project-editor "$env:USERPROFILE\.claude\skills\powerbi-project-editor"
```

macOS / Linux:

```bash
git clone https://github.com/GoCloudToday/powerbi-project-editor ~/.claude/skills/powerbi-project-editor
```

**4. Load the overlay with the core.** Claude Code has no skill-to-skill dependency, and the
core's skills trigger on their own, so add this line to `~/.claude/CLAUDE.md`:

```text
When a powerbi-authoring skill (semantic-model-authoring or powerbi-report-cli) is used, also load the powerbi-project-editor skill and apply its overrides; they win on conflict.
```

The overlay requires the core. If it is missing, Claude stops and asks you to install it.

## Update

The overlay:

```powershell
git -C "$env:USERPROFILE\.claude\skills\powerbi-project-editor" pull
```

(macOS/Linux: `git -C ~/.claude/skills/powerbi-project-editor pull`)

The core: move the pin deliberately. Run the core update check in SKILL.md (self-iteration
step 8) first, then add the marketplace again at the new tag and update the plugin.

## Versioning

Semantic tags on `main`: **minor** bump (`v1.1.0`) for each batch of new learnings or
rule changes, **patch** for typo/sanitization fixes, **major** reserved for breaking
restructures of the skill contract (v2.0.0: the overlay restructure). `git tag -l` for
history; pin a machine to a version with `git checkout vX.Y.Z` if you need
reproducibility over freshness.

## Contents

| File | Purpose |
|---|---|
| `SKILL.md` | The contract Claude loads: load order, overrides of the core, environment notes, 10 Iron Rules, red flags, situation index, self-iteration protocol |
| `references/*.md` | Situation rows by topic: model surgery, verification, reports (PBIR), row-level security, writeback pages, sources and Power Query, DAX logic, service and publishing, Desktop and harness |
| `reference.md` | Deep procedures: parsing recipes, dependency layers, verification stack, engine ops, playbooks, dated evidence log |

## Philosophy

The skill follows a strict self-iteration protocol: every rule cites what was measured
or what broke; "I think" entries are forbidden; edits are probe-tested on fresh agents
before being trusted. If you extend it, keep that bar — and keep entries generalized
(no client names, tenants, or business figures; see protocol step 5).

## License

MIT — see [LICENSE](LICENSE).

Power BI, Power Query, and related marks are trademarks of Microsoft Corporation.
This project is not affiliated with or endorsed by Microsoft.
