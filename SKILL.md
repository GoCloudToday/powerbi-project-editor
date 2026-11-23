---
name: powerbi-project-editor
description: Field-earned overlay for any Power BI semantic model or report work (PBIP, TMDL, PBIR, DAX, M, Gen1 dataflows, RLS, publishing, verification). Use it together with, and on top of, Microsoft's powerbi-authoring skills (semantic-model-authoring, powerbi-report-cli), which are the core; where they disagree this overlay wins. Covers safe edits to existing models (M cache invalidation, deletion closures, stale Desktop sessions), engine-backed verification (msmdsrv, Invoke-ASCmd, DISCOVER_CALC_DEPENDENCY, cache.abf, captured-query replay), report binding fidelity, DAX-to-M ports, RLS designs, writeback pages and publish traps.
---

# Power BI Project Editor (overlay on the Microsoft core)

Operational rules for programmatic Power BI work, earned through verified failures. Since v2.0.0
this skill is an overlay: Microsoft's `powerbi-authoring` plugin from skills for Fabric
(`microsoft/skills-for-fabric`) is the core for workflows, tooling and greenfield guidance, and
this skill adds the rules that kept real models alive. Situation rows by topic:
[references/](references/) (index at the end). Deep procedures, DMV queries and the dated
evidence log: [reference.md](reference.md). A companion toolchain (extractor, inventory,
engine-backed keep/kill graph, gate, deleters, fingerprint suite) is worth carrying per repo
under `tools/` — if present, adapt it; the recipes in reference.md are sufficient to rebuild it.

## Load order

1. **Load the core skill for the task** and follow its routing: model, DAX, refresh, deploy and
   connection work → `powerbi-authoring:semantic-model-authoring`; report planning, design,
   PBIR pages and visuals, preview and publish → `powerbi-authoring:powerbi-report-cli`.
2. **Apply this overlay on top:** the overrides, the environment notes, the Iron Rules and the
   topic file whose situation matches (index at the end).
3. **On conflict this overlay wins.** Each override cites a verified failure or a documented
   product limit.
4. **Core not in the skill list:** check `claude plugin details powerbi-authoring@fabric-collection`
   (a cache folder alone can be a leftover). Installed but not loaded yet (plugins load at session
   start): read the core skill from the plugin cache
   (`~/.claude/plugins/cache/fabric-collection/powerbi-authoring/<version>/skills/<skill>/SKILL.md`)
   and follow it from there. If the plugin is not installed, this skill still works alone, as it
   did before v2.0.0; say once that the core is missing and point to the README install steps.

Tested against core **0.3.23** (`microsoft/skills-for-fabric`, tag `v0.3.23`). The core ships
weekly and has renamed skills before. If the installed version differs (`claude plugin details
powerbi-authoring@fabric-collection`, or the version folder under
`~/.claude/plugins/cache/fabric-collection/powerbi-authoring/`), finish the user's task with the
overrides as written, tell the user the core moved, and run the core update check
(self-iteration step 8) as separate maintenance.

## Overrides of the core (this overlay wins)

