# PROJECT_CONTEXT.md — health-analytics-fabric-project

> Living session-handoff doc. Read at the start of every session alongside
> TEACHING_PREFERENCES.md and PROJECT_PLAN.md. Updated at every session close.

---

## Current state (as of Phase 4, 2026-06-14)

- **Phase:** 4 (Power BI) — IN PROGRESS. Semantic models + relationships +
  10 documented DAX measures DONE (both the Direct Lake web model and the
  Import-mode .pbix). REMAINING for next session: the 3 dashboard pages, then
  screenshots + README. Phases 0-3 COMPLETE.

## Repo / commit hygiene (read before staging at session close)

- These files are gitignored by design and must NOT be staged or pushed —
  they are personal working docs, updated on disk but kept local only:
  LEARNINGS.md, TEACHING_PREFERENCES.md, LEARNING_ROADMAP.md.
- MINI3_KICKOFF.md is excluded locally via .git/info/exclude (also never
  pushed).
- So at session close: LEARNINGS.md gets updated on disk (the ledger), but a
  clean `git status` will NOT list it — that is correct, not a missed file.
  Don't re-check whether LEARNINGS needs committing; it never does.
- Public-facing artifacts that DO get committed each session: notebooks/,
  pipelines/, powerbi/, docs/, README.md, PROJECT_CONTEXT.md, PROJECT_PLAN.md,
  ENGINEERING_STANDARDS.md, and the *_NOTEBOOK.md / *_PIPELINE.md walkthroughs.
- **Environment gate: PASSED 2026-06-12.** Fabric 60-day trial active,
  expires ~2026-08-11 (items deleted ~7 days after expiry).
- **Build clock:** day 1 done. Bronze snapshot is frozen; the API is out of
  the demo path from here on.
- **Bronze inventory:** Files/bronze (5 list-endpoint JSONs) +
  Files/bronze/data_items (33 data-items JSONs, ~1.2 GB total, two
  legitimately-empty envelopes: MYH0037/38). Data version of record:
  2026052802.

## Environment + identities

| Container | Name | Notes |
| --- | --- | --- |
| Entra tenant | Pheluciam | Existing tenant on the personal Azure account |
| Azure portal identity | pheluciam@outlook.com | Tenant admin; used for Entra user management only |
| Fabric identity | phil-fabric@Pheluciam.onmicrosoft.com | ALL Fabric portal work; Capacity admin of the trial |
| Trial capacity | Trial-20260612T082220Z (Australia Southeast) | F4-class trial, 60 days from 2026-06-12 |
| Workspace | health-analytics-fabric | Fabric Trial license mode; LARGE semantic model storage format |
| Lakehouse | lh_health_analytics | Schemas enabled; dbo default schema present |

Sign-in note: Fabric sessions run in a Firefox Private window so the
personal-account cookie doesn't hijack the sign-in.

## Data source status

- AIHW MyHospitals API verified live 2026-06-12 (second verification; first
  was the scoping pre-flight the same day).
- Data version at audit: 2026031701 (date_uploaded 2026-03-17).
- README audit paragraph: banked in README.md (section "Data audit").
- Dashboard-page measures confirmed live: MYH0005, MYH0010, MYH0011,
  MYH0035 (ED); MYH0007, MYH0008, MYH0009 (elective surgery); plus
  MYH0006, MYH0013, MYH0036 available.

## Decisions made at Phase 0

- Used the existing Pheluciam tenant rather than creating a fresh one
  (years old, so immune to the community-reported sub-90-day trial block).
- Trial capacity region: Australia Southeast (Melbourne — Phil's choice,
  local; default offered was Australia East).
- Workspace on LARGE semantic model storage format (Direct Lake story).
- Lakehouse schemas enabled (bronze/silver/gold namespacing).

## Session log

### Session 1 — 2026-06-12 (Phase 0)

- Environment gate: phil-fabric Entra user created; Fabric (Free) licence
  signup; 60-day trial activated (Australia Southeast); workspace +
  Lakehouse created. Gate PASSED — clock can start.
- Forward-verify pass: the Entra-tenant workaround is now an officially
  documented Microsoft path ("Start a Fabric trial with a personal email",
  learn.microsoft.com, updated 2026-04-08).
- Data audit re-run: measure-categories + measures endpoints live;
  audit paragraph banked.
- PROJECT_PLAN.md + PROJECT_CONTEXT.md authored.
- Repo: git init (main) run locally after a sandbox lock-file snag (M3-2);
  public GitHub repo Pheluciam/health-analytics-fabric-project created;
  remote wired. README skeleton + .gitignore + folder scaffolding
  (pipelines/ notebooks/ powerbi/ docs/) in place.
