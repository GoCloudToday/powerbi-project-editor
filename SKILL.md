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
   **Never touch a NestedJoin's nested column per row before `ExpandTableColumn`**
   (`Table.RowCount([Nested])`, `Table.IsEmpty`, `[Nested]{0}`…): each row then
   re-evaluates the whole right side including its source reads — measured 12,382
   downloads of each dataflow CSV for a 19k-row dimension, 3.5 h, machine crash. For
   a "matched" flag, add a constant `each true` column to the RIGHT table and expand
   it (unmatched → null). reference.md §2026-08-28.

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
| Porting M Number.Round to SQL | M defaults to **RoundingMode.Up** (half away from zero), NOT bankers (.NETs default). Port it explicitly as `SIGN(x) * FLOOR(ABS(x) + 0.5)`: plain T-SQL `ROUND()` is half-away-from-zero only for DECIMAL/NUMERIC - on a FLOAT argument it rounds ties to EVEN (`ROUND(CAST(0.5 AS float), 0)` = 0) and reproduces the very bug it was meant to fix. Detect a mismatch by diffing MARGINAL bucket distributions: adjacent buckets swap near-identical counts at the .5 boundaries (311k/61k/28k rows here) while row totals match. reference.md §2026-09-03. |
| Porting a calculated column that looks up another table | CALCULATE in a calc column does a CONTEXT TRANSITION: the value can be per-entity, not global, when the filter reaches the lookup table through a bidirectional relationship. Splicing one constant into SQL moved 92% of a bucket. Walk the relationship path first; keep it DAX if the lookup is contextual (or unreachable from SQL). |
| Engine→M port acceptance | EVALUATE-export the loaded table BEFORE the swap; after refresh, diff per column with a DIRECTION cross-tab (old==old-fallback vs new==new-fallback). It separates source drift, representation upgrades (variant serial → typed date), translation bugs, and latent OLD-side defects (blank-key LOOKUPVALUE mislabeling 17k rows was the old model's bug). |
| Inlined per-row logic recalcs forever | A calc column doing per-row FILTER(bigTable) × inner CALCULATE per iterated row is pathological — hoist the inner lookup to a calc column on the ITERATED table (relationship-filtered O(n) scan), then aggregate that. |
| ADDING a table to a PBIP model | Write ONLY the table file. `ref table` lines and PBI_QueryOrder are Desktop-cosmetic — Desktop's own save strips hand-added entries while the table stays loaded; discovery is from `tables/*.tmdl`. (Removal rules unchanged: a stale ref to a DELETED file is what breaks.) |
| Authoring a calc table by hand | Prototype the EXACT expression as EVALUATE against the user's live Desktop engine first (DATATABLE/SWITCH stand-ins for tables that don't exist yet). Var-table aggregates need GROUPBY/CURRENTGROUP — SUMMARIZECOLUMNS can't group by var-table columns and returns empty. |
| Scaffolding a NEW PBIP from scratch | Template every metadata part (schema versions, theme plumbing incl. StaticResources BaseThemes file) from a sibling Desktop-authored project — don't guess versions. Gate with offline `TmdlSerializer::DeserializeDatabaseFromFolder` (check the TYPE exists, not the DLL version — ALM Toolkit ships a capable build when SSMS's is too old). Desktop normalizes on first save (fenced M → plain `source =`, adds `active: true`) — re-diff before further hand edits. reference.md §2026-08-04. |
| Translytical input slicer / data-function button | Both fully hand-authorable. Input box = `textSlicer` with NO projections. Button = `actionButton` + `visualLink` `{show, type 'DataFunction', dataFunction{...}}` — the `dataFunction` node (structured, not `{expr}`) carries itemId/workspaceId literals + full parameter bindings (column params wrapped in `Aggregation` Function 3; slicer params reference the input slicer's visual NAME). itemId/workspaceId are environment-bound — rewrite on cross-workspace promotion. Full shape: reference.md §2026-08-04 round 3. |
| Second calculation group | Give it explicit `precedence:` (existing groups default 0; duplicates are rejected at load). |
| Hand-adding an object with `queryGroup:` | The group must ALSO exist as a bare `queryGroup <name>` object in model.tmdl (+ `PBI_QueryGroupOrder` annotation) or Desktop refuses the whole project ("Property QueryGroup ... refers to an object which cannot be found"). Don't disprove by grepping for a `PBI_QueryGroups` annotation — that's the dataflow-JSON spelling, not TMDL's. reference.md §2026-07-28. |
| Cloning a report page onto a parallel model | Rebind `queryRef` + Entity/Property nodes; KEEP `nativeQueryRef` — custom-visual embedded state (Zebra BI) keys on it, so old aliases preserve formatting without touching opaque blobs. Slicer selections live in `objects.general[].properties.filter` literals, NOT filterConfig. Regenerate page/visual/filter names (folder name must equal visual.json `name`). Verify: TOM field-inventory cross-check — every bound (Entity, Property) and queryRef prefix must resolve; zero legacy entities remain. |
| "Copy this page but on a different date column" | Don't rebind visuals. Inactive relationship `Fact[X]→Date[Date]` (+ `joinOnDateBehavior: datePartOnly` if X has a time part) + a two-item calc group (`SELECTEDMEASURE()` / `CALCULATE(SELECTEDMEASURE(), USERELATIONSHIP(...))`, explicit `precedence:`) + ONE hidden page-level filter on the group column in the clone. Composes with time-intelligence groups; no M change so no refresh. Sweep the page's measure queryRefs against the model first — cloned pages propagate phantom bindings. reference.md §2026-08-28 round 2. |
| Date-hierarchy field on a copied page | The `.Variation.Date Hierarchy.Year` binding can be swapped to a plain int column by replacing the whole HierarchyLevel node with a Column node. But the slicer's SELECTION filter also carries the LocalDateTable entity in `From` + `Annotations.filterExpressionMetadata` (incl. a display `valueMap`) — an entity-collection audit catches what eyeballs miss. |
| "Version selector broken" (all deltas zero) | Data may simply be flat: sweep per-snapshot sums FIRST, then emulate the slicers end-to-end with TREATAS inside CALCULATETABLE. Zero delta with working machinery is a data fact, not a bug. |
| "Filter out X from the report" | Default to a visible report-level `filterConfig` filter (hand-authorable; shape in reference.md §2026-07-21), not a hard exclusion inside a calc table — and never both: a hard filter under a pane card makes the toggle lie. Verify parity engine-side (`CALCULATE` emulation vs old counts). |
| Retargeting a field parameter | THREE attachment points: calc-table NAMEOF, the bound visual's Category projection, AND the parameter slicer's stored selection literals (`'''T''[C]'` appears twice in its visual.json). A stale literal renders ONE unsegmented bar, zero errors. Grep the report for the old NAMEOF after any retarget. |
| Dynamic multi-line or mixed-color text | Native cards can't: no word wrap, `UNICHAR(10)` renders as a space, one fontColor per value, multi-tile stacks show only tile 1 plus an overflow pill. Use the HTML Content (lite) custom visual (queryState role key `content`) with a measure returning styled HTML spans; park replaced native cards with visualContainer `"isHidden": true`. Recipes: reference.md §2026-07-22 rounds 2–5. |
| Nudging text inside a hand-built card | Use `visualContainerObjects.padding.top` (`…D` literal) — measured reliable. `cardCalloutArea.paddingTop` moved text on a Desktop-authored card but NOT on a hand-cloned one; card heights below ~24px render nothing. |
| Period-over-period comparison asks | Gate everything through ONE hidden date measure returning BLANK unless the selection is a single whole (edge-clipped) period — every consumer, and even footnote wording, blanks in lockstep. Fact-bounded calendars clip whole-period selections at the data edge; accept bounds equal to global MIN/MAX via `ALL('Calendar')` and anchor shifted windows at the period start. |
| "PBI shows wrong item split vs the ERP document" | EVALUATE the loaded rows for that document key FIRST — table visuals sum identical duplicate rows into one, so a screenshot lies about grain. Then audit any per-(doc, line) latest-version dedup in the feed: a renumbered/dropped line yields ghost + duplicated content with plausible totals. Dedup at document grain. No ADOMD DLL? `System.Data.OleDb` + `Provider=MSOLAP` works. Details: reference.md §2026-07-22 round 6. |
| Hand-authoring a thin report's `byConnection` | Desktop and the service parse the connectionString DIFFERENTLY: the service binds by `semanticmodelid=` alone (bogus Data Source still renders online); Desktop dials `Data Source=` for real, and **My Workspace has no XMLA path** (`powerbi://...myorg/My workspace` → "workspace is not found"). Universal shape = the one Desktop writes: `Data Source=pbiazure://api.powerbi.com;initial catalog=<datasetGuid>;identity provider="https://login.microsoftonline.com/common, https://analysis.windows.net/powerbi/api, 7f67af8a-fedc-4b08-8b4e-37c4d127b6cf";integrated security=ClaimsToken;semanticmodelid=<datasetGuid>` (+optional `modelid=<n>`). reference.md §2026-08-07 round 5. |
| Report stuck on "Loading your report..." after a trim | Binding/pages endpoints healthy but render spins forever → grep BOOKMARKS for dropped table names: explorationState still snapshots deleted visuals filtered on deleted tables. Prune `explorationState.sections.<page>.visualContainers.<name>` for every visual no longer on the page. reference.md §2026-08-07 round 5. |
| Initial load of a model with an incremental-refresh policy | pplyRefreshPolicy:true FORCES commitMode:transactional (partialBatch rejected); 	ype:clearValues creates NO partitions (only ull materialises a policy); the service kills any request running past **5 h** and rolls it back - so load ONE fact table per request (objects:[{table}]). The policy sourceExpression is validated TEXTUALLY for the literal names RangeStart/RangeEnd, so a window-splicing helper must take them as ARGUMENTS. getDefinition never exports policy-generated partitions - verify from the refresh record. reference.md §2026-09-02. |
| Fixed the fact query but the numbers did not move | `applyRefreshPolicy:true` refreshes ONLY the incremental window; historical partitions keep data from the OLD query, so a check against an older month shows no change and the model silently holds a MIX of both query versions. Propagate a query change with `applyRefreshPolicy:false` + `objects:[{table}]` (re-queries every partition). Always verify one partition inside the incremental window AND one outside. reference.md §2026-09-03. |
| Rebuild is BIGGER than the model it replaces | Check whether the original loads old partitions through a *historical* query variant that blanks high-cardinality detail columns. Reproducing it is worth GBs (one document-number column: 38M distinct with the projection, ~55M without); dropping it hit 
ew dataset of size 11634 MB exceeds the limit of 10240 MB. Splice one __ARCHIVE__ token per partition instead of maintaining two queries; 	argetStorageMode: PremiumFiles lifts the 10 GB ceiling but is not the fix. Also: etryCount:3 on a 2.5-hour query turns one mistake into 8.5 h of capacity. |
| Freezing a retiring source into a one-time Gen1 dataflow | Author `model.json` in the export dialect (queriesMetadata MAP, entities = loadEnabled set, no partitions/connectionOverrides; service reassigns dataflowId on import). Three enforced traps: declared attributes are a PROJECTION (missing physical column fails refresh), an any-typed column fails SAVE (`DataflowObjectModelTypeNotSupportedException`), and dataflows evaluate EVERY column (no Desktop laziness — garbage cells Desktop never touched fail the refresh). Bulletproof query shape + headless REST verification: reference.md §2026-07-27. |
| Measuring a published model's memory footprint / finding the expensive columns | REST `executeQueries` rejects EVERY `INFO.*` function on some datasets (even `INFO.TABLES()`; detail is just "Failed to execute the DAX query"), so storage DMVs need XMLA - and XMLA refuses an Azure-PowerShell token ("Authentication failed for all authenticators"). Working route: a recent ADOMD (DAX Studio's build, assembly 19.84, has `AccessToken`; an old SSMS one does not) under **PowerShell 5.1** (pwsh7 = `CallContext` type-load error), **preload the whole DLL folder** (a lazy AssemblyResolve handler = StackOverflow), connect with NO credentials for an interactive prompt in a REAL window, and read rows in a manual loop (`DataTable.Load` throws "Failed to enable constraints"). Size = SUM(USED_SIZE) over segments + SUM(DICTIONARY_SIZE) over columns; strip `\s*\(\d+\)$` from COLUMN_ID before diffing two models. reference.md §2026-09-04. |
| Reading RLS roles off a PUBLISHED model | `INFO.ROLES()`/`INFO.TABLEPERMISSIONS()` via REST `executeQueries` are BLOCKED (as is EVERY `INFO.*` function on some datasets - see the footprint row) (400, AS code 3239575574) though other INFO functions work; ADOMD/XMLA with an Azure-PowerShell Power BI token 401s despite a correct `aud`. Use the items API: `POST …/semanticModels/{id}/getDefinition?format=TMDL` → poll `Location` → `/result` (base64 TMDL incl. `roles/*.tmdl`). ADOMD needs PowerShell 5.1 (`CallContext` type-load error under 7). reference.md §2026-08-25. |
| Downloading a Gen1 dataflow definition | Fabric `…/dataflows/{id}/getDefinition` returns **500 UnknownError** (Gen2-only, not transient). `GET /v1.0/myorg/groups/{ws}/dataflows/{id}` returns model.json with all M in `pbi:mashup.document` — the no-operator equivalent of the portal's Export .json. |
| Authoring a NEW Gen1 dataflow (import model.json) from SQL, no DB access | Keep each entity's T-SQL in a file, splice parameters as `" & Param & "`, build the JSON by script and validate offline with ScriptDom (SqlServer PS module `coreclr\` DLL under pwsh 7): parse + outermost `SelectElements` == declared attributes (name AND order). Inside a foldable native query: no `--` comments, `;`, `"`, `ORDER BY` or CTE — the service wraps it as a derived table; climb hierarchies with fixed-depth self-joins on a level column. Header-vs-line integrity counters = `UNION ALL` of a header-attributed aggregate and an orphan-line aggregate, then GROUP BY (NULL periods group; a FULL OUTER JOIN would not). Columns you cannot prove from production queries get a `-1` sentinel + documented switch. **Emit `allowNativeQueries: false`** — the service refuses to import `true` ("should be set to false on import"); the operator approves the native queries once in PQ Online (Edit tables → Edit permission → Run → Save) after import. AFTER import + approval, re-GET the definition and assert every entity KEPT its attributes — one can come back empty (M and refresh fine, but Desktop's navigator won't offer it); repair = open that query in PQ Online and re-save. reference.md §2026-08-27 (+ round 2). |
| Auditing PBIR JSON with a PowerShell walker | On `ConvertFrom-Json -AsHashtable` output, `$node.Values` (also `.Keys`/`.Count`) resolves to a JSON KEY of that name when present — a pivotTable's queryState HAS a "Values" key, so the walker skips the Rows role and every matrix reports false unbound fields (measures fine = the signature). Enumerate with `$node.GetEnumerator()`; methods are never shadowed. reference.md §2026-08-27 round 2. |
| Generating PBIR visuals from PowerShell | Members of `objects`/`visualContainerObjects` must be ARRAYS — a helper function returning a single-element array gets unwrapped on return (`"title": [{…}]` → `"title": {…}`) and Desktop refuses the report ("Property /visual/visualContainerObjects/title … was not provided as the correct type", one line per visual). Return ONE object, wrap at the call site with `@(...)`, and shape-audit every member for IList. Also: a validator step that COPIES resources must normalize (BOM-strip) at copy time, or each rerun reintroduces the bug it checks for. reference.md §2026-08-31 round 2. |
| Desktop refresh of many dataflow-fed tables fails "A cyclic reference was encountered during evaluation" | If the failing table ROTATES between runs and every query is an acyclic navigation, it is the KNOWN Desktop parallel-loading bug (community-confirmed; surfaced here going from 7 to 9 identical dataflow navigations), not your M — set `.pbi\editorSettings.json` `parallelQueryLoading: false` (per-project equivalent of Options → Data Load → parallel loading) and the same refresh completes. Don't burn cycles diffing definitions once engine partitions, expressions (0) and the dataflow document all check clean. |
| A dataflow entity's aggregate query dies server-side ("A severe error occurred on the current command", ~25 min) | A per-combo GROUP BY over a multi-billion-row snapshot table exhausts the server. Chunk BY MONTH in the M layer: token the WHERE as `__WINDOW__`, splice per-month bounds in a `(MonthStart, IncludeUndated) => Value.NativeQuery(...)` lambda, `List.Transform({0..MonthsBack}, ...)` + `Table.Combine` — identical schema/semantics, 1/Nth the aggregate state per query; undated rows ride with chunk 0. Reach the fix WITHOUT reimport by pasting the one query in PQ Online (dataflowId and native-query approval survive). |
| Verifying Desktop work while the machine is LOCKED | `run.it`, `wnd.find`, and `Elm[...].Find/Invoke` all work without the input desktop — click Refresh by element, then watch completion via OleDb DMV polling (TMSCHEMA_PARTITIONS states) and read results with DAX; only screenshots/keyboard/save need the unlocked session. |
| Copying a static resource (theme) into a PBIR project | The file must be UTF-8 WITHOUT BOM — Desktop refuses the whole project open ("Only text with UTF8 encoding without BOM (byte order marks) is supported. Detected BOM: 'UTF-8'") and falls back to an "Untitled" window with an EMPTY workspace DB, so a headless cycle sees a catalog with 0 TMSCHEMA tables forever (dialog-blocked, not slow — screenshot the desktop and read the dialog instead of polling blind). `Copy-Item` preserves the source's BOM: strip EF BB BF and BOM-sweep the project. TMSCHEMA DMVs also need `Initial Catalog` (discover via DBSCHEMA_CATALOGS) or they return 0 rows without error. reference.md §2026-08-31. |
| "RLS users see less data than write users" | RLS on a dimension DELETES fact rows whose FK doesn't resolve — the unmatched row sits on the relationship's blank member, which every `tablePermission` excludes. Isolate by applying each role predicate alone via `TREATAS`. **Never** relax it to `… \|\| ISBLANK(bridge[key])`: measured, that leaked every other tenant's dangling rows into one tenant (9,639 → 1,683,563 rows). Fix by backfilling placeholder dimension rows carrying the tenant key — after verifying each dangling key maps to exactly ONE tenant. |
| Blaming the source for a dangling FK | Read the query the MODEL actually binds to, not the dataflow's same-named entity — derived dims can inherit a date window, sentinel exclusions and INNER JOINs that cause the loss locally (1.78M rows here). Prove source fault with a superset argument (the trim's key list is unfiltered over the full line tables ⊇ the facts, so absence proves the master lacks the row), and corroborate against the source's own unknown-member convention. |
| `executeQueries` reproducing an RLS identity | `impersonatedUserName` 401s with `RLSNotAuthorizedForModel` unless you own/administer the dataset — simulate the role with `TREATAS` instead (matched the real report output exactly). Read the `DetailsMessage` detail for the verbatim DAX error before guessing at a 400. |
| Importing a Gen1 dataflow fails "Linked tables cant be modified … {0}" | A LOAD-ENABLED query navigating PowerPlatform.Dataflows to another dataflows entity is a linked table and may hold ONLY the navigation steps ({0} never names it). Split: pure navigation in a load-disabled query + a load-enabled computed entity with the transformations (the pattern a working production dataflow uses). Validator rule: load-enabled + PowerPlatform.Dataflows in the text ⇒ must end at {[entity = …, version = ""]}[Data]. reference.md §2026-09-01 round 2. |
| Aggregating one Gen1 dataflow entity inside another | Referencing a load-enabled sibling makes it a computed-entity SOURCE evaluated by the enhanced compute engine — SQL-strict typing can fail the untouched source entity ("We cannot convert the value 1 to type Text"). Put the join/group in the consuming MODEL's M instead. Also: charge feeds with fixed + percentage columns zero out percentage-configured charges if "value" maps only the fixed column; collapse charge tables to one row per document before header joins. reference.md §2026-07-27 round 2. |
| "Refresh fails: duplicate value on the key column" after an M dedup (drop volatile cols → `Table.Distinct` → re-join aggregates) | No loaded copy shows the rows — the uniqueness violation ROLLS BACK the refresh, so local cache and published dataset both look clean. Diagnose from the CURRENT source file: reproduce the M parse, group by key, count keys with >1 distinct value per KEPT column; the offenders extend the removal + aggregate lists (`List.Max` for counters). `Csv.Document(QuoteStyle.None)` still honors quotes around delimiter-bearing fields — a naive split misaligned 99.9% of rows and invented 20 phantom offenders. reference.md §2026-09-01 round 3. |
| Proving a REBOUND report shows the same numbers on a rebuilt model | Rebuild each visual's DAX from `visual.query.queryState` (Column → grouping, Measure/Aggregation → value; `TOPN(300, SUMMARIZECOLUMNS(...), every grouping col, ASC)`), de-duplicate the text (805 visuals → 116 queries), run the SAME text on both models via `executeQueries`, split keys/values by name shape (`T[C]` vs `[v1]`). Traps: a measure-less `SUMMARIZECOLUMNS` IGNORES its filter argument (446,432 towns = all history vs 310,250 with a row-count value — add `"n", COUNTROWS(fact)`); field-parameter tables reject one grouped column (compare with `SELECTCOLUMNS` over the table); `Aggregation.Function` 5 = ROW count (`COUNTA`), 2 = `DISTINCTCOUNTNOBLANK`; export the inventory BEFORE any `-OnlyIds` filter; a PowerShell helper named `Diff` silently runs `Compare-Object` (aliases beat functions); `pwsh -File` binds `[string[]]` as one string. 20-column detail tables with a selector table in the grouping cannot run over REST even for one day — cover their columns with per-month aggregates + one document's line detail. reference.md §2026-09-04 round 2. |
| A KPI differs only on the first days after a partition edge, row counts identical | A per-row calculation that reads a SECOND table (non-working-day calendar for a working-day lead time) inherited the fact's `RangeStart/RangeEnd`, so rows just after an edge cannot see the weekend before it → misclassified Late (1,699 of 30,450 headers on one edge day, +8.7 pp). The original model had it too (moving 3-month windows recomputed at refresh → the wrong days MOVE monthly); a quarter-window + month-grain policy has edges at every quarter AND month start, and merged monthly partitions are never re-queried → the error is FROZEN into history. Fix: give the lookup its own window padded 45 days each side (`__WD_START__`/`__WD_END__` tokens next to the range tokens). Audit every `RangeStart` occurrence in a partitioned query, not just the fact's WHERE. Acceptance = (event date, reference date) bucket-signature equality, not a per-day threshold (0.5 pp hid weekday edges). reference.md §2026-09-04 round 2. |
| Ported an exactness / divisibility test and a "level" column drifts | M arithmetic on decimal-typed inputs is DECIMAL precision; `CAST(q AS float) / CAST(r AS float) = FLOOR(...)` misses exact decimal quotients. Signature: per-level LINE counts move by tens while per-level QUANTITIES move by hundreds of thousands (26 of 11.96M lines, every quantity card off ~0.035 %) — easy to misfile as refresh lag. Port as `CAST(q AS decimal(38,18)) % CAST(r AS decimal(38,18)) = 0`. reference.md §2026-09-04 round 2. |
| Before writing "parity proven" | Commission an adversarial second-model review with the read-only ids and the REST recipe: in ~40 min it overturned "the rebuild is right wherever it differs", "these differences are refresh lag" and "same window on both sides". Its winning methods: bucket-signature diffs keyed by two dates, per-first-character set fingerprints (count, Σlength, Σchar codes) instead of distinct-count equality, and reading the ORIGINAL partition generator instead of assuming its grain. Also only a reviewer noticed: dictionaries keep FIRST-SEEN casing (`ANYTOWN` vs `Anytown` in slicers) and a per-quarter archive projection lags a month-precise rule by a quarter. reference.md §2026-09-04 round 2. |

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