| Core default (0.3.23, where) | Overlay rule | Why |
|---|---|---|
| semantic-model-authoring, Tool Selection Priority, Tier 1: `powerbi-modeling-mcp` edits whatever model it connects to, a model published in a workspace included, with immediate effect | MCP **writes** go only to a local Desktop model or a PBIP. A published model gets read-only use (List/Get, DAX queries, a query-scoped `DEFINE MEASURE` to try a fix); change it with `updateDefinition` of a reviewed local definition. A live write only on the owner's explicit request, after telling them both side effects, with the same change put into the PBIP | After an XMLA write the service no longer offers the model as a PBIX download (reference.md §2026-10-23 round 2), and the next Desktop publish of the project replaces the model, live change included (references/service-publish.md, "A model you published comes back with your change reverted") |
| Same section: never read TMDL files while the MCP is connected; Must/Prefer/Avoid: hand-authoring TMDL is an anti-pattern once the MCP is registered | When Desktop or the MCP has saved, the files on disk are the truth for gates, diffs and audits. Diff the definition folder after every such save and revert rewrites nobody asked for, partition M first (Iron Rules 1 and 2). Hand-authored TMDL stays allowed behind the offline deserializer gate (references/model-surgery.md, "Scaffolding a NEW PBIP from scratch" for the gate, "Hand-authoring TMDL / DAX from scratch" for what it misses) | Every gate here (dependency closure, selector audit, byte identity) reads files. Desktop rewrites files on its first save (fenced M to plain `source =`, `active: true` added), so a save can carry changes nobody asked for |
| Workflow: Deploy to Fabric, "Deploy exactly once": "If deploying a model that may already exist, delete the existing same-named model(s) before the single deploy" | **Never delete a published model to redeploy it.** Resolve its item id and `updateDefinition` in place (the core's own branch for files on disk). Two items with one name: stop and ask | Deleting the item drops its id with everything attached: bound reports, credentials, refresh schedule, role members, permissions |
| tmdl-guidelines.md, "Task: Setting Descriptions in Power Query / M Code": comment each step and rename the steps | Comments count as M text. Only on queries whose tables get a real source refresh in the same change, reference closure included (Iron Rule 1). Without a planned refresh, leave the M untouched and put the explanation outside the partition text | Any change to partition M text drops that table's data on the next open, transitively; only a source refresh restores it |
| tmdl-guidelines.md, TMDL Syntax Rules and "Task: Creating Measures in TMDL": measures before columns, `///` descriptions, `formatString` on every measure | Apply them to the object you add. In an existing file touch only the lines the change needs (insert the new object, delete what was asked): no reordering, no reformatting, no tidying of other objects (Iron Rule 2) | Reordering turns a one-line change into a whole-file diff and breaks anchor-based patch scripts (references/model-surgery.md, "Generator / patch scripts on TMDL") |
| tmdl-guidelines.md, Format String Reference: scaling strings such as `#,##0,,M` and `#,##0,K` | Let exactly one side scale. Default: display units on the visual and a plain `#,##0` on the measure; a scaling string on the measure only when every visual that shows it has display units off. Before changing a measure's format string, list the visuals that use it in every report on the model | A scaling format string on a card with Auto display units rendered `42.5,,MM` (reference.md §2026-09-20; references/report-pbir.md, "A number renders as `42.5,,MM`") |
| semantic-model-authoring, Must/Prefer/Avoid, DENY: never change RLS/OLS role membership | Allowed only when the model owner explicitly asks. Prefer the model's Security page in the service (no XMLA write, the PBIX download stays). TOM over the XMLA endpoint only when the owner accepts losing the PBIX download, and not on a model with Query Scale-Out. Either way refuse a role without table permissions, and run the impersonation gate after every later publish (references/security-rls.md) | Membership grants data access, so it is the owner's decision rather than a blanket refusal. An XMLA role write ends the PBIX download, and members of a role without table permissions see everything (reference.md §2026-10-23 round 2); a Query Scale-Out model refused the XMLA save while its Security page worked (reference.md §2026-11-23 round 28) |
| powerbi-report-cli, authoring.md completion gate (validate, load, screenshot every affected page, review) and management-part-03.md: PowerShell part walking "is not a supported executable workflow" | Use the CLI whenever it is installed. Without it, the PowerShell route and gates in references/report-pbir.md are the fallback; say which route ran | The CLI catches schema, property and encoding errors; it does not catch selector drift, lost bindings or bookmark state (Iron Rule 8) |
| powerbi-report-cli, authoring/preview.md: the Desktop host opens or attaches the Desktop instance for the folder | If any other Desktop window is open, idle or not, do not start Desktop: ask the user to close it, hand the open to the user, wait, or use the CLI's service host (renders the local PBIR against the published model, with the workspace and model ids the user gives, and publishes nothing). The "if you must share the machine" steps in references/harness-desktop.md limit damage; they are not permission | Every Desktop window shares one WebView2 data folder; a second instance left the other window showing a WebView2 error over a black canvas (reference.md §2026-10-23 round 7: don't run the cycle next to another Desktop at all) |
| README Authentication and the auth recipes: `az login`, service preview and MCP sign-ins ride browser single sign-on | Confirm account and tenant before the first call (`az account show`, the page's account menu) | Single sign-on picks the device's default account, which can belong to another tenant (reference.md §2026-11-23 round 28) |
| powerbi-report-cli, design.md: tone, signature, archetypes | The owner's or the project's design rules (palette, wording, simplicity) override the core's defaults | Owners asked for the least busy version, without decorative labels (reference.md §2026-11-23 round 28) |

## Environment notes

- **Tools the core expects:** Node.js 20+ for the `powerbi-report-author` CLI
  (`npm install -g @microsoft/powerbi-report-authoring-cli`) and the `powerbi-modeling-mcp`
  server (run through `npx` by the plugin), and the Azure CLI for every Fabric call (`az rest`).
  Without admin rights, install both per user: the Node.js zip from nodejs.org (check its
  SHA-256 against `SHASUMS256.txt`) and the Azure CLI zip package, each added to the user PATH;
  restart the editor so the agent's environment sees the new PATH.
- **Sign-in:** where conditional access blocks the device-code flow,
  `az config set core.enable_broker_on_windows=false`, then
  `az login --tenant <tenant id> --allow-no-subscriptions` signs in through the browser.
  Without the Azure CLI, get the token with Az PowerShell
  (`Get-AzAccessToken -ResourceUrl https://api.fabric.microsoft.com`, WAM off) and send the
  core's request with `Invoke-RestMethod`: same URL, body and headers. Keep sign-in and write
  calls in separate commands.
- **The core's attribution header** (`x-ms-fabric-skill: <skill name>` on every
  `api.fabric.microsoft.com` call) is a core rule; keep it when translating a call to PowerShell.