- Dashboard pages confirmed against the live measures fetch (PLAN §6).
- Phase-boundary audit: caught + fixed three informal-label leaks on
  public-bound files (M3-3). LEARNINGS M3-1..M3-4 banked.
- Phase 0 COMPLETE. Next session: Phase 1 ingestion (forward-verify pass
  on Fabric Data Pipeline REST copy first).

### Session 2 — 2026-06-13 (Phase 1)

- Forward-verify pass: REST-to-Lakehouse-Files JSON copy path confirmed on
  current Microsoft Learn docs; projected risks banked first (M3-5, M3-6).
  Mid-session UI-drift reset → full doc sweep of every UI step (M3-7).
- Built pl_ingest_myhospitals_bronze: two parameter-driven sequential
  ForEach loops (5 list endpoints; 33 measure-code data-items), anonymous
  REST connection conn_myhospitals_api, JSON sink to Bronze Files, empty
  Mapping (hierarchical as-is copy), 10-min timeout / 2 retries per copy.
- Run: 38/38 copies succeeded, ~16.5 min wall clock, zero retries used.
- Verification PASSED: 5 + 33 files, sizes eyeballed, two 189 B files
  traced to genuinely empty API results (M3-10).
- Pipeline definition exported to pipelines/pl_ingest_myhospitals_bronze.json;
  INGEST_PIPELINE.md walkthrough authored (three-layer pattern).
- Payload-size reality vs scoping banked (M3-9); clone-drops-source-connection
  gotcha banked (M3-8); data_version of record 2026052802 (M3-11).
- Structural audit: leak-grep run; one informal-label restatement fixed in
  this file. LEARNINGS M3-5..M3-11 banked.
- Phase 1 COMPLETE. Next session: Phase 2 Silver (forward-verify pass on
  PySpark/Delta in Fabric first; explicit schemas for the big data-items
  reads per M3-9).

### Session 3 — 2026-06-13 (Phase 2 — Silver)

- Forward-verify pass: PySpark/Delta write path + schema-enabled Lakehouse
  confirmed on current Microsoft Learn docs. Projected risks banked first
  (M3-12, M3-13).
- Built nb_silver_build: 6-cell PySpark notebook. Cell 1 imports + CREATE
  SCHEMA IF NOT EXISTS silver + BOM-safe wholeTextFiles reader. Cells 2–5
  flatten 5 small list-endpoint Bronze files to Silver Delta via schema
  inference + explode("result"). Cell 6 flattens 33 data-items files using
  an explicit PySpark schema (1.7 M rows).
- Path fix mid-session: BRONZE path corrected from "/lakehouse/default/Files/bronze"
  to "Files/bronze" after a 400 Bad Request (M3-14). Confirmed via
  mssparkutils.fs.ls before retrying.
- Old discovery/skeleton cell deleted from the notebook before shipping.
- All 6 Silver tables written and verified:
    silver.datasets              7,199 rows  | data_set_id nulls: 0
    silver.measures_ref             33 rows  | measure_code nulls: 0  (renamed from silver.measures Phase 4: 'measures' is reserved in the semantic-model engine and broke SQL endpoint sync)
    silver.measure_categories       12 rows  | measure_category_code nulls: 0
    silver.reporting_unit_types      6 rows  | reporting_unit_type_code nulls: 0
    silver.reporting_units       1,427 rows  | reporting_unit_code nulls: 0
    silver.data_items        1,700,870 rows  | value nulls: 653,363 (expected suppressed rows — M3-15)
- LEARNINGS M3-14, M3-15 banked.
- nb_silver_build.ipynb saved to notebooks/ with confirmed row counts in comments.
- Phase 2 COMPLETE. Next session: Phase 3 Gold (star schema build —
  fact_measure_value + dim_hospital + dim_measure + dim_period +
  dim_reported_measure; datasets period join at this layer).

### Session 4 — 2026-06-13 (Phase 3 — Gold)

- Forward-verify pass: Kimball load pattern + surrogate-key mechanics
  confirmed on current Microsoft Learn docs (dimensional-modeling-load-tables,
  updated 2025-04-06). Full UI-path audit of every screen this session touches
  (import notebook, pin lakehouse + restart, run cells, view schema tables) —
  M3-16, M3-17, M3-18 banked before any code.
- Built nb_gold_build: 7-cell PySpark notebook. Cell 1 discovery (silver is the
  schema source of truth — 4 of 5 silver tables had no enumerated columns in the
  repo). Cells 2-5 build the four dims, Cell 6 the fact, Cell 7 verification.
  Surrogate keys via row_number() over the business key (deterministic 1..N).
