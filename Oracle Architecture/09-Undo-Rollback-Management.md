# Oracle Architecture: Undo and Rollback Management - Reversing Changes

## Introduction: The Safety Net

Imagine you're editing a Word document and make a mistake. You can press **CTRL+Z** to undo it. Oracle has a similar system:

- **Undo logs** remember the old values
- When you ROLLBACK, undo logs restore the data
- Without undo, you'd lose ability to rollback

Think of it like a security camera recording everything, so you know exactly what happened before the "mistake".

---

## What is Undo?

### Definition

**Undo** is a **record of old values** before changes are made. It allows:

- Rolling back uncommitted transactions
- Supporting read consistency (other users see old data)
- Flashback operations (recovering accidentally deleted data)

### Undo vs Redo

```
Comparison:

REDO Logs:
├── Stores: NEW values (after change)
├── Purpose: Forward recovery
├── Action on crash: REPLAY changes
├── Use: "Make it happen again"

UNDO Logs:
├── Stores: OLD values (before change)
├── Purpose: Backward recovery (rollback)
├── Action on ROLLBACK: UNDO changes
├── Use: "Pretend it never happened"

Both needed for consistency ✓
```

### Real-World Analogy: Backup System

```
Your Document: "Sales: 1000 units"

You edit: "Sales: 5000 units"

REDO log records: "Changed from 1000 to 5000"
├── Use case: If computer crashes mid-save
├── Replay: "Change to 5000"
└── Result: Recovers the new value

UNDO log records: "Previous value was 1000"
├── Use case: If you press CTRL+Z
├── Undo: "Back to 1000"
└── Result: Restores the old value

Together = Complete recovery capability ✓
```

---

## Undo Segments

### What is an Undo Segment?

An **Undo Segment** is a **storage area that holds undo data**.

```
Undo Tablespace:
├── UNDO tablespace (dedicated to undo)
├── Contains: Multiple undo segments
├── Default name: UNDOTBS1
├── Automatic management: YES
└── Size: 1 GB - 100 GB (depends on workload)

Undo Segments:
├── Segment 0 (system)
├── Segment 1 (for transactions)
├── Segment 2 (for transactions)
├── ...
└── Segment 10 (for transactions)

Why Multiple Segments?
├── Multiple transactions happening simultaneously
├── Each transaction needs undo space
├── Multiple segments reduce contention
├── Better concurrency
```

### Undo Segment Assignment

```
When user connects:

Transaction 1 starts:
├── Oracle assigns: Undo Segment 1
├── All changes go to Segment 1
├── Transaction scope includes Segment 1

Transaction 2 starts (different user):
├── Oracle assigns: Undo Segment 2
├── All changes go to Segment 2
├── Transaction scope includes Segment 2

Benefit:
├── No overlap (no contention)
├── Each transaction has dedicated space
└── Concurrent transactions independent
```

---

## How Undo Works

### Step-by-Step: Creating Undo Data

```
Original Data:
Employee 100: Salary = 5000

User executes:
UPDATE employees SET salary = 6000 WHERE emp_id = 100;

Process:

Step 1: Read block from disk
├── Block 1050 contains employee 100
├── Load into Database Buffer Cache
└── Block image: "Salary: 5000"

Step 2: Create Undo Entry
├── Before applying change
├── Record old value in Undo Segment 1
├── Undo entry: "Column salary_id: OLD 5000, NEW 6000"
├── Stored in UNDO tablespace
└── Entry in Undo Log Buffer (similar to Redo Log Buffer)

Step 3: Create Redo Entry
├── Record the change in Redo Log Buffer
├── Entry: "Block 1050: salary changed to 6000"

Step 4: Apply Change
├── Modify block in Database Buffer Cache
├── Block state: "Salary: 6000"
├── Block state: DIRTY (modified)

Step 5: Both Buffers Ready
├── Undo entry in Undo Log Buffer (has old value)
├── Redo entry in Redo Log Buffer (has new value)
└── Dirty block in Database Buffer Cache

Result:
├── New value: 6000 (in memory, will write to disk)
├── Old value: 5000 (in undo, for rollback or read consistency)
└── Change record: 5000→6000 (in redo, for recovery)
```

### Lifecycle of Undo Data

