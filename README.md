# Hospital System Performance Analytics — Microsoft Fabric End-to-End

An end-to-end analytics platform build on **Microsoft Fabric**: Australian
hospital performance data (emergency department waits, elective surgery,
admissions) ingested from the AIHW MyHospitals open REST API into a
**medallion Lakehouse** (Bronze → Silver → Gold), transformed with
**PySpark**, modelled as a **star schema** in Delta, and served through a
**Direct Lake Power BI semantic model** and a 3-page dashboard.

> 🚧 In progress — Phase 2 (Silver) complete. Phase 3 (Gold) next.

## Architecture

```
AIHW MyHospitals REST API
        │  Fabric Data Pipeline (Copy activity — courier only)
        ▼
Lakehouse BRONZE — raw JSON files, one per endpoint (snapshot, frozen)
        │  PySpark notebook — flatten, conform, period join
        ▼
Lakehouse SILVER — Delta tables, one per entity
        │  PySpark notebook — star schema build + tests
        ▼
Lakehouse GOLD — fact_measure_value + dims (hospital, measure, period,
        │       reported_measure)
        ▼
Power BI semantic model (Direct Lake) → 3-page dashboard
```

All compute and storage live in a single Fabric workspace on OneLake —
pipeline, notebook, Lakehouse, semantic model, and report.

## Dashboard pages

1. **ED performance** — 4-hour departure rate, waiting times by triage
   urgency, presentation volumes; trends since 2011-12; Victoria vs
   national benchmark.
2. **Elective surgery** — median waiting time, % treated within clinically
   recommended time, % waiting >365 days; by hospital and peer group.
3. **Hospital activity & patient flow (Victoria)** — admissions volume and
   case-mix. Five KPI cards (hospital count, total admissions, emergency /
   surgical / childbirth share); top-10 hospitals by admissions split
   emergency vs planned; an average-length-of-stay gauge comparing Victoria
   to the national hospital average; and a total-admissions trend over time
   with a median reference line. (The original benchmarking-map concept was
   dropped — Azure/Bing maps and custom visuals are blocked on the trial
   tenant, and peer group is not a hospital attribute in the data.)

A note on the admissions data: AIHW publishes the same admissions under
several overlapping breakdowns in one field, so totals are taken from the
explicit "Total" reported measure (or a single mutually-exclusive partition)
to avoid double-counting. The emergency/planned care-type split is only
reported for two years, so it appears as a current-state split (cards, by
hospital) rather than a trend; length of stay has no national rollup, so the
benchmark is the average across all Australian hospitals. Full data-quirk
notes are in `LEARNINGS.md` (M3-34..M3-41) and `powerbi/SEMANTIC_MODEL.md`.

_Screenshots land here at ship._

## Data audit

Data audit (2026-06-12): The AIHW MyHospitals API
(myhospitalsapi.aihw.gov.au, open REST, no auth, CC-BY) was verified live
before build start. 12 measure categories and 33 measures with full units
metadata were confirmed, covering ED presentations, ED waiting times, time
in ED, elective surgery, admissions and length of stay. Facts arrive as
flat JSON data-items keyed by measure, reporting unit and data_set_id;
reporting periods (2011-12 to current, ~14 annual periods) attach via the
datasets endpoint join. Hospital reporting units carry latitude/longitude,
public/private flag and state/LHN/peer-group mappings. Volume is ~33 API
calls for a full national snapshot; institutional aggregates only, zero
PII. API data version at audit: 2026031701.

## Repository structure

```
PROJECT_PLAN.md           — locked scope, architecture rules, phase breakdown
PROJECT_CONTEXT.md        — environment, identities, decision log
ENGINEERING_STANDARDS.md  — the 10-criteria quality bar every script meets
INGEST_PIPELINE.md        — Bronze ingestion pipeline walkthrough
SILVER_NOTEBOOK.md        — Silver transformation notebook walkthrough (Phase 2)
pipelines/                — Fabric Data Pipeline definitions (exported JSON)
notebooks/                — PySpark notebooks (Bronze→Silver→Gold)
powerbi/                  — Import-mode .pbix export + documented DAX measures
docs/                     — dashboard screenshots
```

## How this project was built

This project was built using AI-assisted pair programming (Claude by
Anthropic). All architecture decisions, technology selections, and final
design choices are my own; the AI accelerated implementation and acted as a
senior-DE code reviewer. The intent of the project is portfolio learning —
every component was built with explicit understanding of what it does and
why. Walkthrough docs are in the docs/ folder; the engineering bar each
script is held to is in ENGINEERING_STANDARDS.md.

## Licence + attribution

Source data: Australian Institute of Health and Welfare (AIHW) MyHospitals
API, licensed CC-BY. This project is not affiliated with or endorsed by
the AIHW.
