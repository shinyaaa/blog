---
title: "How PostgreSQL Determines Which Tuples Are Visible: A Deep Dive"
published_at: draft
tags: postgres
---

PostgreSQL's MVCC (Multi-Version Concurrency Control) gives each transaction a consistent view of the database without locking reads against writes. But how does PostgreSQL actually decide, at runtime, whether a given tuple is visible to a given transaction? This post digs into the two-level visibility system: **page-level** checks and **tuple-level** MVCC evaluation.

> Based on investigation of PostgreSQL 17devel source code.

---

## Level 1: Page-Level Visibility

### Table Structure

A PostgreSQL table is stored as a file made up of 8 kB pages. Each page contains:

- `PageHeaderData` — metadata about the page
- A series of `ItemId` pointers
- The actual `Item` (tuple) data
- A special space area

```plaintext
+----------------+--------+--------+--------+
| PageHeaderData | ItemId | ItemId |  ...   |
+----------------+--------+--------+--------+
|      Item      |      Item      | Special|
+----------------+--------+--------+--------+
        <----------- 8 kB ----------------->
```

### The `pd_flags` Field

The `PageHeaderData` struct (`src/include/storage/bufpage.h`) contains a `pd_flags` field with several important flag bits:

| Flag | Description |
|---|---|
| `PD_HAS_FREE_LINES` | Page has unused line pointers |
| `PD_PAGE_FULL` | Not enough free space for a new tuple |
| `PD_ALL_VISIBLE` | All tuples on this page are visible to every transaction |
| `PD_VALID_FLAG_BITS` | OR of all valid flag bits |

The most performance-critical flag is **`PD_ALL_VISIBLE`**.

### Why `PD_ALL_VISIBLE` Matters

If a page is marked `PD_ALL_VISIBLE`, PostgreSQL can skip per-tuple visibility checks entirely for that page. This is a significant optimization in read-heavy workloads.

The flag is set via `PageSetAllVisible()` and is written during **VACUUM** and **`COPY FREEZE`**.

### Seeing It in Action

Let's create a table, insert some rows, and observe how `pd_flags` changes.

```sql
CREATE TABLE t (id INT PRIMARY KEY, data TEXT);
INSERT INTO t SELECT generate_series(1,300), md5(clock_timestamp()::TEXT);

CREATE EXTENSION pageinspect;

-- Check flags before VACUUM
SELECT flags FROM page_header(get_raw_page('t', 0));
--  flags
-- -------
--      0   ← not all-visible yet
```

After running `VACUUM`:

```sql
VACUUM t;

SELECT flags FROM page_header(get_raw_page('t', 0));
--  flags
-- -------
--      4   ← PD_ALL_VISIBLE is set (bit 2 = 4)
```

Now update a row on page 0:

```sql
UPDATE t SET data = 'hoge' WHERE id = 1;

SELECT flags, prune_xid FROM page_header(get_raw_page('t', 0));
--  flags | prune_xid
-- -------+-----------
--      2  |  91108    ← PD_ALL_VISIBLE cleared, prune_xid populated
```

The `PD_ALL_VISIBLE` flag is immediately cleared when any tuple on the page is modified.

---

## Level 2: Tuple-Level Visibility (HeapTupleSatisfiesMVCC)

When a page is not fully visible, PostgreSQL falls back to evaluating each tuple individually. This is handled by `HeapTupleSatisfiesMVCC()` and related functions.

### The Call Stack

```plaintext
heap_prepare_pagescan
  └── all_visible = PageIsAllVisible(page) && !snapshot->takenDuringRecovery
        ├── all_visible == true  → skip per-tuple check
        └── all_visible == false → page_collect_tuples
              └── HeapTupleSatisfiesVisibility
                    └── dispatches to:
                          HeapTupleSatisfiesMVCC          (normal queries)
                          HeapTupleSatisfiesNonVacuumable (VACUUM)
                          HeapTupleSatisfiesSelf
                          HeapTupleSatisfiesAny
                          HeapTupleSatisfiesToast
                          HeapTupleSatisfiesDirty
                          HeapTupleSatisfiesHistoricMVCC
```

Which function is called depends on the `SnapshotType`.

### Three Inputs to Visibility Evaluation

#### 1. Tuple Header Data

The `HeapTupleHeaderData` struct (`src/include/access/htup_details.h`) stores:

| Field | Type | Description |
|---|---|---|
| `t_xmin` | TransactionId | XID of the inserting transaction |
| `t_xmax` | TransactionId | XID of the deleting transaction |
| `t_cid` | CommandId | Command ID (cmin for insert, cmax for delete) |
| `t_infomask` | uint16 | Hint bits and other flags |

The key fields for inter-transaction visibility are `xmin` and `xmax`. For intra-transaction visibility, `cid` (command ID) is used.

**Hint bits** (stored in `t_infomask`) cache the results of CLOG lookups to avoid repeated reads:

- `HEAP_XMIN_COMMITTED` / `HEAP_XMIN_INVALID`
- `HEAP_XMAX_COMMITTED` / `HEAP_XMAX_INVALID`
- `HEAP_COMBOCID`

#### 2. CLOG (Commit Log)

CLOG tracks the commit status of every transaction. Each transaction gets 2 bits:

