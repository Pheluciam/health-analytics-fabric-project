# Gold notebook — nb_gold_build

Walkthrough for the Gold layer of the health-analytics-fabric project. The
notebook (`notebooks/nb_gold_build.ipynb`) builds a Kimball star schema in the
`gold` schema of the `lh_health_analytics` lakehouse from the conformed Silver
Delta tables.

## What it builds

A five-table star, schema-namespaced under `gold`:

| Table | Grain | Rows | Notes |
| --- | --- | --- | --- |
| `fact_measure_value` | one row per AIHW data item | 1,700,870 | value + bounds + suppression/caveat counts + four FKs |
| `dim_hospital` | one reporting unit of type H | 1,166 | lat/long, public/private, state, LHN, PHN, plus an Unknown member |
| `dim_measure` | one measure code | 33 | category + units |
| `dim_period` | one distinct reporting period | 54 | 15 financial years + 39 audit windows |
| `dim_reported_measure` | one reported-measure code | 690 | carries the `is_total` flag |

## Design decisions

**Surrogate keys.** Each dimension gets an integer surrogate key generated with
`row_number()` over a window ordered by the business key. This is deterministic
(stable 1..N across re-runs) and consecutive — preferable to
`monotonically_increasing_id()` for a full-reload snapshot build, and the
dimensions are small enough that a single-partition window is cheap. The fact
resolves each surrogate by left-joining Silver on the business key, per the
Kimball load pattern documented for Fabric.

**Fact grain and the Unknown member.** The AIHW `data-items` feed publishes
values not just for hospitals but for peer-group, state, national and LHN
aggregates (type codes PG / S / NAT / LHN — 318,595 of the 1.7M rows). Those
benchmark rows are required by the dashboard's "Victoria vs national" and
peer-group comparisons, so the fact retains every reporting-unit type.
`dim_hospital` stays hospital-only (type H) per the model spec, and a synthetic
Unknown member (`hospital_key = -1`) absorbs the non-hospital facts. The fact
also carries `reporting_unit_code` and `reporting_unit_type_code` as degenerate
dimensions so benchmark rows remain sliceable by type.

**Overlapping reported measures — a downstream double-count trap (banked Session 7).**
`dim_reported_measure` holds every reported-measure variant across all 33 measures,
and a single measure can publish the SAME values under several independent groupings
in this one field — e.g. MYH0024 admissions carries a `"Total"` row PLUS a care-type
partition (Medical/Surgical emergency & non-emergency, Childbirth, Mental health) that
each re-sum to the total. Summing `value` across the whole `reported_measure_name`
field therefore multi-counts (we saw 78.9M / 23.8M vs the true 21.25M). Downstream
(DAX/SQL) must aggregate either the `is_total` row or ONE mutually-exclusive partition,
never the raw field. The `is_total` flag (name = "total" / starts with "all ") is the
safe anchor; some measures (length of stay, % overnight, ED triage) have NO `is_total`
row and must be averaged across their MECE partition instead. See LEARNINGS M3-35.

**Period derivation.** The source has no financial-year column; periods are
derived from the reporting start/end dates. A true Australian financial year
must start in July **and** end in June — AIHW also publishes shorter
hand-hygiene audit windows (Nov–Mar, Apr–Jun, Jul–Oct), some of which also start
in July, so a start-month-only test misclassifies them. Genuine FYs get a
`2010-11`-style label; audit windows get an honest date-range label with
`financial_year = NULL` and `period_type = "Audit period"`.

**is_total.** Reported measures include both disaggregations (e.g. by triage
category) and roll-up totals. Summing across all variants double-counts, so
`dim_reported_measure.is_total` flags the genuine aggregates — rows named
`Total`, `All`, or `All …`. A naive "contains the word total" test is wrong here:
it would flag procedures like *Total hip replacement* while missing *All
patients*, so the flag matches exact aggregate names instead. Power BI measures
filter on `is_total` (and on `suppression_count = 0`) to keep aggregations
correct.

## Cell map

1. **Cell 1 — setup + discovery.** Imports, `CREATE SCHEMA IF NOT EXISTS gold`,
   and a `printSchema` / sample pass over the Silver tables. Silver is the schema
   source of truth: four of the five Silver tables were written with
   `explode(...).select("r.*")` last phase, so their columns are confirmed here
   before any dimension is coded against them.
2. **Cell 2 — dim_period.** Derives the period label/type from the dataset dates.
3. **Cell 3 — dim_hospital.** Filters Silver reporting units to type H, flattens
   the nested `mapped_reporting_units` array to pull state / LHN / PHN, and unions
   the Unknown member.
4. **Cell 4 — dim_measure.** Takes the first category from the measure's category
   array and joins `measure_categories` for the canonical name; flattens units.
5. **Cell 5 — dim_reported_measure.** Distinct reported-measure codes from the
   facts, named from `datasets.reported_measure_summary`, with the `is_total` flag.
6. **Cell 6 — fact_measure_value.** Left-joins the data items to each dimension on
   the business key (period via the dataset's derived period label), coalesces the
   hospital key to the Unknown member, and projects the measures plus degenerate
   columns.
7. **Cell 7 — verification.** Row counts, FK-null checks (Fabric does not enforce
   foreign keys, so the load process must verify integrity itself), the
   hospital-key −1 breakdown by reporting-unit type, the period and is_total
   listings, and a double-count test that confirms reported-measure variants would
   over-count without the `is_total` flag.

## Verification result

All four foreign-key null counts are 0; `hospital_key = -1` is exactly 318,595
and appears only on the non-hospital types; every type-H row (1,382,275) resolves
to a real hospital; the fact row count equals the Silver source exactly (no loss,
no fan-out). The double-count test returns measure/hospital/period grains with up
to 127 reported-measure variants, confirming the `is_total` flag is load-bearing
for downstream aggregation.
