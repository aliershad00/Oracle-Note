# Oracle Architecture: Locks and Concurrency - Sharing the Database

## Introduction: The Concurrency Problem

Imagine a bank account with $1000:

```
Without Locks:

User 1: Read balance = $1000
User 2: Read balance = $1000

User 1: Withdraw $300 → Balance = $700
User 2: Withdraw $200 → Balance = $800

Problem: Which value is correct?
├── Should be: $500 ($1000 - $300 - $200)
├── But shows: $800 (last write wins)
└── $500 lost! ✗

With Locks:

User 1: Lock the account
User 1: Read balance = $1000
User 1: Withdraw $300 → Balance = $700
User 1: Write to disk
User 1: Release lock

User 2: Wait for lock
User 2: Lock the account
User 2: Read balance = $700
User 2: Withdraw $200 → Balance = $500
User 2: Write to disk
User 2: Release lock

Result: $500 ✓ (correct)
```

This is why **locks** are critical for database consistency.

---

## What Are Locks?

### Definition

A **Lock** is a **mechanism to control access** to database resources:

- Prevents concurrent users from interfering
- Ensures data consistency
- Allows serialization of access

### Types of Locks

```
Lock Hierarchy (from highest to lowest restriction):

1. Exclusive Lock (X)
   ├── Only one user can hold it
   ├── Others must wait
   ├── Used during: UPDATE, DELETE, INSERT
   └── Example: Withdrawing money from account

2. Shared Lock (S)
   ├── Multiple users can hold it simultaneously
   ├── Prevents exclusive locks by others
   ├── Used during: SELECT with FOR UPDATE
   └── Example: Multiple people reading same file

3. Row-Level Lock
   ├── Locks one row
   ├── Other rows available for update
   ├── Highest granularity
   └── Example: One customer record locked

4. Table-Level Lock
   ├── Locks entire table
   ├── All rows unavailable for update
   ├── Lowest granularity
   └── Example: Entire table locked during maintenance

Locking Modes:
├── Row Exclusive (RX): Lock one row
├── Row Share (RS): Multiple readers, exclusive writers
└── Table Exclusive (X): Exclusive access to entire table
```

---

## Row-Level Locks (Most Common)

### How Row Locks Work

```
Scenario: Updating a customer record

Before Lock:
SELECT * FROM customers WHERE cust_id = 100;
├── Result: John, $5000 balance
└── NO lock held

During UPDATE (Lock Acquired):
UPDATE customers SET balance = 4000
WHERE cust_id = 100;
├── Oracle acquires exclusive lock on row 100
├── Other users can read row 100 (might see old value)
├── Other users cannot update row 100 (must wait)
└── Lock held until COMMIT or ROLLBACK

Example Concurrent Activity:

Timeline:

User 1: UPDATE row 100 to balance=4000
        ├── Lock acquired on row 100
        └── Waiting: COMMIT or ROLLBACK

User 2: Tries to UPDATE row 100 to balance=3500
        ├── Lock check: Is row locked?
        ├── Answer: YES (by User 1)
        ├── Action: WAIT for User 1 to release lock
        └── Blocked until User 1 commits

User 1: COMMIT
        ├── Lock released
        ├── Balance = 4000 (written to disk)

User 2: Lock acquired (now available)
        ├── Update row 100 to balance=3500
        ├── Lock held
        └── Waiting: COMMIT or ROLLBACK

User 2: COMMIT
        ├── Balance = 3500 (written to disk)
        ├── Lock released
        └── Update complete
```

### Row Lock Duration

```
Lock Lifecycle:

Step 1: Lock Acquired
├── Acquired on: INSERT, UPDATE, or DELETE
├── Acquired on: SELECT FOR UPDATE
├── Scope: Single row
└── Process: Automatic (no user action needed)

Step 2: Lock Held
├── Duration: Entire transaction
├── User can do other operations while holding lock
├── Other locks may be acquired (more rows)
├── Lock remains until transaction ends
└── NO explicit "release lock" command needed

Step 3: Lock Released
├── Released on: COMMIT
├── Released on: ROLLBACK
├── Automatic: No user action needed
├── Scope: All locks in transaction released
└── Other users: Can now access locked rows

Example:

10:00:00 - Lock Acquired
10:00:01 - Still holding lock
10:00:02 - Still holding lock
10:00:05 - COMMIT issued
10:00:05 - Lock Released (automatic)
10:00:06 - Lock no longer held
```

