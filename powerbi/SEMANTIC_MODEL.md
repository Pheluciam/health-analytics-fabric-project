# Power BI semantic model — sm_health_analytics_gold

Canonical definition of the star schema, relationships, and DAX measures for the
health-analytics-fabric Gold layer. Built twice from this spec:

- **Direct Lake** semantic model `sm_health_analytics_gold` in the Fabric workspace
  (web modeling) — the live platform-story deliverable, reads Gold Delta tables
  directly from OneLake.
- **Import-mode `.pbix`** in Power BI Desktop (connected to the lakehouse SQL
  analytics endpoint, Import storage mode) — the durable artifact that opens after
  the Fabric trial capacity is deleted (~2026-08-11).

Both point at the same five Gold tables in `lh_health_analytics`.

## Tables

| Table | Role | Notes |
| --- | --- | --- |
| fact_measure_value | Fact | 1,700,870 rows; grain = one AIHW data item |
| dim_hospital | Dimension | 1,166 rows (1,165 type-H + Unknown member key −1) |
| dim_measure | Dimension | 33 rows |
| dim_period | Dimension | 54 rows (15 financial years + 39 audit windows) |
| dim_reported_measure | Dimension | 690 rows (10 `is_total` aggregates) |
| _Measures | Measure group | Disconnected `{BLANK()}` table; hosts all measures; Value column hidden |

## Relationships

All single-direction, many-to-one, active, assume referential integrity.
From `fact_measure_value` (many) to each dimension (one):

| From column (fact) | To table | To column |
| --- | --- | --- |
| hospital_key | dim_hospital | hospital_key |
| measure_key | dim_measure | measure_key |
| period_key | dim_period | period_key |
| reported_measure_key | dim_reported_measure | reported_measure_key |

## Measures

