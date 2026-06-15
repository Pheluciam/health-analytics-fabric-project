# PROJECT_PLAN.md — health-analytics-fabric-project

> Authored 2026-06-12 at Phase 0, from the project scoping decision record.
> Locks scope, architecture, and the phase breakdown. Budget: 3-5
> build days at ~6 hrs/day. Cut scope rather than extend.

---

## 1. What this project is

**Microsoft Fabric end-to-end: hospital system performance analytics on the
AIHW MyHospitals API.** The final portfolio build before the
consolidation/interview-prep block. Lead theme: Fabric-the-platform —
Lakehouse, OneLake, Data Pipeline, PySpark notebook, semantic model, and
Power BI all in one workspace. The data is deliberately simple; the pipeline
is the star.

## 2. Pipeline architecture

MyHospitals REST API → Fabric Data Pipeline (Copy) → Lakehouse Bronze (raw
JSON files) → PySpark notebook (flatten + conform) → Silver Delta tables →
Gold star schema → Power BI semantic model (Direct Lake) → 3-page dashboard.

## 3. Environment (gate PASSED 2026-06-12, pre-build)

- Tenant: Pheluciam (existing Entra ID tenant on the personal Azure account)
- Fabric identity: phil-fabric@Pheluciam.onmicrosoft.com (created at Phase 0;
  Capacity admin of the trial)
- Trial: Fabric 60-day trial, activated 2026-06-12, capacity region
  Australia Southeast → expires ~2026-08-11; items deleted ~7 days after
  expiry (durable artifacts: repo, Import-mode .pbix, screenshots)
- Workspace: health-analytics-fabric (Fabric Trial license mode, large
  semantic model storage format)
- Lakehouse: lh_health_analytics (schemas enabled)

## 4. Data source (audit re-verified 2026-06-12)

- AIHW MyHospitals API — myhospitalsapi.aihw.gov.au, open REST, JSON, no
  auth, licence CC-BY. Data version at audit: 2026031701.
- Endpoints: measure-categories (12), measures (33, with units),
  datasets (measure-variant × financial-year period; carries reporting
  dates), measures/{code}/data-items (facts: value × reporting unit ×
  data_set_id, with caveats/suppressions), reporting-units (hospitals with
  lat/long, public/private, state/LHN/PHN/peer-group mappings),
  reporting-unit-types (H / PG / LHN / NAT / S / PHN).
- Volume: ~33 polite calls = full national snapshot; low hundreds of
  thousands of values. Zero PII (institutional aggregates).
- Known wrinkles (banked as risks): period requires the data_set_id →
  datasets join; reported-measure sub-variants need a dim with is_total
  flag; docs page is JS-rendered (work from endpoints); data-items payloads
  are a few hundred KB each.

## 5. Architecture rules (locked at kickoff)

- Medallion in ONE Lakehouse: Bronze = raw JSON files per endpoint
  (Files area); Silver = flattened Delta tables, one per entity; Gold =
  star schema (Tables, schema-namespaced).
- Fabric Data Pipeline does the API copy only (courier mode). The PySpark
  notebook does ALL transformation — flattening, joins, star build.
- Snapshot pattern: pull once during build, freeze raw JSON in Bronze. The
  API stays out of the live demo path.
- Gold star: fact_measure_value (value, suppression flag, FKs);
  dim_hospital (lat/long, public/private, state, LHN, peer group);
  dim_measure (category, units, variant); dim_period (financial year);
  dim_reported_measure (is_total flag).
- Power BI: semantic model on Gold inside the workspace (Direct Lake is the
  platform story) PLUS an Import-mode .pbix exported to the repo as the
  durable artifact. Documented DAX measures; screenshots in README.
- Scope: Victoria focus with national/peer-group benchmark context. ED +
  elective surgery + admissions are core; financial performance and
  infection measures are stretch.

## 6. The three dashboard pages (confirmed against the 2026-06-12 audit)

1. **ED performance** — 4-hour departure rate (MYH0005), waiting times by
   triage urgency (MYH0010), presentations volume (MYH0011/MYH0035); trend
   since 2011-12; Vic vs national.
2. **Elective surgery** — median wait (MYH0009), % within clinically
   recommended time (MYH0008), % waited >365 days (MYH0007); by hospital
   and peer group.
