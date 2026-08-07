# powerbi-project-editor — operational reference

Deep procedures backing SKILL.md. Everything here was executed and verified on real
production models (a multi-phase refit + rewrite, 2026-07). Adaptable implementations
belong in the host repo's `tools/` (pbir_usage.py, inventory.py, model_graph.py, gate.py,
delete_objects.py, fingerprint_*, m_prune.py, tmdlref.py).

## PBIP anatomy (what's safe to touch)

- `*.pbip` — pointer only. `*.Report/definition/` — PBIR JSON (report.json, pages/*/page.json,
  pages/*/visuals/*/visual.json, bookmarks/). `*.SemanticModel/definition/` — TMDL
  (model.tmdl, database.tmdl, relationships.tmdl, expressions.tmdl, tables/*.tmdl, roles/, cultures/).
- Gitignore (MS-documented, Desktop auto-creates): `**/.pbi/cache.abf`, `**/.pbi/localSettings.json`.
  `core.autocrlf=true` is correct; do NOT add `* -text` gitattributes (mass-dirties the repo).
- Editable externally: TMDL, PBIR JSON, expressions. NOT: diagramLayout.json,
  mobileState.json, report.json (legacy format).
- Desktop writes CRLF, UTF-8 no BOM. `open(p, encoding='utf-8-sig', newline='')` to read;
  write with `newline=''` and explicit `\r\n` restoration.

## TMDL parsing rules (each regex trap was hit)

- Block structure: declarations at 1 tab (`\tcolumn|\tmeasure|\tpartition|\thierarchy`),
  properties at exactly 2 tabs, expression continuation at 3+. `sourceColumn: [X]` is a
  calc-table OUTPUT reference, not a DAX ref — scanning properties as DAX false-positives.
- Declaration regex: `^\t(column|measure)\s+'?([^'=\n]+?)'?\s*(=.*)?$` — the name group is
  lazy; ONLY the `(=.*)?$` tail forces full-name consumption. Any trailing `.*` escape hatch
  truncates unquoted names to one character. Unit-test against the real file before use.
- Relationship endpoints: `fromColumn:`/`toColumn:` with dot notation; either part quotable:
  `Sales.Amount`, `'Order Lines'.src_x`, `Accounts.'Universal Name'`. Quoted tables may
  contain dots — try quoted alternative first. Test: 100% of the file's endpoints must parse
  AND resolve, or do not proceed.
