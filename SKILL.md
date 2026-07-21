---
name: powerbi-project-editor
description: Use when editing Power BI Project (PBIP) files outside Power BI Desktop — TMDL semantic models or PBIR report definitions — including trimming/refactoring models, deleting columns/measures/tables, dependency or usage analysis, editing M partitions, diagnosing slow visuals, or verifying model changes against a live Desktop engine (msmdsrv, Invoke-ASCmd, cache.abf, DISCOVER_CALC_DEPENDENCY).
---

# Power BI Project Editor

Operational rules for editing PBIP folders (TMDL model + PBIR report) programmatically,
earned through verified failures. Deep procedures, DMV queries and parsing recipes:
[reference.md](reference.md). A companion toolchain (extractor, inventory, engine-backed
keep/kill graph, gate, deleters, fingerprint suite) is worth carrying per repo under
`tools/` — if present, adapt it; the recipes in reference.md are sufficient to rebuild it.

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
7. **Checkers must not share code with the thing they check**, and must be calibrated:
   the gate must FAIL a known-bad input; integrity must PASS a known-good model —
   before either verdict is trusted. Never chain gate→apply unconditionally.
8. **The frontend attaches by strings, and fails silently.** Conditional formatting
   (RAG icons, backColor/fontColor, webURL), columnWidth, alignment and bars-only
   settings bind via `selector: {metadata: "Table.Object"}` STRINGS that must equal
   the visual's own projection queryRef. A rename/rebind that updates projections but
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

## Quick reference