```
Phase 1: ACTIVE (Transaction In Progress)
├── Undo data: Used by current transaction
├── Usage: "If user RBACKs, we have old values"
├── Location: Undo Segment (active)
├── Example: "Before rollback"

Phase 2: COMMITTED (Transaction Committed)
├── Undo data: No longer needed for this transaction
├── But: Kept briefly for read consistency
├── Usage: "Other users may still read old values"
├── Duration: Retained based on UNDO_RETENTION

Phase 3: EXPIRED (Retention Period Over)
├── Undo data: Safe to overwrite
├── Usage: "New transactions can reuse space"
├── Oracle marks space as: "Available for reuse"
├── Duration: Space held until needed

Phase 4: REUSED (Overwritten by New Transactions)
├── New undo data: Overwrites old data
├── Old data: No longer available
├── Space: Reused efficiently
└── Cycle continues
```

---

## Rollback Process

### When You Issue ROLLBACK

```
Transaction in Progress:

UPDATE employees SET salary = 6000 WHERE emp_id = 100;
UPDATE departments SET budget = 50000 WHERE dept_id = 50;
INSERT INTO audit_log VALUES (...);

ROLLBACK;  ← User issues this command

Step 1: Find Transaction Identifier
├── Get Transaction ID from session
├── Transaction: TX_12345
└── Undo Segment: Segment 1

Step 2: Locate Undo Data
├── Find all undo entries for TX_12345
├── In Undo Segment 1
└── Entries:
    ├── Undo 1: "salary: 5000 (old value)"
    ├── Undo 2: "budget: 45000 (old value)"
    └── Undo 3: "Insert: NULL (deletion)"

Step 3: Apply Undo in Reverse Order
├── Undo 3: Delete the inserted audit log row
├── Undo 2: Change budget back to 45000
├── Undo 1: Change salary back to 5000
└── Order matters! (reverse of execution)

Step 4: Update Transaction Status
├── Mark transaction: ROLLED BACK
├── Release transaction locks
├── Free undo segment space
└── Transaction cleared

Step 5: Session Continues
├── Session still connected
├── Transaction ended successfully
├── User can start new transaction
└── "SQL> " prompt returns

Result:
├── Database as if transaction never happened ✓
├── Salary: Back to 5000 ✓
├── Budget: Back to 45000 ✓
├── Audit log: Row removed ✓
```

### Savepoint Rollback

```
Partial Rollback Using Savepoint:

UPDATE employees SET salary = 6000 WHERE emp_id = 100;
SAVEPOINT SP1;  ← Create rollback point

UPDATE departments SET budget = 50000 WHERE dept_id = 50;

ROLLBACK TO SAVEPOINT SP1;
← Roll back only to SP1 (not entire transaction)

Step 1: Find Savepoint SP1
├── Remember: Previous savepoint
├── Undo only changes AFTER SP1
└── Keep changes BEFORE SP1

Step 2: Apply Undo
├── Undo: "budget: 45000 (old value)"
├── Keep: "salary: 6000 (before SP1)"
└── Selective rollback ✓

Result:
├── Department budget: Back to 45000 ✓
├── Employee salary: Stays 6000 ✓
├── Transaction still active: YES
├── Can commit or rollback more
└── Partial rollback achieved ✓
```

---

## Read Consistency with Undo

### The Problem: Dirty Reads

```
User 1 (In Transaction):
UPDATE employees SET salary = 6000 WHERE emp_id = 100;
← Salary changed to 6000
← Not yet committed

User 2 (Different Transaction):
SELECT salary FROM employees WHERE emp_id = 100;
← What should User 2 see?

Option A (Without Undo - Dirty Read):
├── User 2 sees: 6000 (uncommitted value)
├── User 1 rolls back
├── User 2's result now wrong ✗
└── Inconsistent data!

Option B (With Undo - Read Consistency):
├── User 2 sees: 5000 (committed value from undo)
├── User 2 data consistent even if User 1 rolls back
├── No dirty reads ✓
└── Each user sees consistent view!
```

### How Undo Provides Consistency

```
Timeline:

14:00:00 - User 1 begins transaction
14:00:05 - User 1: UPDATE salary = 6000
          ├── Undo records: OLD 5000
          └── Block shows: NEW 6000

14:00:10 - User 2 queries same employee
          ├── Sees block: 6000 (uncommitted)
          ├── Checks: "Is this value committed?"
          ├── Block shows: "Transaction TX_12345 in progress"
          ├── Oracle action: Use UNDO data
          ├── Reconstructs: OLD VALUE 5000
          └── User 2 sees: 5000 ✓

14:00:15 - User 1 issues ROLLBACK
          ├── Block restored to 5000
          ├── Transaction cleared
          └── Both users see same value ✓

Result:
├── User 2 never saw uncommitted data ✓
├── User 2 data never inconsistent ✓
└── Oracle automatically handled via undo ✓
```

