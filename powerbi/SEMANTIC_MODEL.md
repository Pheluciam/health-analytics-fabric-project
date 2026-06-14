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

## Measures (10)

All numeric measures filter `suppression_count = 0` (clean values only) and, where
a single aggregated grain is wanted, `dim_reported_measure[is_total] = TRUE()` to
avoid double-counting across up to 127 reported-measure variants.

Percentages are stored on a 0–100 scale by AIHW (e.g. 73.4 = 73.4%), so the three
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
