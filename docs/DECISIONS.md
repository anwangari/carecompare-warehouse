# Decisions

The choices that shaped this warehouse, and what each one costs.

---

**1. Raw tables are all `TEXT`.**
CMS marks suppressed values with blanks, `.`, footnote codes, and `*`, and the
convention changes by column and month. A typed loader breaks on the first
surprise. Loading text always succeeds; casting happens in dbt, where a bad value
fails a test instead of the pipeline.
*Cost:* no type checks at the boundary. They moved to dbt, not away.

**2. Each load deletes its partition, then rewrites it, in one transaction.**
CMS ships complete files per month (or fiscal year), so a period is a natural
partition. Re-running any load leaves the same result: no duplicates, no partial
state.
*Cost:* rewrites a whole period instead of changed rows. Irrelevant at this size.

**3. Facility history is SCD Type 2, on what a facility *is*, not how it performs.**
Name, ownership, chain, and beds are versioned. Ratings and staffing change every
month by design, so they live in fact tables. Versioning them would create about
14,700 new rows a month and turn the dimension into a fact table.
*Cost:* "what was the rating in March?" needs a fact join. That is the standard shape.

**4. "Slipping" is measured on staffing and quality-measure ratings only.**
Inspection stars are graded on a curve within each state. The data confirms it:
across states, the share of 5-star inspection ratings varies by 2 points, against 12
for staffing and 27 for quality measures. And 64% of chain facilities sit in chains
that cross state lines. Rules, each from EDA on 14,690 facilities: chains keyed on
Chain ID (one-to-one with names), at least 5 facilities (keeps 98% of chain
facilities), at least 80% rated, and at least 2 facilities declining before a chain
counts as slipping.
*Cost:* ignores inspection results, which consumers weigh heavily. Stated in every finding.

**5. Monthly quality and annual cost are never joined directly.**
Joining a monthly table to an annual one repeats each annual payment twelve times.
Quality is rolled up to the CMS fiscal year first, then joined on facility and year.
*Cost:* one extra aggregation step. That step is the point.

**6. Each dataset checks its own freshness.**
Quality data changes monthly, cost data yearly. Each dataset skips itself when CMS
has not changed it, and dbt runs if at least one dataset loaded (Airflow trigger
rule `none_failed_min_one_success`).
*Cost:* trigger rules are less obvious than the default, so the DAG documents them.

**7. Tests fail the run.**
A quality check that only reports gets ignored. Grain violations, orphan facts, and
malformed IDs stop the pipeline. Volume swings and match rates warn.
*Cost:* a flaky test can block a good load, hence the two severities.

---

## Known limits

- Cost data covers traditional Medicare only and lags about two years.
- Cost data is summarized per facility, not claim lines.
- Ownership and chain data are self-reported and incomplete.
- Single-node Postgres and a local Airflow executor: right for this size, not for 100x.