### Undo Usage for Read Consistency

```
Concept: Consistent Read Image (CR Block)

User 1 Update:
├── Block changed from 5000 to 6000
├── Undo link points to undo segment

User 2 Reads:
├── Want to read block
├── Check block header: "SCN in progress"
├── Go to undo
├── Reconstruct old image from undo
├── Read old image (5000)
└── Consistent with transaction start time

Benefit:
├── User 2 doesn't wait for User 1
├── No blocking
├── Both work independently
├── Read consistency guaranteed
```

---

## Undo Retention and Flashback

### Undo Retention Policy

```
UNDO_RETENTION Parameter:
├── Default: 900 seconds (15 minutes)
├── Defines: How long to keep undo data
├── Purpose: Enable flashback queries

Example Setup:
UNDO_RETENTION = 1800  (30 minutes)

Behavior:
├── Transaction committed at 14:00
├── Undo data kept until 14:30
├── At 14:30: Marked as expired
├── At 14:45: Space might be reused
├── At 15:00+: Definitely reused

User tries flashback query at 14:35:
├── "Show me data from 14:05"
├── Undo data available ✓
├── Flashback succeeds

User tries flashback query at 15:30:
├── "Show me data from 14:05"
├── Undo data OVERWRITTEN ✗
├── Flashback fails "ORA-01555: snapshot too old"
```

### Flashback Query

```
Use Undo Data to See Historical Data:

Current Time: 14:30

Customer record updated:
├── Previous name: "ABC Corp"
├── Current name: "XYZ Corp"

Flashback Query:
SELECT * FROM customers
AS OF TIMESTAMP (SYSDATE - INTERVAL '15' MINUTE)
WHERE customer_id = 100;

Result:
├── Old value: "ABC Corp"
├── Uses undo data
├── Shows state 15 minutes ago ✓

How It Works:
├── Find block's current state: "XYZ Corp"
├── Check undo: "Was ABC Corp 15 min ago"
├── Reconstruct old image
├── Return old data to user
└── Read-only query (no modification)
```

---

## Undo Tablespace Management

### Automatic Undo Management (AUM)

```
Modern Oracle (11g and later):
├── Undo management: AUTOMATIC
├── DBA doesn't manage individual segments
├── Oracle handles allocation/deallocation
└── Size automatically managed

Setup:
CREATE UNDO TABLESPACE UNDOTBS1
  DATAFILE '/u01/undotbs01.dbf' SIZE 5G
  AUTOEXTEND ON NEXT 100M MAXSIZE 50G;

ALTER SYSTEM SET UNDO_TABLESPACE = UNDOTBS1;
ALTER SYSTEM SET UNDO_RETENTION = 1800;  (30 min)
```

### Undo Tablespace Sizing

```
How Much Undo Space Do You Need?

Formula (approximate):
Undo Size = (Transaction_Rate × Avg_Undo_Size × Retention_Time)

Example:

Peak Transactions: 1000 per second
Average Undo per Transaction: 50 KB
Retention Required: 30 minutes (1800 seconds)

Calculation:
500 TPS × 50 KB × 1800 seconds = 45 GB

But not all 1000 TPS generates undo:
├── Reads don't generate undo
├── Only write-intensive operations
├── Average: 500 TPS write (estimated)

Result:
├── Undo needed: 500 × 50 KB × 1800 = 45 GB
├── Add buffer: 20% more = 54 GB
├── Recommended: 60 GB undo tablespace
```

### Monitoring Undo Space

```sql
-- View undo tablespace info
SELECT TABLESPACE_NAME, EXTENT_MANAGEMENT, STATUS
FROM DBA_TABLESPACES
WHERE TABLESPACE_NAME LIKE 'UNDO%';

-- Check undo space usage
SELECT TABLESPACE_NAME, SUM(BYTES)/1024/1024 AS FREE_MB
FROM DBA_FREE_SPACE
WHERE TABLESPACE_NAME LIKE 'UNDO%'
GROUP BY TABLESPACE_NAME;

-- View undo statistics
SELECT * FROM V$UNDOSTAT;

-- Check current undo retention
SHOW PARAMETER undo_retention;

-- Monitor undo usage
SELECT SUM(BYTES)/1024/1024 AS UNDO_USED_MB
FROM DBA_UNDO_EXTENTS
WHERE STATUS = 'ACTIVE';
```

---

## Undo Issues and Solutions

### Issue 1: "ORA-01555: Snapshot Too Old"

