---
title: "A guide to Database Isolation level"
layout: single
date: 2026-07-18
permalink: /posts/mysql-isolation/
author_profile: false
tags:
  - Computer Science
  - Database
---

This post is a quick guide to MySQL Isolation level, but the same concept can still be applied to other relational databases. Think of this post as a **quick reference** to refresh your memory.

{% include toc %}

I wrote this post based on my own conceptual understanding of the topic, if you notice any mistakes or have any suggestions for improvement, feel free to leave a comment or contact me directly. I will be happy to update the post accordingly.

## 0. Introduction

The Isolation level (I in ACID) is the setting that fine-tunes the **balance/trade-off between performance and reliability, consistency** when multiple transactions are performing queries simultaneously.

From highest amount of consistency and protection to the least, the isolation levels supported by MySQL InnoDB are: `SERIALIZABLE`, `REPEATABLE READ` (default), `READ COMMITTED`, and `READ UNCOMMITTED`.

## 1. READ UNCOMMITTED

In this mode, a transaction is allowed to READ UNCOMMITTED data from other transactions. This is the lowest isolation level, and it can lead to **DIRTY READS** (visible uncommitted changes) and other anomalies that we will cover in the next sections.

![alt text]({{ site.baseurl }}/images/blogs/mysqlisolation/read-uncommitted.png)

## 2. READ COMMITTED

A transaction can only READ COMMITTED data from other transactions. This **prevents DIRTY READS**, but it can still lead to:

- **NON-REPEATABLE READS** (data can change between reads in the same transaction).
- **PHANTOM READS** (#rows changed between 2 reads in the same transaction).

This example illustrates the **NON-REPEATABLE READ** phenomenon, but similar logic can be applied to phantom reads:

![alt text]({{ site.baseurl }}/images/blogs/mysqlisolation/read-committed.png)

### How MySQL implements READ COMMITTED?

MySQL takes a **"snapshot"** (latest committed data) of the database's current state **for each query** in a transaction.

When a transaction reads data, it **accesses the snapshot instead of the source table**. A snapshot is refreshed and synced with committed data for subsequent queries within the same transaction. For example Tx1 COMMIT could refresh the snapshot for Tx2, and Tx2 will see the changes made by Tx1, just like the diagram above.

## 3. REPEATABLE READ

TLDR, `REPEATABLE READ`:

- Prevents **NON-REPEATABLE READs** by reusing the snapshot created by the transaction’s **first plain SELECT**. Rows committed by other transactions afterward remain invisible to this transaction.
- Prevents **phantom reads** by using **gap locks** and **next-key locks** for range scans.
- But there are edge cases

Compare to `READ COMMITTED`, there are 2 main differences at this level: (1) when does snapshot refresh? (2) locking strategy.

### (1) Snapshot refresh

In this mode, a snapshot is taken **at the transaction's first read**.

Transaction only sees 1 snapshot at its first read, and that snapshot is **used for all subsequent reads** within that transaction. ==> This ensures consistent subsequent READs throughout the transaction's execution and avoids **NON-REPEATABLE READS**.

### (2) Locking strategy

Under `READ COMMITTED`, InnoDB uses **record locks** for locking: it locks index records but generally does not lock the surrounding gaps. Because gaps remain unlocked, another transaction can insert a new row into the searched range, **allowing phantom reads**.

Meanwhile, `REPEATABLE READ` adds **gap locks** and **next-key locks** for range scans. I covered these in detail here [MySQL locks post](/posts/mysql-locks/).

Basically, when a transaction executes a locking range operation (e.g., `SELECT ... FOR UPDATE`, `UPDATE`, or `DELETE`), InnoDB locks the scanned index range using next-key or gap locks. This prevents other transactions from inserting new rows and therefore **prevents phantom reads**.

```sql
-- Tx1
BEGIN;
SELECT * FROM users WHERE age BETWEEN 20 AND 30 FOR UPDATE;

-- Tx2: waits if age = 25 falls inside Tx1's locked index range
INSERT INTO users(age) VALUES (25);
```

### Edge Case: Mixing Snapshot Reads and Current Reads

Under `REPEATABLE READ`, plain `SELECT` statements use a fixed MVCC snapshot, while write operation like `UPDATE` and `DELETE` operate on the latest committed version of the data directly.

``` sql
-- Tx1
BEGIN;
SELECT * FROM users WHERE age = 25; -- Returns 0 rows; snapshot is established.

-- Tx2
INSERT INTO users(age) VALUES (25);
COMMIT;

-- Tx1
UPDATE users SET status = 'active' WHERE age = 25; -- Affects 1 row.
SELECT * FROM users WHERE age = 25; -- Returns 1 row; Tx1 can now see its own modification.
```

Although Tx1’s snapshot cannot see the row inserted by Tx2, the `UPDATE` uses the latest committed data (bypassing its snapshot), so it can find and modify the newly committed row. Note that the `UPDATE` statement **does not refresh the snapshot**, but Tx1 can see its own modification afterward.

### Lost Update problem

This problem occurs when applications use a **read-modify-write pattern** instead of performing the modification atomically in the database.

For example, 2 transactions can **read** the same data, **update** it in its own memory, and **write** back to the database. The 2nd transaction wins and **overwrites** the first transaction's update, resulting in a **LOST UPDATE**.

![alt text]({{ site.baseurl }}/images/blogs/mysqlisolation/lost-update.png)

So how to prevent this? There are 2 approaches:

- **Pessimistic locking**: use `SELECT ... FOR UPDATE` to lock the row before reading it. This prevents other transactions from modifying the row until the transaction is committed or rolled back.
- **Optimistic locking**: use a `version` column to detect if the row has been modified by another transaction before updating it. If the version number has changed, the transaction can abort or retry.
  - In implementation, the `version` should be compared between **client's data vs the database's data** (etags or if-match HTTP pattern).

## 4. Serializable

This level is like `REPEATABLE READ`, but InnoDB implicitly converts all plain `SELECT` statements to `SELECT ... FOR SHARE` (shared locks). The exact locks depend on the query:

- UNIQUE index lookup: acquires a **shared record lock** on the found row.
- Range or NON-UNIQUE index scan: acquires **shared next-key locks** over the scanned index range.

Shared locks **allow concurrent reads**, but **conflicting writes must wait**. Next-key locks also prevent new rows from being inserted into the protected gaps.

As a result, `SERIALIZABLE` prevents dirty reads, non-REPEATABLE READs, phantoms, and typical lost-update patterns.

**The trade-off** is lower concurrency and **potential deadlocks**. Therefore in practice, applications often remain on `REPEATABLE READ` and use targeted `SELECT ... FOR UPDATE` queries where serialization is required to reduce locking scope.

![alt text]({{ site.baseurl }}/images/blogs/mysqlisolation/deadlock.png)

## Summary

| Isolation Level | Dirty Read | Non-REPEATABLE READ | Phantom Read | How |
| :--- | :---: | :---: | :---: | :--- |
| `READ UNCOMMITTED` | ❌ | ❌ | ❌ | No snapshot, reads live data directly |
| `READ COMMITTED` | ✅ | ❌ | ❌ | Snapshot refreshed **per query** |
| `REPEATABLE READ` | ✅ | ✅ | ✅ | Snapshot fixed at **first read** + gap/next-key locks |
| `SERIALIZABLE` | ✅ | ✅ | ✅ | Plain `SELECT` implicitly becomes `SELECT ... FOR SHARE` |

**Key takeaways:**

- Isolation level is fundamentally a **trade-off** between consistency and performance ⇒ higher isolation means more locking, less throughput.

- `READ UNCOMMITTED`
  - best performance, lowest isolation level. It can lead to **DIRTY READS** and other anomalies.
  
- `READ COMMITTED`
  - Use snapshot mechanism: prevents **DIRTY READS**, but it can still lead to **NON-REPEATABLE READS** and **PHANTOM READS**.
  
- `REPEATABLE READ` (InnoDB default)
  - Blocks the most common anomalies (dirty read, non-REPEATABLE READ, phantom read on locking scans) while still being reasonably concurrent.
  - Beware for Mixing Snapshot Reads and Current Reads: plain `SELECT` reads the snapshot, `UPDATE`/`DELETE`/`SELECT ... FOR UPDATE` read the latest committed data.
  - Lost updates problem: Use pessimistic or optimistic locking to prevent this.

- `SERIALIZABLE`
  - Gives strongest data consistency, but trades off performance, hence reach for targeted `SELECT ... FOR UPDATE` on `REPEATABLE READ` instead.

## References

- [MySQL ACID](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- [A straightforward guide for isolation levels](https://dev.to/eyochen/a-straightforward-guide-for-isolation-levels-3h66)
- [MySQL locks: A quick guide](/posts/mysql-locks/)
