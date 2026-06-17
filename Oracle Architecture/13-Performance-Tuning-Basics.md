# Oracle Architecture: Performance Tuning Basics - Making Databases Fast

## Introduction: Why Performance Matters

```
Real-World Impact:

Scenario: Web Application Query

Query 1 (Original):
└─ Execution time: 10 seconds
└─ 1000 users online
└─ Total wait time per second: 10,000 seconds = 167 minutes wasted!

Query 2 (After tuning):
└─ Execution time: 0.5 seconds
└─ 1000 users online
└─ Total wait time per second: 500 seconds = 8 minutes
└─ Time saved: 159 minutes per second = 166X improvement

Business Impact:
├─ Unhappy users: Reduced from 90% to 5%
├─ Page abandonment: Reduced by 80%
├─ Server load: Reduced by 95%
├─ Infrastructure costs: Can serve 20X more users on same hardware
└─ Revenue impact: Millions saved
```

---

## Performance Tuning Layers (Bottom Up)

### Layer 1: Database I/O (Disk Access)

```
Slowest operations:

Read from disk: 10 milliseconds
Read from cache: 0.1 milliseconds
Read from CPU: 0.001 milliseconds

Strategy: Minimize disk I/O

Disk Reads Per Query:
├─ Bad: 1000 disk reads
├─ Good: 100 disk reads
├─ Excellent: 10 disk reads

Techniques:
├─ Add indexes (find data faster)
├─ Increase buffer cache (hold more data in memory)
├─ Optimize queries (read fewer blocks)
└─ Archive old data (reduce search space)
```

### Layer 2: CPU Usage

```
CPU Processing:

Once data in memory, CPU processes it

Optimization:
├─ Reduce rows processed
├─ Reduce columns selected
├─ Use better algorithms
├─ Parallel processing (if available)

Example:

Query 1:
SELECT * FROM employees;
├─ Processes: ALL columns
├─ Processes: ALL rows
├─ Discards: 99% of data
└─ CPU: WASTED on unwanted data

Query 2:
SELECT emp_id, name FROM employees WHERE dept_id = 50;
├─ Processes: Only needed columns
├─ Processes: Only needed rows (filtered)
├─ Discards: Only truly unwanted
└─ CPU: USED efficiently
```

### Layer 3: Memory Usage

```
Memory Efficiency:

Goal: Keep frequently used data in memory

Strategy:
├─ Larger buffer cache: More data cached
├─ Larger shared pool: More SQL cached
├─ Better memory management: Less waste

Trade-offs:
├─ More memory = Fewer disk reads
├─ More memory = Higher cost
├─ Need to balance

Monitoring:
├─ Cache hit ratio: >95% ideal
├─ Memory utilization: 80-90% ideal
└─ Swapping: 0% ideal (using disk for memory)
```

### Layer 4: Application Logic

```
Application Optimization:

Best query still slower than:
├─ Not running the query at all
├─ Running query earlier (cache result)
├─ Running query less frequently
└─ Running query in batch (combine operations)

Examples:

Bad: Run query for every user view
├─ 1000 views = 1000 queries
└─ Load: HIGH

Good: Run query once, cache result
├─ 1000 views = 1 query
└─ Load: 1000X less

Bad: Run query individually
├─ 1000 items = 1000 queries
└─ Time: Hours

Good: Batch queries (bulk operation)
├─ 1000 items = 1 query
└─ Time: Seconds
```

---

## Finding Performance Problems

### Wait Events - Where Time is Spent

```
Wait Event Analysis:

Most time spent in:
├─ Disk I/O waits (db file sequential read)
├─ Network waits (SQL*Net message from client)
├─ Lock waits (enq: TX - row lock contention)
├─ CPU (not a wait, but processing time)

Steps:
1. Identify TOP wait events
2. Determine which are tunable
3. Focus on highest impact

Example:

Top Wait Events:
1. db file sequential read: 70% of time ← Focus here!
2. db file scattered read: 15% of time ← Secondary
3. log file sync: 10% of time ← Lower priority
4. db cpu: 5% of time ← CPU already optimal

Action: Reduce disk reads (add indexes, optimize queries)
```

### Common Wait Events

