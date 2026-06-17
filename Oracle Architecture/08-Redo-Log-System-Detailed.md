# Oracle Architecture: Redo Log System (Detailed) - Complete Change History

## Introduction: Why Redo Logs are Critical

Redo logs are Oracle's **complete record of every change**. Without them:

- Lost transactions after crash = Data loss
- No way to recover to any point in time
- Backups alone cannot recover all changes

Think of it like a flight recorder (black box) - it records everything so that even if something goes wrong, you know exactly what happened.

---

## Redo Log Fundamentals

### What is a Redo Entry?

A **Redo Entry** is a **record of a single database change**.

```
Example Redo Entry:

[14:23:45.123]
├── Operation: UPDATE
├── Table ID: 45678 (internal reference)
├── Block: 2050 (block number in data file)
├── Offset: 256 (position within block)
├── New Value: 6000 (salary amount)
├── SCN: 50000123 (System Change Number - timestamp)
└── Transaction ID: TX_12345

Every change (INSERT, UPDATE, DELETE) = One redo entry
```

### Why Redo Logs Matter

```
Scenario: Database Crash at 2:15 PM

Before Crash:
├── Committed Transaction 1: "Update salary to 5000" ✓
├── Committed Transaction 2: "Insert new employee" ✓
├── Uncommitted Transaction 3: "Delete customer" ✗
└── Transaction 3 in progress, not committed

In Database Buffer Cache (Memory - LOST):
├── Transaction 1 data: Updated
├── Transaction 2 data: Updated
├── Transaction 3 data: Updated
├── None written to disk yet

On Disk (Data Files):
├── Transaction 1: NOT there
├── Transaction 2: NOT there
├── Transaction 3: NOT there
└── All lost!

BUT In Redo Log (On Disk - SAVED):
├── Transaction 1: "Update salary to 5000" ✓ SAVED
├── Transaction 2: "Insert new employee" ✓ SAVED
├── Transaction 3: "Delete customer" ✗ SAVED
└── All recorded

After Crash Recovery:
├── SMON reads redo logs
├── Replays: Transaction 1 ✓ (COMMITTED - restore)
├── Replays: Transaction 2 ✓ (COMMITTED - restore)
├── Rollback: Transaction 3 ✗ (NOT COMMITTED - undo)
└── Database restored to 2:15 PM state ✓

Without redo logs: Transactions 1 and 2 would be LOST ✗
```

---

## Redo Log File Architecture

### Redo Log Groups and Members

```
Redo Log Groups:
├── Group 1
│   ├── Member 1: /u01/redo01a.log (primary)
│   ├── Member 2: /u02/redo01b.log (copy)
│   └── Member 3: /u03/redo01c.log (copy)
│
├── Group 2
│   ├── Member 1: /u01/redo02a.log
│   ├── Member 2: /u02/redo02b.log
│   └── Member 3: /u03/redo02c.log
│
└── Group 3
    ├── Member 1: /u01/redo03a.log
    ├── Member 2: /u02/redo03b.log
    └── Member 3: /u03/redo03c.log

Why Multiple Groups?
├── Group 1 is ACTIVE (current)
├── Group 2 is STANDBY (ready)
├── Group 3 is STANDBY (ready)
└── When Group 1 full: Switch to Group 2

Why Multiple Members (Copies)?
├── Member 1 on disk 1
├── Member 2 on disk 2 (copy)
├── Member 3 on disk 3 (copy)
├── If disk 1 fails: Still have members 2 & 3
└── Protection against disk failure
```

### Redo Log Multiplexing

```
Concept: Write same redo entry to multiple locations

Without Multiplexing:
├── LGWR writes to: /u01/redo01.log
├── Disk 1 fails
├── Redo log lost ✗
└── Cannot recover!

With Multiplexing:
├── LGWR writes to:
│   ├── /u01/redo01a.log (disk 1)
│   ├── /u02/redo01b.log (disk 2)
│   └── /u03/redo01c.log (disk 3)
│
├── All three identical copies
├── Disk 1 fails: Still have copies 2 & 3
└── Can recover ✓

Oracle's behavior:
├── Writes simultaneously to all members
├── Continues if 1 member fails
├── Errors if ALL members fail
```

---

## Redo Log Sequence and Cycling

### How Redo Logs Cycle

