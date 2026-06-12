# PROJECT_CONTEXT.md — health-analytics-fabric-project

> Living session-handoff doc. Read at the start of every session alongside
> TEACHING_PREFERENCES.md and PROJECT_PLAN.md. Updated at every session close.

---

## Current state (as of Phase 0, 2026-06-12)

- **Phase:** 0 (setup) — in progress.
- **Environment gate: PASSED 2026-06-12.** Fabric 60-day trial active,
  expires ~2026-08-11 (items deleted ~7 days after expiry).
- **Build clock:** not yet started. Phase 1 (ingestion) is next.

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
- Phase-boundary audit: caught + fixed three "mini" leaks on public-bound
  files (M3-3). LEARNINGS M3-1..M3-4 banked.
- Phase 0 COMPLETE. Next session: Phase 1 ingestion (forward-verify pass
  on Fabric Data Pipeline REST copy first).