```
1. db file sequential read
   ├─ Single block read from disk
   ├─ Index lookup, one block needed
   ├─ Tuning: Add index, use existing index
   ├─ Example: SELECT * FROM table WHERE id = 100;

2. db file scattered read
   ├─ Multiple blocks read at once
   ├─ Full table scan, multiple blocks
   ├─ Tuning: Add index, optimize query
   ├─ Example: SELECT * FROM table WHERE salary > 5000;

3. log file sync
   ├─ Waiting for redo logs to write to disk
   ├─ On every COMMIT
   ├─ Tuning: Faster disks, batch commits
   ├─ Example: Many commits, no batching

4. enq: TX - row lock contention
   ├─ Waiting for row lock held by another user
   ├─ Contention on same rows
   ├─ Tuning: Optimize lock order, reduce lock time
   ├─ Example: Two users updating same customer

5. CPU
   ├─ Not a wait event
   ├─ Active processing
   ├─ High CPU usually good (working)
   ├─ Very high CPU might indicate: Complex queries, poor algorithms

Monitoring:
SELECT * FROM V$SESSION_EVENT
WHERE SID = (SELECT SID FROM V$MYSTAT)
ORDER BY TIME_WAITED DESC;
```

---

## Indexing Strategy

### How Indexes Help

```
Scenario: Finding employee by ID

Without Index:
├─ Read entire table: 100,000 rows
├─ Check each row: "Is this emp_id = 100?"
├─ Found at row 50,000
├─ Time: 5 seconds (full scan)

With Index:
├─ Use index B-tree
├─ Navigate tree: 3-4 disk reads
├─ Find row directly
├─ Time: 0.05 seconds (100X faster!)

Index Structure (B-tree):
Root
├─ 1-50000: Left branch
├─ 50001-100000: Right branch

Left branch
├─ 1-25000: Left branch
├─ 25001-50000: Right branch

... continue until found
```

### When to Add Indexes

```
Good Index Candidates:

1. Search Conditions
   └─ SELECT * FROM employees WHERE emp_id = 100;

2. Join Columns
   └─ JOIN departments ON emp.dept_id = dept.dept_id;

3. Sort Columns
   └─ ORDER BY salary DESC;

4. Filter Conditions
   └─ WHERE salary > 5000;

5. Foreign Keys
   └─ Improve join performance

Bad Index Candidates:

1. Columns with low selectivity
   └─ Gender (M/F only): 50% matches
   └─ Boolean column: 50% matches

2. Columns rarely used
   └─ Unused in WHERE clauses
   └─ Index overhead not justified

3. Columns with many NULLs
   └─ Index doesn't store NULLs
   └─ Reduced benefit

4. Small tables
   └─ Full scan might be faster
   └─ Index overhead not justified
```

### Index Tuning

```
SQL Statement:
SELECT name, salary
FROM employees
WHERE dept_id = 50
AND salary > 5000
ORDER BY salary DESC;

Without Index:
├─ Time: 10 seconds
├─ Disk reads: 1000

Strategy 1: Single Column Index
CREATE INDEX idx_dept ON employees(dept_id);
├─ Result: Find dept_id = 50 fast
├─ But: Still scan all 800 rows from that dept
├─ Still: Need to filter salary > 5000
├─ Time: 8 seconds (20% improvement)

Strategy 2: Composite Index
CREATE INDEX idx_dept_salary
ON employees(dept_id, salary DESC);
├─ Result: Find dept_id = 50 AND salary > 5000 directly
├─ Both conditions in index
├─ Return in sort order
├─ Time: 1 second (10X improvement) ✓

Strategy 3: Covering Index
CREATE INDEX idx_covering
ON employees(dept_id, salary DESC, name);
├─ Include all needed columns
├─ No need to access table at all
├─ Get everything from index
├─ Time: 0.5 seconds (20X improvement) ✓✓

Monitoring:
SELECT * FROM V$DB_OBJECT_CACHE
WHERE TYPE = 'INDEX'
ORDER BY HITS DESC;
```

---

## Query Optimization

### Query Rewriting Techniques

```
Technique 1: Use WHERE Instead of HAVING

Bad:
SELECT dept_id, COUNT(*) as emp_count
FROM employees
GROUP BY dept_id
HAVING COUNT(*) > 10;

├─ Process: All 100,000 rows
├─ Group: All departments
├─ Filter: Remove small groups
└─ Time: 5 seconds (processes too much)

Good:
SELECT dept_id, COUNT(*) as emp_count
FROM employees
WHERE emp_id IS NOT NULL  ← Filter early
GROUP BY dept_id
HAVING COUNT(*) > 10;

├─ Process: Already filtered rows
├─ Group: Fewer rows to group
├─ Filter: Remove small groups
└─ Time: 2 seconds ✓
```