> **CANONICAL SOURCE: `powerbi/measures.tmdl` (22 measures as of Session 7, 2026-06-15).**
> The per-measure DAX listed further below is the original 10-measure set and is now
> PARTLY STALE — trust `measures.tmdl`, not the blocks below, where they differ.
>
> **Session-7 changes (Page 3 build):** removed the 5 benchmarking measures and 2
> unused general measures; added the Page-3 set — Total Admissions, Admissions Value,
> Emergency Share %, Surgical Share %, Childbirth Admissions, Childbirth Share %,
> Planned Share %, Emergency Admissions, Planned Admissions, Length of Stay,
> National Length of Stay. The computed shares (Emergency / Surgical / Childbirth /
> Planned) inline the Total-Admissions denominator inside `DIVIDE` rather than
> referencing the `[Total Admissions]` measure — a measure-to-measure reference failed
> to resolve on TMDL Apply ("cannot find Total Admissions"), so each share is
> self-contained. Computed shares use `formatString: "0.0%"` (built-in percent, ×100,
> because `DIVIDE` returns a 0–1 fraction) — NOT the `0.0\%` literal-percent form used
> by the raw 0–100 AIHW percentage measures.
>
> **National Length of Stay** = MYH0014 averaged across ALL Australian hospitals
> (`REMOVEFILTERS(dim_hospital)` + `reporting_unit_type_code = "H"`). NOT pinned to
> `"NAT"` — MYH0014 has no national rollup row (length of stay is published per
> hospital only), so a `"NAT"` filter returns BLANK. Used as the gauge target vs
> Victoria's 3.9 days (national 3.7).
>
> **Page 3 = Hospital Activity & Patient Flow — Victoria.** Page filters: type H +
> Victoria. Final layout: 5 KPI cards (Hospital Count, Total Admissions, Emergency
> Share %, Surgical Share %, Childbirth Share %); top row = Top-10 hospitals stacked
> bar (Emergency vs Planned Admissions, % on tooltip) + LOS gauge (Victoria vs
> national); bottom full-width = Total Admissions trend line by financial year with a
> median reference line. Financial Year slicer is synced across all three report pages;
> each page's trend line is exempted from the slicer via Edit interactions so a
> year-pick doesn't collapse it.
>
> **Data quirks found this session (see LEARNINGS M3-34..M3-39):** MYH0024 admissions
> publishes the SAME admissions under several overlapping reported-measure groupings in
> one column — summing the whole column triple-counts (the spurious 78.9M / 23.8M we
> hit). Only `reported_measure_name = "Total"` (21.25M) or one MECE partition is safe.
> The care-type split sums to 23.8M ≠ 21.3M (differential suppression coverage). The
> emergency/planned care-type breakdown only exists for 2015-16 & 2016-17, so it cannot
> be a time series. `dim_hospital[private]` is mostly null → unusable. MYH0018 bed days
> and MYH0015 % overnight have no `is_total` row → no clean total to trend.
>
> **Session-6 corrections (verified against the live AIHW API):**
> - `ED Median Waiting Time` (MYH0010) was MISLABELLED — MYH0010 is a percentage
>   ("% commenced treatment within recommended time"), not minutes. Renamed
>   **ED Treatment Started On Time**, reformatted `0.0\%`.
> - MYH0010 and MYH0011 (ED presentations) have **no is_total row** (reported only by
>   triage category) — their measures DROP the `is_total` filter.
> - Real ED time metrics added: **ED Median Departure Time** (MYH0036) and
>   **ED 90% Departure Time** (MYH0013).
> - Elective measures filter by `reported_measure_name` to a MECE partition, not
>   is_total: MYH0006/0008/0009 use the 3 clinical-urgency names; MYH0007 uses the 11
>   surgical-specialty names (it has no urgency split). **Elective Surgeries** (MYH0006)
>   added.
> - Benchmarking measures added for the (now-superseded) Vic-vs-national page:
>   **ED 4-Hour Rate Vic**, **ED 4-Hour Rate National**, **ED 4-Hour Gap vs National**,
>   **Hospitals Beating National**, **Vic ED Rank** (returns "X of 8" text; RANKX over
>   the type-"S" state rows). These are slated for removal/repurposing when Page 3 is
>   rebuilt as Hospital Activity & Patient Flow (see PROJECT_CONTEXT Session 6).
> - All formats set deterministically via TMDL, NOT the format dropdown. Percentages
>   use `0.0\%` (literal %, no x100).

All numeric measures filter `suppression_count = 0` (clean values only) and, where
a single aggregated grain is wanted, `dim_reported_measure[is_total] = TRUE()` to
avoid double-counting across up to 127 reported-measure variants.

Percentages are stored on a 0–100 scale by AIHW (e.g. 73.4 = 73.4%), so the
percentage measures use the custom format string `0.0\%` — the backslash makes the
`%` a literal sign that is NOT multiplied by 100 (a bare `0.0%` would render 7340%).
Confirmed against Microsoft Learn custom-format-strings docs.

### ED performance

**ED 4-Hour Departure Rate** — custom format `0.0\%` (MYH0005)

```dax
ED 4-Hour Departure Rate =
CALCULATE (
    AVERAGE ( fact_measure_value[value] ),
    fact_measure_value[suppression_count] = 0,
    dim_measure[measure_code] = "MYH0005",
    dim_reported_measure[is_total] = TRUE ()
)
```

**ED Presentations** — Whole number, thousands separator (MYH0011)

```dax
ED Presentations =
CALCULATE (
    SUM ( fact_measure_value[value] ),
    fact_measure_value[suppression_count] = 0,
    dim_measure[measure_code] = "MYH0011",
    dim_reported_measure[is_total] = TRUE ()
)
```

**ED Median Waiting Time** — Whole number (MYH0010)

```dax
ED Median Waiting Time =
CALCULATE (
    AVERAGE ( fact_measure_value[value] ),
    fact_measure_value[suppression_count] = 0,
    dim_measure[measure_code] = "MYH0010",
    dim_reported_measure[is_total] = TRUE ()
)
```