```
Scenario: Three redo log groups

Time 1:00 - Group 1 ACTIVE
├── LGWR writes all changes to Group 1
├── Entries accumulate
└── When Group 1 full: Switch

Time 1:05 - Switch to Group 2 ACTIVE
├── CKPT checkpoint marker written to Group 1
├── Group 1 now INACTIVE (can be archived)
├── LGWR starts writing to Group 2
├── Old entries in Group 1: Safe to archive

Time 1:10 - Switch to Group 3 ACTIVE
├── CKPT checkpoint marker written to Group 2
├── Group 2 now INACTIVE
├── LGWR starts writing to Group 3
└── When Group 3 full: Back to Group 1?

Time 1:15 - Attempt to reuse Group 1
├── Check: Is Group 1 safe to overwrite?
├── CKPT verification: All data from Group 1 written to disk?
├── YES: Overwrite Group 1 ✓
├── NO: WAIT for checkpoint to complete

Result: Circular cycling
├── Group 1 → Group 2 → Group 3 → Group 1 ...
└── Each contains complete change history before overwrite
```

### Redo Log Sequence Numbers

```
Each Group Has a Sequence Number:

Group 1:
├── Sequence: 1
├── Entries: "Update salary to 5000", "Insert employee"
└── Status: ARCHIVED

Group 2:
├── Sequence: 2
├── Entries: "Update dept budget", "Insert customer"
└── Status: ARCHIVED

Group 3:
├── Sequence: 3
├── Entries: "Delete old records", "Update totals"
└── Status: CURRENT (being written to)

Archiver Process:
├── Copies Group 1 to: /archive/1_1_XXX.arc
├── Copies Group 2 to: /archive/2_1_XXX.arc
├── Keeps historical record
└── Enable point-in-time recovery
```

---

## Redo Log Writing Process (LGWR)

### When LGWR Writes

```
LGWR Triggers:

1. Every 3 Seconds
   └── Automatic timer-based flush

2. When Buffer 1/3 Full
   └── Don't wait, write now

3. On COMMIT
   └── All session's changes written immediately
   └── User receives confirmation only after

4. On ROLLBACK
   └── Changes not committed still logged
   └── Undo logs handle rollback

5. On Checkpoint
   └── All dirty blocks must have redo entries written
   └── Before DBWn writes data blocks

6. Before Database Shutdown
   └── All pending entries flushed
   └── Clean shutdown
```

### LGWR Performance Characteristics

```
LGWR is FAST:
├── Sequential writes (not random)
├── Redo log block size: 512 bytes (OS block)
├── Write speed: Microseconds per write
├── Write latency: 1-5 milliseconds

Why Sequential?
├── Redo logs are circular
├── Always write to "next" location
├── No seeking back
├── Optimal for disk performance

Comparison:
Database Buffer Cache Writes (DBWn):
├── Random reads/writes (accessing any block)
├── May require seeking
├── Slower (5-10 milliseconds)

Redo Log Writes (LGWR):
├── Sequential writes (always next block)
├── No seeking
├── Faster (1-2 milliseconds)
```

---

## Redo Log Components: What Gets Logged?

### Everything Logged

```
Data Changes:
├── INSERT statements
├── UPDATE statements
├── DELETE statements
├── Merge statements
└── Direct path operations

Structural Changes:
├── CREATE TABLE
├── ALTER TABLE
├── CREATE INDEX
├── DROP TABLE
└── All DDL operations

Transaction Control:
├── COMMIT statements
├── ROLLBACK statements
├── SAVEPOINT creation
└── Transaction markers

System Operations:
├── User login/logout (if auditing on)
├── Privilege grants
├── Password changes
└── Tablespace operations

Block Changes:
├── Row header information
├── Transaction ID
├── Lock information
├── Undo pointers
└── Change vectors (exact bytes changed)
```

### What's NOT Logged?

```
Things NOT in Redo Logs:

1. Query Results (SELECT)
   └── Queries don't modify data
   └── No redo needed

2. Temporary Segment Changes
   └── Temporary tablespace data
   └── Lost anyway on shutdown

3. Certain Background Operations
   └── Some internal operations optimized
   └── Not needed for recovery

4. Block Reads
   └── Only writes logged
   └── Reads don't need recovery

Principle: Only changes needed for recovery
```

---

## Redo Log Entry Structure

### Inside a Redo Entry

```
Redo Entry Layout:

Redo Entry Header:
├── Entry length: 256 bytes
├── Header byte flag
├── Version: 10.2.0.5
└── Redo byte flags

Operation Code:
├── 5 = UPDATE
├── 2 = INSERT
├── 3 = DELETE
├── 7 = LOB operation
└── Etc.

Data Block Information:
├── File ID: 3 (data file 3)
├── Block #: 1050 (block within file)
├── Slot: 5 (row location within block)
├── Table ID: 45678
└── Index ID: 56789

Change Vector:
├── Column ID: 7
├── Old Value: 5000
├── New Value: 6000
├── Length: 4 bytes
└── Value: 6000

Transaction Information:
├── Transaction ID: 00001234
├── Timestamp: 14:23:45.123
├── User ID: 50
├── Session ID: 123
└── Serial #: 4567
```

