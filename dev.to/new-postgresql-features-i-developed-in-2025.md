---
title: "New PostgreSQL features I developed in 2025"
published_at: 2025-12-25
tags: postgres
---

## Introduction
I started contributing to PostgreSQL around 2020. This year I wanted to work harder, so I will explain the PostgreSQL features I developed and committed in 2025.

I also committed some other patches, but they were bug fixes or small document changes. Here I explain the ones that seem most useful.

**These are mainly features in PostgreSQL 19, now in development. They may be reverted before the final release.**

## Added a document that recommends default psql settings when restoring pg_dump backups
- Title: Doc: recommend "psql -X" for restoring pg_dump scripts.
- Committer: Tom Lane
- Date: Sat, 25 Jan 2025
- Link: https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d83a108c10a3ec886a24c620a915aa2c5bc023aa

When you restore a dump file made by pg_dump with psql, you may get errors if psql is using non-default settings (I saw it with AUTOCOMMIT=off). The change is only in the docs. It recommends using the psql option `-X` (`--no-psqlrc`) to avoid reading the psql config file.

For psql config file `psqlrc`, see my past blog:
https://zenn.dev/shinyakato/articles/543dae5d2825ee

Here is a test with `\set AUTOCOMMIT off`:
```bash
-- create a test database
$ createdb test1

-- dump all databases to an SQL script file
-- -c issues DROP for databases, roles, and tablespaces before recreating them
$ pg_dumpall -c -f test1.sql

-- restore with psql
$ psql -f test1.sql
~snip~
psql:test1.sql:14: ERROR:  DROP DATABASE cannot run inside a transaction block
psql:test1.sql:23: ERROR:  current transaction is aborted, commands ignored until end of transaction block
psql:test1.sql:30: ERROR:  current transaction is aborted, commands ignored until end of transaction block
psql:test1.sql:31: ERROR:  current transaction is aborted, commands ignored until end of transaction block
~snip~
```

`DROP DATABASE` from `-c` cannot run inside a transaction block, so you get these errors. Also, by default, when a statement fails inside a transaction block, the whole transaction aborts, so later statements also fail.

Other statements also cannot run inside a transaction block, so you will see similar errors when restoring them.

## Added backup_type column to pg_stat_progress_basebackup view
- Title: Add backup_type column to pg_stat_progress_basebackup.
- Committer: Masahiko Sawada
- Date: Tue, 5 Aug 2025
- Link: https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=deb674454c5cb7ecabecee2e04ca929eee570df4

PostgreSQL 17 added incremental backup to pg_basebackup. But the pg_stat_progress_basebackup view had no column to tell full or incremental.

So I added `backup_type`, which shows `full` or `incremental`.

Run pg_basebackup and check pg_stat_progress_basebackup from another terminal:

```bash
$ pg_basebackup -D full
```
`backup_type` shows `full`.
```
=# TABLE pg_stat_progress_basebackup;
-[ RECORD 1 ]--------+-------------------------
pid                  | 853626
phase                | streaming database files
backup_total         | 1592460800
backup_streamed      | 622124544
tablespaces_total    | 1
tablespaces_streamed | 0
backup_type          | full                     <- new!
```

Now take an incremental backup with pg_basebackup -i.
```bash
$ pg_basebackup -i full/backup_manifest -D incl
```
`backup_type` shows `incremental`.
```
=# TABLE pg_stat_progress_basebackup;
-[ RECORD 1 ]--------+-------------------------
pid                  | 854435
phase                | streaming database files
backup_total         | 1613615104
backup_streamed      | 726617088
tablespaces_total    | 1
tablespaces_streamed | 0
backup_type          | incremental              <- new!
```

## COPY FROM now supports files with multi-line headers
- Title: Support multi-line headers in COPY FROM command.
- Committer: Fujii Masao
- Date: Thu, 3 Jul 2025
- Link: https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bc2f348e87c02de63647dbe290d64ff088880dbe

The COPY command has a `HEADER` option. For COPY FROM it controls skipping header rows. For COPY TO it controls outputting column names. Before, COPY FROM could not skip multi-line headers. It only accepted a boolean.

Now it accepts an integer, so it skips the given number of header lines.

```
=# \! cat /tmp/copy.csv
first header
second header
1,one
2,two
3,three

=# COPY t FROM '/tmp/copy.csv' WITH (HEADER 2, FORMAT csv);
COPY 3

=# TABLE t;
 id | data  
----+-------
  1 | one
  2 | two
  3 | three
(3 rows)
```

By the way, MySQL, SQL Server, and Oracle SQL*Loader already had similar functions.
- MySQL: `LOAD DATA ... IGNORE N LINES` [^1]
- SQL Server: `BULK INSERT … WITH (FIRST ROW=N)` [^2]
- Oracle SQL*Loader: `sqlldr … SKIP=N` [^3]

## Split the autovacuum log settings for VACUUM and ANALYZE
- Title: Add log_autoanalyze_min_duration
- Committer: Peter Eisentraut
- Date: Wed, 15 Oct 2025
- Link: https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd3ae378301f7e84c18f7a90f183c3cd4165c0da

