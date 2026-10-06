# CMS Care Compare → Nursing Home Quality Warehouse

An orchestrated, monthly-refreshing analytics warehouse built on CMS's public
Nursing Home Care Compare data. Tracks how the quality, staffing, and compliance
profile of every Medicare/Medicaid-certified nursing home in the United States
changes over time.

**Data:** 100% public, published by the Centers for Medicare & Medicaid Services.
No PHI. ~14,700 facilities, refreshed monthly.

**Status:** In development.

---

## The question this answers

Nursing home quality is not a fixed attribute — it moves. A facility loses its
Director of Nursing, staffing hours drop, the next state survey finds more
deficiencies, and six months later its Five-Star rating has fallen from 4 to 2.
CMS publishes a fresh snapshot every month but **overwrites the previous one** —
the history is not in the data, it has to be built.

That is the whole point of this project. It builds the history.

Once the history exists, questions that were impossible become one query:

- Which facilities' overall star ratings declined for three consecutive months?
- Which ownership chains are deteriorating fastest across their portfolio?
- Does a drop in RN hours-per-resident-day predict a rating drop, and with what lag?
- How does a given facility compare to its state benchmark, month over month?
- Which facilities changed owners, and what happened to their ratings afterward?

For anyone working in senior living, long-term care analytics, or provider
network management, this is the shape of the real job.

---

## Architecture

![architecture](diagrams/architecture.png)

```
CMS Provider Data Catalog API
        │
        │  1. resolve current download URL from metastore
        ▼
   Python extract  ──────────►  S3 raw zone (immutable, partitioned by snapshot month)
        │                              │
        │                              │  2. load exactly as-is
        ▼                              ▼
                              Postgres  raw.*  (all TEXT + lineage columns)
                                       │
                                       │  3. dbt
                                       ▼
                              staging → snapshots → marts
                                       │
                                       ▼
                        dim_facility (SCD Type 2), fct_facility_rating_monthly,
                        fct_deficiency, fct_penalty, agg_state_month

     Airflow orchestrates all of it, monthly, idempotently.
```

Full design in [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

---

## Source data

Six datasets from the CMS Provider Data Catalog, theme *"Nursing homes including
rehab services"*:

| Dataset | ID | Grain | Refresh |
|---|---|---|---|
| Provider Information | `4pq5-n9py` | one row per facility | monthly |
| Health Deficiencies | `r5ix-sfxw` | one row per citation | monthly |
| Penalties | `g6vv-u9sr` | one row per fine / payment denial | monthly |
| Ownership | `y2hd-n93e` | one row per owner–facility relationship | monthly |
| Survey Summary | `tbry-pc2d` | one row per facility per survey type | monthly |
| State US Averages | `xcdc-v8bm` | one row per state | monthly |

Provider Information alone is ~14,690 rows × 98 columns. Every term in it is
explained in plain English in [`docs/DATA_DICTIONARY.md`](docs/DATA_DICTIONARY.md).

**The download URL is not stable.** CMS publishes each month's file under a
hashed path, e.g. `.../resources/328596835e6db31b2564cd733c3795f4_1786724150/NH_ProviderInfo_Aug2026.csv`.
The pipeline resolves it from the metastore at runtime instead of hardcoding it —
one of the small realities that separates a script from a pipeline.

---

## Stack

| Layer | Tool | Why |
|---|---|---|
| Extraction | Python (`requests`, `boto3`) | Simple, no framework needed |
| Raw storage | AWS S3 | Immutable landing zone; replay without re-downloading |
| Warehouse | PostgreSQL 16 (Docker) | Free, reproducible, SQL that transfers anywhere |
| Transformation | dbt-postgres | Models, tests, snapshots, lineage, docs |
| Orchestration | Apache Airflow 2 (Docker, LocalExecutor) | The scheduler interviewers ask about |
| Environment | Docker Compose | One command, identical on any machine |

Everything except S3 runs locally and costs nothing.

---

## How to run

```bash
# 1. Configuration
cp .env.example .env          # add AWS keys and S3 bucket name

# 2. Bring up Postgres + Airflow
docker compose up -d

# 3. Airflow UI
open http://localhost:8080    # unpause the `carecompare_monthly` DAG

# 4. Or run one snapshot by hand
python -m src.extract --snapshot-month 2026-08
python -m src.load    --snapshot-month 2026-08
dbt snapshot && dbt run && dbt test

# 5. Model documentation and lineage graph
dbt docs generate && dbt docs serve
```

---

## Sample output

<!-- paste one query and its result here once marts are built -->

---

## What this project demonstrates

Written out plainly, because the point of building it is to be able to talk
about it.

**Orchestration.** A real DAG with dependencies, task groups, retries with
exponential backoff, a short-circuit when the source has not changed, and
`catchup=False` with a deliberate reason.

**Idempotency.** Every task is keyed on `snapshot_month`. Re-running any task,
or the whole DAG, for a month that already loaded produces the identical result —
no duplicates, no drift. This is the single most common senior-level interview
probe and this pipeline has a concrete answer to it.

**Slowly Changing Dimensions.** `dim_facility` is SCD Type 2 via dbt snapshots.
When a facility is renamed, sold, or changes ownership type, the old row is
closed out and a new one opened, so a fact from March 2026 still joins to the
facility as it was in March 2026.

**Dimensional modeling.** A star schema with explicitly stated grain for every
fact table, conformed dimensions, and surrogate keys.

**Data quality as a gate, not a report.** dbt tests run *inside* the DAG and
failing tests fail the run. Includes volume-anomaly and referential-integrity
checks, not just `not_null`.

**Incremental loading and watermarks.** A `meta.source_watermark` table records
the CMS `modified` timestamp per dataset; the DAG skips work when nothing
upstream has changed.

**Separation of raw and modeled data.** Raw lands untyped and untouched, with
lineage columns. All interpretation happens in dbt, in version control, where it
can be reviewed and re-run.

---

## Repository layout

```
.
├── dags/                     Airflow DAG definitions
├── src/
│   ├── extract.py            CMS API → S3
│   ├── load.py               S3 → Postgres raw schema
│   └── config.py             Dataset registry
├── dbt/
│   ├── models/staging/       Typed, renamed, one model per source
│   ├── models/marts/         Dimensions, facts, aggregates
│   ├── snapshots/            SCD Type 2 definitions
│   └── tests/                Custom data quality tests
├── sql/                      Analytical queries answering the questions above
├── docs/
│   ├── ARCHITECTURE.md       Technical design
│   ├── DIAGRAM_BRIEF.md      Spec for the architecture diagram
│   ├── DATA_DICTIONARY.md    Plain-English glossary of every CMS term
│   ├── DECISIONS.md          Why things were built this way
│   └── BUILD_PLAN.md         Day-by-day build schedule
├── diagrams/
├── docker-compose.yml
└── .env.example
```

---

## Data source and license

All data is published by the Centers for Medicare & Medicaid Services in the
public domain via the [Provider Data Catalog](https://data.cms.gov/provider-data/).
This project is an independent analysis and is not endorsed by or affiliated
with CMS.