### Execution Plan Analysis

```
Goal: Check if Oracle using optimal plan

Bad Plan Example:
ID | OPERATION              | ROWS | COST
───┼────────────────────────┼──────┼──────
 0 | SELECT STATEMENT       |      | 1000
 1 | SORT ORDER BY          | 8000 | 1000
 2 | TABLE ACCESS (FULL)    | 8000 | 500
 3 | FILTER                 |      |
 4 | TABLE ACCESS (FULL)    |100000| 500

Interpretation:
├─ Full table scan on table 2: 100,000 rows
├─ Filter reduces to 8000 rows
├─ Sort 8000 rows
├─ Cost: 1000 (high)
└─ Problem: Full scan, sorting expensive

Better Plan (with index):
ID | OPERATION              | ROWS | COST
───┼────────────────────────┼──────┼──────
 0 | SELECT STATEMENT       |      | 100
 1 | INDEX RANGE SCAN       | 8000 | 100

Interpretation:
├─ Use index to find rows
├─ 8000 rows from index directly
├─ Already sorted (by index)
├─ Cost: 100 (10X better)
└─ Solution: Add appropriate index
```

---

## Database Configuration Tuning

### Memory Parameters

```
SGA_TARGET Parameter:
├─ Controls total SGA size
├─ Default: 1 GB (too small for production)
├─ Typical: 4-50 GB

Setting:
ALTER SYSTEM SET SGA_TARGET = 16G SCOPE = BOTH;

Distribution:
├─ Database Buffer Cache: 50-70% of SGA
├─ Shared Pool: 20-30% of SGA
├─ Redo Log Buffer: 1-2% of SGA
├─ Large Pool: 5-10% of SGA

Monitoring:
SELECT * FROM V$MEMORY_TARGET;
SELECT * FROM V$SGA_DYNAMIC_COMPONENTS;
```

### Database Block Size

```
Standard: 8 KB (configured at database creation)

When to adjust:
├─ Smaller (2-4 KB): OLTP systems
│  ├─ Many small random reads
│  ├─ Reduces block overhead
│  ├─ Reduces memory per block
│  └─ Cannot change after creation
│
├─ Standard (8 KB): Default, general purpose
│  ├─ Good balance
│  ├─ Standard across industry
│  └─ Recommended for most systems
│
├─ Larger (16-32 KB): Data warehouse
│  ├─ Few large sequential reads
│  ├─ Efficient for bulk operations
│  ├─ More data per block
│  └─ Cannot change after creation

Warning: Block size set at database creation
```

---

## Application Tuning

### Connection Pooling

```
Problem: Too Many Connections

Without Pooling:
├─ 1000 concurrent users
├─ 1000 server processes (one each)
├─ 1000 × 200 MB PGA = 200 GB
├─ System runs out of memory ✗

With Connection Pooling:
├─ Connection pool: 100 connections
├─ 1000 users share 100 connections
├─ 100 × 200 MB PGA = 20 GB
├─ Memory saved: 180 GB ✓

Implementation:
├─ Oracle Connection Manager
├─ Application server pooling
├─ Third-party connection poolers
└─ Cloud managed services
```

### Batch Operations

```
Problem: Many Single Operations

Bad Approach:
FOR i = 1 TO 1000000
  INSERT INTO table VALUES (...);
END FOR;

├─ 1 million individual transactions
├─ 1 million COMMIT commands
├─ 1 million redo log writes
├─ Time: Hours

Good Approach:
FOR i = 1 TO 1000000
  INSERT INTO table VALUES (...);
  IF i MOD 1000 = 0 THEN
    COMMIT;
  END IF;
END FOR;
COMMIT;

├─ 1 million inserts
├─ 1000 COMMIT commands (batch)
├─ 1000 redo log writes
├─ Time: Minutes (100X faster) ✓

Principles:
├─ Batch commits
├─ Reduce context switches
├─ Bulk operations when possible
└─ Minimize transaction overhead
```

---

## Monitoring and Diagnostics

### Key Metrics to Monitor