We already had `log_autovacuum_min_duration`. It logged VACUUM and ANALYZE by autovacuum when they took more time than the setting (default unit: ms). But ANALYZE mostly reads sampled rows, while VACUUM reads and writes table and index pages, so VACUUM often takes longer (it depends on table design, data size, stats target, workload, extended stats, etc.). We could not set them separately.

Now `log_autovacuum_min_duration` is VACUUM-only, and new `log_autoanalyze_min_duration` is ANALYZE-only. At first we proposed `log_autovacuum_vacuum_min_duration` and `log_autovacuum_analyze_min_duration` for consistency, but changing the existing name would break backward compatibility and affect pg_dump/pg_upgrade, so we stopped.

```
=# CREATE TABLE t (i int, d text) WITH (
  -- autoanalyze settings
  autovacuum_analyze_threshold = 1,
  autovacuum_analyze_scale_factor = 0,
  log_autoanalyze_min_duration = 0,
  -- autovacuum settings
  autovacuum_vacuum_threshold = 1,
  autovacuum_vacuum_scale_factor = 0,
  log_autovacuum_min_duration = 100_000_000
);
=# INSERT INTO t VALUES (1, 'a');
=# DELETE FROM t WHERE i = 1;
```
```
2025-12-03 15:15:39.608 JST [401368] LOG:  automatic analyze of table "postgres.public.t"
avg read rate: 18.229 MB/s, avg write rate: 0.000 MB/s
buffer usage: 155 hits, 7 reads, 0 dirtied
WAL usage: 1 records, 0 full page images, 530 bytes, 0 buffers full
system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s
```

## Show compressed full-page image size in pg_stat_wal
- Title: Add wal_fpi_bytes to pg_stat_wal and pg_stat_get_backend_wal()
- Committer: Michael Paquier
- Date: Tue, 28 Oct 2025
- Link: https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f9a09aa2952039a9956b44d929b9df74d62a4cd4

When `wal_compression` is on, full-page images (FPI) in WAL are compressed. Before, to measure the effect you had to:

1. Run the same benchmark with compression on/off and compare `wal_bytes` in pg_stat_wal. But this is for all WAL, not only FPI, so it is not the exact compression ratio.
2. Run `pg_waldump --fullpage` on the server to see compressed WAL sizes and calculate the ratio. This needs WAL files and server access, so it was not usable in clouds where you cannot log in. (If you can use pg_wal_inspect, maybe you can calculate in the cloud, but I have not checked.)

So I added `wal_fpi_bytes` to pg_stat_wal.

```
=# TABLE pg_stat_wal;
-[ RECORD 1 ]----+-----------------------------
wal_records      | 2031667
wal_fpi          | 288581
wal_bytes        | 6346674376
wal_fpi_bytes    | 1932610356                   <- new!
wal_buffers_full | 424447
stats_reset      | 2025-12-02 19:31:44.16184+09
```

There are also patches to add this info to EXPLAIN (WAL) and VACUUM logs. [^4][^5]

## Added mode and started_by columns to pg_stat_progress_vacuum view
- Title: Add mode and started_by columns to pg_stat_progress_vacuum view.
- Committer: Masahiko Sawada
- Date: Tue, 9 Dec 2025
- Link: https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f9a09aa2952039a9956b44d929b9df74d62a4cd4

VACUUM has several reasons to start (auto, manual, wraparound) and several modes (normal, aggressive, fail-safe). The pg_stat_progress_vacuum view for progress did not show these.

So I added `mode` (`normal`, `aggressive`, `failsafe`) and `started_by` (`manual`, `autovacuum`, `autovacuum_wraparound`).

Here is pg_stat_progress_vacuum:

```
=# TABLE pg_stat_progress_vacuum;
-[ RECORD 1 ]--------+--------------
pid                  | 362895
datid                | 5
datname              | postgres
relid                | 24602
phase                | scanning heap
heap_blks_total      | 8850
heap_blks_scanned    | 5327
heap_blks_vacuumed   | 0
index_vacuum_count   | 0
max_dead_tuple_bytes | 67108864
dead_tuple_bytes     | 0
num_dead_item_ids    | 0
indexes_total        | 0
indexes_processed    | 0
delay_time           | 0
mode                 | normal       <- new!
started_by           | autovacuum   <- new!
```

A similar patch adds `started_by` (`manual`, `autovacuum`) to pg_stat_progress_analyze. [^6]

## Conclusion
I introduced the PostgreSQL features I developed in 2025. I also proposed some other patches, but they are still under discussion. I hope to share them in next year.


[^1]: https://dev.mysql.com/doc/refman/8.4/en/load-data.html#load-data-field-line-handling
[^2]: https://learn.microsoft.com/en-us/sql/t-sql/statements/bulk-insert-transact-sql?view=sql-server-ver17#firstrow--first_row
[^3]: https://docs.oracle.com/en/database/oracle/oracle-database/23/sutil/oracle-sql-loader-commands.html#SUTIL-GUID-84244C46-6AFD-412D-9240-BEB0B5C2718B
[^4]: https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5ab0b6a248076bf373a80bc7e83a5dfa4edf13aa
[^5]: https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ad25744f436ed7809fc754e1a44630b087812fbc
[^6]: https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ab40db3852dfa093b64c9266dd372fee414e993f
