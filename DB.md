# Database Concurrency, Locking & High-Scale Design: Interview Prep Guide

This guide compiles high-frequency database-focused system design and coding interview questions. It covers **Transactions (ACID)**, **Isolation Levels**, **Locking Mechanisms**, **Race Conditions**, **Deadlocks**, **PostgreSQL Internals (MVCC)**, and **High-Scale System Designs**.

---

## 📌 Table of Contents
1. [Transactions & ACID](#1-transactions--acid)
2. [Transaction Isolation Levels](#2-transaction-isolation-levels)
3. [Locking Mechanisms](#3-locking-mechanisms)
4. [Database Race Conditions](#4-database-race-conditions)
5. [Deadlocks](#5-deadlocks)
6. [PostgreSQL-Specific Concurrency Internals](#6-postgresql-specific-concurrency-internals)
7. [High-Scale System Design Scenarios](#7-high-scale-system-design-scenarios)

---

## 1. Transactions & ACID

### Q1: What is a database transaction?
*   **The Core Concept:** A transaction is a single logical unit of database work containing one or more SQL statements (e.g., `INSERT`, `UPDATE`, `DELETE`).
*   **Detailed Explanation:** The database guarantees that all statements within the transaction block (enclosed in `BEGIN` and `COMMIT`) execute successfully as a single unit, or none of them execute (aborted and rolled back via `ROLLBACK`).

---

### Q2: What problems do transactions solve?
*   **The Core Concept:** Transactions protect data integrity against two primary challenges: **System Failures** and **Concurrent Access**.
*   **Specific Problems Solved:**
    1.  **Partial Updates (Atomicity Failures):** For example, in a bank transfer, money is debited from Account A but a server crash occurs before it is credited to Account B. Transactions roll back the debit, preventing money from vanishing.
    2.  **Concurrency Anomalies:** Multiple users modifying the same data simultaneously, which would otherwise lead to corrupted balances, duplicate reservations, or dirty reads.

---

### Q3: What are ACID properties?
*   **The Core Concept:** ACID is a set of database engine design properties that guarantee data validity despite errors, power failures, or concurrent operations.
*   **The Breakdown:**
    *   **Atomicity ("All or Nothing"):** If any part of the transaction fails, the entire transaction is rolled back, leaving the database unchanged.
    *   **Consistency:** A transaction can only transition the database from one valid state to another, enforcing all database schemas, constraints (e.g., foreign keys, unique indexes), and triggers.
    *   **Isolation:** The execution of concurrent transactions is isolated from one another. The intermediate state of a running transaction is invisible to other transactions.
    *   **Durability:** Once a transaction commits, its changes are written to non-volatile storage (usually via a Write-Ahead Log / WAL) and will survive subsequent power losses or crashes.

---

### Q4: Why are transactions important in concurrent systems?
*   **The Core Concept:** In systems with high write concurrency, multiple application instances query and update the same data records. Without transaction guarantees, these operations interleave arbitrarily, resulting in data corruption, race conditions, and race states. Transactions provide boundaries that allow the database engine to safely serialize modifications.

---

## 2. Transaction Isolation Levels

### Q1: What are transaction isolation levels?
*   **The Core Concept:** Isolation levels are database configurations that define the degree to which a transaction's operations are isolated from concurrent transactions.
*   **The Trade-off:** High isolation prevents concurrency anomalies but reduces performance by holding locks longer or forcing serialization. Low isolation improves throughput but exposes the system to anomalies.

---

### Q2: What is a Dirty Read?
*   **Definition:** Transaction A reads data that has been modified by Transaction B but is **not yet committed**. If Transaction B rolls back, Transaction A's read data is invalid ("dirty").
*   **Example:**
    1. Transaction 1 updates account balance from $100 to $150.
    2. Transaction 2 reads balance: $150.
    3. Transaction 1 aborts and rolls back. Balance reverts to $100. Transaction 2 continues with incorrect $150 data.

---

### Q3: What is a Non-Repeatable Read?
*   **Definition:** Transaction A reads a row once. Transaction B updates or deletes that row and **commits**. Transaction A reads the row again and finds the data has changed.
*   **Example:**
    1. Transaction 1 reads balance: $100.
    2. Transaction 2 updates balance to $150 and commits.
    3. Transaction 1 re-reads balance: $150. (The read was not repeatable).

---

### Q4: What is a Phantom Read?
*   **Definition:** Transaction A queries a range of rows matching a condition. Transaction B **inserts** new rows matching that condition and **commits**. Transaction A runs the range query again and sees new "phantom" rows.
*   **Example:**
    1. Transaction 1 queries: `SELECT COUNT(*) FROM orders WHERE status = 'pending';` (returns 5).
    2. Transaction 2 inserts a new pending order and commits.
    3. Transaction 1 queries again: returns 6.

---

### Q5: Explain Read Committed, Repeatable Read, and Serializable.

| Isolation Level | Dirty Reads | Non-Repeatable Reads | Phantom Reads | Mechanism (Standard RDBMS) |
| :--- | :--- | :--- | :--- | :--- |
| **Read Uncommitted** | Allowed | Allowed | Allowed | No locks held for reads. |
| **Read Committed** | **Prevented** | Allowed | Allowed | Read locks released immediately. Only reads committed data at statement-start. |
| **Repeatable Read** | **Prevented** | **Prevented** | Allowed / Prevented* | Read locks held until transaction ends. *PostgreSQL prevents Phantoms here using MVCC snapshots.* |
| **Serializable** | **Prevented** | **Prevented** | **Prevented** | Full logical serialization (locks ranges/uses SSI). |

*   **Read Committed:** The default in Postgres/SQL Server. A statement only sees data committed *before the statement began*.
*   **Repeatable Read:** A statement inside a transaction only sees data committed *before the transaction began*.
*   **Serializable:** The strictest level. It acts as if transactions run sequentially. If a concurrent conflict is detected during execution, the database aborts the transaction with a serialization failure, requiring the application to retry.

---

### Q6: Which isolation level would you choose for a payment system and why?
*   **The Recommended Choice:** **Repeatable Read** (with explicit pessimistic row locks like `SELECT ... FOR UPDATE`) or **Serializable**.
*   **The Rationale:**
    *   Payment systems require zero tolerance for write skews, double spending, or incorrect balance readings.
    *   If using **Serializable**, the database engine handles safety checks automatically. However, your application code *must* have built-in retry mechanisms to catch and rerun aborted transactions caused by serialization failures.
    *   If using **Read Committed** or **Repeatable Read**, you must use explicit pessimistic locking (`FOR UPDATE`) to lock account balances during reads, forcing concurrent transactions to block and serialize.

---

## 3. Locking Mechanisms

### Q1: What is pessimistic locking?
*   **The Core Concept:** An eager locking strategy where you prevent conflicts by acquiring an exclusive lock on a resource (row or table) *before* modifying it, blocking any concurrent readers or writers.
*   **Database Syntax:**
    `SELECT * FROM accounts WHERE id = 1 FOR UPDATE;`

---

### Q2: What is optimistic locking?
*   **The Core Concept:** A lazy concurrency control strategy that does not acquire locks. Instead, it allows concurrent reads and writes, but verifies data integrity (using a version number or timestamp) before committing. If the version has changed, the write fails.
*   **Database Flow:**
    1. Read row: `SELECT balance, version FROM accounts WHERE id = 1;` (returns `version = 5`).
    2. Modify in application memory.
    3. Write back: `UPDATE accounts SET balance = 150, version = 6 WHERE id = 1 AND version = 5;`.
    4. If another thread updated the row, the row version would be `6`. The `UPDATE` query updates 0 rows. The application detects this and retries.

---

### Q3: When would you choose optimistic locking over pessimistic locking?
*   **Choose Optimistic Locking (OCC) when:**
    *   **Low Contention:** Write conflicts are rare (e.g., users updating their own profiles).
    *   **High Read Throughput:** You cannot afford the performance bottleneck of holding database locks on reads.
    *   **No DB Connection Blocking:** Prevents holding database connections open during long client-side think-times.
*   **Choose Pessimistic Locking when:**
    *   **High Contention:** Conflicts are frequent (e.g., booking a hot concert ticket). OCC would cause constant retries, wasting CPU.
    *   **Strict Safety:** Retries are unacceptable, and you want requests to queue up and execute sequentially.

---

### Q4: How does `SELECT ... FOR UPDATE` work?
*   **The Mechanism:** It acts as an exclusive write lock on the queried rows. 
    *   Any other transaction executing a `SELECT ... FOR UPDATE`, `UPDATE`, or `DELETE` on those rows will block until the locking transaction finishes (`COMMIT` or `ROLLBACK`).
    *   *Note:* Standard non-locking `SELECT` reads are still allowed to query the rows under MVCC (they read the snapshot), unless the isolation level is set to Serializable.

---

### Q5: Row-Level Locks vs. Table-Level Locks
*   **Row-Level Locks:** Lock specific records (e.g., `FOR UPDATE`). Highly concurrent; other users can edit different rows in the same table.
*   **Table-Level Locks:** Lock the entire table (e.g., `LOCK TABLE users IN EXCLUSIVE MODE`). Used during schema alterations (`ALTER TABLE`) or bulk migrations. Destroys concurrency because all writes to the table are blocked.

---

### Q6: What is lock contention?
*   **Definition:** Lock contention occurs when multiple concurrent transactions wait for the same database lock to be released.
*   **Impact:** Causes system latency to spike, connection pools to exhaust, and throughput to drop, as database worker threads sit idle waiting in lock queues.

---

## 4. Database Race Conditions

### Q1: Two users withdraw money from the same account simultaneously. How do you prevent inconsistent balances?
*   **The Scenario:** Account has $100. User A tries to withdraw $80, User B tries to withdraw $50. Both check if `balance >= amount` simultaneously. Both succeed, resulting in a negative balance ($100 - $80 - $50 = -$30).
*   **Prevention Strategies:**
    1.  **Pessimistic Locking (Serializer):**
        ```sql
        BEGIN;
        SELECT balance FROM accounts WHERE id = 1 FOR UPDATE; -- locks row
        -- application checks balance --
        UPDATE accounts SET balance = balance - 80 WHERE id = 1;
        COMMIT;
        ```
    2.  **Single Atomic Check Constraint:**
        ```sql
        UPDATE accounts SET balance = balance - 80 WHERE id = 1 AND balance >= 80;
        -- check rows affected: if 0, throw error.
        ```

---

### Q2: Multiple users update the same record. How do you prevent lost updates?
*   **The Prevention:** Use Optimistic Concurrency Control (OCC) versioning in the application layer, or execute atomic SQL updates (`UPDATE table SET counter = counter + 1`) which do not require a separate read step.

---

### Q3: How would you ensure inventory never goes below zero?
*   **Database Constraint (The Ultimate Guard):** Add an unsigned integer or a database-level `CHECK` constraint:
    `ALTER TABLE inventory ADD CONSTRAINT check_stock_non_negative CHECK (stock >= 0);`
*   **Atomic Query Execution:**
    `UPDATE inventory SET stock = stock - qty WHERE product_id = 123 AND stock >= qty;`
    If the stock is insufficient, the database constraint raises a SQL error, triggering a safe rollback.

---

### Q4: How would you prevent double booking of a seat?
*   **Double-Guard Strategy:**
    1.  **Database Unique Index Constraint (Primary Guard):**
        Create a composite unique index on the table:
        `CREATE UNIQUE INDEX unique_seat_booking ON bookings (flight_id, seat_number) WHERE status = 'CONFIRMED';`
        If two queries try to insert the same seat, the database throws a duplicate key violation, rolling back the second transaction.
    2.  **Pessimistic State Locking:** Lock the seat row status during validation:
        ```sql
        SELECT status FROM seats WHERE flight_id = 123 AND seat_number = '12A' FOR UPDATE;
        ```

---

## 5. Deadlocks

### Q1: What is a deadlock?
*   **Definition:** A deadlock is an execution state where two or more transactions are permanently blocked because each holds a lock that the other needs to proceed.
*   **Visualizing a Deadlock:**
    ```
    Transaction A                               Transaction B
    ---------------------------------------------------------
    1. Lock Row 1 (Acquired)                    1. Lock Row 2 (Acquired)
    2. Try to Lock Row 2 (Blocks...)            2. Try to Lock Row 1 (Blocks...)
                     \                         /
                      +--- [ Permanent Wait ] +
    ```

---

### Q2: How do deadlocks occur?
*   **The Cause:** They occur when transactions lock resources in different sequences. If Transaction A updates User 1 then User 2, while Transaction B updates User 2 then User 1, they will deadlock if their operations interleave.

---

### Q3: How can deadlocks be prevented?
*   **Best Practices for Prevention:**
    1.  **Enforce Strict Lock Ordering:** Always lock resources in a consistent, sorted order (e.g., sort user IDs numerically in the application before issuing SQL queries).
    2.  **Keep Transactions Short:** Avoid putting third-party API calls or long computations inside a database transaction. Commit as fast as possible.
    3.  **Acquire Locks Eagerly:** Lock everything you need at the beginning of the transaction rather than acquiring locks incrementally.
    4.  **Use Optimistic Concurrency Control (OCC):** Eliminates write locks during execution, removing the possibility of lock-based deadlocks.

---

### Q4: How does PostgreSQL handle deadlocks?
*   **Deadlock Detection:** PostgreSQL runs a background deadlock detector. When a transaction blocks waiting on a lock, Postgres starts a timer. If the wait exceeds `deadlock_timeout` (default: 1 second), it analyzes the lock dependency graph for cycles.
*   **Resolution:** If a cycle is detected, Postgres selects one of the transactions (usually the blocker that triggered the check) and aborts it with a `40P01` error (`deadlock_detected`), allowing the other transaction to proceed.

---

### Q5: How would you detect deadlocks in production?
*   **Detection Plan:**
    1.  **Application Logs:** Monitor backend logs for PostgreSQL error code `40P01`.
    2.  **Database System Views:** Periodically check `pg_stat_activity` to find blocked transactions:
        `SELECT query, wait_event, wait_event_type FROM pg_stat_activity WHERE pid != pg_backend_pid() AND state = 'active';`
    3.  **APM Metrics:** Set alerts on database metrics tracking transaction lock wait times.

---

## 6. PostgreSQL-Specific Concurrency Internals

### Q1: What is MVCC (Multi-Version Concurrency Control)?
*   **The Core Concept:** MVCC is a concurrency control method where PostgreSQL maintains multiple physical versions of the same data row instead of locking records for reads.
*   **How it Works:**
    *   Every row tuple in Postgres has hidden system columns: `xmin` (the Transaction ID that created the row) and `xmax` (the Transaction ID that deleted/replaced the row).
    *   When you update a row, Postgres does not overwrite the data in place. It marks the old row tuple as deleted (`xmax = current_tx_id`) and inserts a completely new row tuple (`xmin = current_tx_id`).

---

### Q2: Why can PostgreSQL serve reads without blocking writes?
*   **The Answer:** Because of MVCC. Since updates create new versions of rows rather than overwriting them in place, read queries simply scan the table and read the appropriate historical version of the row based on the transaction's snapshot.
*   **Interview Slogan:** **"Readers never block writers, and writers never block readers."**

---

### Q3: What happens when two transactions update the same row?
*   **The Behavior:**
    1.  The first transaction acquires an exclusive lock on the row.
    2.  The second transaction attempts to update the row and blocks, waiting for the first transaction to complete (`COMMIT` or `ROLLBACK`).
    3.  **If Tx 1 Commits:**
        *   *Under Read Committed:* Tx 2 wakes up, re-evaluates the query condition against the *newly updated* row version, and executes its update.
        *   *Under Repeatable Read/Serializable:* Tx 2 aborts immediately with a serialization failure error: `could not serialize access due to concurrent update`.
    4.  **If Tx 1 Aborts:** Tx 2 wakes up and applies its update to the original row version.

---

### Q4: Explain row locking in PostgreSQL.
*   **The Mechanism:** Postgres does not store row locks in shared memory. (This is a major architectural difference from engines like MS SQL Server, which can run out of memory and escalate row locks to table locks).
*   **How Postgres Stores Locks:** Postgres writes lock ownership bits directly onto the row header (`infomask` bits in the tuple) in the physical data page.
*   **Lock Types:**
    *   `FOR UPDATE`: Exclusive write lock (blocks updates, deletes, and other locks).
    *   `FOR NO KEY UPDATE`: Weaker update lock (does not block foreign key checks).
    *   `FOR SHARE`: Shared read lock (blocks modifications but allows other reads).
    *   `FOR KEY SHARE`: Weaker read lock.

---

### Q5: Explain advisory locks in PostgreSQL.
*   **The Core Concept:** Advisory locks are application-defined locks that have no physical table or row relations. They exist purely in the database's memory space, and their semantics are defined entirely by the developer.
*   **Use Cases:** Perfect for implementing distributed locking (e.g., ensuring a background report only runs on one Node.js server).
*   **Syntax:**
    *   `SELECT pg_advisory_lock(42);` -- Block until lock 42 is free (session level).
    *   `SELECT pg_try_advisory_lock(42);` -- Returns `true` if acquired, `false` immediately if locked.
    *   `SELECT pg_advisory_xact_lock(42);` -- Transaction-level lock (automatically released on commit).

---

## 7. High-Scale System Design Scenarios

### Q1: How would you design a wallet service?
*   **Key Requirements:** 100% financial accuracy, no balance corruption, high transaction throughput.
*   **Design Solution:**
    1.  **Immutable Ledger (Double-Entry Bookkeeping):** Never use simple updates like `UPDATE accounts SET balance = balance + 50`. Instead, use an append-only `transactions` table containing `debit` and `credit` rows.
    2.  **Compute Balance from Ledger:** The account balance is the sum of all transaction entries. Optimize reads using a cached materialized view or a running balance table.
    3.  **Strict Transaction Scope:**
        ```sql
        BEGIN;
        SELECT balance FROM account_balances WHERE account_id = A FOR UPDATE;
        -- Validate A has sufficient funds --
        INSERT INTO ledger (account_id, amount, type) VALUES (A, -50, 'debit');
        INSERT INTO ledger (account_id, amount, type) VALUES (B, 50, 'credit');
        UPDATE account_balances SET balance = balance - 50 WHERE account_id = A;
        UPDATE account_balances SET balance = balance + 50 WHERE account_id = B;
        COMMIT;
        ```

---

### Q2: How would you design a banking transaction system?
*   **Key Requirements:** Highly consistent, audit-trail compliant, resilient to hardware crashes.
*   **Design Solution:**
    1.  **Event Sourcing:** Store every transaction as an immutable event in an Event Store. Reconstruct current account state by replaying events.
    2.  **Relational Database for Ledger:** PostgreSQL with Serializable isolation to enforce strict consistency and prevent write skews.
    3.  **Outbox Pattern:** When a transaction occurs, write both the transaction record and an "outgoing event" message in the same ACID transaction. A background publisher polls the outbox table and writes events to Kafka, ensuring zero event loss.

---

### Q3: How would you design an inventory management system?
*   **Key Requirements:** High read throughput for product pages, accurate stock allocation during checkouts, protection against selling non-existent items.
*   **Design Solution:**
    1.  **Split Inventory into Cache and DB:**
        *   *Redis:* Stores virtual inventory counts for fast read allocations during checkout.
        *   *RDBMS (Postgres):* Stores the physical source-of-truth inventory counts.
    2.  **Two-Phase Reservation:**
        *   When a user clicks checkout, allocate stock in Redis using a Lua script (atomic check and decrement).
        *   Hold the virtual reservation in Redis for 10 minutes.
        *   When payment succeeds, write the final transaction to Postgres:
            `UPDATE inventory SET stock = stock - :qty WHERE product_id = :id AND stock >= :qty;`

---

### Q4: How would you design an order management system?
*   **Key Requirements:** State transition safety, reliability under high write loads.
*   **Design Solution:**
    1.  **State Machine Database Schema:**
        Maintain an `orders` table with a `status` column (`CREATED` -> `PAID` -> `PROCESSING` -> `SHIPPED`).
    2.  **State Transition Lock (Optimistic Locking):**
        When updating state, enforce the previous state in the query:
        `UPDATE orders SET status = 'PAID', version = version + 1 WHERE id = :orderId AND status = 'CREATED' AND version = :version;`
        This ensures concurrent workers cannot push the same order to different workflows.

---

### Q5: How would you design a ledger-based accounting system?
*   **Key Requirements:** Immutable audit trail, fast range query reporting (e.g., quarterly balance sheet generation).
*   **Design Solution:**
    1.  **Append-Only Schema:** Block all `UPDATE` and `DELETE` SQL statements on the ledger tables. Enforce this via database user permissions or database triggers.
    2.  **Periodic Snapshotting:** Calculating the balance of an account with 10 million ledger entries is too slow. Store weekly/monthly snapshots of balances in a `snapshots` table:
        `Balance = Snapshot (End of May) + Sum of Ledger Entries (June 1 to present)`.

---

### Q6: How would you design a reservation system handling high concurrency?
*   **Key Requirements:** Prevent double booking of tickets/seats, handle spikes during hot events (e.g., ticket releases).
*   **Design Solution:**
    1.  **Unique Constraints:** Enforce uniqueness at the database level on `(event_id, seat_number)` or `(room_id, date)`.
    2.  **Pessimistic Row Reservation:**
        ```sql
        BEGIN;
        SELECT status FROM seats WHERE event_id = :eventId AND seat_no = :seatNo FOR UPDATE NOWAIT;
        -- If status is 'available', continue booking, else abort --
        UPDATE seats SET status = 'reserved', reserved_by = :userId, expires_at = NOW() + INTERVAL '10 minutes' WHERE id = :seatId;
        COMMIT;
        ```
        *Note:* `NOWAIT` tells Postgres to fail immediately if another transaction holds the lock, rather than waiting, keeping API response times low.