---

## Redo Log Scenarios and Examples

### Scenario 1: Committed Transaction

```
Time 14:00 - User Transaction

SQL: UPDATE employees SET salary = 6000 WHERE emp_id = 100;

Step 1: Execution
├── PGA sort area: SQL parsed
├── Database buffer cache: Block containing employee 100 loaded
├── Memory: Salary changed to 6000
├── Block becomes DIRTY

Step 2: Redo Entry Created
├── Redo Log Buffer entry added:
│   ├── Operation: UPDATE
│   ├── Block: 1050
│   ├── Old value: 5000
│   ├── New value: 6000
│   └── Transaction: TX_12345

Step 3: User Issues COMMIT

Step 4: LGWR Triggered
├── Redo Log Buffer flushed to disk
├── Entry written to: /u01/redo01a.log
├── Entry written to: /u02/redo01b.log (copy)
├── Entry written to: /u03/redo01c.log (copy)
├── Disk write confirmed

Step 5: User Receives Success
├── Message: "1 row updated"
├── Transaction complete ✓

Step 6: Later DBWn Writes (Independent)
├── DIRTY block eventually written to data file
├── May be seconds/minutes later
└── But redo log already safe on disk

If Crash Between Step 4 and Step 6:
├── Redo log has entry (on disk)
├── Data file doesn't (in memory)
├── SMON replays: "Update salary to 6000"
└── Transaction restored ✓
```

### Scenario 2: Uncommitted Transaction (Rollback)

```
Time 14:05 - Another User Transaction

SQL: DELETE FROM customers WHERE cust_id = 50;

Step 1: Execution
├── Customer 50 record deleted from buffer cache
├── Redo entry created: "DELETE from CUSTOMERS, block 2050"
├── Entry in Redo Log Buffer

Step 2: User Issues ROLLBACK (instead of COMMIT)

Step 3: Undo Log Used
├── Undo data contains old values
├── Redo log entry NOT removed (stay in log)
├── But marked as "rolled back"
└── Data restored in buffer cache

Step 4: What's in Redo Log?
├── Entry still exists: "DELETE from customers cust_id = 50"
├── But marked as: "Transaction rolled back"
├── Recovery-wise: This is OK

Step 5: If Crash Happens
├── SMON reads redo log
├── Sees: "DELETE from customers"
├── Checks: "Transaction committed?"
├── Answer: "NO - rolled back"
├── Result: Deletion is UNDONE
└── Customer 50 restored ✓

Important:
├── Redo logs store both committed AND uncommitted changes
├── Undo logs handle "which to keep"
├── Recovery reads both to be consistent
```

### Scenario 3: Database Crash

```
Before Crash (14:15 PM):
├── Committed Transaction 1: "Salary update"
│   └── In redo log: YES ✓
│   └── In data file: NO (not yet written)
│
├── Committed Transaction 2: "New employee"
│   └── In redo log: YES ✓
│   └── In data file: NO (not yet written)
│
├── Uncommitted Transaction 3: "Customer delete"
│   └── In redo log: YES
│   └── In data file: NO
│   └── Status: NOT COMMITTED
│
└── Server crashes

After Crash (14:16 PM):
├── Oracle restarts
├── Detects abnormal shutdown
└── SMON starts recovery process

SMON Recovery Process:

Step 1: Open Control Files
├── Read last checkpoint: "Up to block 50000"
├── Read last redo sequence: 15

Step 2: Open Data Files
├── Verify file headers
├── All files consistent with checkpoint

Step 3: Check Redo Logs
├── Start from sequence 15, block 50001
├── Find all entries after last checkpoint

Step 4: Cache Recovery Phase
├── Read redo entries from redo log files
├── Transaction 1: Committed ✓ → REPLAY
├── Transaction 2: Committed ✓ → REPLAY
├── Transaction 3: Uncommitted ✗ → ROLLBACK
└── Apply changes to buffer cache

Step 5: Verify Consistency
├── All committed data now in memory
├── All uncommitted data removed
├── Database consistent ✓

Step 6: Write to Data Files
├── DBWn writes all changes to disk
├── Data file now consistent with redo log

Step 7: Open Database
└── Users can now connect

Result:
├── Transaction 1: Restored ✓
├── Transaction 2: Restored ✓
├── Transaction 3: Rolled back ✓
└── No data loss for committed transactions ✓
```

---

## Redo Log Sizing

### How Large Should Redo Logs Be?