| Status | Value |
|---|---|
| `TRANSACTION_STATUS_IN_PROGRESS` | in flight |
| `TRANSACTION_STATUS_COMMITTED` | committed |
| `TRANSACTION_STATUS_ABORTED` | rolled back |
| `TRANSACTION_STATUS_SUB_COMMITTED` | subtransaction committed |

Defined in `src/include/access/clog.h`, persisted to `$PGDATA/pg_xact`.

#### 3. Snapshot

The `SnapshotData` struct (`src/include/utils/snapshot.h`) captures the transaction landscape at a point in time:

| Field | Description |
|---|---|
| `xmin` | Lowest XID of any in-progress transaction |
| `xmax` | Highest XID of any completed transaction |
| `xip[]` | Array of XIDs that were in-progress at snapshot time |
| `curcid` | Command ID of the command that took the snapshot |

**Snapshot acquisition timing** depends on isolation level:

- **READ COMMITTED**: A new snapshot is taken per SQL statement
- **REPEATABLE READ / SERIALIZABLE**: One snapshot per transaction

---

## MVCC Visibility Logic

For each tuple, PostgreSQL independently evaluates whether the **insert** is valid and whether the **delete** is valid.

A tuple is visible if: **insert is valid** AND **delete is NOT valid**.

### Insert Validity

The insert is considered **invalid** (tuple not yet visible) in these cases:

| Condition | Reason |
|---|---|
| `xmin < snapshot.xmin` AND xmin aborted | Inserting tx rolled back |
| `xmin IN snapshot.xip` | Inserting tx was in-progress at snapshot time — not committed yet from this snapshot's perspective |
| `xmin > snapshot.xmax` | Inserting tx started after snapshot — future insert |
| `xmin == current_xid` AND `cmin > snapshot.curcid` | Self-insert, but command hasn't run yet from cursor's perspective |

The insert is considered **valid** in these cases:

| Condition | Reason |
|---|---|
| `xmin < snapshot.xmin` AND xmin committed | Completed before snapshot, committed |
| `xmin NOT IN snapshot.xip` AND xmin committed | Completed before snapshot, committed |
| `xmin == current_xid` AND `cmin < snapshot.curcid` | Self-insert, already executed |

### Delete Validity

The delete is considered **invalid** (deletion not visible → tuple still live) in these cases:

| Condition | Reason |
|---|---|
| `xmax < snapshot.xmin` AND xmax aborted | Deleting tx rolled back |
| `xmax IN snapshot.xip` | Deleting tx was in-progress at snapshot time |
| `xmax > snapshot.xmax` | Deleting tx started after snapshot |
| `xmax == current_xid` AND `cmax > snapshot.curcid` | Self-delete, command not yet executed |

The delete is considered **valid** (tuple is deleted → not visible) in these cases:

| Condition | Reason |
|---|---|
| `xmax < snapshot.xmin` AND xmax committed | Completed before snapshot, committed |
| `xmax NOT IN snapshot.xip` AND xmax committed | Completed before snapshot, committed |
| `xmax == current_xid` AND `cmax < snapshot.curcid` | Self-delete, already executed |

---

## Putting It All Together

Here's a concrete example to make this concrete. Suppose Tx104 takes a snapshot:

```plaintext
Snapshot (taken by Tx104):
  xmin   = 101
  xmax   = 105
  xip    = [101, 103, 104, 105]
  curcid = 5
```

And we have a tuple with `xmin=102, cmin=3`:

1. Is `xmin (102)` in `xip`? No → check CLOG → Tx102 committed → **insert is valid**
2. Is `xmax` set? Suppose `xmax=0` (not deleted) → **delete is invalid**
3. Result: **tuple is visible** ✓

Now suppose another tuple has `xmin=103, cmin=2`:

1. Is `xmin (103)` in `xip`? Yes → **insert is invalid** (Tx103 was in-progress at snapshot time)
2. Result: **tuple is not visible** ✗

---

## Summary

PostgreSQL's visibility system operates at two levels:

1. **Page level**: `PD_ALL_VISIBLE` in `PageHeaderData` lets PostgreSQL skip entire pages worth of per-tuple work. Set by VACUUM, cleared on any modification.
2. **Tuple level**: `HeapTupleSatisfiesMVCC` evaluates `xmin`/`xmax` against a snapshot, using CLOG lookups (cached via hint bits) and the snapshot's `xmin`, `xmax`, `xip`, and `curcid`.

Understanding this mechanism is essential for reasoning about VACUUM behavior, read performance, and the semantics of different isolation levels.

---

## References

- [PostgreSQL Docs: Database Page Layout](https://www.postgresql.org/docs/devel/storage-page-layout.html)
- [MVCC Unmasked (EDB)](https://edbjapan.com/webinar/MVCC_Unmasked_211110.pdf)
- [A Deeper Look Inside PostgreSQL Visibility Check Mechanism (Highgo)](https://www.highgo.ca/2024/04/19/a-deeper-look-inside-postgresql-visibility-check-mechanism/)
- `src/include/storage/bufpage.h`
- `src/include/access/htup_details.h`
- `src/include/utils/snapshot.h`
- `src/include/access/clog.h`