```
1. Database CPU Usage
   ├─ Should be: 50-80% (optimal)
   ├─ Too high: Query optimization needed
   ├─ Too low: Waiting on I/O
   └─ Command: TOP, vmstat, iostat

2. Disk I/O Usage
   ├─ Should be: <80% (balanced)
   ├─ Too high: Increase buffer cache, add indexes
   ├─ I/O pattern: Sequential (good) vs Random (bad)
   └─ Command: iostat, V$SYSSTAT

3. Cache Hit Ratio
   ├─ Should be: >95% (target)
   ├─ Too low: Increase buffer cache
   ├─ Query: SELECT bytes_cached / bytes_read FROM cache_stats;
   └─ Low ratio indicates: Disk activity

4. Lock Waits
   ├─ Should be: Zero or minimal
   ├─ Frequent: Concurrency issue
   ├─ Query: SELECT * FROM V$LOCK;
   └─ High waits: Reduce contention

5. Query Response Time
   ├─ Should be: <1 second (typical)
   ├─ >5 seconds: Needs optimization
   ├─ Measure: End-to-end user time
   └─ Query: SELECT elapsed_time FROM V$SQL;
```

### Diagnostic Tools

```
AWR (Automatic Workload Repository):
├─ Captures database statistics hourly
├─ Reports on performance
├─ Identifies top SQL, waits, events
├─ Available in Enterprise Edition
└─ Command: @$ORACLE_HOME/rdbms/admin/awrrpt.sql

TKPROF (SQL Trace Analysis):
├─ Analyzes SQL trace files
├─ Shows CPU, I/O, execution time
├─ Identifies slow queries
└─ Command: tkprof <trace_file> <output_file>

EXPLAIN PLAN:
├─ Shows query execution plan
├─ Identifies full table scans
├─ Suggests indexes
└─ Command: EXPLAIN PLAN FOR SELECT ...;

V$SESSION_WAIT:
├─ Current wait events
├─ Which users waiting
├─ How long waiting
└─ Query: SELECT * FROM V$SESSION_WAIT;
```

---

## Real-World Tuning Example

### Scenario: Slow Report (30 seconds)

```
Initial Analysis:

Query:
SELECT e.name, e.salary, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > 5000
ORDER BY e.salary DESC;

Execution Time: 30 seconds
Disk Reads: 5000
Wait Events: 90% db file sequential read

Step 1: Check Indexes
QUERY: SELECT INDEX_NAME FROM USER_INDEXES;
Result: No indexes on salary or dept_id

Step 2: Check Execution Plan
EXPLAIN PLAN shows: FULL TABLE SCAN on employees

Step 3: Find Root Cause
├─ Query missing index on salary
├─ Full scan 100,000 rows
├─ Then filter salary > 5000
└─ Very inefficient

Step 4: Add Index
CREATE INDEX idx_emp_salary ON employees(salary);

Re-run: 3 seconds (10X improvement)
Disk Reads: 500

Step 5: Further Optimization
EXPLAIN PLAN shows: Sort on ORDER BY

Step 6: Add Composite Index
CREATE INDEX idx_emp_sal_dept
ON employees(salary DESC, dept_id);

Re-run: 0.5 seconds (60X improvement vs original!)
Disk Reads: 50

Result:
├─ Original: 30 seconds, 5000 disk reads
├─ After tuning: 0.5 seconds, 50 disk reads
├─ Improvement: 60X faster
├─ User satisfaction: 99% (was 10%)
└─ Infrastructure cost: Can serve 100X more users
```

---

## Performance Tuning Checklist

```
Startup:
□ Set SGA_TARGET appropriately
□ Set PGA_AGGREGATE_TARGET
□ Set DB_CACHE_SIZE
□ Enable automatic statistics gathering
□ Enable SQL tracing (development only)

Development:
□ Create indexes on search columns
□ Create indexes on join columns
□ Review execution plans
□ Use bind variables
□ Batch operations where possible

Production:
□ Monitor cache hit ratios
□ Monitor wait events
□ Monitor CPU/disk usage
□ Review top SQL by elapsed time
□ Archive old data
□ Gather statistics regularly
□ Review AWR reports
□ Test backups

Ongoing:
□ Monthly performance review
□ Quarterly optimization assessment
□ Annual infrastructure review
□ Plan capacity additions
□ Keep Oracle patched
□ Test on test system before production
```

---

## Key Takeaways

1. **Disk I/O** = Biggest bottleneck (focus here first)
2. **Wait events** = Show where time spent
3. **Indexes** = Most powerful optimization tool
4. **Execution plan** = Understand query path
5. **Cache hit ratio** = Target >95%
6. **Batch operations** = Reduce overhead
7. **Connection pooling** = Reduce memory
8. **Bind variables** = Enable soft parse (reuse)
9. **Monitor constantly** = Catch issues early
10. **Test changes** = Before production deployment