---

## Transaction Isolation Levels

### The Four Isolation Levels

```
Isolation Level: Determines what other users can see

Level 1: READ UNCOMMITTED
├── Can read: Uncommitted changes
├── Problem: Dirty reads possible ✗
├── Speed: Fastest
├── Consistency: Lowest
└── Use: Very rare (avoid)

Level 2: READ COMMITTED (Default in Oracle)
├── Can read: Only committed changes
├── Problem: Non-repeatable reads possible
├── Speed: Fast
├── Consistency: Good
└── Use: Most common (recommended)

Level 3: REPEATABLE READ
├── Can read: Committed changes
├── Problem: Phantom reads possible
├── Speed: Medium
├── Consistency: Better
└── Use: When needed for consistency

Level 4: SERIALIZABLE
├── Can read: Committed changes
├── Problem: None (but may serialize access)
├── Speed: Slowest
├── Consistency: Highest
└── Use: For critical transactions
```

### READ COMMITTED (Default)

```
How It Works:

User 1 Transaction:
UPDATE employees SET salary = 6000 WHERE emp_id = 100;
(Not yet committed)
└── Block in buffer cache shows: salary = 6000

User 2 Transaction:
SELECT salary FROM employees WHERE emp_id = 100;
(Isolation: READ COMMITTED)

What User 2 Sees:
├── Check: Is current value committed?
├── Answer: NO (User 1 hasn't committed)
├── Action: Use undo data to get old value
├── Result: Sees salary = 5000 (old committed value)
├── User 2 is NOT blocked

Later:
User 1: COMMIT
User 2: SELECT salary FROM employees WHERE emp_id = 100;
(Queries again)
└── Result: Now sees salary = 6000 ✓
```

### SERIALIZABLE

```
How It Works (Strict Isolation):

User 1 Transaction (SERIALIZABLE):
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

Range Query:
SELECT * FROM employees
WHERE dept_id BETWEEN 10 AND 20
AND salary > 5000;
(Returns 50 employees)

User 2 Transaction (attempts):
INSERT INTO employees (dept_id, salary)
VALUES (15, 6000);
(Tries to insert in User 1's range)

What Happens:
├── User 1 locked range: dept_id 10-20, salary > 5000
├── User 2 tries to insert: dept_id=15, salary=6000
├── Insertion in User 1's range!
├── Conflict detection
├── User 2 blocked (must wait)

User 1: COMMIT
├── Range lock released
├── User 2 attempt allowed

Result:
├── User 1 query is serialized
├── No phantom reads (rows added during query)
└── Strictest consistency but slower
```

---

## Deadlocks

### What is a Deadlock?

A **Deadlock** occurs when:

- User A waits for lock held by User B
- User B waits for lock held by User A
- Neither can proceed = DEADLOCK

```
Deadlock Scenario:

User 1                          User 2
├─ Lock Employee 100            ├─ Lock Department 50
├─ Needs Department 50          ├─ Needs Employee 100
└─ WAIT for Department 50 ┐     └─ WAIT for Employee 100
                         │
                    DEADLOCK!
```

### Classic Deadlock Example

```
User 1 Code:
BEGIN
  UPDATE employees SET salary = 6000 WHERE emp_id = 100;
  UPDATE departments SET budget = 50000 WHERE dept_id = 50;
  COMMIT;
END;

User 2 Code:
BEGIN
  UPDATE departments SET budget = 45000 WHERE dept_id = 50;
  UPDATE employees SET salary = 5500 WHERE emp_id = 100;
  COMMIT;
END;

Timeline:

Time 1:
├─ User 1: Lock Employee 100 ✓
├─ User 2: Lock Department 50 ✓

Time 2:
├─ User 1: Try to lock Department 50
│         ├─ Already locked by User 2
│         └─ WAIT
├─ User 2: Try to lock Employee 100
│         ├─ Already locked by User 1
│         └─ WAIT

Result: DEADLOCK!
├─ User 1 waiting for User 2
├─ User 2 waiting for User 1
└─ Neither can proceed ✗

Error: ORA-00060: Deadlock detected while waiting for resource
```