- All five Gold tables written and verified:
    gold.dim_period            54 rows  (15 financial years + 39 audit windows)
    gold.dim_hospital       1,166 rows  (1,165 type-H + Unknown -1 member)
    gold.dim_measure           33 rows
    gold.dim_reported_measure 690 rows  (10 is_total aggregates)
    gold.fact_measure_value 1,700,870 rows  (= silver.data_items; no loss/fan-out)
- FK integrity PASSED: all four FK null counts 0; hospital_key -1 = 318,595 and
  only on non-H types; every H row resolved; variant double-count test fired.
- Two bugs caught and fixed mid-build (Phil drove a full one-at-a-time rerun
  that surfaced them): dim_period "starts in July" misclassified Jul-Oct audit
  windows as FYs — fixed to start-July AND end-June (M3-19); is_total "contains
  total" flagged hip/knee procedures and missed "All patients" aggregates —
  fixed to exact total/all-prefix match (M3-20). LEARNINGS M3-19..M3-21 banked.
- Locked design decisions (M3-17): fact grain = one row per AIHW data item (all
  reporting-unit types retained for benchmark context); dim_hospital H-only +
  Unknown -1; degenerate reporting_unit_code/type on the fact.
- nb_gold_build.ipynb saved to notebooks/ with confirmed row counts/logic.
- Phase 3 COMPLETE. Next session: Phase 4 Power BI (Direct Lake semantic model
  on Gold, relationships, documented DAX measures filtering suppression_count=0
  and is_total, the three dashboard pages; Import-mode .pbix to repo).

### Session 5 — 2026-06-14 (Phase 4 — Power BI semantic models + measures)

- Forward-verify pass on current Power BI / Direct Lake / semantic-model docs
  before any clicks; projected Phase 4 risks banked first (LEARNINGS M3-22, M3-23).
