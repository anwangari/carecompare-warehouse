# carecompare-warehouse

CMS publishes the quality of every U.S. nursing home monthly, then overwrites
it. This warehouse keeps the history, joins it to what Medicare pays, and
answers one question per release.

[Live model docs](#) · [3-minute walkthrough](#) · [Decisions](docs/DECISIONS.md)

---

## Findings

| Release | Question | Finding |
|---|---|---|
| v0.1 | Which nursing home chains are slipping across their portfolio? | *Pending* |
| v0.2 | Does rising staff turnover predict a star drop, and how far ahead? | *Pending* |
| v0.3 | Do higher-rated facilities cost traditional Medicare more or less per stay? | *Pending* |

<!-- Each finding: one sentence, one number, link to its query and chart. Newest on top. -->

---

## How it works

![architecture](diagrams/architecture.png)

Python pulls seven CMS datasets into an immutable S3 landing zone. Postgres
loads them untyped, one partition per period, idempotently. dbt types them,
keeps facility history as a Type 2 dimension, tests every build, and serves one
analysis per question. Airflow runs it monthly and skips any dataset CMS has not
changed.

**Stack:** Python · AWS S3 · PostgreSQL · dbt · Airflow · Docker · GitHub Actions

---

## Read before trusting the numbers

- **Inspection stars are scored within each state**, so they are not compared
  across states. "Slipping" uses staffing and quality-measure ratings, which use
  national thresholds.
- **Cost data is traditional Medicare only.** Medicare Advantage stays are not
  in it.
- **Suppressed values** (small denominators) are treated as missing, never as zero.

---

## Run it

Tested on WSL 2 with Docker Desktop.

```bash
git clone https://github.com/anwangari/carecompare-warehouse.git && cd carecompare-warehouse
cp .env.example .env && echo "AIRFLOW_UID=$(id -u)" >> .env   # add AWS keys
docker compose up -d                                            # Airflow at localhost:8080
```

No AWS account: set `LOCAL_MODE=true` and raw files land in `./data`.

---

## What broke

<!-- Three entries from docs/WHAT_BROKE.md, in my own words. -->

---

Data: public domain, from the CMS [Provider Data Catalog](https://data.cms.gov/provider-data/)
and [data.cms.gov](https://data.cms.gov/). Not affiliated with or endorsed by CMS.