| Situation | Rule |
|---|---|
| Removing a table | Also: its relationship blocks, `ref table` line in model.tmdl, `PBI_QueryOrder` entry, role/culture entries. Verify not a bridge / M-consumed first. |
| Relationship endpoints | Dot notation `'Table'.Column` (either side quotable). NOT `Table[Column]`. Root endpoints iteratively — only while BOTH tables are kept. |
| Calc-table columns (`isNameInferred`, `sourceColumn: [X]`) | Block deletion is meaningless — the DAX source recreates them. Remove via source surgery or keep. |
| Kept table, all columns dead | Keep ≥1 column; zero-column tables are degenerate (COUNTROWS errors). |
| Broken (non-compiling) measures | Engine reports only PARTIAL deps for them — take their dependencies from static text. |
| `Stage`/`Team`-like words in M | Static M-consumption greps false-positive on column names (`[Sales Stage]`). Engine `DISCOVER_CALC_DEPENDENCY` (rot=PARTITION rows) is authoritative. |
| Verifying a trim | Numeric fingerprint (values, ordered, volatility-flagged) + engine recalc + partitions all `State=1` + captured-query perf replay. |
| Timed-out `Invoke-ASCmd` | The server keeps executing. Sweep orphans: `<Cancel><SPID>n</SPID></Cancel>`. Never run suites while the user works in Desktop. |
| "Report uses X" | PBIR truth = visuals + filters + sorts + field-param `NAMEOF` + RLS + `From`-alias resolution. Bookmarks are false-positive roots (validate `activeSection` against real pages). |
| Prod safety | A carve-out report's trim is NEVER prod-safe. Union sibling reports' usage into the contract first. |
| No operator available | Desktop cycles work headlessly: `Start-Process <pbip>` → poll msmdsrv port → verify → `Stop-Process -Force` (kill is cache-safe). Data refresh is NOT headless (dataflow auth fails out-of-band) — batch M edits for one user refresh. |
| Desktop won't load an edit (invisible dialog) | Validate offline: TOM `TmdlSerializer` parse + throwaway deploy to a live engine (see reference.md §2026-07-17). Desktop opens cache.abf's OLD schema first — poll for the NEW table count before judging. |
| Calc table referencing a related table | Blank-row circular dependency (error names the relationship GUID). Route references through `ALLNOBLANKROW(T)`. |
| Recalc says "no errors" but partitions not Ready | The recalc result lies; read TMSCHEMA_PARTITIONS states: **5 = EvaluationError (root cause)**, 7 = its cascaded victims, 3 = NoData awaiting refresh. Fix the State-5 object's formula. |
| Slimming an M-consumed producer | Its TMDL columns may drop below what consumers read (they re-run the M query, not the stored table) — but the M output must keep union(own TMDL cols, consumer-read cols), and calc columns ON the producer pin their inputs. |
| Model won't shrink under keep/kill | Reachability ≠ necessity on calc-table webs. Sever chains first (hidden visuals are look-free deletion roots), then trim. |
| Folding tables away | Order by REFRESH COST, not size: (1) FREE — M-consumed-only table → shared expression under the SAME query name (byte-identical M keeps consumer caches); embedded static list → inline DAX constant; single-consumer calc table → inline VAR in its consumer. (2) ONE REFRESH — small-entity M fusions; calc table → M partition. (3) LAST — data carriers read by calc tables must stay loaded until their DAX consumers convert first (converting Account to M dissolved five carriers at once). Full playbook: reference.md §Folding ladder. |
| Converting table → expression | Sweep it for DAX calc columns FIRST — they vanish with the table and only fail at the user's refresh ("column X of the table wasn't found"); port each into consumers' M. Also confirm: no relationships, no RLS, no frontend refs, no local M step named like any document member (self-cycle). |
| Partition type change (calc ↔ m) | Rejected in place ("Changing the partition type ... is not allowed") — RENAME the partition so Desktop sees drop+add. |
| Engine→M port acceptance | EVALUATE-export the loaded table BEFORE the swap; after refresh, diff per column with a DIRECTION cross-tab (old==old-fallback vs new==new-fallback). It separates source drift, representation upgrades (variant serial → typed date), translation bugs, and latent OLD-side defects (blank-key LOOKUPVALUE mislabeling 17k rows was the old model's bug). |
| Inlined per-row logic recalcs forever | A calc column doing per-row FILTER(bigTable) × inner CALCULATE per iterated row is pathological — hoist the inner lookup to a calc column on the ITERATED table (relationship-filtered O(n) scan), then aggregate that. |
| ADDING a table to a PBIP model | Write ONLY the table file. `ref table` lines and PBI_QueryOrder are Desktop-cosmetic — Desktop's own save strips hand-added entries while the table stays loaded; discovery is from `tables/*.tmdl`. (Removal rules unchanged: a stale ref to a DELETED file is what breaks.) |
| Authoring a calc table by hand | Prototype the EXACT expression as EVALUATE against the user's live Desktop engine first (DATATABLE/SWITCH stand-ins for tables that don't exist yet). Var-table aggregates need GROUPBY/CURRENTGROUP — SUMMARIZECOLUMNS can't group by var-table columns and returns empty. |
| Second calculation group | Give it explicit `precedence:` (existing groups default 0; duplicates are rejected at load). |
| Cloning a report page onto a parallel model | Rebind `queryRef` + Entity/Property nodes; KEEP `nativeQueryRef` — custom-visual embedded state (Zebra BI) keys on it, so old aliases preserve formatting without touching opaque blobs. Slicer selections live in `objects.general[].properties.filter` literals, NOT filterConfig. Regenerate page/visual/filter names (folder name must equal visual.json `name`). Verify: TOM field-inventory cross-check — every bound (Entity, Property) and queryRef prefix must resolve; zero legacy entities remain. |
| Date-hierarchy field on a copied page | The `.Variation.Date Hierarchy.Year` binding can be swapped to a plain int column by replacing the whole HierarchyLevel node with a Column node. But the slicer's SELECTION filter also carries the LocalDateTable entity in `From` + `Annotations.filterExpressionMetadata` (incl. a display `valueMap`) — an entity-collection audit catches what eyeballs miss. |
| "Version selector broken" (all deltas zero) | Data may simply be flat: sweep per-snapshot sums FIRST, then emulate the slicers end-to-end with TREATAS inside CALCULATETABLE. Zero delta with working machinery is a data fact, not a bug. |
| "Filter out X from the report" | Default to a visible report-level `filterConfig` filter (hand-authorable; shape in reference.md §2026-07-21), not a hard exclusion inside a calc table — and never both: a hard filter under a pane card makes the toggle lie. Verify parity engine-side (`CALCULATE` emulation vs old counts). |

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

## Self-iteration protocol (this skill must grow)

After ANY session using this skill where something surprised you — a new trap, a rule
that was wrong or incomplete, a better procedure:

1. Append a dated entry to the **Learnings log** in [reference.md](reference.md) with:
   what happened, the evidence (measured numbers / verbatim errors), and the rule derived.
2. If it changes an Iron Rule or Quick-reference row, edit that row — corrections beat
   additions; keep SKILL.md tight and move depth to reference.md.
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
   name. Before committing an edit, run the banned-token grep audit — the token list
   itself lives OUTSIDE this skill (host repo private notes), because listing the
   banned names here would leak them.
6. **Version every learning push.** After the commit passes the audit and is pushed,
   tag it: minor bump (`vX.Y+1.0`) for new learnings or rule changes, patch for
   typo/sanitization fixes, major only for breaking restructures. Push the tag
   (`git push origin <tag>`). An unpushed or untagged learning batch is unfinished work.
