# PROJECT_CONTEXT.md — health-analytics-fabric-project

> Living session-handoff doc. Read at the start of every session alongside
> TEACHING_PREFERENCES.md and PROJECT_PLAN.md. Updated at every session close.

---

## Current state (as of Phase 1, 2026-06-13)

- **Phase:** 1 (ingestion) — COMPLETE. Phase 2 (Silver) is next.
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
    silver.measures                 33 rows  | measure_code nulls: 0
    silver.measure_categories       12 rows  | measure_category_code nulls: 0
    silver.reporting_unit_types      6 rows  | reporting_unit_type_code nulls: 0
    silver.reporting_units       1,427 rows  | reporting_unit_code nulls: 0
    silver.data_items        1,700,870 rows  | value nulls: 653,363 (expected suppressed rows — M3-15)
- LEARNINGS M3-14, M3-15 banked.
- nb_silver_build.ipynb saved to notebooks/ with confirmed row counts in comments.
- Phase 2 COMPLETE. Next session: Phase 3 Gold (star schema build —
  fact_measure_value + dim_hospital + dim_measure + dim_period +
  dim_reported_measure; datasets period join at this layer).
