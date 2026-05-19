---
title: "5 PostgreSQL locking behaviors that trip people up"
published: false
tags: postgres, database, lock
---

## Introduction

PostgreSQL uses MVCC (Multi-Version Concurrency Control) for concurrency control: reads never block writes, and writes never block reads.

Its locking system has 8 table-level lock modes and 4 row-level lock modes, and the conflict tables in the documentation tell you exactly which lock modes conflict with which.

In practice, though, once you actually operate PostgreSQL, locks end up conflicting in places you never expected. Queries take far longer than anticipated, and in the worst case you end up with an outage.

This article walks through five of these counterintuitive locking behaviors.

## Environment
- Version: PostgreSQL 18
- Transaction isolation level: READ COMMITTED (the default)

## 1. Once an `ACCESS EXCLUSIVE` request is queued, subsequent queries get blocked in a chain

The first one: an `ALTER TABLE` that should finish instantly can bring your entire service to a halt.

Suppose one session is running a long `SELECT` on table `t`, and another session runs the following `ALTER TABLE`:

**Session 1**

```sql
SELECT pg_sleep(600) FROM t LIMIT 1; -- a long-running SELECT
```

**Session 2**

```sql
ALTER TABLE t ADD COLUMN name text;
```

Since Session 1 holds an `ACCESS SHARE` lock on table `t`, the `ACCESS EXCLUSIVE` lock that Session 2's `ALTER TABLE` requires is forced to wait. So far, this is expected behavior.

But PostgreSQL's lock waiting works like a FIFO queue. While the `ACCESS EXCLUSIVE` lock is waiting, any `SELECT` issued against table `t` afterward gets stuck behind it — even though that `SELECT` does not conflict with Session 1's currently running `SELECT` at all.

![Diagram showing subsequent SELECTs also forced to wait in the lock queue](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wmbfw7f1502kcb2ih7p8.png)

> Besides a long-running `SELECT`, the same thing happens with a session that ran a `SELECT` inside a `BEGIN` transaction and then left it open without `COMMIT`/`ROLLBACK` (a so-called *idle in transaction* session). An `ACCESS SHARE` lock is held until the transaction ends, so even if the `SELECT` itself finished in an instant, simply forgetting to close the transaction will keep blocking `ALTER TABLE`. This is often caused by a missing commit in the application, or by a connection left idle after fetching its results — so watch out for it.

The result is the following chain:

1. A long-running `SELECT` is executing.
2. An `ALTER TABLE` (or any statement that acquires an `ACCESS EXCLUSIVE` lock) is forced to wait due to a lock conflict.
3. Every subsequent `SELECT` lines up behind the lock wait in step 2.

This chained blocking makes your `SELECT`s wait far longer than expected, which can lead to an outage.

### Mitigations

- In transactions that run statements acquiring an `ACCESS EXCLUSIVE` lock, set `lock_timeout` so the lock wait can't drag on indefinitely.
- Check in advance for long-running queries via the `pg_stat_activity` view.

## 2. The "invisible" deadlock caused by foreign key constraints

The second one comes from the locks that a foreign key constraint acquires implicitly.

When you run an `INSERT`, PostgreSQL takes a `FOR KEY SHARE` lock on the referenced row in every parent table that the foreign keys point to. This mechanism prevents the situation where deleting or updating the referenced row in the parent table would break the referencing side's integrity.

For example, when you `INSERT` into table `t`, a `FOR KEY SHARE` lock is automatically taken on the referenced row in the parent table `s`. Because `FOR KEY SHARE` and `FOR UPDATE` conflict, a deadlock arises if two sessions each "lock a different row in `s` with `FOR UPDATE`, then run an `INSERT` whose foreign key references the other session's row" in opposite orders — a circular wait forms.

**Session 1**

```sql
BEGIN;
SELECT * FROM s WHERE id=1 FOR UPDATE; -- lock row s.id=1 with FOR UPDATE
```

**Session 2**

```sql
BEGIN;
SELECT * FROM s WHERE id=2 FOR UPDATE; -- lock row s.id=2 with FOR UPDATE
```

**Session 1**

```sql
INSERT INTO t (s_id) VALUES (2); -- wants FOR KEY SHARE on row s.id=2 (waits for Session 2)
```

**Session 2**

```sql
INSERT INTO t (s_id) VALUES (1); -- wants FOR KEY SHARE on row s.id=1 (deadlock detected)
```

What's counterintuitive here is:

- Application code that contains nothing but `INSERT` statements is implicitly locking rows in the foreign key's parent table.
- Looking at the application's SQL, the lock acquisition never surfaces.

### Mitigations

- Avoid a design where multiple sessions lock a row in a foreign key's parent table with `FOR UPDATE` and then `INSERT` a different row referencing it, in opposite orders — that causes deadlocks.
- If you really need it, make the lock acquisition order consistent across the entire application (for example, always lock rows in `s` in ascending `id` order — enforce a single, one-directional locking order).
- Deadlocks are always reported as errors, so implement with retries in mind.

## 3. A deadlock between two `INSERT`s caused by the unique constraint duplicate check

The third one: two transactions that only do `INSERT`s can deadlock simply by inserting each other's already-inserted values in opposite orders.

Run SQL like the following:

**Session 1**

```sql
BEGIN;
INSERT INTO t (id) VALUES (1);
```

**Session 2**

```sql
BEGIN;
INSERT INTO t (id) VALUES (2);
```