### Elective surgery

**Elective Median Wait** — Whole number (MYH0009)

```dax
Elective Median Wait =
CALCULATE (
    AVERAGE ( fact_measure_value[value] ),
    fact_measure_value[suppression_count] = 0,
    dim_measure[measure_code] = "MYH0009",
    dim_reported_measure[is_total] = TRUE ()
)
```

**Elective % Within Recommended Time** — custom format `0.0\%` (MYH0008)

```dax
Elective % Within Recommended Time =
CALCULATE (
    AVERAGE ( fact_measure_value[value] ),
    fact_measure_value[suppression_count] = 0,
    dim_measure[measure_code] = "MYH0008",
    dim_reported_measure[is_total] = TRUE ()
)
```

**Elective % Waited Over 365 Days** — custom format `0.0\%` (MYH0007)

```dax
Elective % Waited Over 365 Days =
CALCULATE (
    AVERAGE ( fact_measure_value[value] ),
    fact_measure_value[suppression_count] = 0,
    dim_measure[measure_code] = "MYH0007",
    dim_reported_measure[is_total] = TRUE ()
)
```

### Benchmark / general

**Hospital Count** — Whole number, thousands separator

```dax
Hospital Count =
CALCULATE (
    DISTINCTCOUNT ( fact_measure_value[hospital_key] ),
    fact_measure_value[hospital_key] <> -1,
    fact_measure_value[suppression_count] = 0
)
```

**National Benchmark Value** — Decimal number, 1 dp

```dax
National Benchmark Value =
CALCULATE (
    AVERAGE ( fact_measure_value[value] ),
    fact_measure_value[suppression_count] = 0,
    fact_measure_value[reporting_unit_type_code] = "NAT",
    dim_reported_measure[is_total] = TRUE ()
)
```

**Selected Value** — Decimal number, 1 dp (generic clean value for flexible visuals)

```dax
Selected Value =
CALCULATE (
    AVERAGE ( fact_measure_value[value] ),
    fact_measure_value[suppression_count] = 0
)
```

**Suppressed Data Points** — Whole number, thousands separator (data-quality indicator)

```dax
Suppressed Data Points =
CALCULATE (
    COUNTROWS ( fact_measure_value ),
    fact_measure_value[suppression_count] > 0
)
```

## Build notes

- **Reserved-name fix (Phase 4):** the Silver table `silver.measures` broke the SQL
  analytics endpoint metadata sync — `measures` is a reserved object name in the
  Analysis Services / semantic-model engine ("Invalid AS model object name … the
  name of the object 'Table' cannot be the reserved string 'measures'"). The whole
  endpoint sync failed, so no Gold/Silver schema surfaced to either model picker.
  Fixed by renaming the table to `silver.measures_ref` (notebooks + context doc
  updated to match). The pipeline's `measures` API endpoint name is unchanged.
- The `_Measures` table is a DAX `{BLANK()}` calculated table. Direct Lake allows
  calculated tables only when they do not reference Direct Lake columns — a blank
  measure-holder qualifies.
- **Bulk measure load via TMDL (Import .pbix):** all 10 measures + their formats
  are defined once in `powerbi/measures.tmdl` and loaded into the Import model in a
  single TMDL View paste + Apply (Power BI Desktop). The `formatString` property on
  each measure carries the number format ("0.0" = 1-dp decimal, "#,0" = whole with
  thousands, "0" = whole), so no manual per-measure formatting is needed. Re-usable:
  paste `measures.tmdl` into TMDL View to recreate the whole measure set.
- **TMDL gotcha (banked M3-25):** the script must start with the `createOrReplace`
  command. And `createOrReplace` cannot convert an existing import-mode table's
  partition to `calculated` ("Changing the partition type … is not allowed"). If a
  placeholder `_Measures` already exists (e.g. created via Enter data), delete it
  first, then Apply — `createOrReplace` then builds it fresh as a calculated table.