- DAX object resolution is CASE-INSENSITIVE (`col_x(Calculated)` == `...(calculated)`).
- Strip `//` and `/* */` comments before scanning DAX (commented refs are phantoms).
- Bare `[x]` in DAX legally binds cross-table in row context (`FILTER(Pipeline,[PipelineSource]=..)`
  from another table's measure) — a static closure MUST over-approximate bare refs to
  any-table; use the ENGINE graph for precision instead.
- Block deleter: find declaration line, extend to next 1-tab sibling or column-0 line,
  trim trailing blanks INSIDE the span only. Never post-process other lines.

## Dependency layers (a keep/kill computation needs all six)

1. **Report contract** (PBIR): visual projections, visual/page/report `filterConfig`,
   sorts, `pageBinding` (drillthrough/tooltip). Resolve `SourceRef.Source` aliases against
   enclosing `From` arrays (`{"Name":"c","Entity":"Contracts"}`). Bookmarks persist
   state for DELETED objects — treat as roots only after validating their `activeSection`
   exists; orphaned bookmark groups betray carve-out reports.
2. **Engine DAX graph** — `$SYSTEM.DISCOVER_CALC_DEPENDENCY` (exact; supersedes static
   text closure). Row semantics: `ot=PARTITION, rot=PARTITION` = table-to-table M consumption;
   `ot=M_EXPRESSION` rows map expression deps; `ROWS_ALLOWED` = RLS deps. EXCEPTION: measures
   in error state have partial engine deps — union their static-text refs.
3. **Relationships** with `crossFilteringBehavior`/`isActive`; endpoints rooted iteratively
   (only while both tables kept), else dead tables' keys keep them "alive" forever.
4. **Filter-bridge policy**: keep any table on a directed filter-flow path
   (edges: `to`-table → `from`-table; both ways if bothDirections) from {RLS tables ∪ queried
   tables} THROUGH to a queried table. Pure dims nothing selects (no incoming edge) are inert.
   Evidence: `Team` (zero references anywhere) carried Users⇒Team⇒Pipeline slicer counts —
   only a numeric fingerprint caught its removal.
5. **Structural roots**: sortByColumn, groupByColumn (`relatedColumnDetails`), hierarchy
   levels, field-param `NAMEOF('T'[X])` targets (via calc-table sources).
6. **RLS roles**: `tablePermission` expressions + the tables they bind.

Keep-set iterates to fixpoint. M-consumed keeper tables keep ALL columns (zero-column
tables are degenerate). Calc-table inferred columns are exempt from block deletion.

## Verification stack (in order, cheapest first)

1. **Static integrity** (calibrated on known-good first): every relationship endpoint,
   sortBy/groupBy/hierarchy target, report ref resolves case-insensitively.
2. **Independent gate** on every delete set — separate parser from the closure, scans raw
   text; MUST fail a preserved known-bad set (regression test). Ambiguous bare-ref flags are
   adjudicated by the engine graph. Gate exit code checked BEFORE apply, never chained.
3. **Engine shape check** after reopen: INFO counts match disk; all partitions `State=1`
   (`State=3` = cache invalidated → an M text change escaped); `RefreshedTime` (UTC) equal to
   the last real refresh proves cache reuse — equal to the open time proves invalidation.
4. **Recalc** `{"refresh":{"type":"calculate",...}}` — 0 errors; warning count must not grow.
   Recalc rebuilds calc tables/columns/hierarchies but NEVER loads data.
5. **Numeric fingerprint**: measure totals + sliced by top contract columns + per-visual
   SUMMARIZECOLUMNS + per-column `VALUES()` + rowcounts. `ORDER BY` everything (row order is
   not guaranteed → hash flap). Flag TODAY()/NOW()-dependent measures via transitive closure
   (calc-table sources count) — compare by hash within a data-day, status-only across refreshes.
   Error statuses are baseline facts (error must stay error). Prove determinism with a double
   run before trusting. After a source refresh, only status-level comparison is meaningful.
6. **Captured-query perf replay**: while the visual renders, pull full `COMMAND_TEXT` from
   `$SYSTEM.DISCOVER_COMMANDS` (it truncates nothing server-side; save immediately); replay
   verbatim with `-QueryTimeout` on candidate models. Two Desktop instances = live A/B harness.

## Engine operations

- Port: `Get-NetTCPConnection -OwningProcess (Get-Process msmdsrv).Id -State Listen`.
  Catalog: `SELECT [CATALOG_NAME] FROM $SYSTEM.DBSCHEMA_CATALOGS`. Parse via
  `([xml]$r).return.root.row` (values inside escaped element names).
- Every Desktop reopen = new port + new catalog GUID. PBIX save = a fork of the PBIP.
- `Invoke-ASCmd` param `-Db` collides with `-Debug`; PowerShell 7 here-strings in
  `python -c` mangle `\"` — put queries in files.
- `-QueryTimeout` aborts the CLIENT only; the server keeps crunching. After any timeout,
  sweep: `<Cancel xmlns="...analysisservices/2003/engine"><SPID>n</SPID></Cancel>`.
- Never run query suites while the user interacts with Desktop (mutual starvation; on a
  memory-starved machine msmdsrv gets OOM-killed mid-query and the visual "spins forever").
- VertiPaq size = `DISCOVER_STORAGE_TABLE_COLUMNS.DICTIONARY_SIZE` +
  Σ`DISCOVER_STORAGE_TABLE_COLUMN_SEGMENTS.USED_SIZE`. Column count ≠ bytes: in one model
  86% of the dead weight was ONE table's 20 columns. Always size before trimming.
- Slow-visual triage order: (1) own background queries / orphaned SPIDs, (2) free RAM +
  msmdsrv alive, (3) calc-table recalc-on-open blocking, (4) THEN capture the real query
  and A/B. Component probes (per-measure at grain) rule out data cost; identical components
  with divergent whole-query time = FE plan difference (see isAvailableInMdx below).

## M-query pruning (refresh-time/cache-size wins)

- Deleting a TMDL column does NOT stop the M query fetching it. To reclaim fetch/cache,
  append `#"Kept Columns" = Table.SelectColumns(<last step>, {<kept sourceColumn names>})`.
- Leaf tables (no M consumers) prune freely. Non-leaf producers CAN be pruned too, but
  column-level M consumption isn't tracked by any graph — you must READ each consumer's M
  and keep union(own TMDL columns, consumer-read columns) in the SelectColumns list
  (see Learnings 2026-07-17 "Slimming an M-consumed producer"). No enumerated consumer
  list = no prune.
- Every M edit invalidates that partition's cache → batch all M edits, one scheduled refresh.
- Waste detection by quoted-name scan false-positives on data values and intermediate step
  columns; harmless (the SelectColumns projects FINAL names) but expect noise.

## Windows/tooling traps

- MSYS bash mangles POSIX paths passed inside `python -c` strings (fine as argv). Python
  `print` emits CRLF → `while read` loops on its output need `tr -d '\r'`.
- Exit codes die in pipelines (`cmd | tail; echo $?` reports tail). Check `$PIPESTATUS`/run
  checks standalone. NEVER `gate && apply` in one unconditional chain — a failing gate
  applied anyway once; git saved it.
- Ref-only merges while Desktop holds files open: `git fetch . branch:master`.

## isAvailableInMdx / attribute hierarchies

`isAvailableInMdx: false` (the standard "memory optimization") flipped the FE plan of a
"Show items with no data" visual from short-circuit (2.2s) to catastrophic (150s timeout):
the `__DS0PrimaryShowAllCompat` block (nested GENERATEALL/GENERATE + NOT(ISBLANK) OR-chains)
depends on attribute hierarchies for plan choice. Component probes were identical on both
models — only whole-query time diverged. Before ANY such pass, grep the report for ShowAll
visuals; if present, don't. Value fingerprints cannot detect this class.

## Learnings log

Append new dated entries here (see SKILL.md §Self-iteration protocol). Every entry:
what happened → evidence → rule.

- **2026-07-17 (single-page rework, autonomous session):**
  - **Headless Desktop cycle is safe and complete.** `Start-Process <pbip>` → poll
    msmdsrv port (`Get-NetTCPConnection`) → DMV/DAX verify → `Stop-Process -Force
    PBIDesktop` (msmdsrv dies with it; no save dialog; no orphans). ~8 cycles, cache.abf
    intact every time (all partitions State=1 on reopen). Autonomous editing loops are
    viable with no operator.
  - **PBIP open = cache.abf restore FIRST (old schema!), TMDL apply SECOND.** Polling
    the engine right after open can see the OLD model (45 partitions where 31 expected)
    or an empty catalog. Poll until the EXPECTED table/partition count appears; a
    premature snapshot misdiagnoses "TMDL failed" (it did, once, for ~7 wasted minutes).
  - **Autonomous data refresh is impossible on Desktop.** TMSL `refresh type=full`
    executes (no auth prompt) but `PowerPlatform.Dataflows(null)` returns an empty
    workspace list out-of-band → `Expression.Error: The key didn't match any rows`.
    Batch every M edit; the user does ONE refresh.
  - **Offline TMDL validation without the Desktop UI**: Desktop's schema-error dialogs
    are invisible headlessly and its Traces logs are empty. NuGet
    `Microsoft.AnalysisServices.NetCore.retail.amd64` (TOM 19.84) `TmdlSerializer`
    parses the folder; deploying the deserialized DB to a live Desktop engine under a
    throwaway name (`Databases.Add` + `Update(ExpandFull)`, then Drop) surfaces the
    engine's exact semantic error. A ~40-line `tools/validate_tmdl.ps1` wrapping this is worth carrying.
  - **Calc-table blank-row circularity**: a calc table that references a table it has an
    active relationship to (even via LOOKUPVALUE) trips "A circular dependency was
    detected" naming the relationship GUID — the related table's virtual blank row
    depends on the calc table's keys. Fix: reference through `ALLNOBLANKROW`
    (`MAXX(FILTER(ALLNOBLANKROW(T), T[k]=[key]), T[v])`, or wrap predicate filters
    as `FILTER(ALLNOBLANKROW(T), ...)`). Verified: model loaded after the rewrite.
  - **Deleting a relationship deletes its virtual blank member** — fingerprint
    `VALUES()`/slice row counts drop by exactly 1 on the former one-side (e.g. a dim's
    VALUES() 16→15 after a snapshot fact's rel died). Expected-diff class, not a regression —
    but confirm rendered visuals exclude nulls before accepting.
  - **Audit the delete list against the engine graph BEFORE applying.** A bare `[teamid]`
    inside `FILTER(Pipeline,...)` in another calc table survived static qualified-ref
    integrity and broke recalc (EvaluationError cascading as Incomplete states). The
    engine-graph consumer check (rt/ro of every deleted object vs surviving consumers)
    finds these in seconds; running it post-hoc cost a Desktop cycle. Note: expanded-table
    noise exists (a calc column "using" an unrelated table's key) — formula text
    adjudicates.
  - **Bridge-table → boolean-column rewrites are provable**: express the old
    relationship-filter semantics (including blank-member inclusion for `<>` predicates,
    exclusion for `=`) as explicit set logic in one DAX query and count disagreement rows
    (must be 0). Evidence: `verification/bridge_equiv.dax`, 4 predicates, 0/0/0/0.
  - **Don't span midnight with fingerprint runs**: TODAY()-baked calc tables re-bake on
    recalc; a baseline taken before 00:00 and a comparison after produces legitimate
    volatile-class drift entangled with real changes (2 contracts crossed a C1 date
    boundary here). Run baseline+comparison inside one data-day, or prove semantics
    separately as above.
  - **Partition State decode + the silent recalc**: TMSCHEMA_PARTITIONS State 1=Ready,
    3=NoData (M cache dropped, awaiting refresh), 4≈calculation pending (a `calculate`
    refresh resolves it), **5=EvaluationError — THE ROOT CAUSE**, 7=Incomplete —
    cascaded victims of a 5. TMSL `type=calculate` can return "no errors" while
    partitions stay 5/7 (observed twice) — verify STATES after recalc, never trust the
    empty error list. Diagnose by reading the State=5 object's formula (here: a mapping calc
    table's bare `[teamid]` over a deleted fact column).
  - **Desktop cascades cache invalidation through M references**: pruning
    a detail dim's M also flipped its workbook-sourced M consumer to NoData —
    budget the refresh scope for consumers, not just the queries you edited.
  - **Slimming an M-consumed producer (refines Iron Rule 4)**: consumers re-evaluate the
    producer's M QUERY, not its stored table — so the producer's TMDL columns can drop
    BELOW what consumers read, as long as the M output (its SelectColumns list) keeps
    union(own TMDL columns, consumer-read columns). Evidence: a source table's
    `universal_name` TMDL column deleted while its M kept it for a consumer's
    join — refresh succeeded. Constraint: CALC columns on the producer pin their input
    columns in both TMDL and M (a parent-group lookup column needed
    `parent_id` + `alt_group_id`) — read their formulas before listing.
  - **Blank lines between top-level TMDL blocks are OPTIONAL** (myth-buster): a
    relationships.tmdl with all 13 inter-block blank lines removed parses fine (TOM
    19.84, 14/14 relationships). A load failure after block surgery is NOT the spacing —
    look for a semantic error and check you didn't poll the engine prematurely.
  - **Reachability ≠ necessity**: on a calc-table-woven model the keep/kill fixpoint
    keeps nearly everything (here: a single HIDDEN slicer visual chained to 5 tables and
    15 measures). Restructure first — sever chains — then trim. Hidden visuals are
    look-free deletion roots: deleting an invisible measure-selector slicer freed the
    whole subsystem with zero rendered change.
  - **Stale persisted selectors are harmless; retarget only moved fields**: visual.json
    keeps `columnWidth`/formatting selectors for long-removed fields (metadata:
    "Warehouses.name" etc.) — they don't block table deletion or rendering. But a
    field you MOVE must have its selectors retargeted with the projection or its width
    resets.
  - **Verify verification inputs' freshness**: fingerprint_build silently consumed a
    stale 3-page report_usage.csv because the extractor writes to CWD while the builder
    reads verification/ — the baseline covered a contract that no longer existed
    (harmlessly broader here, but the reverse direction would under-verify). Pin tool
    output paths; check the input's page list before building a query set.
  - **After a real source refresh, compare status-level; hash-identity is a bonus
    signal**: 209/318 hashes survived the refresh (stable dims/scenario metadata),
    109 hash-only diffs (fresh facts + TODAY re-bake), 0 status regressions = clean.
  - Windows/tooling additions: `Get-Process X -ErrorAction SilentlyContinue` still exits
    1 when nothing matches (breaks `$?`-chains — wrap in `@(Get-Process | Where-Object …)`);
    env-var-configured tools leak (`$env:OUT` set for inventory.py hijacked
    pbir_usage.py's output path) — `Remove-Item Env:X` between runs.

- **2026-07-17b (render-baseline capture before V3 rewrite):**
  - **Rendered-visual baseline = AMO `QueryEnd` server trace during a headless open**,
    not `$SYSTEM.DISCOVER_COMMANDS` (which keeps only the LAST command per connection —
    a 21-query render left 13 rows, mostly duplicates). Tool: `tools/capture_render_trace.ps1`
    (start it BEFORE Desktop; it waits for the new msmdsrv port, settles on idle).
    Trace trap: **QueryBegin rejects the Duration column** ("The event Id=9 does not
    contain the column Id=5") — subscribe QueryEnd only; it carries TextData too.
  - **Dropdown slicers fire NO query at page render** — they query on expand only.
    Evidence: 3 visible dropdown slicers absent from all 21 render events while every
    card/table/range-slicer fired. Baseline them with synthetic item-list queries cloned
    from the Desktop idiom (page filters + `NOT(ISBLANK([DisplayFilter]))`, no TOPN).
  - **Mapping captured queries to visuals: strip the filter scaffold first.** Page/
    field-param filter columns appear in EVERY query's DEFINE block and mis-assign
    naive ref-overlap (a card was scored as the slicer its filter referenced). Score on
    full-text refs minus refs present in ≥⅔ of queries; EVALUATE-only refs also fail
    (tableEx projections live in `VAR __DS0Core` inside DEFINE).
  - **Table visuals run one BATCH with two EVALUATEs** (min/max scan for bar scaling +
    the TOPN-windowed rows); the XMLA reply then nests roots in
    `<results xmlns=…xmla-multipleresults>` instead of `<return><root>` — parse both
    shapes or you silently get 0 resultsets. Keep BOTH resultsets in the baseline.

- **2026-07-15/16 (founding session, first refit):** all rules above. Headline evidence:
  whitespace "tidy" in M dropped 47k rows (refresh required); relationship dot-notation
  assumption deleted 7 endpoints (gate shared the broken regex and passed it); CRLF stripped
  by Python universal newlines across 30 files; `Team` bridge caught only by numeric
  fingerprint; `isAvailableInMdx` 75× plan cliff caught only by captured-query replay.
  Refit result: 248→54 measures, 591→345 columns, cache 139→62 MB, recalc 49.6→7s,
  heaviest visual 2.2→1.5s, 0 regressions across 319 checks.

### 2026-07-17 (V3 hybrid port session)

- **Quoted single-word table headers crash TOM/Desktop silently.** Generator wrote
  `table 'Account'` (name needs no quotes). TOM 19.x: "An internal error has occured"
  (Verify crash in MergeAndGroupChildObject — doc fails to match its `ref table Account`
  line, creating a phantom duplicate); Desktop rejects the apply invisibly and serves the
  old cache.abf schema. Evidence: exactly the 4 quoted single-word tables (Account,
  Opportunity, Contract, Scenario) were the bisect culprits; unquoting fixed the parse.
  Quote TMDL object names ONLY when they contain spaces/special chars.
- **The same "internal error" fires for refs to missing table docs** — a subset/bisect
  harness must delete `ref table` lines together with the docs, or every configuration
  fails and the bisect lies. Missing ROLE docs too (`ref role`).
- **queryGroup references need definitions.** Partitions carrying `queryGroup: Dynamics`
  with no `queryGroup Dynamics` block in model.tmdl → the same opaque merge error.
- **Cross-dialect rename corruption classes** (porting DAX+M with a table rename map):
  (1) DAX-quoted `'New Name'` leaking into M (M needs `#"New Name"`); (2) M record field
  access `[Alias.Col]` — expanded-column names with the join-alias prefix must NEVER be
  renamed: `[#"New".Col]` is invalid M ("Token Literal expected"); (3) bare `[col]` refs
  in row context break when the column was renamed but a same-named unmapped column on
  another table suppressed the safe-bare rule. Renames must respect string literals,
  partition-header lines, and be line-ending-agnostic (`\r?\n` — mixed-EOL files silently
  skip `\r\n`-anchored regexes).
- **Pinpointing bad M without Desktop dialogs:** TOM in-memory bisect — deserialize once,
  set each suspect script to `let x = 1 in x` (partition `Source.Expression` / expression),
  throwaway-deploy per subset. 35 scripts localized to 2 culprits in ~70 deploys/minutes.
- **Engine recalc warnings name the missing columns** (`Column X cannot be found`) —
  much faster than eyeballing; fix roots (the calc TABLE partitions at State 5), the
  "in table 'DQ Items'" errors on dependents are cascade.
- **PQ-first architecture verdict (hybrid retreat):** moving cross-entity joins/aggregates
  into M multiplied refresh memory ~4×+ vs the same logic as engine-side calc columns/
  tables (mashup containers stream uncompressed; every loaded table re-evaluates its full
  chain; Table.Buffer pins whole entities per container). Same data/hardware: V2 shape
  refreshed fine, V3 PQ-first OOM-thrashed 3×. Keep M row-local (trims, typing,
  conditionals, small-dim fusions); keep lookups/aggregations/set-logic in DAX.

### 2026-07-17 (T10 verification of the V3 hybrid)

- **PBIR selectors are rename-map surface.** Rebinding projections is not enough:
  conditional formatting (icons/backColor/webURL) and columnWidth attach via `selector:
  {metadata: "Table.Object"}` STRINGS. Stale strings silently no-op (Desktop keeps the
  JSON, the rule just never matches) — the RAG icons vanished with zero errors anywhere.
  Translate every `metadata` string through the rename map, including aggregation-wrapped
  forms like `CountNonNull(T.C)`. Ancient truly-dead selectors are harmless; translate
  what the map covers, leave the rest.
- **Renaming can flip DAX bare-ref binding.** Before: `FILTER(ALLNOBLANKROW(Countries),
  Countries[country_id] = [countryid])` — the bare ref could only bind to the outer
  (fact) column. After renaming both sides to `Country ID`, the bare ref bound to the
  ITERATED table instead: X=X, true everywhere, MAXX returned max ISO2 ("ZZ") on all
  10,720 rows. Hoist outer values into VARs before any iterator whenever a rename makes
  column names collide across tables. Grep ported DAX for `= [` after renames.
- **Retargeting scripts corrupt string literals.** A TREATAS filter value "Pipeline" got
  table-renamed inside the quotes -> query silently returned 0 rows. Verification queries
  are code too: value-diff them against the baseline BY VALUES (alias names differ by
  design), and treat a 0-row resultset as a query bug before a model bug — the rendered
  visual (user screenshot) adjudicates which side is wrong.

### 2026-07-17 (table folding batches)

- **Partition type cannot change in place.** Desktop: "Changing the partition type from
  or to PartitionType.Calculated is not allowed; partition 'RLS' of table 'RLS'".
  Converting calc<->m requires RENAMING the partition (drop old + add new identity).
- **M cache invalidation is TRANSITIVE, refining Iron Rule 1.** The cache key is the
  effective inlined query closure, not just the partition's own text: fusing a small
  detail dim invalidated a workbook-sourced accounts table (references it), a revenue
  extract (references the changed dim) AND the opportunity fact (three hops away via a
  shared workbook expression) though none of their texts changed. Converting a table to a same-named
  expression WITHOUT changing its M does NOT invalidate consumers (Batch A: all Ready).
- **Table->expression conversion is the free fold.** M-consumed-only staging tables
  (no DAX refs, no rels) become shared expressions under the same query name: consumer
  M text and caches untouched, zero refresh cost. DAX-consumed tables must stay loaded.

### 2026-07-18 (Account-to-M fold)

- **Local M step names must never equal document member names.** A step named like its
  own query ("Master Account Dict" containing step #"Master Account Dict", introduced by
  table renames hitting inner identifiers) is tolerated in TABLE queries but detonates
  when the query becomes a SHARED EXPRESSION: the engine resolves the step as the member
  -> self-cycle -> whole mashup document poisoned with misattributed "cyclic reference"
  and "import X matches no exports" on unrelated queries. Scan step DEFINITIONS against
  the member-name set after any rename; record fields ([#"X" = ...]) are exempt.
- **Refresh errors reproduce locally without credentials.** Document-level errors (cycles,
  unresolvable imports) surface on a throwaway-deploy + RequestRefresh probe BEFORE auth;
  reaching "The key didn't match any rows" (dataflow auth) IS the local success signature.
  Fast bisect: probe with Account source swapped for `let a = #"X" in a` per import.

## Folding ladder (model refactoring playbook)

Proven on a 31→19-table fold with every rung value-gated. Order candidates by
REFRESH COST, then per-table decide by asking: **who reads it?**

| Consumer kind | Detection | Consequence |
|---|---|---|
| DAX (calc tables/columns, measures) | grep `'Table'[` in DAX + engine DISCOVER_CALC_DEPENDENCY | Must stay LOADED — DAX cannot read unloaded queries. Fold only after converting the DAX consumers themselves |
| M (other queries reference it) | grep bare/`#""` refs in partitions+expressions | Convert to shared expression under the SAME query name: consumer M text unchanged → caches survive → ZERO refresh |
| Relationships / RLS / frontend | relationships.tmdl, roles, PBIR usage | Blockers until retargeted |

Rungs, cheap → expensive:
1. **Free (no refresh):** M-consumed-only staging → expression (same name!);
   embedded static lists → inline DAX constants (decode the Base64/Deflate blob to
   get the literal values); single-consumer calc tables → inline VAR in the consumer
   column. Hoist any per-row inner CALCULATE onto the iterated table first or the
   recalc goes pathological (a naive StageDuration inline never finished; hoisted to
   a rel-filtered Opportunity calc column it settled in seconds).
2. **One refresh:** small-entity M fusions (Country + Country Detail); calc→M
   partition conversions (RENAME the partition — type change in place is rejected).
   The refresh bill = the transitive M-closure of every changed query, so BATCH all
   rungs of this class into one refresh.
3. **The unlock move:** a big calc table (Account) converted to M dissolves every
   data-carrier table it was pinning (five at once) — but port its calc columns and
   respect the memory split: M = row-local + small hash joins; DAX calc columns =
   big-entity aggregation over loaded tables. Full-scale cross-entity M (the
   PQ-first architecture) OOM-thrashed an 8GB machine three times; the same logic
   engine-side refreshed fine.

## DAX↔M translation fidelity

Every one of these diverged in production; snapshot-diff caught them:

| DAX behavior | M default | Faithful M |
|---|---|---|
| Text equality ignores case AND trailing spaces | ordinal | `Text.Upper(Text.Trim(x ?? ""))` both sides — one missing Trim dropped 17k of 19k lookup matches |
| `IN {..}` / `CONTAINSSTRING` case-insensitive | case-sensitive | pre-uppercased ordinal lists / `Text.Contains(x, s, Comparer.OrdinalIgnoreCase)` |
| `LOOKUPVALUE(col, key, BLANK())` MATCHES blank-key rows | null joins nothing | Decide intent first: the blank-key match fed a junk dictionary row to 17k accounts — the DAX was the bug; the M "miss" was the fix |
| number↔text coercion in comparisons/UNION variant columns | typed | `Text.From` at the projection; expect representation diffs (serial 4.38e4 vs typed date) that are upgrades, not regressions |
| `LOOKUPVALUE`/`Table.First` single match on dup keys | `Table.Join` MULTIPLIES rows | `Table.Distinct(lookup, {key})` before the join |

**Acceptance:** EVALUATE-export the loaded table before the swap (`EVALUATE T ORDER BY key`,
save the rowset XML); after the refresh, per-column diff with a DIRECTION cross-tab:
classify each mismatch by (old==old-fallback?, new==new-fallback?). The four cells
separate: translation misses (False,True — new falls back where old matched), old-side
defects (the reverse), content drift (False,False), and noise. Sampling one full row
old-vs-new adjudicates fastest.

## PBIR frontend fidelity surface

The report look attaches to the model by NAME STRINGS at these points — every rename or
rebind must chase ALL of them, and misses are SILENT no-ops (nothing errors; the look
degrades):

1. Projections: `queryRef` + `nativeQueryRef` captions (incl. single-space captions
   that hide a column header).
2. `selector: {metadata: "Table.Object"}` on: conditional formatting (icon/RAG rules,
   backColor, fontColor incl. bars-only number hiding, webURL), columnWidth (incl. the
   near-zero widths that hide columns), columnFormatting alignment, dataBars.
   Selectors must equal the visual's OWN projection strings — per visual, not per
   rename map: rebind tooling left `CountNonNull(Old.Col)` labels untranslated, so a
   uniformly-translated selector broke in the opposite direction (doubled counts,
   lost widths). Align to whatever string the projection actually uses; try
   old/new/aggregation-wrapped variants.
3. Filters (visual/page/report incl. pane card order), sorts, field-parameter NAMEOF
   lists, bookmarks, RLS role tablePermissions, drill/tooltip bindings.
4. Ancient dead selectors (pre-carve-out tables) are harmless — leave them; translating
   only map-covered strings reproduces the old semantics exactly.

Audit recipe: for each visual, set(projection queryRefs) vs every selector metadata;
report selectors that name CURRENT-model objects but match no projection. Rendered
screenshots (user-supplied) are the only ground truth for "did the dots come back".

### 2026-07-18 (fold ladder finale — evidence log)

- Calc columns on tables converted to expressions vanish and fail ONLY at the
  credentialed refresh: a parent-group id (2-level parent-chain
  lookup) and `rn` (per-ID max-Vertical/max-country dedupe flag) each cost a round
  trip. Sweep `column X =` on every table before converting; port into consumers' M.
- Trailing-space semantics: Master Segment matches old 18,973 vs new 1,837 rows
  ≠ Industry before Trim; direction cross-tab after fix: {(F,T):17,161 = old blank-key
  matches (OLD defect), (T,F):29 + (F,F):10 + (T,T):2 = dictionary drift}.
- NestedJoin over Table.Buffer = nested loop: 3,860 rows loaded in ~10 min, CPU 20%,
  memory 59%, disk 1%. Table.Join LeftHash + Distinct: same chain in seconds.
- Comparer-lambda membership: 92k rows × 4 × ~200-item lists ≈ 70M invocations;
  pre-uppercased buffered lists made it native.
- Local refresh probes stop at the FIRST credentialed source: "key didn't match any
  rows" = local success, but column-not-found/join-key errors past that point only
  surface on the credentialed machine. Static sweeps are the countermeasure.
- Pooled shells give nondeterministic assembly state — run TOM probes via
  `pwsh -NoProfile -File script.ps1` (fresh process) when LoadFrom/type resolution
  flip-flops.
- Refresh triage by resource signature: memory+disk pegged = container thrash
  (parallelism/buffers); CPU-busy + memory-healthy + slow rows/min = algorithmic
  (join/lambda); instant block = document/static error (reproduce locally).

### 2026-07-20 (model.tmdl ref lines are Desktop-cosmetic, not load-bearing)

- Hand-added a `ref table` line + a PBI_QueryOrder entry for a new
  M table; Desktop refreshed/saved the PBIP and its save REMOVED both while the
  table stayed fully loaded (data present, engine queries fine). TMDL folder
  deserialization (TOM `DeserializeDatabaseFromFolder` and Desktop alike) discovers
  tables from `tables/*.tmdl` regardless of `ref` lines.
- Rule: when ADDING tables to a PBIP model, write only the table file; touch neither
  the ref list nor PBI_QueryOrder (matching what Desktop's next save would produce).
  The Quick-reference row about removing refs applies to DELETION only (a stale ref
  to a nonexistent file is what breaks parses, not a missing ref to an existing one).
- Same session: old bundled TOM (16.0.117, TE2) rejects `ref cultureInfo` — control-
  parse the untouched sibling model to separate tool limitation from real defect;
  validate by scratch-copying the definition minus cultures + that one line.

### 2026-07-21 (parallel-model page clone onto a new source, zero-defect first render)

Task: replicate a report page against a brand-new source (Fabric lakehouse view) as a
parallel model layer, legacy untouched. Worked first try after offline verification;
evidence and reusable recipe:

- Live-engine prototyping of calc tables: ran the exact UNION/SELECTCOLUMNS/LOOKUPVALUE
  expression as EVALUATE on the user's open Desktop session BEFORE writing TMDL, with
  DATATABLE/SWITCH stand-ins for mapping tables that didn't exist yet. Caught semantics
  early; prototype total matched an independent hand-computed FX
  cross-check to 4 significant digits. Gotcha: SUMMARIZECOLUMNS cannot group by var-table columns
  (returns EMPTY, no error) — use GROUPBY(...CURRENTGROUP()) for validation aggregates.
- nativeQueryRef is the preservation key: rebind changed queryRef + Entity/Property but
  kept nativeQueryRef verbatim. Zebra BI visuals' embedded state (cardTitle blobs)
  references those aliases; all Zebra formatting survived without editing opaque blobs.
- Slicer selection state lives in visual objects.general[].properties.filter (a full
  Version-2 filter with From/Where/Literals), not in filterConfig. Default selections =
  literal swaps there. The filterConfig entry for a slicer is just a hidden placeholder.
- Date-hierarchy slicer trap: swapping the HierarchyLevel projection to a plain int
  Column works, but the slicer's SELECTION filter still referenced
  LocalDateTable_<guid> in From[] AND in Annotations.filterExpressionMetadata (with a
  valueMap {"0": "2025"} display string). A distinct-Entity audit over the copied page
  caught it; grep/eyeballs had missed it. Audit = collect every "Entity" value
  recursively; assert subset of intended tables.
- Verification stack that made first-render clean: (1) TOM parse of model scratch copy
  (culture stripped for old TOM); (2) dump model field inventory via TOM to JSON;
  (3) resolve every page binding (Column/Measure nodes + queryRef prefixes) against it —
  49/49 resolved; (4) folder-name == visual.json name; (5) pages.json referential check.
- Second calculation group coexisted with the legacy one after setting precedence: 1
  explicitly (legacy group has none = 0); model loaded and both groups' items worked
  (verified via TREATAS-emulated end-to-end measure test).
- Diagnosis pattern for "selector not working" (user report, deltas all 0.0): FIRST
  sweep the fact per-snapshot (SUMMARIZECOLUMNS by DataSet) — all 22 snapshots byte-
  identical for the filtered category; THEN emulate selections end-to-end
  (CALCULATETABLE + TREATAS on both selector tables + switch table) — A/B differed materially
  with no category filter. Verdict: machinery fine, data flat in that slice.
  Exhaustive mixed-sign sweep (231 snapshot pairs x category x year x gross/weighted)
  proved no red+green combo existed yet — reported as data fact, not bug.
- Invoke-ASCmd XML: property-style access ($row.'colname') fails on typed/nil elements
  ("Cannot convert XmlElement to Double") and attribute enumeration returns xmlns
  noise. Robust: iterate $_.ChildNodes LocalName/InnerText, or save raw XML and parse
  in Python (ElementTree, namespace urn:schemas-microsoft-com:xml-analysis:rowset).

### 2026-07-21 (missing business attribute — dataflow export closes the gap without a probe refresh)

Request: "filter out prospect accounts" from a DQ report whose model had NO account-type
field anywhere. Resolved same-session with one user refresh. The reusable procedure:

- **Prove absence at engine level before designing.** Enumerated every column of every
  table via TMSCHEMA_COLUMNS + TMSCHEMA_TABLES on the live Desktop engine (not just
  TMDL grep — calc tables' inferred columns don't all appear as named blocks). Also
  killed the plausible proxy definition with data: "account without an ERP number =
  prospect" fell because zero such accounts had revenue under ANY of three revenue
  columns (a 92k-account model). Report the definition gap; don't guess a proxy.
- **The upstream dataflow's exported .json is the decisive schema oracle, no refresh
  needed.** Have the user download it from the service (dataflow → Export .json). Its
  `pbi:mashup.document` holds the complete M of every entity, including native SQL
  passed to Value.NativeQuery — there we found the source already SELECTed the
  account-type option-set label (LocalizedLabel joined from OptionsetMetadata) that the
  model's own SelectColumns was dropping. Outcome flipped from "vendor must extend the
  dataflow" to "local M edit + one refresh".
- **Filter-by-label edits: verify the exact label AFTER refresh** with a distinct-values
  sweep (SUMMARIZECOLUMNS on the new column). Here the guessed label matched verbatim;
  if it hadn't, the calc-table filter would silently no-op (DAX equality forgives case
  and trailing spaces, nothing else).
- Calc-table exclusion shape that validated cleanly: build the union as before, then
  `VAR ids = SELECTCOLUMNS(FILTER(ALLNOBLANKROW('Dim'), <label test>), "@k", key)` and
  `FILTER(__base, NOT([Type]="X" && [Key] IN ids))` before the final ADDCOLUMNS —
  ALLNOBLANKROW per the blank-row circular-dependency rule.
- Environment recovery: Desktop's own AdomdClient DLL under `C:\Program Files\
  WindowsApps\...` is ACL-denied even for reading (Add-Type "Access is denied"), and no
  SqlServer module existed in either PowerShell edition. `Install-Module SqlServer
  -Scope CurrentUser -Force` from PSGallery restored Invoke-ASCmd in ~1 min — faster
  than any DLL-copy workaround; matches what the repo tools already import.

### 2026-07-21 (hand-authored report-level filter replaces a calc-table hard exclusion)

Follow-up on the same request: after shipping the exclusion INSIDE the calc table, the
user asked for it as a visible report-level filter in the pane instead. Both learnings:

- **"Filter out X from the report" usually wants a PANE filter, not a hard exclusion.**
  Convert, don't stack: a hard calc-table filter under a pane filter makes the toggle
  lie (unchecking the pane card restores only the rows the calc table kept). Remove the
  hard filter when the pane filter lands; verify parity by emulating the pane filter
  engine-side (`CALCULATE(COUNTROWS(T), T[Col] <> "X")`) and matching the old hard-
  filtered counts exactly (here: same 7,114 for the key scenario). Bonus of the pane
  route: it naturally covered sibling record types the hard filter had deliberately
  skipped (886 rows total vs 7) — and that widening is one visible click to undo.
- **PBIR report-level filter, authored by hand, accepted by Desktop first try** —
  minimal working shape inside report.json `filterConfig.filters[]`:
  `{name: <unique-hex>, displayName, ordinal: 0, field: {Column: {Expression:
  {SourceRef: {Entity}}, Property}}, type: "Categorical", filter: {Version: 2, From:
  [{Name, Entity, Type: 0}], Where: [{Condition: {Not: {Expression: {In: {Expressions:
  [{Column: {Expression: {SourceRef: {Source}}, Property}}], Values: [[{Literal:
  {Value: "'X'"}}]]}}}}}]}, howCreated: "User", objects: {general: [{properties:
  {isInvertedSelectionMode: {expr: {Literal: {Value: "true"}}}}}]}}`.
  `isInvertedSelectionMode: true` is what renders the card as "is not X" (matches how
  Desktop saves an excluded categorical selection). NOT-IN keeps blank/null rows —
  correct default when the column is a lookup that can miss. The filter column was
  added to the calc table itself (lookup via ALLNOBLANKROW MAXX-FILTER), so no
  relationship path to the filtered visuals is needed.
- Calc-table-only DAX edits (new ADDCOLUMNS output column, removed FILTER wrapper) plus
  a report.json edit needed NO refresh — reopen recalculated from loaded data. The full
  cycle: user saves+closes Desktop → edit files → offline TOM parse + JSON parse →
  relaunch pbip → poll new port/catalog → poll until the new column answers a DAX probe
  (cache.abf's old schema answers first) → verify counts → user eyeballs pane and saves.


### 2026-07-22 (field-parameter retarget + period-comparison measures on a survey dashboard)

Task: on a survey-dashboard PBIP, fold blank axis members into an "Others" category
(field-parameter-driven chart) and add previous-period comparison cards driven by a
timeline slicer's quarter/year granularity. Four learnings, all verified live:

- **Retargeting a field parameter has THREE attachment points, and the third fails
  silently as a single collapsed bar.** Changing a Selector-style calc table's
  `NAMEOF` entry requires updating (1) the calc-table source, (2) the bound visual's
  Category projection (Property/queryRef), AND (3) the parameter SLICER's stored
  selection — the `'''Table''[Col]'` literal appears TWICE in its visual.json (the
  `Where...In.Values` literal and `Annotations.filterExpressionMetadata.
  decomposedIdentities`). A stale literal filters the parameter table to zero rows and
  the chart renders ONE unsegmented total bar — zero errors anywhere. Grep the report
  for the old NAMEOF literal (`Table''\[Col\]`) after any parameter retarget.
- **SWITCH/IF branches cannot return table expressions, and TMDL won't tell you.** A
  measure with `VAR w = SWITCH(TRUE(), cond1, DATESBETWEEN(...), ...)` loads fine and
  fails only at query time: `Calculation error in measure '...': The True/False
  expression does not specify a column`. Compute scalar start/end dates in the SWITCH,
  then one `DATESBETWEEN(col, start, end)`.
- **A fact-bounded calendar breaks whole-period detection at the data edge.** With
  `CALENDAR(MIN(fact), MAX(fact))`, a timeline's whole-quarter selection clips to the
  last fact date (Apr 1–Jun 30 became Apr 1–Jun 18), so `MAX = EOMONTH(...)` tests fail
  for the current period — the most common selection. Accept bounds equal to the
  calendar's global MIN/MAX (`CALCULATE(MIN/MAX('Calendar'[Date]), ALL('Calendar'))`)
  as period-aligned, and anchor the shifted window at the PERIOD START of the selection
  min (`DATE(YEAR(_minD), 3*QUARTER(_minD)-2, 1)`), not at `_minD` itself (a clipped
  start would shift the window mid-month).
- **To screenshot-verify visuals that are blank in the default state** (e.g. comparison
  measures gated to a sub-period selection), temporarily add a hand-authored PAGE-level
  Advanced date filter to page.json (`filterConfig` with `Version: 2`, two `Comparison`
  nodes, `ComparisonKind` 2/4, `datetime'YYYY-MM-DDT00:00:00'` literals — accepted by
  Desktop first try), render, screenshot, then remove it. Beats fighting a WebView
  custom slicer with synthetic clicks (timeline cell clicks in edit mode never
  registered).
- Practical minimum height for a hand-added value-only cardVisual: an 18px-tall card
  with 8pt text rendered NOTHING (fully on-page, measure verified non-blank); the same
  card at 24px rendered. Treat ~24px as the floor.
- Overlay pattern for dual-audience cards: a cloned card at identical geometry, higher
  z, measures gated to the complementary audience (`NOT [HasRLS]`), `showBlankAs ' '`,
  same tooltip section — each audience sees exactly one populated layer, hit-testing
  and tooltips stay intact.

### 2026-07-22 (round 2 — new-card text limits, RLS detection vs slicer context)

- **The new card (`cardVisual`) cannot render multi-line text — three failures measured:**
  (1) a `wordWrap` literal on the `value` object is silently ignored (text clips with
  an ellipsis); (2) `UNICHAR(10)` in the measure text renders as a SPACE, not a line
  break; (3) stacking N text measures on one card (`columnCount 1`, generous height,
  `cellPadding 0`) rendered only the FIRST tile plus a gray overflow pill — tiles 2..N
  never appeared at any height tried (120→182px for 5 tiles). The reliable pattern for
  a multi-line dynamic footnote: N separate single-measure cards at ~24px vertical
  pitch, one short line-measure each (`'Note 1'..'Note N'`, each `IF`-switched on the
  audience). Bonus: per-line measures also give exact typographic control.
- **RLS detection by comparing filtered rowcounts conflates slicer filters with RLS.**
  `COUNTROWS(Dim) <> SUM(TotalDim[Count])` flips TRUE the moment a user slices the
  region hierarchy, silently switching audience-gated visuals (comparison rows,
  scope footnotes) into the RLS persona. Fix: evaluate both sides under
  `CALCULATE(..., REMOVEFILTERS())` — REMOVEFILTERS cannot undo row-level security,
  so the measure then detects ONLY genuine RLS. Verified: with a region filter +
  quarter window, the RLS-gated reference row stayed blank and the global-audience
  deltas computed over the filtered region (prev-quarter values matched direct
  evaluation exactly).

### 2026-07-22 (follow-up — HTML Content visual supersedes card hacks for dynamic text)

The N-stacked-single-line-cards workaround (above) rendered but the user judged it
"extremely wonky" (uneven rhythm, no wrapping). The right tool: the certified **HTML
Content (lite)** custom visual (`htmlContent443BE3AD55E043BF878BED274D3A6865`) —
one measure returning an HTML string, natural word wrap, full typographic control.
Hand-authoring the instance works first try:
- queryState role key is **`content`** (internal role name from the visual's
  capabilities.json — displayName is "Values"; guessing "Values" as the key would
  bind nothing). When a custom visual's package isn't cached locally, its
  capabilities.json on the vendor's public repo is the fastest role-name ground truth.
- The report must list the visual id in report.json `publicCustomVisuals` (adding the
  visual once via Desktop's store UI does this; hand-adding untested).
- Inline `style` attributes (font-family/size/color/line-height) and `<br/>` survive
  the certified sanitizer and render as authored.
- DAX side: escape inner double quotes by doubling, or use single-quoted HTML
  attributes.

### 2026-07-22 (round 3 — "compare with previous period" means LATEST period, not the whole selection)

User: "why is it not changing?" — the deltas were blank on the DEFAULT view. Cause: I
had implemented previous-period comparison as whole-selected-window vs the equal
window before it; with the default selection covering the entire data range, the
prior window is always empty, so the flagship state showed nothing. The mockup showed
deltas in exactly that state. Correct reading of "Quarter → compare against the
previous quarter": **latest whole period inside the selection vs the one before it**
(Q2 vs Q1 even when Q1–Q2 is selected) — headline totals still describe the full
selection; the small delta describes the latest period. Implementation that keeps it
maintainable: four hidden date-valued helper measures (Cur Start/End, Prev Start/End)
hold ALL the alignment logic once; every consumer is one line —
`CALCULATE([X], DATESBETWEEN('Calendar'[Date], [Prev Start], [Prev End]))` (measure
refs as DATESBETWEEN bounds are fine). Year-vs-quarter disambiguation when both align:
year mode only if the selection spans ≥4 quarters. Color measures should derive from
the display string's arrow character (`LEFT(s,1) = UNICHAR(9650)`) instead of
recomputing the delta — one source of truth, invertible per KPI (risk metrics: up=red).

### 2026-07-22 (round 4 — final comparison rules + card text positioning lever)

- Requirement converged after two rounds of feedback; final rules worth pattern-izing
  for "period comparison" asks: show the delta ONLY when the selection is exactly one
  whole period (multi-period, sub-period, or no prior data → blank), and when a
  geo/dimension filter narrows scope, show BOTH the global-reference figures (original
  divisional row) and the same-scope previous-period delta as stacked lines. Gate via
  a single hidden date-valued 'Prev Start' measure that returns BLANK unless the
  selection is a single (edge-clipped) quarter — every consumer then blanks for free,
  and even the footnote can key its comparison sentence off it.
- **Positioning text inside a hand-built new-card: use `visualContainerObjects.
  padding.top` (…D literal), NOT `cardCalloutArea.paddingTop`.** Measured: on a cloned
  card, callout paddingTop 26L rendered identically to 4L (text did not move), while
  container padding.top 26D shifted the text down exactly 26px. Also: shrinking a
  card to make room (h 61→26) clips the value to a few-pixel sliver — keep the
  original card height and move the TEXT with container padding; transparent
  full-height overlaps are harmless.
- Stacked dual-audience slot recipe (verified rendering): original row text pulled up
  via its callout padding (24→4), overlay clone keeps identical rect, container
  padding.top ≈ 26 puts its line directly beneath; per-line font 11/10pt.

### 2026-07-22 (round 5 — HTML flex row replaces overlay-card gymnastics for KPI sublines)

User verdict on the stacked two-row card solution: "ugly". Final pattern that won:
ONE HTML Content (lite) visual per zone rendering "reference-figure · delta" SIDE BY
SIDE with per-part colors — impossible in native cards (single fontColor per value),
trivial in HTML spans.
- KPI row: a full-width visual with `display:flex` and one `flex:1;text-align:center`
  cell per KPI reproduces the multi-card grid alignment exactly; each cell concatenates
  a blue reference span and a conditionally-colored delta span (hex measures reused
  directly in the style attribute). Blank parts vanish via `IF(NOT ISBLANK(...))`
  concatenation — the same measure serves deltas-only, refs-only, both, or empty.
- Outer div needs `margin:0;overflow:hidden;line-height:1.2` or the visual grows
  scrollbars/clips descenders at tight heights; give the visual ~1.6× the font size in
  height.
- Originals stay recoverable: `"isHidden": true` (top-level visualContainer property)
  parks the replaced native cards instead of deleting them.
- Group-child slot → absolute coords for a replacement visual: group position + child
  position (groups translate only); verified across two group-nested slots.
- Blind `sed 's/"Value": "4L"/.../'` on a visual.json hit `columnCount` as collateral
  (rendered fine only because the visual was hidden) — property values are NOT unique;
  always anchor Edit on the property name block.

### 2026-07-22 (round 6 — "wrong item split" report: visuals hide identical duplicate rows; per-line latest-version dedup mixes document versions)

A business user reported a fact row (one part, qty 200) contradicting the ERP invoice
(two parts, qty 100 + 100). Two independent traps, both verified against the live
Desktop cache (when the ADOMD DLL isn't locatable — WindowsApps ACLs —
`System.Data.OleDb` with `Provider=MSOLAP;Data Source=localhost:<port>` works fine
from PowerShell for DAX EVALUATE and DMVs):
- **Visuals aggregate identical rows invisibly.** The loaded table held TWO
  byte-identical rows (same part, qty 100, same price) — the table visual rendered
  them as one qty-200 row. EVALUATE the loaded rows for the document key BEFORE
  reasoning from a screenshot; a "single wrong row" may be N identical rows summed.
- **Per-(order, line) latest-document-version dedup mixes versions.** The feed kept,
  per (order id, line number), the row from `List.Max(confirmation document ref)`.
  If version 1 had lines {1: partA, 2: partB} and the final version has a single
  renumbered line {1: partB}, the output is partB twice — line 1 from the new version
  plus line 2 surviving as a ghost of the old version. partA and its revenue vanish
  silently while totals stay plausible. Dedup at DOCUMENT grain (pick the winning
  document reference per order, then take all its lines), or source invoiced orders
  from invoice lines.
- A fleet audit for the signature (group the loaded fact by doc+part+qty+price,
  count ≥ 2) is only an UPPER bound when the line-number column isn't imported —
  legitimately repeated lines collide with true duplicates. Import the position id
  if duplicate audits matter.
- CONFIRMED against the raw export: six confirmation versions; versions 1–5 had
  {line 1: partA, line 2: partB}, the final version a single renumbered
  {line 1: partB}. Per-line max-doc picked line 1 from the final version + line 2
  from version 5 — reproducing the loaded duplicate byte-for-byte. Extra twist:
  even "all lines of the newest confirmation" (the obvious fix) still mismatched
  the invoice, because the final confirmation itself dropped a line that WAS
  invoiced — for invoiced orders only the invoice/order-line truth reconciles;
  the current order lines (status Invoiced) matched the invoice exactly.
- Auth recipe that got the raw export headlessly: tenant blocked first-party
  Azure CLI tokens for SPO REST (x-ms-diagnostics `3001003 "App is not allowed to
  call SPO with user_impersonation scope"`), but the SAME refresh token redeemed
  for `graph.microsoft.com/.default` worked via the Graph drive API
  (`/sites/{host}:/sites/{site}` → `/drive/root:/path:/children` → `:/content`).
  A user-run device-code script persisting the token bundle to a local file keeps
  the agent out of the credential path (permission classifiers rightly block
  agent-side token persistence).


### 2026-07-27 (freezing a retiring SQL landing zone into a one-time Gen1 dataflow)

Task: split frozen historical data out of a live import model into a constant Gen1
dataflow, authored as an importable `model.json`, with the model re-pointed via
`PowerPlatform.Dataflows` navigation-only swaps (step names kept; all consumer M
byte-identical). Three service behaviors, each proven by a verbatim failure:

- **Declared entity attributes are an ENFORCED projection, not documentation.**
  Refresh failed with `Expression.Error: The column 'X' of the table wasn't found`
  for a column that existed only in the contract (it was real on a sibling branch,
  guessed onto this one). The service conforms query output to the attribute list.
  Never declare a column you cannot prove exists — or make the query self-conforming
  (below).
- **Untyped columns break SAVE, not refresh.** `Table.SelectColumns(...,
  MissingField.UseNull)` yields type-`any` columns for physically-missing names;
  saving then fails with `DataflowObjectModelTypeNotSupportedException` ("Can't save
  dataflow"). Every column must end explicitly typed.
- **Dataflows materialize every column; Desktop does not.** A raw date column whose
  garbage cells Desktop never evaluates (dropped downstream = lazy skip, plus
  model-level `returnErrorValuesAsNull`) failed the dataflow refresh with
  `DataFormat.Error: We couldn't parse the input provided as a Date value`.

Battle-tested snapshot-entity query shape (fixed all three at once, refresh green
first try afterwards):

1. navigation;
2. `Table.SelectColumns(Raw, {<declared list>}, MissingField.UseNull)` — a missing
   physical column becomes a null column, which is exactly what the model's
   `Table.Combine` union produced for it anyway;
3. `Table.TransformColumns` with `each try Date.From/Number.From/Int64.From(_,
   "<model sourceQueryCulture>") otherwise null` + type ascription for every
   date/number/int64 column (parity with Desktop `returnErrorValuesAsNull`);
4. `Table.TransformColumnTypes(..., {every remaining column, type text}, <culture>)`.

The attribute array must match the SelectColumns list name-for-name AND
order-for-order. Export-dialect facts (verified against a working service export):
`queriesMetadata` is a MAP keyed by query name, `loadEnabled` present only when
true, `entities[]` = exactly the loadEnabled set, `pbi:QueryGroups` annotation is a
double-encoded JSON string, entity `pbi:refreshPolicy` = FullRefreshPolicy with
URL-encoded `<name>.csv` location; OMIT `partitions` and `connectionOverrides` on
import (service-written). On import the service assigns its own dataflowId — the
model-side `dataflowId` must be read back from the created dataflow's URL.

Headless Power BI REST verification recipe (worked in a tenant that blocks the
Azure PowerShell first-party client, AADSTS700016; `/organizations` endpoint gave
AADSTS50059 — use the tenant-named endpoint): device-code flow with the Azure CLI
public client id (`04b07795-8ddb-461a-bbee-02f9e1bf7b46`), scope
`https://analysis.windows.net/powerbi/api/.default`; poll + query in ONE process,
token in-memory only, never persisted. `GET groups/{ws}/dataflows` lists ids;
`GET .../dataflows/{id}` returns the LIVE model.json — per-entity `partitions[]`
presence (with refreshTime) is the materialization proof, and
`.../dataflows/{id}/transactions` gives refresh status. This turns "did the
one-time refresh actually complete, per entity?" into a checkable fact.

### 2026-07-27 (round 2 — cross-entity computation inside a Gen1 dataflow backfires)

Adding a `Table.Group`/join over a LOAD-ENABLED sibling entity inside a Gen1 dataflow
query turned that sibling into a computed-entity SOURCE. On the next refresh the
*source* entity — whose own query was untouched and had refreshed green hours
earlier — failed with `Expression.Error: We cannot convert the value 1 to type
Text.` The enhanced compute engine evaluates computed-entity sources with SQL-strict
type semantics; implicit numeric-vs-text coexistence that pure-M evaluation tolerated
(numeric keys feeding text concatenation/comparison) becomes a hard error, attributed
to the source entity, not the query you edited.

Rule: when a dataflow entity needs an aggregate of another load-enabled entity,
prefer computing it in the CONSUMING MODEL's M — the model's entity read has no
enhanced-compute-engine involvement, and the edit goes through the model's own
verification pipeline. Reverting the dataflow query and moving the identical
join/group/compute into the model's shared expression fixed it first try (validated
per-row against the live Desktop engine via `System.Data.OleDb` + `Provider=MSOLAP`:
computed amount = pct x grouped base, exact to the cent).

Related trap from the same investigation: an ERP exports charges with BOTH a
fixed-amount and a percentage column; percentage-configured charges carry 0 in the
fixed column. A feed that maps "value" from the fixed column silently zeroes every
percentage charge — while a sibling fixed-type charge from the same file flows fine,
which is the diagnostic signature (one charge type present, the other absent, same
join). Also: two charge rows per document expanded through a NestedJoin then deduped
by `Table.Distinct` on the document key drop one charge at random — collapse charge
tables to one row per document BEFORE joining headers.

### 2026-07-28 (a `queryGroup:` reference needs a queryGroup OBJECT in model.tmdl)

Repeating the historical-dataflow swap on a second report's model: added an
expressions.tmdl whose shared expression carried `queryGroup: Historical` — copying
the DONOR model's expression block but not noticing the donor also declares, in
model.tmdl right after the model header:

```text
queryGroup Historical

	annotation PBI_QueryGroupOrder = 8
```

Desktop refused to open the project: "Cannot resolve all the paths while
de-serializing Database. Resolution Errors: Property QueryGroup of object
"expression HistoricalSource" refers to an object which cannot be found." The trap
compounds because a grep for the `PBI_QueryGroups` ANNOTATION (the dataflow-JSON
spelling) finds nothing in any model.tmdl and wrongly "proves" no declaration is
needed — TMDL declares groups as bare `queryGroup <name>` objects (quoted when the
name has spaces, `A\B` for nesting), each with a `PBI_QueryGroupOrder` annotation.
Rule: when hand-adding any table/expression that carries `queryGroup:`, declare the
group object in model.tmdl first (order number unique among that model's groups; `0`
fine for a model's first group). Offline TOM validation of the fix was NOT possible
with the store-app Desktop's own DLLs (WindowsApps ACL denies Add-Type; copied out,
`Server.Tabular` loads but `GetTypes` dies on missing deps) — for store installs,
either keep a NuGet TOM handy or accept Desktop's next open as the validator: this
dialog names the offending object precisely, so the loop is tight.

### 2026-07-28 (excludeFromModelRefresh via TMDL edit drops the table's data)

Adding `excludeFromModelRefresh` to three small frozen dimension tables by editing
their TMDL (property line under `table X`, validated by TOM round-trip) looked like a
pure-metadata change — Rule 1 says only M text invalidates. WRONG for this property:
on next open, Desktop treated it as structural and dropped those tables' data, and
because the tables are now excluded from the global refresh, a full refresh never
reloads them — they sit EMPTY (COUNTROWS blank) while every other table is fine.
Knock-on: DAX calc columns on OTHER tables that RELATED() into them recalculated
against the empty table and went blank, silently poisoning a baseline export taken in
that session.

Rules:
- `excludeFromModelRefresh` belongs to the cache-invalidating class for practical
  purposes: apply it WITH a follow-up load plan.
- The recovery/intended workflow: per-table right-click "Refresh data" in Desktop
  still processes an excluded table (exclusion only gates the global refresh). One
  manual per-table refresh restores the data; the exclusion then keeps it constant.
- After any excludeFromModelRefresh edit, verify with COUNTROWS on the excluded
  tables AND on calc columns that reference them before trusting any engine export
  from that session.

### 2026-07-28 (round 2 — precomputing frozen fact slices as dataflow entities; the CSV null->"" join bomb)

Moved the frozen slice of two fact tables (headers + positions, ~190k/~520k rows,
full processing incl. FX sort/FillUp and margin columns) into two SELF-CONTAINED
one-time dataflow entities; the model now processes only the live-ERP slice per
refresh and combines with entity reads. Acceptance = keyed per-row diff on headers,
per-branch aggregate fingerprints on positions, both against a pre-swap engine
export. Worked — with one landmine:

- **Gen1 CSV round-trips turn TEXT nulls into empty strings.** A LeftOuter-join
  column that was null in-memory comes back from the entity as "". A model-side
  NestedJoin keyed on that column then MATCHES map rows whose key is also ""
  (e.g. synthetic/code-less products) — every unmatched row multiplied by the count
  of empty-key map rows (observed 5x on ~21k rows; branch row count 36k -> 89k,
  margin sums shifted). The old all-in-memory path never matched because genuine
  nulls join nothing. Fix pattern (both halves): `Table.ReplaceValue(entityRead,
  "", null, ...)` on join-key columns AND filter the lookup map to non-empty keys.
  Corollary: after ANY table moves through a dataflow, audit every downstream join
  keyed on a column that can be null — "" is the new null, and "" DOES join.
- Acceptance methodology when a multiset string-diff is defeated by column
  reordering (calc-column conversion) and float re-formatting: per-BRANCH aggregate
  fingerprints over every value column (sums to ~1e-13 relative) + exact row counts
  + keyed per-row diff where a key exists. A branch-sliced fingerprint catches a
  wrong-branch regression that a table-total fingerprint hides.
- Splitting a union also splits dedup/fill scopes: prove cross-branch key collisions
  impossible (ID shape rule + measured zero overlap) before splitting Table.Distinct;
  document FillUp fill-space changes and probe null-counts before/after.
### 2026-07-29 (column name with a trailing space; A/B against an old PBIX snapshot in a second Desktop instance)

Investigating "value missing in PBI" reports against a live Desktop model, two
reusable findings:
- **A TMDL/TMSCHEMA column name can end in a trailing SPACE** (`[Some_Column ]`).
  `$SYSTEM.TMSCHEMA_COLUMNS` output formats it invisibly, and DAX referencing
  `[Some_Column]` (no space) fails with the misleading "Column ... cannot be found
  or may not be used in this expression" even though the DMV "shows" it and
  State=1. Audit by dumping `.Length`/char codes of ExplicitName, then reference
  the bracket name WITH the space. DAX *value* comparison is trailing-space
  insensitive, but *object name* resolution is exact.
- **Old-PBIX A/B is the fastest missing-data triage when the feed reworked
  recently.** A stale .pbix snapshot (saved before the feed rework) opens in a
  SECOND Desktop instance alongside the user's live session (new msmdsrv port —
  poll `Get-NetTCPConnection` excluding the known port, then poll DBSCHEMA_CATALOGS
  until the catalog answers; a ~120MB pbix needs minutes). Querying the same keys
  in both models instantly separates "regression introduced by the rework" (rows
  present in old, gone in new) from "was always missing" (absent in both) — no
  source credentials needed, because the old cache IS the old feed's output.
  Kill only the second instance afterwards: resolve the new port's msmdsrv PID →
  its PARENT PBIDesktop PID → Stop-Process that parent; the user's session and
  its port stay untouched. Fleet-wide old-vs-new diffs need care: if the rework
  also FIXED bugs (e.g. price-quantity inflation), amount deltas mix corrections
  with regressions — verify candidate orders against an independent truth
  (posted-invoice totals) before calling them losses.
### 2026-07-23 (Bing filledMap → azureMap conversion, hand-finished; choropleth recipe)

Deprecated `filledMap` (Bing) replaced with native `azureMap` on a hand-built page.
What was measured to work vs not:
- **Desktop's "Upgrade map" banner button is the right converter** (it authors valid
  azureMap JSON) — but the banner lives in the visual's WebView, invisible to UIA;
  click it by pixel coordinates. The conversion carries `dataPoint.fill` over.
- **Role trap: azureMap interprets query role `Y` as LATITUDE.** The Bing filled map's
  Y (size/tooltip measure) survives conversion and silently feeds a count as a
  coordinate — geocoding breaks, every region renders in the default theme fill, and
  the map world-tiles. Fix: DELETE the Y projection; a country/ISO2 column in
  `Category` (Location) alone geocodes regions.
- Restrict the query with a hidden visual filter `measure > 0` — without any measure
  in the query, ALL dimension rows geocode and no-data countries paint the default
  fill.
- **Conditional region color = `dataPoint.fill` with a measure + dataViewWildcard
  selector** (same shape as Bing). A guessed `filledMap.fillColor` entry was silently
  stripped by Desktop's next save — invalid property, no error (Rule 8 behavior).
- Working `mapControls` properties (Desktop-serialized, verified): `defaultStyle`
  (`'grayscale_light'` gives the clean mockup-style base), `zoom`/`centerLatitude`/
  `centerLongitude` (D literals), **`worldWrap: false`** to stop horizontal world
  repetition, `showStylePicker`/`showNavigationControls`/`showSelectionControl`.
  `filledMap.show true` + `bubbleLayer.show false` for pure choropleth;
  `mapTransparency 40L` (conversion default) washes colors — set `0L`.
- The Format pane IS UIA-automatable for unknown properties: card headers are UIA
  BUTTONs ("Map settings"), toggles clickable; flip the setting in UI, Ctrl+S, and
  diff the visual.json to learn the serialization (that's how `worldWrap` was found).
- The one-time "Bing maps are going away" dialog blocks page navigation clicks until
  dismissed via its UIA button named **"Close Dialog"** (the WinForms `#32770`-style
  finders don't see it).

### 2026-07-23 (page-4 trend suite — programmatic visual generation)

- **Authoring several similar PBIR visuals: build nested hashtables and
  `ConvertTo-Json -Depth 60` from a parametrized PowerShell function** instead of
  hand-writing JSON — five line charts (varying category, optional Series role,
  1–2 measures, per-chart filters) generated and loaded first try. The one trap:
  a Values list-of-lists in an `In` filter needs the unary-comma wrapper
  (`@(,@(@{Literal=...}))`) or ConvertTo-Json flattens it.
- lineChart accepts TWO measures in role `Y` for a two-line chart (score + rate)
  when a Series legend isn't used; Series (legend) and multiple Y measures remain
  mutually exclusive — for "per-period series AND a rate line" asks, pick per widget.
- `sortByColumn:` added by hand to a TMDL column (pointing at an existing
  order column) is honored everywhere after reopen — charts and slicers pick up the
  business ordering with no report-side changes.
- Category axes over relationship-blank members: the blank shows as a "(Blank)"
  axis category; a hidden visual-level Advanced `Not(= null)` filter on the axis
  column removes it (same shape as the not-blank date filter).

### 2026-07-23 (syncing the embedded theme to the org template breaks slot-riding semantic colors)

The report's RegisteredResources theme copy had drifted from the org's source-of-truth
theme JSON (two palette slots: dark greys where the template has greens). Syncing the
embedded file to the template (overwrite content, keep the registered filename;
strip the template's BOM) is the right move — but any visual whose SEMANTIC series
colors ride theme slot order flips with it: the sentiment-category charts' "negative"
series went GREEN. Fix pattern: pin category colors explicitly on affected visuals —
`dataPoint` entries with `selector.metadata = "<Table>.<Measure>"` for per-measure
series, and `selector.data[].scopeId.Comparison` (column = 'Value') for legend-column
series. Both shapes verified rendering; the scopeId shape matches what Desktop itself
serializes. Appending to an existing Desktop-authored `dataPoint` array via
ConvertFrom/To-Json round-trip (-Depth 100) preserved fieldParameters and stale
entries intact. Rule: semantic (good/bad) series must NEVER depend on theme slot
order — pin them at the visual.

### 2026-07-31 (ThemeDataColor ColorId mapping; theme swaps recolor custom visuals silently)

`ThemeDataColor` in visual.json maps as: ColorId 0 = theme `background`, ColorId 1 =
theme `foreground`, ColorId N (N>=2) = `dataColors[N-2]`. Verified by matching a
custom-visual (Tachometer) gauge's rendered colors against the theme file: its axis
labels used ColorId 8 → dataColors[6], pointer ColorId 9 → dataColors[7], hub
ColorId 7 → dataColors[5]. Consequence of the theme-sync rule above extends to
CUSTOM visuals too: after swapping the embedded theme, any Desktop-authored
ThemeDataColor reference (even in third-party visuals) silently changes hue — here
gray axis labels turned light green because the new palette put a green in slot 6.
Fix: replace the `ThemeDataColor` expr with a `Literal` hex (structure:
`"expr": {"Literal": {"Value": "'#RRGGBB'"}}`) wherever the color is semantic or
neutral-by-design. Audit trick: grep edited pages for `ThemeDataColor` after any
theme sync and eyeball each slot against the new palette.

### 2026-07-31 (Desktop-authored manual layout edits: fractional positions are the tell)

When the user hand-resizes visuals in Desktop and saves, positions serialize as long
fractionals (e.g. `403.64741641337389`) vs. programmatic integers — a reliable way to
detect which visuals the user touched before redistributing a layout around their
edits. Redistribute AROUND those visuals (treat them as fixed), and remember inner
`columnWidth` selectors do NOT scale with the container — a user-widened table keeps
its old column widths unless they resized columns too (also fractional if they did).

### 2026-07-30 (Int64-typing a line-number column BANKER'S-ROUNDS fractional lines into collisions; Distinct then eats invoiced rows)

Follow-up to the 2026-07-29 missing-items triage — the confirmed root cause was
NOT the canceled-line filter (my first hypothesis; raw export showed the missing
lines as status Invoiced):
- **M's number→Int64 TransformColumnTypes ROUNDS (round-half-to-even), it does
  not error.** ERP line numbers are decimals — inserted lines get 8.5, 3.75,
  3.875. Typing that column Int64 folds 8.5→8, 13.5→14, 3.75→4 silently onto
  neighboring integer lines; a downstream `Table.Distinct(_, {order, line})`
  then keeps one arbitrary row per collision and real invoiced lines vanish
  with plausible totals. (Text→Int64 errors; number→Int64 rounds — the danger
  path is the numeric one, e.g. Excel-sourced columns.)
- Measured on one company's export: 15,348 active lines → 120 rows in collision
  groups → 62 rows silently dropped (57 of them Invoiced) across 52 orders,
  ~103k in transaction currency.
- Rule: NEVER type a line/position number Int64. Type it `type number` (or
  text preserving the fraction) and keep the typing IDENTICAL on every query
  that joins or dedups on it (anti-join sets, mapping tables, tax-allocation
  groupings) so keys round-trip byte-identically.
- Triage rule derived: when specific child rows are missing, fetch the RAW
  export row for the missing keys FIRST and read the actual status/key values
  before theorizing from the transform code — two plausible mechanisms
  (status filter, rounding collision) predicted the same symptom; only the raw
  row distinguished them. A user screenshot showing header status cannot —
  headers and lines both said Invoiced.

### 2026-07-31 (bridging order-level values to position grain; "history is empty" triage order)

Request: an order-level income stream (charges in the new ERP, a dedicated fee
ARTICLE on invoice lines in the legacy ERP) had to become sliceable by product
dims and continuous across both eras. Learnings:
- **"No history" for a metric usually means the legacy era booked it
  DIFFERENTLY, not that data is missing.** Before declaring a gap, sweep the
  legacy position lines for fee/service articles (grep part names for the fee
  concept in every business language of the tenant, then rank candidates by
  yearly sums — the real fee article stands out by orders of magnitude).
  Legacy-era "charge" concepts often live as line items on a synthetic article.
- **Allocate order-level values to lines with TIERED fallbacks that preserve
  exact parity**, not a single rule: (1) pro-rata over positive non-fee lines;
  (2) else fee lines keep their own value and absorb charges proportionally;
  (3) else allocate by absolute line value, last resort equal split by a row
  count carried on the header. Acceptance = SUM(allocated) vs SUM(sources) to
  the cent, then DECOMPOSE the residual (ours: exactly the orders with charges
  but zero loaded position rows — quantify, don't hand-wave).
- **Expect a large legitimately-unallocatable share.** Over half the legacy fee
  volume sat on fee-ONLY documents (separate freight invoices with no product
  lines) — no bridge can conjure product dims for those; they still slice by
  header dims (customer/subsidiary). Report that share explicitly or the
  product-dim view will be mistaken for a bug.
- Implementation notes that made it cheap: hoist per-order aggregates to calc
  columns on the HEADER (context transition over the related fact — O(n)), the
  line column reads them via RELATED; use the model's CANONICAL value column
  (the consolidated/FX-fallback one the Sales measures use), not the raw one;
  calc columns recalc from loaded data — no source refresh needed.
- DMV caveats hit again: TMSCHEMA DMVs reject LIKE and subqueries; read values
  via GetValue with DBNull checks (GetString throws on null ExplicitName rows).

### 2026-08-03 (summary count ≠ detail row count on the same page: diff visualInteractions FIRST)

A data-quality page showed an error-category count of 10 contracts while the
detail table below listed only 6 rows for the same category/country. Neither
visual's filterConfig explained it (the hidden per-visual filters differed but
were no-ops — they sat on a many-side table with no propagation path to the
counted table). The cause lived in `page.json` → `visualInteractions`: a
numeric-range slicer (revenue ≥ threshold) had `NoFilter` overrides toward two
of the three summary tables but NO entry toward the detail table, so the
default (DataFilter) applied only there. Engine A/B confirmed:
`CALCULATE(COUNT(...), cat, country)` = 10; adding
`KEEPFILTERS(rank[revenue] >= threshold)` = 6.

Rules:
- When two visuals on ONE page disagree on the same slice, diff
  `page.json.visualInteractions` before diffing visual filters. Entries are
  `{source, target, type}` with type `NoFilter`/`DataFilter`/`HighlightFilter`;
  an ABSENT pair means default (filter/highlight applies). Asymmetric overrides
  (some summary visuals exempted, siblings not) are invisible in the filter pane.
- Per-visual hidden filters on a many-side table (e.g. `Contract[Account ID]`
  with single-direction Contract→Account) cannot filter other fact tables —
  rule them out by walking the relationship graph before blaming them.
- Verify with an engine A/B (`CALCULATE` vs `CALCULATE`+`KEEPFILTERS(slicer
  predicate)`) — it reproduces both numbers exactly and names the dropped rows.

### 2026-08-03 round 2 (making a range slicer apply per-category: disconnected clone + TREATAS, verified by A/B)

Follow-up to the visualInteractions finding: the requirement became "the revenue
slicer must filter SOME issue categories (and their detail rows) but not others,
with counts and detail always agreeing." Interaction overrides can't do this —
they are per-visual, and a NoFilter override also blinds the visual's measures
to the slicer selection (SELECTEDVALUE/ISFILTERED see nothing). Working recipe:

1. **Disconnected clone table** of the sliced column:
   `SELECTCOLUMNS(ALLNOBLANKROW('rank'[revenue]), "revenue", 'rank'[revenue])`
   (ALLNOBLANKROW per the calc-table blank-row-circularity rule). Rebind the
   slicer to it — TWO attachment points in the slicer visual.json: the
   projection AND the stored selection in `objects.general.filter.filter.From`
   (a numeric-range slicer keeps its typed bound there + `objects.data.numericStart`).
2. **Category flag** as an int (0/1) calc column on the fact — booleans break
   MAX(); ints filter cleanly in CALCULATE.
3. **Counts** = unflagged part + flagged part, the flagged part under
   `KEEPFILTERS(TREATAS(VALUES(clone[revenue]), rank[revenue]))` when
   `ISFILTERED(clone[revenue])` — TREATAS re-enters the original table so
   relationship propagation replicates the connected slicer EXACTLY (incl.
   bidi paths to sibling denominators).
4. **Detail-table row gating** = hidden `Pass` measure (1/0) + visual-level
   Advanced filter `= 1`; the pass test is
   `CALCULATE(COUNTROWS(fact), KEEPFILTERS(TREATAS(...))) > 0` per row —
   reuses propagation, no LOOKUPVALUE blank-key traps.
5. Swapping a visual's `CountNonNull(...)` aggregation for the new measure
   touches projections, sortDefinition, conditional-formatting expressions
   (RangePercent inner nodes), columnWidth/dataBars selectors, AND any
   aggregation-field visual filter. A JSON round-trip walker (PowerShell
   ConvertFrom/ConvertTo-Json -Depth 100) replacing the exact Aggregation
   subtree + a plain string replace of the queryRef selector strings handled
   all of them; Desktop accepted the re-serialized visual.json.
6. **Acceptance** = engine A/B in the NEW model: old behavior emulated with
   `KEEPFILTERS(rank[revenue] >= x)` vs new measures under a clone-table
   filter — every pair must match (counts, detail pass-sums, ratios,
   denominators), plus the no-selection state (ISFILTERED false ⇒ all three
   representations equal the unfiltered count).

Also: a headless `Start-Process <pbip>` open validated the hand-authored calc
table + measures (poll for the NEW table name, then TMSCHEMA_PARTITIONS all
State=1).

### 2026-08-04 (scaffolding a complete PBIP from scratch — accepted by Desktop on first open)

Built a whole project (DirectQuery model over a cloud SQL source + report) by
hand, no Desktop involved, for a writeback demo. Desktop opened it clean,
signed in, and saved without complaints. What made it work:

- **Template every metadata part from a sibling Desktop-authored project** —
  don't guess versions. The combination that worked together: `.pbip` 1.0,
  `definition.pbism` 4.2, report `definitionProperties` 4.0 (byPath), report
  `versionMetadata` 2.0.0, `report.json` schema 3.3.0, `page` 2.1.0,
  `visualContainer` 2.11.0, `database.tmdl` compatibilityLevel 1606. A minimal
  `model.tmdl` (culture, defaultPowerBIDataSourceVersion powerBI_V3,
  dataAccessOptions, PBI_QueryOrder, `ref table` lines) suffices.
- **report.json must carry the theme plumbing or Desktop has nothing to
  render with**: `themeCollection.baseTheme` (SharedResources) + the matching
  `resourcePackages` entry + the BaseThemes JSON copied into
  `StaticResources/SharedResources/BaseThemes/`. A custom theme carried over
  from the sibling project worked with RegisteredResources item type
  `CustomTheme` (the sibling's Desktop-written file said `Image` — Desktop
  accepts both, so don't cargo-cult the odd one).
- **Hand-authorable visual set (verified rendering)**: slicer (objects.data
  mode 'Dropdown' + selection.singleSelect), card (single measure projection),
  tableEx (column projections + sortDefinition), textbox
  (objects.general.paragraphs[].textRuns[] with textStyle). LF line endings
  everywhere were accepted.
- **Desktop normalizes on first save** — fenced ```` ``` ```` M partition
  sources become plain indented `source =`, slicer projections gain
  `"active": true`, `PBI_ProTooling` annotation appears. Diff AFTER the first
  Desktop save before doing further hand edits, or your next old_string match
  fails.
- **Offline TMDL gate without a modern SSMS**: check for the TYPE, not the DLL
  version — an SSMS 20 `Microsoft.AnalysisServices.Tabular.dll` lacked
  `TmdlSerializer` while ALM Toolkit's build (FileVersion 17.0.39.18) had it.
  `[TmdlSerializer]::DeserializeDatabaseFromFolder($definitionFolder)` is the
  gate; it validated 2 tables / 1 relationship / 5 measures before Desktop
  ever saw the project.
- **PS 5.1 validator trap on non-ASCII paths**: a `.ps1` saved as UTF-8
  *without* BOM gets its path literals mangled by Windows PowerShell
  (`á` → `Ã¡`, "Invalid path ... does not exists"). Save harness scripts with
  `-Encoding utf8BOM` when user profiles contain accented characters.

### 2026-08-04 round 2 (translytical visuals ARE hand-authorable; binding claim CORRECTED in round 3)

Screenshot-verified in Desktop after a reopen of the scaffolded project:

- **Input slicer visualType is `textSlicer`** — hand-authored with NO query
  projections (that's what makes it a pure input box instead of a filter) plus
  a `visualContainerObjects.title`. It rendered and accepted typed input
  first try. (Desktop's save normalizes it by adding an empty
  `objects.general` card.)
- **A data-function button** = `actionButton` with
  `visualContainerObjects.visualLink` `{show: true, type: 'DataFunction'}`.
  With no `dataFunction` node it renders with tooltip "The data function is
  not properly configured" — the unbound state. I initially recorded (from a
  third-party writeup) that PBIR cannot persist the binding at all. **That is
  WRONG for current Desktop — see round 3**, which documents the full
  serialized binding.

### 2026-08-04 round 3 (Desktop DOES serialize the full data-function binding — verified shape, end-to-end write confirmed)

After binding the button in Desktop's Format > Action pane and saving, the
visual.json carried the COMPLETE binding — so a fully-wired translytical
button is hand-authorable after all (the round-2 "unavoidable manual bind"
claim was based on an outdated external writeup; trust the round-trip, not
the blog). Verified end to end: button press inserted the row via the UDF.

Serialized shape under `visualContainerObjects.visualLink` (alongside `show`
and `type: 'DataFunction'`) — note `dataFunction` is a STRUCTURED node, not
an `{expr}` formatting property:

```json
"dataFunction": {
  "kind": "ItemLocation",
  "byReference": {
    "itemId":      { "expr": { "Literal": { "Value": "'<udf-item-guid>'" } } },
    "workspaceId": { "expr": { "Literal": { "Value": "'<workspace-guid>'" } } }
  },
  "metadata": { "dataFunction": {
    "name": "<function_name>",
    "autoRefresh": true,
    "parameters": [
      { "name": "p1", "dataType": "string", "isOptional": false, "defaultValue": null,
        "type": "ValueParameter",
        "value": { "expr": { "Aggregation": { "Expression": { "Column": { "Expression": { "SourceRef": { "Entity": "<dim>" } }, "Property": "<col>" } }, "Function": 3 } } } },
      { "name": "p2", "dataType": "number", "isOptional": false, "defaultValue": null,
        "type": "SlicerParameter", "slicer": "<input-slicer visual name>", "autoClear": false },
      { "name": "p3", "dataType": "string", "isOptional": false, "defaultValue": null,
        "type": "ValueParameter",
        "value": { "expr": { "Measure": { "Expression": { "SourceRef": { "Entity": "<table>" } }, "Property": "<measure>" } } } }
    ]
  } }
}
```

Rules derived:
- Column-bound scalar parameters get wrapped by Desktop in an `Aggregation`
  with `Function: 3` (Min) — mirror that when hand-authoring, don't pass the
  bare Column node.
- `SlicerParameter` references the input slicer by its VISUAL NAME (the
  folder-name id) — a hand-authored `textSlicer` participates fine; keep the
  name stable or the binding dangles silently.
- `byReference.itemId`/`workspaceId` are ENVIRONMENT-BOUND literals — a
  report promoted to another workspace/tenant keeps pointing at the original
  UDF item. Deployment across environments must rewrite these two literals
  (or re-bind in Desktop); nothing errors at load time.
- The measure-bound parameter (e.g. USERPRINCIPALNAME() for audit columns)
  evaluates per-viewer at press time — in Desktop it records the author's
  sign-in identity, in the service each viewer's own UPN.

### 2026-08-07 (two traps: DirectQuery DIVIDE alternate-result, and actionButton label card anatomy)

- **DirectQuery models reject `DIVIDE(a, b, alternate)` with a non-constant
  alternate** at query time: "The alternate result to return on divide by zero
  cases must be a constant numeric value." A measure using
  `DIVIDE(x, rate, x)` (fallback-to-unconverted) errors on every visual. Use
  two-arg `DIVIDE` + `IF(ISBLANK(...))` for variable fallbacks.
- **actionButton label**: the `objects.text` card must contain TWO entries —
  a card-level `{properties: {show: true}}` WITHOUT a selector, then the
  `text`/`fontSize`/`fontColor` properties under `selector: {id: "default"}`.
  Collapsing them into one selector-default entry renders NO label (silently).
  Same split applies to `fill` (card-level `show`, then `fillColor` under the
  state selector). An `icon.shapeType = 'blank'` entry gives a clean
  text-only button.
- Binding-metadata `defaultValue` on a `SlicerParameter` PRE-FILLS the bound
  input slicer with that value in the service — use `null` there even when
  the function's own default is a numeric sentinel.

### 2026-08-07 round 2 (PowerShell JSON generators: single-element array unrolling breaks VCO cards)

A generator that returns formatting-object arrays from helper FUNCTIONS
(`function TitleVCO { @(@{...}) }`) produced Desktop errors "Property
/visual/visualContainerObjects/title ... was not provided as the correct
type": PowerShell unrolls a returned one-element array, so ConvertTo-Json
serialized an OBJECT where the schema wants an ARRAY of cards. Inline
`@(@{...})` inside a hashtable literal keeps the array; a function return
does not. Fix: `return ,@(...)` (comma operator). Audit any PBIR generator
by grepping the OUTPUT for `"title": {` / `"padding": {` — card properties
must always open with `[`.
Also verified: slicer `objects.header.show=false` removes the redundant
field-name chrome under a VCO title; a `shape` visual panel = objects
`shape.shapeType 'rectangle'` + two-entry fill/outline cards (selector
default + card-level show), same split as buttons.

### 2026-08-07 round 3 (input slicer `textSlicer` formatting card names — captured from a Desktop save)

Formatted ONE input slicer in Desktop's Format pane, saved, and diffed the
visual.json — the discovery pattern of choice when object/property names are
undocumented (public schemas leave visual objects untyped). Cards written:

- `applyButton`: `iconSize` (`0D` hides the submit arrow — the documented
  "Icon size = 0" trick), `spacing`, `paddingLeft/Right` (`0L`).
- `slicerSettings`: `numericInputOnly` (`true`) — free input validation for
  numeric-only task-flow inputs.
- `inputText` (the committed-value pill): `backShow false` kills the pill's
  highlight background, `dismissSize 0D` hides the pill's X, plus paddings.
- `filterOperator`: `backShow`.
- `inputTextBox`: `paddingLeft/Right` (`4L`).

Propagation recipe: read the hand-formatted visual's `visual.objects` subtree
and stamp it onto sibling visuals IN PLACE (ConvertFrom/ConvertTo-Json),
rather than re-running a full page generator over Desktop-saved files — that
preserves Desktop's other normalizations and any manual edits.

### 2026-08-07 round 4 (full-report clone onto a new binding; hand-built DQ-to-AS chains do NOT surface remote measures)

Cloned a 130-visual production report (2 pages, bookmarks, reportExtensions,
theme resources) by copying the .Report folder, deleting the embedded-app
visuals, and generating replacement input visuals into the app's exact slot.
Two binding strategies tested:

- **Hand-built composite chain (DQ-to-AS proxy tables) — columns work,
  measures DON'T.** Proxy tables generated from the source model's TMDL
  (columns + `partition X = entity` with `expressionSource`) loaded fine:
  slicers and column-bound visuals rendered with live data. But every
  measure-consuming visual errored ("field was deleted from the model" /
  "you don't have required permissions" naming remote measures). A
  Desktop-authored chain exposes source measures; a hand-built proxy set
  evidently needs more than tables+columns (untested what — likely
  Desktop-generated proxy metadata). Also: a TMDL-parsing proxy generator
  must handle `column X = expr` calc columns (header has `= expr` on the
  same line — a `column NAME\n` regex silently skips them).
- **Thin live-bound report (byConnection) — everything resolves by
  definition.** For an exact clone of a report whose model still exists,
  copy the old `definition.pbir` byConnection verbatim; report-layer
  measures go in `reportExtensions.json` (entities[].measures[] with
  name/dataType/expression — verified shape). A data-function button works
  identically in a live-bound report; bind its scalar params to source-model
  columns (Aggregation-wrapped) and extension measures.

Practical rule: for "exact copy of page X with its setup", clone thin-bound
first (zero field risk), and only go composite when local tables must join
the canvas — via a Desktop-authored chain whose TMDL you then harvest.

### 2026-08-07 round 5 (byConnection connectionString: Desktop vs service resolve it DIFFERENTLY; My Workspace is not XMLA-addressable)

Two independent resolvers read `definition.pbir` → `byConnection.connectionString`,
and they disagree:

- **The service resolves by `semanticmodelid=` alone** and ignores the
  `Data Source=` host/path. A report published via REST with a bogus
  workspace path but a correct semanticmodelid binds and renders fine online.
- **Desktop actually dials the `Data Source=`.** The XMLA-style
  `powerbi://api.powerbi.com/v1.0/myorg/<workspace>` form only works for
  NAMED org workspaces; **"My workspace" has no XMLA path** — Desktop fails
  with "The specified Power BI workspace is not found. Workspace:
  'My%20workspace'" (Cannot load model, ModelMode ExternalOnPrem).

The shape Desktop itself writes when you pick a dataset via
Home → Data hub (works for My Workspace and org workspaces alike):

```
Data Source=pbiazure://api.powerbi.com;initial catalog=<datasetGuid>;identity provider="https://login.microsoftonline.com/common, https://analysis.windows.net/powerbi/api, 7f67af8a-fedc-4b08-8b4e-37c4d127b6cf";integrated security=ClaimsToken;semanticmodelid=<datasetGuid>;modelid=<numericId>
```

- `initial catalog` = `semanticmodelid` = the dataset GUID.
- The identity-provider triple is generic (common authority, PBI resource,
  first-party Desktop client id) — copy verbatim, nothing tenant-specific.
- `modelid` is a service-internal numeric id Desktop adds; the service
  doesn't need it (semanticmodelid wins). If you don't know it, omit and
  let Desktop rebind once, then harvest the string it saves.

Rule: when hand-authoring `definition.pbir` that must open in BOTH Desktop
and the service, use the `pbiazure://` shape (or, for a named org workspace,
the `powerbi://.../<workspace name>` shape copied from a Desktop-saved file).
Never invent `powerbi://...myorg/My workspace`.

Related load-failure signature: a published PBIR report stuck forever on
"Loading your report..." (binding + pages endpoints healthy) can be caused by
**bookmarks whose explorationState still snapshots DELETED visuals filtered
on DELETED tables**. Slimming a report/model must prune
`explorationState.sections.<page>.visualContainers.<visualName>` entries for
every visual no longer on the page — grep bookmarks for dropped table names
after any trim.