```
Cause:
├── Long-running query
├── Undo data for old snapshot expired
├── Oracle cannot reconstruct old image

Scenario:
├── Undo retention: 15 minutes
├── Query started: 14:00
├── Query reading: 14:20
├── Undo data aged out: 14:15
├── Block needed from 14:00: NOT IN UNDO ✗

Error Message:
ORA-01555: snapshot too old within rollback
segment "_SYSSMU1_1246973$" (rollback segment too small)

Solutions:
1. Increase UNDO_RETENTION
   ALTER SYSTEM SET UNDO_RETENTION = 3600;  (1 hour)

2. Increase UNDO tablespace size
   ALTER TABLESPACE UNDOTBS1 ADD DATAFILE ...;

3. Optimize query (make it faster)
   └── Shorter query = less undo data needed

4. Schedule long queries in low-activity time
   └── Less concurrent transactions = less undo pressure
```

### Issue 2: "ORA-30036: Unable to extend segment"

```
Cause:
├── Undo tablespace full
├── Cannot allocate more undo space
├── High transaction volume

Scenario:
├── Undo tablespace: 50 GB, MAXSIZE 50 GB (hit limit)
├── Peak activity: 1000 TPS
├── Undo generated: 100 GB/hour (exceeds allocation)
└── Error on new transaction

Solutions:
1. Increase MAXSIZE parameter
   ALTER DATABASE DATAFILE '/u01/undotbs01.dbf'
   RESIZE 100G;

2. Add new undo datafile
   ALTER TABLESPACE UNDOTBS1 ADD DATAFILE '/u02/undotbs02.dbf' SIZE 20G;

3. Reduce undo generation
   └── Optimize transactions (batch operations)

4. Increase retention if appropriate
   └── May need more time for read consistency
```

---

## Undo Performance Implications

### Write Performance

```
Update Transaction Impact:

Without Undo Optimization:
├── Generate undo: 1 ms
├── Generate redo: 1 ms
├── Write to buffer cache: 1 ms
├── Total: 3 ms per transaction

With Undo Optimization:
├── Parallel undo generation: 0.5 ms
├── Redo generation: 1 ms
├── Buffer cache: 1 ms
└── Total: 2.5 ms per transaction

Impact at Scale:
1000 TPS = 3000 ms vs 2500 ms = 16% faster ✓
```

### Read Performance

```
Query with Read Consistency:

Scenario 1: No Uncommitted Data
├── Read block directly: 0.1 ms
└── Return result

Scenario 2: Uncommitted Data Exists
├── Read block: 0.1 ms
├── Check transaction state: 0.05 ms
├── Follow undo chain: 0.2 ms
├── Reconstruct old image: 0.15 ms
├── Return result: 0.5 ms total

Small overhead for consistency ✓
```

---

## Real-World Example: E-commerce Database

### Scenario: Customer Updates Order

```
Customer updates order quantity:
└── "Change order from 10 units to 20 units"

Time 14:00:00:
├── User clicks "Update Order"
├── System: UPDATE orders SET quantity = 20 WHERE order_id = 100;

Time 14:00:01:
├── Undo recorded: OLD quantity = 10
├── Redo recorded: NEW quantity = 20
├── Block marked: DIRTY

Time 14:00:02:
├── Customer realizes mistake
├── Clicks "Undo/Cancel"
├── ROLLBACK command issued

Time 14:00:03:
├── Oracle reads undo data
├── Quantity restored to 10
├── Transaction rolled back ✓

Time 14:00:05:
├── Customer tries again with correct quantity
├── UPDATE orders SET quantity = 15;
├── New undo recorded for 15

Time 14:00:06:
├── Customer confirms: COMMIT;
├── Undo becomes "committed"
├── Retained for read consistency

Time 14:00:07:
├── Another user runs report
├── Queries same order
├── Sees quantity: 15 ✓

Time 14:30:00:
├── 30 minutes passed
├── Undo retention expired
├── Undo space marked for reuse
└── Space recycled
```

---

## Key Takeaways

1. **Undo** = Record of old values before changes
2. **Undo Segment** = Storage area for undo data
3. **Rollback** = Use undo to reverse uncommitted changes
4. **Savepoint** = Mark point for partial rollback
5. **Read Consistency** = Use undo to provide committed values
6. **UNDO_RETENTION** = How long to keep undo data
7. **Flashback Query** = Use undo to see historical data
8. **Automatic Management** = Oracle handles undo automatically
9. **Undo Tablespace** = Dedicated storage for undo data
10. **Balance needed** = Enough undo for consistency but not wasteful