**Session 1**

```sql
INSERT INTO t (id) VALUES (2); -- waits for Session 2 to commit
```

**Session 2**

```sql
INSERT INTO t (id) VALUES (1); -- deadlock detected
```

The reason this happens is that the duplicate check for `PRIMARY KEY` and `UNIQUE` constraints **waits for the other transaction to finish if that transaction is currently inserting the same value**. This is how PostgreSQL settles the outcome: "if the other transaction commits, it's a unique constraint violation; if it rolls back, the insert succeeds."

So in the example above:

- Session 1 tries to insert `id=2` → Session 2 is inserting `id=2`, so it waits.
- Session 2 tries to insert `id=1` → Session 1 is inserting `id=1`, so it waits.
- They wait on each other's commit → deadlock detected.

### Mitigations

- Avoid a design where the same value can be inserted concurrently from multiple sessions (using a sequence such as `SERIAL`/`IDENTITY` avoids duplicates entirely).
- As mentioned in the previous section's mitigations, deadlocks are reported as errors, so implement with retries in mind.

## 4. Only the transaction-ID-wraparound-prevention autovacuum refuses to yield on conflict

The fourth one is about an exceptional behavior of autovacuum.

When the `SHARE UPDATE EXCLUSIVE` lock that autovacuum acquires conflicts with another statement, autovacuum is normally the one that gets canceled, so it doesn't block other statements for long. That's why the common understanding is "it's fine to just leave autovacuum running."

There is one exception, however: the autovacuum running to prevent transaction ID wraparound (the one whose `query` column in `pg_stat_activity` ends with `(to prevent wraparound)`) **is not canceled automatically, even when there is a conflict**.

Here's what actually happens:

1. A table's `relfrozenxid` age exceeds `autovacuum_freeze_max_age`.
2. The wraparound-prevention autovacuum starts (holding `SHARE UPDATE EXCLUSIVE`).
3. Another process issues an `ALTER TABLE` to create a partition.
4. The `ALTER TABLE` waits for autovacuum ← **it would normally be canceled, but it isn't**.
5. Every `SELECT` lines up in the queue ← the chained blocking described in #1 occurs.

When this combines with the first behavior, it can turn fatal in a hurry — that's the key point.

### Mitigations

- If you see `autovacuum: VACUUM ... (to prevent wraparound)` in `pg_stat_activity`, recognize that an autovacuum that won't be canceled is running, and wait for it to complete.
- Prevention matters more than reacting after the fact. Periodically monitor `age(relfrozenxid)` in `pg_class` to know which tables are approaching `autovacuum_freeze_max_age` (200 million transactions by default).

  ```sql
  SELECT relname, age(relfrozenxid)
  FROM pg_class
  WHERE relkind = 'r'
  ORDER BY age(relfrozenxid) DESC;
  ```

- If you have DDL such as `ALTER TABLE` planned, check the target table's `age(relfrozenxid)` beforehand, and if it's close to the threshold, run a manual `VACUUM FREEZE` before the DDL.

## 5. VACUUM's hidden `ACCESS EXCLUSIVE` phase

The last one is also about an exceptional behavior of VACUUM.

Normally VACUUM operates with `SHARE UPDATE EXCLUSIVE`, but **the final truncate phase** — truncating empty pages at the end of the table and returning the disk space to the OS — is the one exception: it acquires an `ACCESS EXCLUSIVE` lock.

As in the earlier examples, if a long-running `SELECT` is executing while VACUUM waits to acquire that `ACCESS EXCLUSIVE` lock, subsequent queries all get blocked in the same chained-blocking pattern described in #1.

And on a streaming replication standby, the situation is different. When `ACCESS EXCLUSIVE` is acquired and released on the primary, that operation reaches the standby as WAL. The standby's WAL replay process tries to apply that operation, but a long-running `SELECT` executing on the standby conflicts with it, so **WAL replay stalls**. While WAL replay is stalled, the apply lag for already-received WAL keeps growing, and once that lag exceeds `max_standby_streaming_delay`, the conflicting `SELECT` is **forcibly canceled** and replay resumes.

On the primary, a `SELECT` is allowed to wait until it finishes naturally; on a standby, the `SELECT` is forcibly canceled by an external factor (WAL replay). That's the big difference.

### Mitigations

- On PostgreSQL 12 and later, you can disable VACUUM's truncate phase per table with `vacuum_truncate = false`.
- PostgreSQL 18 and later also adds a server-wide `vacuum_truncate` parameter.
- If you run long-running queries on a standby, set `max_standby_streaming_delay` (30 seconds by default) higher to extend how long WAL replay is allowed to wait (`-1` for unlimited).

## Conclusion

Every one of these cases is behavior that emerges because PostgreSQL's lock design faithfully implements the mechanisms needed to keep things correct. Natural as a specification, but a pitfall from an operational point of view.

In particular, the chained blocking in #1 can rapidly escalate into a fatal incident when combined with the other cases, so setting `lock_timeout` and continuously monitoring `pg_stat_activity` are the minimum defensive lines you'll want in place.

## References

- https://www.postgresql.org/docs/current/explicit-locking.html
- https://www.postgresql.org/message-id/d7df81620708101038k772a2cderb52bb09f5440bd1b@mail.gmail.com
- https://leosjoberg.com/blog/lock-propagation-postgres/
- https://xata.io/blog/migrations-and-exclusive-locks
