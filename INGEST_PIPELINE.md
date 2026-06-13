# Ingestion pipeline walkthrough — pl_ingest_myhospitals_bronze

Companion doc to [pipelines/pl_ingest_myhospitals_bronze.json](pipelines/pl_ingest_myhospitals_bronze.json),
the definition exported from the Fabric pipeline editor (the `</>` view-code panel
on each activity node). This doc explains what every block does and why it is
configured that way.

## What the pipeline does

One run = one full national snapshot of the AIHW MyHospitals API landed as raw
JSON files in the Bronze layer of the `lh_health_analytics` Lakehouse:

| Loop | Items | Calls | Lands in |
| --- | --- | --- | --- |
| ForEach_ListEndpoints | 5 list endpoints | 5 | `Files/bronze/*.json` |
| ForEach_MeasureDataItems | 33 measure codes | 33 | `Files/bronze/data_items/data_items_MYH*.json` |

The pipeline is a courier, nothing more: no mapping, no flattening, no type
conversion. All transformation belongs to the Silver PySpark notebook (Phase 2).
This is the snapshot pattern — the API is called once during the build, the raw
responses are frozen in Bronze, and the live demo path never touches the API again.

## Design decisions

- **Two ForEach loops, parameter-driven.** The endpoint list and the measure-code
  list live in pipeline parameters (`p_list_endpoints`, `p_measure_codes`), not
  hard-wired into activities. Changing scope means editing a default value, not
  rewiring the canvas. The measure codes are a static array rather than a dynamic
  Lookup because the snapshot is meant to be deterministic and reviewable — the
  exact 33 codes pulled are visible in version control.
- **`p_list_endpoints` is an array of objects** (`endpoint` + `file`) so one loop
  carries both the relative URL and the destination file name per item.
  `p_measure_codes` is a plain string array because the URL and file name are both
  derived from the code with `@concat()`.
- **Sequential loops** (`"isSequential": true`). One request at a time — polite to
  a free public API. The two loops themselves start in parallel (no dependency
  line between them), which is acceptable: at most two concurrent requests.
- **Mapping tab left empty.** REST source returning JSON to a JSON file sink is a
  hierarchical-to-hierarchical copy; Fabric copies the payload as-is. Touching the
  Mapping tab would force tabular flattening — exactly what Bronze must not do.
- **Anonymous REST connection** (`conn_myhospitals_api`, base URL
  `https://myhospitalsapi.aihw.gov.au/api/v1`). The API is open, CC-BY licensed,
  no keys — so no secrets exist anywhere in this pipeline.
- **Timeout 10 minutes, retry 2 at 30 s** on every Copy activity. The default
  12-hour activity timeout is pointless for sub-200 MB payloads; a tight timeout
  surfaces a hung call fast, and two retries absorb transient API hiccups.

## Reading the JSON, block by block

### Parameters block

`p_list_endpoints` maps each list endpoint to its Bronze file name. Underscores in
file names (API hyphens swapped out) keep downstream Spark/SQL references clean.

### ForEach_ListEndpoints

- `items` → `@pipeline().parameters.p_list_endpoints` — the loop iterates the
  parameter array; inside the loop, `@item()` is the current object.
- Source `relativeUrl` → `@item().endpoint` — appended to the connection base URL,
  e.g. `measures` becomes `https://myhospitalsapi.aihw.gov.au/api/v1/measures`.
- Sink `fileName` → `@item().file`, `folderPath` → `bronze`.

### ForEach_MeasureDataItems

- `items` → `@pipeline().parameters.p_measure_codes` — here each `@item()` is a
  plain string like `MYH0005`.
- Source `relativeUrl` → `@concat('measures/', item(), '/data-items')`.
- Sink `fileName` → `@concat('data_items_', item(), '.json')`,
  `folderPath` → `bronze/data_items`.

### Blocks that landed as defaults (not deliberate choices)

- `requestInterval: 00.00:00:00.010` — the 10 ms floor between *pagination pages*.
  MyHospitals returns complete payloads (no paging), so this never fires.
- `paginationRules: supportRFC5988` — connector default; inert for the same reason.
- `enableStaging: false` — no interim staging store needed for direct
  API-to-Lakehouse copies.
- The `workspaceId` / `artifactId` / `connection` GUIDs are environment-specific
  references for the trial workspace, not secrets — they are unusable without an
  authenticated identity in the tenant.

## Run results (2026-06-13, first snapshot)

- 38 copy activities, 38 succeeded, zero retries consumed. Wall clock ~16.5 min.
- Bronze landing verified: 5/5 list files, 33/33 data-items files, all non-zero.
- API data_version at snapshot: 2026052802 (uploaded 2026-05-28).
- Payload reality check: scoping estimated "a few hundred KB" per data-items call;
  actual files run up to 170 MB (MYH0007), with several in the 60-170 MB band.
  Raw JSON is verbose (repeated key names per record) — Delta/Parquet will shrink
  this dramatically in Silver.
- MYH0037 and MYH0038 (relative stay index) return an empty `result` array from
  the API itself (verified directly) — 189 B envelope files, faithful copies of
  what the source publishes. Both are stretch-scope measures; no action.

## Re-running

The pipeline is idempotent: file names are deterministic, so a re-run overwrites
the snapshot in place. Re-run only if a fresh snapshot is wanted — the data_version
captured in each file's `version_information` envelope will move accordingly.
