# CLAUDE.md — Fleet Street Analytics

Guidance for Claude Code when working in `fleet_street_analytics/`. Read `docs/PRD.md` first. It is the source of truth for scope.

## What this is
A multi-party political analytics platform for Ireland. It predicts party vote share per street segment (groups of 10 or more households) from generalised public data plus party canvass data, and shows the results on a map to guide canvassing.

## The three-tier data model (hard invariants)

| Tier | Owner | Holds | Can be read by |
|---|---|---|---|
| **Master** | Fleet Street Analytics | Public/scraped data, generalised to area level. **No personal data.** | FSA staff, analytics engine |
| **Party** (1 per party) | That party | Generalised aggregates from its TDs' DBs | That party, analytics engine (that party's jobs only) |
| **TD** (1 per TD) | That TD | Address/person-level canvass data | That TD (and staff they delegate) |

Allowed data flows. **Anything not listed here is forbidden:**
1. Public sources → quarantine → **generalise** → Master
2. TD → **generalise** → Party (same party only)
3. Master + one Party → analytics engine → predictions written to **that Party only**

Never write code that:
- writes Party or TD data (raw or aggregated) into Master;
- lets one party read another party's data, or lets a Party read raw TD records;
- stores personal data (name, exact house number, per-household leaning, individual vehicle records) in Master;
- outputs or displays any group with fewer than `K_MIN` households (default 10, floor 5) outside a TD's own view;
- does a cross-tier write that bypasses the generalisation pipeline;
- puts data from two parties into one analytics job.

If a task seems to need one of these, stop and ask. Do not work around it.

## Generalisation pipeline
- It is the only path for cross-tier writes. It is pure, deterministic and unit-tested.
- Steps: strip direct identifiers → map to spatial key (street segment ID / CSO Small Area ID) → coarsen quasi-identifiers (age bands, month-level dates) → aggregate → enforce `K_MIN` (merge with neighbours or suppress) → write an audit record.
- Any change to it needs tests that show `K_MIN` is still enforced and no identifiers leak.

## Proposed stack (not yet built; confirm before deviating)
- **DB:** PostgreSQL + PostGIS. Separate databases per tier. Per-party and per-TD isolation through separate schemas/DBs plus row-level security. Per-tenant encryption keys.
- **Backend:** Python 3.12, FastAPI, SQLAlchemy, Alembic.
- **Ingestion / analytics:** Python (pandas/geopandas, scikit-learn / PyMC for small-area estimation). Jobs run isolated per party.
- **Frontend:** React + TypeScript, MapLibre GL, vector tiles.
- **Hosting:** EU region only.

Proposed layout:
```
fleet_street_analytics/
  docs/PRD.md
  backend/        # API, auth, tenancy
  ingestion/      # one connector per public source (with licence/ToS metadata)
  generalise/     # generalisation pipeline + tests
  analytics/      # models, backtests
  frontend/       # map UI
```

## Conventions
- Every ingestion connector declares: source URL, licence/ToS note, refresh schedule, and retention for raw data (default ≤7 days in quarantine).
- Every TD/Party data access goes through the audit log helper. No direct queries that skip it.
- The TD schema is fixed and explicit. Do not add free-form "any data" fields. New fields need a PRD update and a note on the DPIA impact.
- Data-subject rights (access, erasure, objection) must work end to end. Erasing a TD record triggers regeneration of the affected Party aggregates.
- Tests use synthetic data only. Never commit real personal data, scraped dumps, or credentials.
- Use Irish terms consistently: TD, Eircode, CSO Small Area, constituency, tally.

## Compliance context (for design decisions, not legal advice)
- Political opinions are GDPR Art. 9 special-category data. Parties/TDs process them under the Irish DPA 2018 electoral provisions. FSA is a **processor** for Party/TD data and a **controller** only for Master.
- Individual vehicle registration data and full Eircode (ECAD) data are not free public data. Use licensed or aggregate sources only.
- Open legal questions are listed in PRD §10. Do not settle them in code by assumption.

## Commands
None yet. Add build/test/lint commands here as the project is scaffolded.