### Deadlock Prevention

```
Solution 1: Lock in Same Order

User 1 Code:
BEGIN
  UPDATE departments SET budget = 50000 WHERE dept_id = 50;
  UPDATE employees SET salary = 6000 WHERE emp_id = 100;
  COMMIT;
END;

User 2 Code:
BEGIN
  UPDATE departments SET budget = 45000 WHERE dept_id = 50;
  UPDATE employees SET salary = 5500 WHERE emp_id = 100;
  COMMIT;
END;

Timeline:

Time 1:
├─ User 1: Lock Department 50 ✓
├─ User 2: Lock Department 50?
│         ├─ Already locked by User 1
│         └─ WAIT

Time 2:
├─ User 1: Lock Employee 100 ✓
├─ User 1: COMMIT (release all locks)
├─ User 2: Lock Department 50 ✓ (now available)

Time 3:
├─ User 2: Lock Employee 100 ✓
├─ User 2: COMMIT
└─ No deadlock! ✓

Key Principle:
└─ Both lock in same order: DEPT then EMP
└─ Prevents circular wait condition
```

---

## Monitoring Locks

### Detecting Lock Contention

```sql
-- View all current locks
SELECT * FROM V$LOCK;

-- View session locks
SELECT * FROM V$LOCK
WHERE SID IN (SELECT SID FROM V$SESSION WHERE USERNAME = 'SCOTT');

-- View blocking sessions
SELECT * FROM V$SESSION_BLOCKER;

-- Find who's blocking whom
SELECT
  BLOCKING_SESSION,
  SID,
  SERIAL#,
  USERNAME,
  WAIT_CLASS
FROM V$SESSION
WHERE BLOCKING_SESSION IS NOT NULL;

-- Kill blocking session (if needed)
ALTER SYSTEM KILL SESSION 'SID,SERIAL#';
```

### Lock Wait Times

```
Example Output:

USERNAME  | SID | STATUS    | WAIT_TIME | EVENT
──────────┼─────┼───────────┼───────────┼──────────────────
SCOTT     | 100 | ACTIVE    |   -1      | db file sequential read
JOHN      | 101 | WAITING   | 5000 ms   | enq: TX - row lock contention

Interpretation:
├─ John (SID 101) is WAITING
├─ Waiting for: TX (transaction) lock
├─ Wait time: 5000 ms (5 seconds already)
├─ Probably blocked by: SCOTT (SID 100)
└─ Reason: Row lock held by Scott
```

---

## Pessimistic vs Optimistic Locking

### Pessimistic Locking (Oracle Default)

```
Assumption: Conflicts will happen frequently

Approach:
├─ Lock before reading
├─ Hold lock while modifying
├─ Release lock on commit

Example (Bank Transfer):
BEGIN
  SELECT balance INTO balance
  FROM accounts
  WHERE account_id = 1
  FOR UPDATE;  ← Acquire lock immediately

  new_balance := balance - 100;

  UPDATE accounts
  SET balance = new_balance
  WHERE account_id = 1;

  COMMIT;  ← Lock released
END;

Pros:
├─ Prevents conflicts
├─ No conflicts on commit
├─ Safe for high-contention scenarios

Cons:
├─ Can cause lock waiting
├─ Potential deadlocks
├─ Lower concurrency
└─ Slower if conflicts rare
```

### Optimistic Locking (Alternative)

```
Assumption: Conflicts rare, minimize locking

Approach:
├─ Read without lock
├─ Check for changes at commit time
├─ Retry if conflict detected

Example (using version number):
BEGIN
  SELECT balance, version
  INTO balance, version
  FROM accounts
  WHERE account_id = 1;
  ← NO lock acquired

  new_balance := balance - 100;

  UPDATE accounts
  SET balance = new_balance, version = version + 1
  WHERE account_id = 1
  AND version = version;  ← Check not changed

  IF SQL%ROWCOUNT = 0 THEN
    -- Row changed by another user, retry
    ROLLBACK;
  ELSE
    COMMIT;
  END IF;
END;

Pros:
├─ Higher concurrency
├─ No lock waiting
├─ Fewer deadlocks

Cons:
├─ Must handle conflicts
├─ Potential retries
├─ Code complexity
└─ Risk if conflicts common
```

