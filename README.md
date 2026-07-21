# powerbi-project-editor

A battle-tested [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills)
for **programmatic Power BI editing** — working on PBIP folders (TMDL semantic models +
PBIR report definitions) outside Power BI Desktop.

Every rule in this skill was earned on real production models: model trims and full
rewrites, a 31→19-table folding campaign, DAX↔M ports verified row-by-row against
92k-row snapshots, silent frontend regressions caught and root-caused. Nothing here is
theory — each Iron Rule broke a real model first.

## What it teaches Claude

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

## Install (one command)

**Windows (PowerShell):**

```powershell
git clone https://github.com/GoCloudToday/powerbi-project-editor "$env:USERPROFILE\.claude\skills\powerbi-project-editor"
```

**macOS / Linux:**

```bash
git clone https://github.com/GoCloudToday/powerbi-project-editor ~/.claude/skills/powerbi-project-editor
```

That's it. Claude Code discovers user-level skills automatically; the next session will
load it whenever you work on PBIP/TMDL/PBIR files, or invoke it explicitly with
`/powerbi-project-editor`.

## Update

```powershell
git -C "$env:USERPROFILE\.claude\skills\powerbi-project-editor" pull
```

(macOS/Linux: `git -C ~/.claude/skills/powerbi-project-editor pull`)

## Versioning

Semantic tags on `main`: **minor** bump (`v1.1.0`) for each batch of new learnings or
rule changes, **patch** for typo/sanitization fixes, **major** reserved for breaking
restructures of the skill contract. `git tag -l` for history; pin a machine to a
version with `git checkout vX.Y.Z` if you need reproducibility over freshness.

## Contents

| File | Purpose |
|---|---|
| `SKILL.md` | The contract Claude loads: 10 Iron Rules, quick-reference table, red flags, self-iteration protocol |
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