```
Factors:
1. Transaction Volume
   ├── More transactions = More redo
   └── Peak TPS (transactions per second)

2. Average Transaction Size
   ├── Small updates: ~1 KB
   ├── Large batch operations: ~10 MB
   └── Mix in your workload

3. Checkpoint Interval
   ├── More frequent checkpoints = Smaller redo logs needed
   └── Less frequent = Larger redo logs needed

4. Recovery Time Requirement
   ├── Need to recover in 5 minutes?
   ├── Need to recover in 1 hour?
   └── Affects log sizing

Example Calculation:

Peak Activity: 1000 transactions/second
Average TPS: 500 transactions/second
Average transaction size: 5 KB

Redo per second: 500 × 5 KB = 2.5 MB/sec
Redo per minute: 2.5 MB × 60 = 150 MB/min
Redo per hour: 150 MB × 60 = 9 GB/hour

For 5-minute Recovery Window:
Redo per 5 minutes: 150 MB × 5 = 750 MB

Recommended Redo Log Group Size:
├── Group 1: 1 GB (can hold 7 minutes)
├── Group 2: 1 GB
├── Group 3: 1 GB
├── Switch every 4-5 minutes
└── Allows time for archiving

With 3 Groups:
├── Total redo log storage: 3 GB
├── Rotation: Every 12-15 minutes
└── Comfortable for recovery
```

### Redo Log Issues

```
Problem: Redo Logs Too Small
├── Switch groups very frequently
├── Checkpoint activity high
├── Contention on redo allocation
└── Performance SLOW ✗

Problem: Redo Logs Too Large
├── Recovery takes very long
├── Archiving takes longer
├── More data to manage
└── Recovery time HIGH ✗

Sweet Spot:
├── Logs fill every 5-10 minutes
├── 3-4 groups minimum
├── Archiver can keep up
├── Recovery reasonable time
└── Performance optimal ✓
```

---

## Redo Log Archiving

### What is Archiving?

```
Redo Log Lifecycle:

State 1: CURRENT
├── Actively being written to by LGWR
├── Only one current group at a time

State 2: ACTIVE (before switching)
├── Just switched out, but not yet archived
├── Checkpoint marker not yet written
├── Cannot be overwritten yet

State 3: ARCHIVABLE
├── Ready to archive
├── Can be copied to archive location

State 4: ARCHIVED
├── Copied to /archive/ directory
├── Protected for long-term recovery
├── Can now be overwritten

Archive Naming:
├── Original: redo01.log (sequence 1)
├── Archive: 1_1_12345678.arc
│   └── Sequence#_Group#_Timestamp
```

### ARCHIVELOG vs NOARCHIVELOG Mode

```
NOARCHIVELOG Mode (Default):
├── Redo logs NOT archived
├── Redo logs overwritten after checkpoint
├── Storage: Minimum
├── Recovery: Only to last backup
├── Use: Development, test environments
└── Cannot do point-in-time recovery ✗

ARCHIVELOG Mode (Recommended for Production):
├── Redo logs ARE archived
├── Redo logs NOT overwritten
├── Storage: Large (depends on activity)
├── Recovery: To any point in time
├── Use: Production databases
└── Full recovery capability ✓

Example:

NOARCHIVELOG:
├── Redo logs: 3 GB (3 groups of 1 GB)
├── Space used: 3 GB (just cycling)

ARCHIVELOG:
├── Redo logs: 3 GB (just cycling)
├── Archive location: 100+ GB (complete history)
└── Space used: 100+ GB (all archives kept)
```

---

## Monitoring Redo Logs

```sql
-- View redo log groups
SELECT GROUP#, MEMBERS, BYTES/1024/1024 AS SIZE_MB, STATUS
FROM V$LOG;

-- View redo log members (files)
SELECT GROUP#, MEMBER, TYPE, STATUS
FROM V$LOGFILE;

-- Check redo log activity
SELECT * FROM V$SYSSTAT WHERE NAME LIKE '%redo%';

-- Monitor log switches
SELECT * FROM V$LOG_HISTORY
ORDER BY FIRST_TIME DESC
FETCH FIRST 10 ROWS ONLY;

-- Check if in ARCHIVELOG mode
SELECT NAME, LOG_MODE FROM V$DATABASE;

-- View archive destination
SHOW PARAMETER db_recovery_file_dest;
```

---

## Key Takeaways

1. **Redo logs** = Complete record of all database changes
2. **LGWR** = Writes redo entries to disk BEFORE data writes
3. **Groups** = Multiple redo log groups cycle continuously
4. **Members** = Multiple copies of each group (redundancy)
5. **Multiplexing** = Write to multiple locations simultaneously
6. **Sequential writes** = Fast and efficient disk I/O
7. **Three states**: CURRENT (writing) → ACTIVE (switching) → ARCHIVED (protected)
8. **Recovery** = Replay redo logs to restore committed transactions
9. **Archiving** = Long-term protection for point-in-time recovery
10. **Size matters** = Too small = performance issues, Too large = slow recovery