3. **Hospital activity & patient flow** (Victoria) — RE-SCOPED at Session 6,
   BUILT at Session 7. The original "benchmarking map" is dead: Azure Maps + Bing
   maps + custom AppSource visuals are all blocked by tenant settings, and
   peer-group is not a hospital attribute in the data. As built: 5 KPI cards
   (hospital count, total admissions, emergency / surgical / childbirth share);
   top-10 hospitals stacked by emergency vs planned admissions; an average
   length-of-stay gauge (Victoria 3.9 vs national hospital average 3.7); and a
   total-admissions trend line over time with a median reference. The treemap and
   the LOS-by-condition bar from the Session-6 plan were dropped after a data audit
   — MYH0024 stores overlapping reported-measure groupings that multi-count, and
   several candidate breakdowns (care-type-over-time, private, bed days) proved
   sparse or absent. Full detail in PROJECT_CONTEXT.md Session 7 and LEARNINGS
   M3-34..M3-41.

Every page is built from live-verified measures only.

## 7. Phase breakdown (3-5 days)

- **Phase 0 — setup (2026-06-12, pre-build + day 0).** Environment gate
  (PASSED), data audit re-run (PASSED), plan + context docs, git + GitHub
  repo + README skeleton, dashboard pages confirmed, standards audit.
- **Phase 1 — ingestion (day 1).** Forward-verify pass on Fabric Data
  Pipeline REST copy. Pipeline lands raw JSON per endpoint into Bronze
  (Files): measure-categories, measures, datasets, reporting-units,
  reporting-unit-types, data-items per measure (~33 calls). Verify file
  counts + sizes.
- **Phase 2 — Silver (day 2).** Forward-verify pass on PySpark/Delta in
  Fabric. Notebook flattens each Bronze entity to a Silver Delta table;
  data_set_id → datasets period join applied here; suppression flags kept
  as columns. Row-count + null-rate verification cells.
- **Phase 3 — Gold (day 2-3).** Star schema build in the notebook:
  fact_measure_value + dim_hospital + dim_measure + dim_period +
  dim_reported_measure (is_total flag, double-count test). Verification:
  FK integrity, no orphan facts, variant-totals test.
- **Phase 4 — BI (day 3-4).** Semantic model on Gold (Direct Lake);
  relationships + documented DAX measures; the three pages; Vic filter +
  national benchmark context. Export Import-mode .pbix to repo.
- **Phase 5 — ship (day 4-5).** README final (architecture diagram,
  screenshots, audit paragraph, AI-disclosure block), repo About + tags +
  profile entry + pin, screen recording of the live workspace, LEARNINGS
  consolidation, final standards audit, v1.0.

Phase-kickoff forward-verify pass (15-30 min, authoritative docs only) at
every phase boundary, findings banked in LEARNINGS.md before work ships.

## 8. Risks + mitigations (banked at kickoff, updated Phase 0)

1. ~~Tenant/trial environment~~ — RESOLVED: gate passed 2026-06-12 on the
   existing Pheluciam tenant; officially documented personal-email path.
2. Trial-clock pressure — 60 days vs 3-5 build days; schedule build days
   adjacent. Expiry ~2026-08-11.
3. data_set_id → period join missed = facts with no dates. Mitigation:
   ingest /datasets in Bronze from the start; locked Silver step.
4. Reported-measure variants double-counting. Mitigation:
   dim_reported_measure with is_total flag; test for it.
5. Suppressed values arriving as nulls/flags. Mitigation: keep suppression
   flag as a fact column; PBI measures respect it.
6. Direct Lake vs Import confusion at demo time. Mitigation: .pbix Import
   export is what reviewers open; Direct Lake is what the recording shows.
7. Data-version drift: API reported 2026031701 at Phase 0 audit (kickoff
   noted 2026-05-28). Snapshot pattern makes this moot post-ingestion; the
   version lands in Bronze metadata.

## 9. Scope discipline

- ONE API, ~33 calls, ONE Lakehouse, flat star schema.
- Feature-creep watchlist (all OUT unless days 1-3 land early): LHN/PHN
  analysis, financial deep-dive, infection pages, OneLake shortcuts,
  Dataflows Gen2, workspace git integration.
- Past 5 days: cut scope, do not extend.

## 10. Conventions

- README per the portfolio template (Project #3 canonical); AI-assistance
  disclosure block; professional presentation on all public surfaces.
- ENGINEERING_STANDARDS.md ten-point audit per script; structural audit +
  forward-verify pass at phase boundaries.
- One bundled commit per session close; no pushes unless asked.
- Ship-first debugging; non-trivial bugs banked in LEARNINGS.md at close.