- **Windows PowerShell 5.1 harnesses** (ADOMD, AMO) and Desktop process handling:
  references/harness-desktop.md.

## The Iron Rules (each one broke a real model)

1. **M partition text is a cache key.** Any content change to a partition's M —
   even removing a blank line — makes Desktop drop that table's data on next open.
   Recalc cannot restore it; only a real source refresh can. Line-ending changes and
   column/measure block deletions do NOT invalidate. Edit M only with a refresh scheduled. Invalidation is TRANSITIVE through the
   query-reference closure (consumers of a changed query drop too, even with their own
   text unchanged).
2. **Deletion-only edits.** Never reformat, collapse blank lines, or "tidy" TMDL you
   aren't explicitly asked to change. Read files with `newline=''` in Python — default
   universal newlines silently strips CRLF and your "CRLF preservation" check always
   sees LF (Desktop writes CRLF).
3. **Desktop never sees disk edits until reopened, and saving from a stale session
   overwrites them.** Close (or discard) Desktop before editing; reopen after; never
   let anyone save from a pre-edit session.
4. **A table with no references can still be load-bearing.** Three invisible ways:
   filter-propagation **bridge** between two queried tables (filters flow dim→fact
   through it); **M-consumed** by another query (`Table.NestedJoin` etc. — DAX analysis
   is blind to this); shared-expression dependency. Removal safety is computed
   (see reference.md §Dependency layers), never eyeballed.
5. **Delete the dead set whole, to fixpoint.** Partial deletion strands survivors
   (dead columns referencing deleted measures; sort-by targets whose parent died).
   Re-run the closure until nothing new dies.
6. **Value verification cannot see plan regressions.** `isAvailableInMdx: false` on
   257 columns left every measure value identical and made a "Show items with no data"
   visual go 2s → 150s timeout. Perf acceptance = replay the CAPTURED Desktop query
   (`$SYSTEM.DISCOVER_COMMANDS` while the visual renders) verbatim, A/B.
