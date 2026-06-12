# Hospital System Performance Analytics — Microsoft Fabric End-to-End

An end-to-end analytics platform build on **Microsoft Fabric**: Australian
hospital performance data (emergency department waits, elective surgery,
admissions) ingested from the AIHW MyHospitals open REST API into a
**medallion Lakehouse** (Bronze → Silver → Gold), transformed with
**PySpark**, modelled as a **star schema** in Delta, and served through a
**Direct Lake Power BI semantic model** and a 3-page dashboard.

> 🚧 In progress — Phase 0 (environment + scaffolding) complete.

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
3. **Hospital benchmarking map** — Victorian hospitals plotted by
   location, peer-group comparison, drill-through to a hospital scorecard.

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
PROJECT_PLAN.md       — locked scope, architecture rules, phase breakdown
PROJECT_CONTEXT.md    — environment, identities, decision log
ENGINEERING_STANDARDS.md — the 10-criteria quality bar every script meets
pipelines/            — Fabric Data Pipeline definitions (exported JSON)
notebooks/            — PySpark notebooks (Bronze→Silver→Gold)
powerbi/              — Import-mode .pbix export + documented DAX measures
docs/                 — walkthrough docs + dashboard screenshots
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
