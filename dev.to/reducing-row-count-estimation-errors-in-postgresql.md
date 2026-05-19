---
title: "Reducing row count estimation errors in PostgreSQL"
published_at: 2026-02-06
tags: postgres
---

## Introduction

PostgreSQL's query planner relies on table statistics to estimate the number of rows (estimated rows) each operation will process, and then selects an optimal execution plan based on these estimates. When the estimated rows diverge significantly from the actual rows, the planner can choose a suboptimal plan, leading to severe query performance degradation.

This article walks through four approaches I used to reduce row count estimation errors, ordered from least to most invasive. Due to confidentiality constraints, I cannot share actual SQL or execution plans, so the focus is on the diagnostic thought process and the techniques applied.

## Prerequisites

- The approaches in this article are applicable to any modern PostgreSQL version, as the underlying mechanisms (autovacuum, `pg_statistic`, extended statistics) have been stable across versions
- The target table had a high update frequency
- Actual SQL and execution plans cannot be shared; this article focuses on methodology

## Approach 1: Tuning autovacuum auto-ANALYZE frequency per table

The target table was known to have a very high update rate, so the first hypothesis was that the statistics were simply stale.

In PostgreSQL, the autovacuum daemon automatically runs `ANALYZE` to update statistics stored in `pg_statistic`. However, for tables with heavy write activity, auto-`ANALYZE` may not keep up with the rate of change, causing the statistics to drift from reality.

To address this, I adjusted the auto-`ANALYZE` frequency for the specific table rather than changing the server-wide settings in `postgresql.conf`.

The two key parameters are:

- `autovacuum_analyze_threshold`: The minimum number of tuple modifications before auto-`ANALYZE` is triggered (default: 50)
- `autovacuum_analyze_scale_factor`: The fraction of the table size added to the threshold (default: 0.1, i.e., 10%)

```sql
ALTER TABLE table_name SET (
    autovacuum_analyze_threshold = 0,
    autovacuum_analyze_scale_factor = 0.01
);
```

In this example, `autovacuum_analyze_threshold` is set to 0 (default: 50) and `autovacuum_analyze_scale_factor` is reduced from 0.1 to 0.01, so that auto-`ANALYZE` triggers after just 1% of the table has been modified.

> **Note:** You can verify whether auto-`ANALYZE` is keeping up by querying the `pg_stat_user_tables` view. Check the `last_autoanalyze` column for the timestamp of the last auto-`ANALYZE`, and `n_mod_since_analyze` for the number of row modifications since the last `ANALYZE`.

For the full list of per-table storage parameters, see the [PostgreSQL documentation on storage parameters](https://www.postgresql.org/docs/current/sql-createtable.html#SQL-CREATETABLE-STORAGE-PARAMETERS).

## Approach 2: Increasing the statistics sampling target per column

After adjusting autovacuum frequency, the next hypothesis was that `ANALYZE` was running often enough but the sample size was too small to produce accurate statistics.

PostgreSQL's `ANALYZE` collects samples from each column and stores most-common values (MCVs) and histograms in `pg_statistic`. The precision of this information is governed by `default_statistics_target` (default: 100), which controls the number of histogram buckets and MCV entries.

Rather than changing the server-wide setting, I increased the statistics target on a per-column basis for the affected table:

```sql
ALTER TABLE table_name ALTER COLUMN column_name SET STATISTICS 500;
```

As a general guideline, setting the target to 500-1000 for columns frequently used in `WHERE` clauses is a common tuning step. However, higher values increase `ANALYZE` execution time and the size of `pg_statistic`, so there is a trade-off to consider.

> **Note:** After changing the statistics target with `SET STATISTICS`, you must run `ANALYZE` (or wait for the next auto-`ANALYZE`) for the new setting to take effect.

## Approach 3: Using extended statistics

Even after improving the freshness and precision of the base statistics, row count estimation errors can persist. This happens when the planner's estimation model itself has structural limitations.

By default, PostgreSQL's planner assumes that conditions on different columns are independent. When this assumption is violated, the planner multiplies selectivities independently, often resulting in a dramatic underestimation of the actual row count.

This occurs when:

- A table has two columns `a1` and `a2`
- There is a functional dependency between them (e.g., `a1` determines `a2`)
- The `WHERE` clause includes conditions on both columns

A concrete example would be columns like `country` and `city` -- knowing the country largely determines the set of possible cities. The planner treats the selectivities of each condition as independent, producing an estimate far lower than the actual row count.

To address this, I created extended statistics on the correlated columns:

```sql
CREATE STATISTICS stat_name ON a1, a2 FROM table_name;
```

`CREATE STATISTICS` supports three kinds of statistics: `ndistinct` (multi-column distinct counts), `dependencies` (functional dependencies), and `mcv` (multi-column most-common values). If you omit the kind specification, all three are collected, which is what I did as a starting point.

> **Note:** `CREATE STATISTICS` only defines the statistics object. The actual statistics are not populated until `ANALYZE` runs on the table.

For more details, see the [PostgreSQL documentation on CREATE STATISTICS](https://www.postgresql.org/docs/current/sql-createstatistics.html).

## Approach 4: Using pg_hint_plan as a last resort

When statistics-based approaches are not sufficient, [pg_hint_plan](https://github.com/ossc-db/pg_hint_plan) provides a way to directly control the planner's behavior through SQL comment-based hints.

For row count estimation issues specifically, the `Rows` hint allows you to override the planner's row estimate for a given table:

```sql
/*+ Rows(table_name #1000) */ SELECT ...
```

In the example above, the estimated row count for `table_name` is fixed at 1000. You can also use `+`, `-`, or `*` instead of `#` to add, subtract, or multiply the planner's original estimate.

However, hint-based approaches come with significant drawbacks:

- **Fragile to data changes**: Fixed row counts become inaccurate as data volumes change
- **Reduced maintainability**: Team members unfamiliar with the hints may be confused by them
- **Masks root causes**: Hints can hide underlying statistics or schema issues that should be properly addressed

For these reasons, I recommend using hints only when statistics-based approaches have been exhausted, or as a temporary measure while investigating the root cause.

## Conclusion

This article covered four approaches to reducing row count estimation errors in PostgreSQL, ordered by increasing invasiveness:

1. **Tune autovacuum frequency**: Are the statistics stale?
2. **Increase the statistics target**: Is the sample size sufficient?
3. **Create extended statistics**: Can the planner account for cross-column correlations?
4. **Apply hint clauses**: A last resort when statistics alone cannot solve the problem

When facing row estimation errors, a systematic approach works best: start with `EXPLAIN ANALYZE` to compare estimated vs. actual row counts, then work through the possible causes in order -- **statistics freshness, then precision, then structural limitations of the estimation model**. I hope this article serves as a useful reference in your troubleshooting process.

## References

- [PostgreSQL documentation: table storage parameters](https://www.postgresql.org/docs/current/sql-createtable.html#SQL-CREATETABLE-STORAGE-PARAMETERS)
- [PostgreSQL documentation: CREATE STATISTICS](https://www.postgresql.org/docs/current/sql-createstatistics.html)
- [pg_hint_plan (GitHub)](https://github.com/ossc-db/pg_hint_plan)