---

## Lock Types in Oracle

### Table Locks

```
Scenarios:

Mode: Exclusive (X)
├─ No other locks allowed
├─ Used during: CREATE INDEX, ALTER TABLE
├─ Duration: Operation duration
└─ Example: Rebuilding index

Mode: Shared (S)
├─ Multiple shared locks allowed
├─ Used during: Fast full table scans
├─ Duration: Until query completes

Automatic vs Manual:

Automatic (default):
├─ Oracle acquires as needed
├─ Usually row-level (preferred)

Manual (if needed):
LOCK TABLE employees IN EXCLUSIVE MODE;
UPDATE employees SET ...;
COMMIT;
```

### Intent Locks

```
Purpose: Show intent to lock rows in table

Example:

Transaction 1:
UPDATE employees SET salary = 6000 WHERE dept_id = 50;

Oracle Automatically:
├─ Acquires Intent eXclusive (IX) lock on table
├─ Then locks specific rows
└─ Allows other sessions to know: "Updates in progress"

Benefit:
├─ Prevents incompatible table locks by others
├─ Without serializing all row updates
└─ Better concurrency
```

---

## Real-World Example: ATM Network

### Scenario: Concurrent Withdrawals

```
Setup:
├─ Account: 10001, Balance: $1000
├─ ATM 1 in New York
├─ ATM 2 in Boston
├─ Network latency: 100 ms

User 1 (ATM 1): Withdraw $500
User 2 (ATM 2): Withdraw $600

Timeline:

T=0:00 - User 1: Insert withdrawal transaction
         ├─ Query: SELECT balance FROM accounts WHERE id=10001
         ├─ Isolation: READ COMMITTED
         ├─ Row lock: NOT acquired yet
         ├─ Sees: balance = $1000

T=0:10 - User 2: Insert withdrawal transaction
         ├─ Query: SELECT balance FROM accounts WHERE id=10001
         ├─ Sees: balance = $1000 (still committed value)
         ├─ No lock conflict
         └─ Both read: $1000

T=0:20 - User 1: UPDATE balance = 500
         ├─ Lock acquired on row 10001
         ├─ Balance set to $500
         ├─ Waiting: COMMIT

T=0:30 - User 2: UPDATE balance = 400
         ├─ Try to lock row 10001
         ├─ Already locked by User 1
         ├─ WAIT for lock
         └─ Blocked

T=0:50 - User 1: COMMIT
         ├─ Lock released
         ├─ Balance = $500 on disk

T=0:52 - User 2: Lock acquired
         ├─ But wait! Read balance was $1000
         ├─ Now balance is $500 (from User 1)
         ├─ User 2 calculates: $1000 - $600 = $400
         ├─ Updates to $400
         └─ Overwrites User 1's $500!

T=0:55 - User 2: COMMIT
         ├─ Balance = $400 on disk
         └─ BUG! Should be $-100 (overdraft)

Problem Analysis:
├─ Both transactions saw initial balance: $1000
├─ Neither blocked the other's read
├─ Both read same value from redo (READ COMMITTED)
├─ Sequential updates overwrote each other
└─ Result: Non-repeatable read issue

Solution: Application Logic
├─ Don't just subtract from read value
├─ Use database semantics:
  UPDATE accounts SET balance = balance - 500
  WHERE id = 10001;
  └─ Let Oracle handle current value
```

---

## Key Takeaways

1. **Locks** = Control concurrent access
2. **Row locks** = Most common (per-row)
3. **Exclusive lock** = Only one user
4. **Shared lock** = Multiple readers
5. **Isolation level** = What others can see
6. **READ COMMITTED** = Default (good balance)
7. **SERIALIZABLE** = Strictest (slowest)
8. **Deadlock** = Circular wait condition
9. **Lock in same order** = Prevent deadlocks
10. **Pessimistic** = Lock early (safe)
11. **Optimistic** = Lock late (fast)
12. **Monitor** = Check V$LOCK and V$SESSION