- BLOCKER hit + fixed: the SQL analytics endpoint metadata sync was failing
  silently because a Silver table was named "measures" — a reserved object name
  in the Analysis Services / semantic-model engine ("Invalid AS model object name
  … reserved string 'measures'"). No Gold/Silver schema surfaced to any model
  picker until fixed. Renamed silver.measures -> silver.measures_ref; updated
  nb_silver_build.ipynb, nb_gold_build.ipynb, and this doc. The pipeline's
  "measures" API endpoint name is unchanged. (LEARNINGS M3-24 area / session note.)
- Built the Direct Lake semantic model sm_health_analytics_gold in the Fabric web
  (platform story): 5 Gold tables, star relationships (single-direction,
  many-to-one, assume referential integrity), _Measures = {BLANK()} holder, 10
  documented DAX measures.
- Built the durable Import-mode .pbix in Power BI Desktop (connected to the SQL
  analytics endpoint via the SQL Server connector, Organizational-account auth,
  Import storage mode). Tables renamed to drop the "gold " prefix. Measures
  bulk-loaded via the TMDL View from powerbi/measures.tmdl in a single Apply
  (formats carried by formatString — no manual per-measure formatting).
- TMDL gotchas banked (M3-25): script must start with createOrReplace;
  createOrReplace can't convert an existing import table's partition to calculated
  (delete the placeholder first). Percentage measures use custom format 0.0\%
  (literal % with no x100) — AIHW stores percentages on a 0-100 scale, confirmed by
  a SQL spot-check (MYH0005 values 59/44/34/46/65).
- Artifacts saved: powerbi/measures.tmdl (re-runnable measure set),
  powerbi/SEMANTIC_MODEL.md (canonical model doc), powerbi/health_analytics_dashboard.pbix
  (Import model; dashboard pages still to come). *_safety.pbix working copy is
  gitignored.
- Process fix locked: PRE-SEND CHECKLIST added to the top of TEACHING_PREFERENCES.md
  (paste-ables -> own code block; no inline backticks; action-only UI steps; never
  assume UI / verify docs).
- Phase 4 IN PROGRESS. Next session: build the 3 dashboard pages (ED performance,
  Elective surgery, Hospital benchmarking map), Vic focus + national/peer benchmark;
  then screenshots + README finalize.

### Session 6 — 2026-06-14 (Phase 4 — Power BI pages 1 & 2 built; page 3 re-scoped)

- **Page 1 ED Performance — DONE.** Title + subtitle, Financial Year slicer, 5 KPI
  cards (ED Presentations, 4-Hour Departure Rate, Treatment Started On Time, Median
  Departure Time, 90% Departure Time), Vic-vs-national trend line (MYH0005), top-10
  hospitals by ED presentations bar. Page filters: type H + Victoria.
- **Page 2 Elective Surgery — DONE.** Title + subtitle, FY slicer, 4 KPI cards
  (Elective Surgeries, Median Wait, % Within Recommended, % Waited >365 Days), median
  wait by hospital bar (top 10), wait-vs-on-time-rate bubble/scatter. Page filters:
  type H + Victoria.
- **Map is DEAD.** Azure Maps AND Bing Map/Filled Map are both blocked by tenant
  settings on this trial tenant. Enabling needs the Fabric Administrator role granted
  to phil-fabric (done this session via portal.azure.com as pheluciam@outlook.com →
  Entra → Roles → Fabric Administrator) BUT the tenant-setting toggle path was
  abandoned. Custom AppSource visuals (incl. circle/packed treemap) are also blocked
  by the same add-in gating. Do NOT attempt map visuals or custom visuals next session.
- **Measure data-structure learnings (verified against live AIHW API this session):**
  - MYH0010/MYH0011 (ED waits/presentations) have NO is_total row — reported only by
    triage category. Measures drop the is_total filter (sum/avg across triage).
  - Elective MYH0006/0008/0009 split by clinical urgency (Urgent/Semi/Non-urgent);
    MYH0007 splits by surgical specialty only. Measures filter by reported_measure_name
    to the relevant MECE partition (urgency names, or the 11 specialty names).
  - MYH0010 was MISLABELLED as "ED Median Waiting Time" — it is actually a %
    ("% commenced treatment within recommended time"). Renamed "ED Treatment Started
    On Time", reformatted 0.0\%. Real ED time metrics added: MYH0036 (median), MYH0013 (90%).
  - State-level rows exist (reporting_unit_type_code = "S": Victoria, NSW, etc.);
    national = "NAT". Used for the (now-superseded) benchmarking page.
  - Percentages are 0-100 scale → format string 0.0\% (backslash before %, NO built-in
    Percentage which x100s). All measures + formats set deterministically via TMDL
    (powerbi/measures.tmdl), not the format dropdown.
- **Page 3 RE-SCOPED — LOCKED PLAN for next session (Option C, Hospital Activity &
  Patient Flow, Victoria).** The benchmarking version (Vic-vs-national cards + gauge +
  state bar) was scrapped — gauge just echoed the cards, too thin. New plan:
  - Theme: Hospital Activity & Patient Flow, Victoria. Title "Hospital Activity &
    Patient Flow — Victoria"; subtitle "Admissions, emergency vs planned, and length
    of stay · AIHW MyHospitals".
  - Data (audited clean this session): admissions MYH0024 has a "Total" reported
    measure PLUS 6 types (Medical emergency/non-emergency, Surgical emergency/
    non-emergency, Childbirth, Mental health); data through 2023-24. Length of stay
    MYH0014 is by medical condition (DRG-like; NO overall total) → only usable as a
    by-condition visual. MYH0015 (% overnight) is ALSO by-condition (no overall total)
    → DROPPED as a KPI. Cost (MYH0032) too thin (3 yrs, ends 2014-15) — rejected. Hand
    hygiene (MYH0019) rejected as too trivial.
  - 5 KPI cards: Total Admissions, Emergency Share %, Surgical Share %, Childbirth
    Admissions, Hospital Count (hospitals reporting).
  - 3 visuals, DISTINCT types (Phil wants no bar/line overload): (1) Treemap —
    admissions by type (6 boxes, native rectangular treemap only); (2) Line — total
    admissions over time, Victoria single line; (3) Bar — average length of stay by
    top conditions (the ONLY bar on the page).
  - Measures ALREADY DRAFTED in measures.tmdl (Session 6 close), ready to apply via
    TMDL View next session: Total Admissions (MYH0024, rm="Total"); Admissions Value
    (raw MYH0024, no rm filter — for the by-type treemap, visual-filter OUT "Total");
    Emergency Share % (Medical+Surgical "emergency" / Total); Surgical Share % (both
    Surgical types / Total); Childbirth Admissions (rm="Childbirth"); Length of Stay
    (raw MYH0014 — for the by-condition bar). Computed shares use format "0.0%"
    (built-in, x100, because DIVIDE returns a 0-1 fraction) — NOT 0.0\%. Scope Page 3
    to Victoria + type H via PAGE filters (no state-level visuals on this page, so a
    page filter is safe here, unlike the dropped benchmarking page).
  - The 5 benchmarking measures (ED 4-Hour Rate Vic/National, ED 4-Hour Gap vs National,
    Hospitals Beating National, Vic ED Rank) and 2 unused general measures (Selected
    Value, Suppressed Data Points) were REMOVED from measures.tmdl this session. NOTE:
    applying the new tmdl next session will break the current (to-be-deleted) Page 3
    benchmarking visuals that reference them — that is intended; delete that page and
    rebuild as Option C.
  - At Page 3 completion: update README, SEMANTIC_MODEL.md, measures.tmdl, this doc.