7. **Checkers must not share code with the thing they check**, and must be calibrated
   on THREE cases, not two: the gate must FAIL a known-bad input, PASS a known-good
   model, and FAIL when a lookup it depends on MISSES. A missing key is the sneaky
   third state: the loop under it walks nothing, so the check is vacuous rather than
   wrong and it prints like a pass (a bookmark whose section key did not match the
   page reported `0 hidden of 0`; an expected-value table with no entry for the
   actual type printed a clean success line for a pairing it never compared). Every
   lookup miss must be reported as UNVERIFIED, and a summary line must say UNVERIFIED
   rather than 0. Never chain gate->apply unconditionally.
8. **The frontend attaches by strings, and fails silently.** Conditional formatting
   (RAG icons, backColor/fontColor, webURL), columnWidth, alignment and bars-only
   settings bind via `selector: {metadata: "Table.Object"}` STRINGS that must equal
   the visual's own projection queryRef (a per cell rule also needs `data: [{dataViewWildcard:
   {matchingOption: 1}}]` beside it, or it colours the column as a whole). A rename/rebind that updates projections but
   not selectors produces ZERO errors — dots and widths just vanish, hidden numbers
   reappear. Selectors must match each visual's ACTUAL projection strings, not the
   rename map globally (rebind tools can leave aggregation labels like
   `CountNonNull(Old.Col)` untranslated — then the selector must stay old too).
   Audit: every live selector metadata ∈ that visual's projection set, trying
   old/new/aggregation-wrapped variants. Full surface: reference.md §PBIR fidelity.
9. **Translating DAX to M silently changes comparison semantics.** DAX text equality
   is case-insensitive AND trailing-space-insensitive (one missed Trim dropped 17k
   lookup matches); `IN`/`CONTAINSSTRING` are case-insensitive; `LOOKUPVALUE` with a
   BLANK key MATCHES blank-key rows (which may itself be a latent bug feeding garbage —
   verify which side is wrong before "fixing" the new one); numbers coerce to text.
   Faithful M keys = `Text.Upper(Text.Trim(x ?? ""))` on BOTH sides. Acceptance for
   any engine→M port = full-table snapshot diff (reference.md §DAX↔M fidelity).
10. **M evaluates what you reference, as many times as you reference it — and joins
   have algorithms.** Buffer each source exactly once (`Table.Buffer`): unbuffered
   multi-reference re-downloads the source per reference. But `Table.NestedJoin` over
   a buffered side degrades to a NESTED LOOP (92k×92k ≈ hours; signature: slow,
   CPU-busy, memory-healthy) — use `Table.Join(..., JoinAlgorithm.LeftHash)` with
   `Table.Distinct` on the lookup key (Table.Join multiplies rows on duplicate keys;
   Distinct restores single-match semantics). Per-row comparer lambdas
   (`List.Contains(xs, x, ci)` × 92k rows ≈ 70M calls) → pre-normalize once into
   buffered ordinal lists. Big-entity aggregation stays DAX calc columns over loaded
   tables (compressed engine scan); M gets row-local transforms and small joins.
   **Never touch a NestedJoin's nested column per row before `ExpandTableColumn`**
   (`Table.RowCount([Nested])`, `Table.IsEmpty`, `[Nested]{0}`…): each row then
   re-evaluates the whole right side including its source reads — measured 12,382
   downloads of each dataflow CSV for a 19k-row dimension, 3.5 h, machine crash. For
   a "matched" flag, add a constant `each true` column to the RIGHT table and expand
   it (unmatched → null). reference.md §2026-08-28.


## Red flags — stop and re-check

- "I'll just tidy the formatting while I'm here" → Rule 2.
- "Nothing references it, safe to delete" → Rule 4.
- "Values match, we're done" → Rule 6.
- "The gate passed" (in the same unconditional command chain as the apply) → Rule 7.
- "The visual is slow, the model must have regressed" → check engine contention,
  orphaned queries, machine memory, and calc-table recalc-on-open FIRST; then capture
  the real query and A/B it (reference.md §Diagnosing slow visuals).
- "The formatting rules are still in the JSON, so the look is fine" → Rule 8: selectors
  that match nothing no-op silently. Compare against a rendered screenshot.
- "Refresh is slow but memory is healthy, must be the source" → Rule 10: that is the
  nested-loop join signature, not a download problem.
- "My port's value differs from the old model, so my port is wrong" → Rule 9: run the
  direction cross-tab first; the OLD side may be the defect.
- "It refreshed locally to the auth wall, so the user's refresh will pass" → only
  document-level errors reproduce locally; evaluation-time errors (missing columns,
  join keys) surface ONLY on the credentialed machine. Sweep for them statically.
- "The KPI only differs on a handful of days, must be source drift" → if those days sit
  just after partition edges with identical row counts, a per-row lookup inherited the
  partition window (reference.md §2026-09-04 round 2). Month-grain policies freeze it.
- "Parity proven" (by the author alone) → commission the adversarial second-model
  review first; it overturned two of five conclusions in the last run.

## Situation index (open the topic file, read the row)

Each line is the situation column of one row in that file. A situation with no line here
is new: handle it with the core and the Iron Rules, then record it (self-iteration step 2).

**[references/model-surgery.md](references/model-surgery.md)**: Model surgery
- Removing a table
- Relationship endpoints
- Calc-table columns (`isNameInferred`, `sourceColumn: [X]`)
- Kept table, all columns dead
- Broken (non-compiling) measures
- `Stage`/`Team`-like words in M
- "Report uses X"
- Prod safety
- Calc table referencing a related table
- Hand-authoring TMDL / DAX from scratch
- Slimming an M-consumed producer
- Model won't shrink under keep/kill
- Folding tables away
- Converting table → expression
- Partition type change (calc ↔ m)
- ADDING a table to a PBIP model
- Authoring a calc table by hand
- Scaffolding a NEW PBIP from scratch
- Second calculation group
- Hand-adding an object with `queryGroup:`
- Generator / patch scripts on TMDL
- Turning a data column into a calculated column of the SAME name
- "One or more calculated objects need to be manually refreshed" + error icon on an edited calc table

**[references/verification.md](references/verification.md)**: Verification
- Verifying a trim
- Timed-out `Invoke-ASCmd`
- Recalc says "no errors" but partitions not Ready
- Measuring a published model's memory footprint / finding the expensive columns
- Proving a REBOUND report shows the same numbers on a rebuilt model
- Inventory of what a report can no longer show after source columns are lost (placeholders in the model)
- Diffing detail sets pulled through `executeQueries`
- Before writing "parity proven"
- Parity gate over FX-converted measures
- Enumerating a report's binding surfaces / trusting a gate that PASSed
- Gating visuals over REST `executeQueries`
- Reviewing a BUILT artefact with no version control
- Emulating a measure fix with query-scoped `DEFINE MEASURE`

**[references/report-pbir.md](references/report-pbir.md)**: Reports (PBIR)
- Styling a hand-generated report
- Hand-built chart: series colour, constant line, dynamic title
- Card visual looks wrong
- Table/matrix cell colours (conditional formatting)
- Table/matrix draws a horizontal scrollbar
- Matrix shows each row level on its own line (a staircase) although stepped layout is off
- Matrix header labels will not centre, or a numeric row field will not right align
- A table shows only one column and every dimension member, no measures, no error
- Button slicer buttons render without text, or show an extra "--" button
- Cloning a report page onto a parallel model
- "Copy this page but on a different date column"
- Date-hierarchy field on a copied page
- "Filter out X from the report"
- Retargeting a field parameter
- Dynamic multi-line or mixed-color text
- Nudging text inside a hand-built card, or one line of button text sits low
- Hand-authoring a thin report's `byConnection`
- Report stuck on "Loading your report..." after a trim
- Generating PBIR visuals from PowerShell
- Copying a static resource (theme) into a PBIR project
- Report-level (extension) measures on a read-only dataset
- Two views on one page switched by bookmarks
- Generating PBIR from PowerShell (data side)
- A number renders as `42.5,,MM`
- Hand-written bookmarks / page visibility
- Selector that must show the same state from several places (n × k)
- Rebinding measures in a report that has been renamed around before (sharpens Rule 8)
- Renaming a measure or a label across a report
- Porting a report's custom visuals
- Field map moves a field to another table (report transport)
- One table row can't be picked, or one name in a link column isn't underlined
- Column widths and alignment for a report people use in the service

**[references/security-rls.md](references/security-rls.md)**: Row-level security and access levels
- RLS that must follow another model's user-to-country table
- Several access levels (per page, read-only) on one model
- Reading RLS roles off a PUBLISHED model
- "RLS users see less data than write users"
- Adding RLS role members to a PUBLISHED model by code
- `executeQueries` reproducing an RLS identity
- Row-level security in a composite model

**[references/writeback-inputs.md](references/writeback-inputs.md)**: Writeback and input pages
- Translytical input slicer / data-function button
- Input box must look unusable (no row picked, no rights)
- "Global" users see everything but every save is refused
- Button that must look disabled

**[references/data-sources-m.md](references/data-sources-m.md)**: Sources and Power Query
- Porting M Number.Round to SQL
- Porting a calculated column that looks up another table
- Engine→M port acceptance
- Dates load blank or swapped, refresh green
- DirectQuery table or matrix slow with text and colour measures
- Initial load of a model with an incremental-refresh policy
- Import a fact from a PUBLISHED semantic model (certified upstream)
- Fixed the fact query but the numbers did not move
- Rebuild is BIGGER than the model it replaces
- Freezing a retiring source into a one-time Gen1 dataflow
- Downloading a Gen1 dataflow definition
- Authoring a NEW Gen1 dataflow (import model.json) from SQL, no DB access
- Desktop refresh of many dataflow-fed tables fails "A cyclic reference was encountered during evaluation"
- A dataflow entity's aggregate query dies server-side ("A severe error occurred on the current command", ~25 min)
- Blaming the source for a dangling FK
- Importing a Gen1 dataflow fails "Linked tables cant be modified … {0}"
- Aggregating one Gen1 dataflow entity inside another
- "Refresh fails: duplicate value on the key column" after an M dedup (drop volatile cols → `Table.Distinct` → re-join aggregates)
- A KPI differs only on the first days after a partition edge, row counts identical
- Ported an exactness / divisibility test and a "level" column drifts
- Fact rows land on the blank member of a dimension after a refresh, evenly across ALL history
- Loads against a warehouse with a nightly key migration
- Composite model: an all-zero empty line in a table grouped by a dual dimension, `(Blank)` in its slicers
- DirectQuery over `Sql.Database`: every visual query takes seconds
- A merge's expand list names a column the right-hand query does not output
- TDS/SQL connections die during pre-login or login TLS ("forcibly closed", "handshake timed out"), worse in parallel, while HTTPS APIs work

**[references/dax-logic.md](references/dax-logic.md)**: DAX logic and numbers
- "Nothing picked" on a slicer dimension that a hidden not-blank filter keeps filtered
- Inlined per-row logic recalcs forever
- "Version selector broken" (all deltas zero)
- Period-over-period comparison asks
- "PBI shows wrong item split vs the ERP document"
- About to divide by a quantity / lot-size field, or a reconstructed cost is absurd
- "Which date drives the FX conversion?"
- One entity amount spread over child rows by several measures
- A budget or target line shows under two parents in a matrix (one parent's subtotal inflated, grand total right)
- Same amount on several rows of a table visual

**[references/service-publish.md](references/service-publish.md)**: Service and publishing
- Promoting a generated project to another workspace
- Publish fails: *Part URI is not valid per rules defined in the Open Packaging Conventions specification*
- A REST call from a PowerShell harness returns 403 with an EMPTY body, every retry
- Column TYPE change on a published model
- Device-code sign-in with no interactive console (agent-driven)
- Local PBIP is older than the published model
- A model you published comes back with your change reverted (someone republished an older copy)
- Publish from Desktop with no operator
- A scheduled flow must read a published model at volume (hundreds of DAX questions a run) into SQL
- Publish while the Windows session is locked (Desktop's publish dialogs hang)

**[references/harness-desktop.md](references/harness-desktop.md)**: Desktop and harness
- No operator available
- Desktop won't load an edit (invisible dialog)
- Headless Desktop screenshots all carry a "Working on it" dialog
- Headless Desktop open of a composite model stays "Untitled", catalogs empty
- Auditing PBIR JSON with a PowerShell walker
- Verifying Desktop work while the machine is LOCKED
- Cloning a model onto its OWN `cache.abf`
- Writing PBIR/model JSON that must stay byte-identical to Desktop's
- Harness scripts in Windows PowerShell 5.1
- Desktop visuals show "This might be caused by a capacity or license issue"
- Need a PBIP model's rows but Desktop cannot run (low RAM) or no sign-in is at hand
- Desktop is already running when a headless cycle starts
- Two runs of a PowerShell generator differ in every JSON file

## Self-iteration protocol (this skill must grow)

After ANY session using this skill where something surprised you — a new trap, a rule
that was wrong or incomplete, a better procedure, a core default that broke something:

1. Append a dated entry to the **Learnings log** in [reference.md](reference.md) with:
   what happened, the evidence (measured numbers / verbatim errors), and the rule derived.
2. If it changes an Iron Rule, an override or a situation row, edit that row in place —
   corrections beat additions. A new situation row goes into the matching topic file under
   `references/`, and its title into the index above. SKILL.md holds only the load order,
   overrides, environment notes, Iron Rules, red flags and the index; depth goes to the topic
   files and reference.md.
3. Per superpowers:writing-skills, test the edit: re-run the scenario that surprised you
   (or a subagent probe) against the updated skill before trusting it.
4. Never record a rule without evidence. "I think" entries are forbidden; every entry
   cites what was measured or what broke.
5. **This skill is PUBLIC — sanitize at write time, not later.** New entries must be
   born generalized: no company/client/vendor names, no person names, no tenant URLs,
   no workspace/dataflow/report GUIDs, no client-specific column prefixes or view
   names, no business figures (revenues, totals, prices). KEEP the evidence that
   teaches: row counts, timings, error strings verbatim, generic table nouns
   (Account, Country, fact/dim), public product names. Describe systems by ROLE
   ("a workbook-sourced accounts table", "the acquired company's extract"), never by
   name. Before committing an edit, run the banned-token grep audit over SKILL.md,
   reference.md and every file in `references/` — the token list itself lives OUTSIDE
   this skill (host repo private notes), because listing the banned names here would
   leak them.
6. **Version every learning push.** After the commit passes the audit and is pushed,
   tag it: minor bump (`vX.Y+1.0`) for new learnings or rule changes, patch for
   typo/sanitization fixes, major only for breaking restructures. Push the tag
   (`git push origin <tag>`). An unpushed or untagged learning batch is unfinished work.
7. **A core default that broke something** becomes an override row (core default,
   overlay rule, why) with its evidence. Also report it upstream as an issue on the
   core's public repository, written to the step 5 bar (the repository is a release
   mirror of an internal one, so issues are the route). Retire the override only after
   a pinned core version ships the fix and a probe confirms it.
8. **Core update check**, before moving the pinned core version: diff the two Power BI
   skills and their references between the tested tag and the new one; re-read every
   override against the new text and keep, adjust or retire it with evidence; re-run a
   probe; record the new tested version under Load order; version the push (step 6).
