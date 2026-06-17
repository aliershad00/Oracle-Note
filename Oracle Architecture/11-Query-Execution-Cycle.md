# Oracle Architecture: Query Execution Cycle - How Oracle Executes Queries

## Introduction: What Happens When You Query?

When you type a SQL command, Oracle doesn't immediately execute it. Instead, it goes through multiple phases:

1. **Parse** - Is it valid SQL?
2. **Validate** - Do the tables exist?
3. **Optimize** - What's the best way to get data?
4. **Compile** - Translate to machine instructions
5. **Execute** - Run the plan
6. **Fetch** - Return results

Each phase is critical. Let's walk through a real query execution.

---

## Query Execution Phases

### Phase 1: Parse

**Purpose**: Check syntax and basic validity

```
Query: SELECT e.name, d.dept_name
       FROM employees e, departments d
       WHERE e.dept_id = d.dept_id;

Parse Phase:
├── Step 1: Lexical Analysis
│   └── Break query into tokens:
│       SELECT, e, name, d, dept_name, FROM, ...
│
├── Step 2: Syntax Checking
│   ├── Is it valid SQL?
│   ├── Check against SQL grammar rules
│   ├── SELECT keyword exists? YES
│   ├── FROM keyword exists? YES
│   ├── WHERE keyword valid? YES
│   └── Syntax valid? YES ✓
│
├── Step 3: Create Parse Tree
│   └── Build internal representation:
│       ├── Query type: SELECT
│       ├── Columns: name, dept_name
│       ├── Tables: employees, departments
│       └── Condition: e.dept_id = d.dept_id

Result: If syntax error, stop here with error
Example: "ORA-00900: invalid SQL statement"

If syntax valid: Move to Phase 2
```

### Phase 2: Validation

**Purpose**: Check if objects and columns exist

```
Validation Phase:

Step 1: Check Tables Exist
├── Dictionary lookup: Does table "employees" exist?
├── Dictionary lookup: Does table "departments" exist?
├── Both exist? YES ✓

Step 2: Check Columns Exist
├── In employees: Does column "dept_id" exist?
├── In employees: Does column "name" exist?
├── In departments: Does column "dept_id" exist?
├── In departments: Does column "dept_name" exist?
├── All exist? YES ✓

Step 3: Check Data Types
├── Column e.dept_id: NUMBER ✓
├── Column d.dept_id: NUMBER ✓
├── Types compatible for join? YES ✓

Step 4: Check User Privileges
├── Can user SELECT from employees? YES
├── Can user SELECT from departments? YES
├── Both allowed? YES ✓

Step 5: Validate Join Condition
├── e.dept_id (NUMBER) = d.dept_id (NUMBER)?
├── Compatible types? YES ✓

Result: If validation fails, stop here with error
Example: "ORA-00942: table or view does not exist"

If validation passes: Move to Phase 3 (Optimization)
```

### Phase 3: Optimization (Query Optimizer)

**Purpose**: Determine best execution plan

```
The Problem:
Multiple ways to execute same query.
Oracle must choose BEST one (fastest).

Example Query:
SELECT * FROM employees WHERE salary > 5000;

Option 1 (Full Table Scan):
├── Read every row in employees table
├── Check: Is salary > 5000?
├── If yes: Include in result
├── Rows to read: 10,000
├── Time: 5 seconds

Option 2 (Index Scan):
├── Use index on salary column
├── Find all rows where salary > 5000
├── Retrieve row details
├── Rows to read: 3,000 (matching rows)
├── Rows indexed: 10,000 (but quick lookup)
├── Time: 0.5 seconds ✓

Oracle Chooses: Option 2 (faster)

How Optimizer Decides:

Statistics Analysis:
├── Table employees: 10,000 rows
├── Salary column: 8000 values where > 5000
├── Index on salary: EXISTS
├── Cost of table scan: 100 units
├── Cost of index scan: 20 units
└── Choose: Index scan (lower cost) ✓

Selectivity Calculation:
├── Rows matching condition: 8000
├── Total rows: 10,000
├── Selectivity: 8000/10000 = 80%
├── If > 70%: Might use full scan anyway
├── If < 10%: Definitely use index
└── Decision tree: Index scan best ✓
```

### Phase 4: Code Generation

**Purpose**: Convert plan to executable code

```
Execution Plan from Optimizer:
├── Use INDEX_SALARY_DESC index
├── Find all entries where SALARY > 5000
├── Retrieve ROWID for each entry
├── Use ROWID to fetch full row from table
├── Return to user

Code Generation:
├── Machine code (object code) generated
├── Bytecode for Oracle VM
├── Optimized loops and jumps
├── Memory addresses pre-calculated
└── Ready for execution ✓

Compilation vs Interpretation:
├── Compilation: Convert once, run many times ✓
├── Interpretation: Convert every time (slow)

Oracle approach:
└── Compiles SQL to bytecode on first execution
└── Subsequent executions: Run compiled version (FAST) ✓
```

### Phase 5: Execution

**Purpose**: Actually run the query

```
Execution Phase:

Query: SELECT name, salary FROM employees WHERE salary > 5000;

Step 1: Initialize Execution
├── Get PGA work areas (sort, hash)
├── Initialize cursors
├── Set up result buffers
└── Ready to execute

Step 2: Execute Index Search
├── Access index on salary column
├── Find first entry where salary > 5000
├── Get ROWID (logical location)
├── Move to next index entry
├── Find all matching entries

Step 3: Fetch Data Rows
├── For each ROWID:
│   ├── Use ROWID to find physical block
│   ├── Read block from buffer cache (or disk)
│   ├── Extract NAME and SALARY columns
│   ├── Store in result buffer
│   └── Continue to next ROWID

Step 4: Filter and Format
├── Apply WHERE conditions (if any not handled by index)
├── Format result columns
├── Order results (if ORDER BY specified)
└── Ready to return

Performance Characteristics:

Database Buffer Cache:
├── Block 1: Employees 100-150 (in cache) → Fast
├── Block 2: Employees 151-200 (not in cache) → Read from disk
├── Block 3: Employees 201-250 (in cache) → Fast
├── Mix of cache hits and misses

Total time:
├── Index scan: 50 ms
├── Row fetch (20 rows): 200 ms (includes disk reads)
├── Result formatting: 10 ms
└── Total: 260 ms
```

### Phase 6: Fetch

**Purpose**: Return results to user

```
Fetch Phase:

Single Result Set:

User Application                 Server Process
    │                                  │
    ├─ Fetch results ────────────────→ │
    │                                  │
    │ ← 100 rows returned ────────────┤
    │                                  │
    ├─ Fetch next batch ────────────→ │
    │                                  │
    │ ← 100 rows returned ────────────┤
    │                                  │
    ... repeat until all results fetched

Array Fetch:
├── Batch size: 100 rows per fetch
├── Network round-trips: Minimized
├── Efficient for large result sets

Example:
Query returns: 50,000 rows
Array fetch size: 100
Network calls: 500 calls (50,000 / 100)
Without array fetch: 50,000 calls ✗

Performance Impact:
With array fetch: 500 network calls
Without: 50,000 network calls (100X slower)
```

---

## Execution Plan Details

### What is an Execution Plan?

An **Execution Plan** is a **step-by-step roadmap** of how Oracle will execute a query.

```
Visual Plan:

Query: SELECT * FROM employees e
       JOIN departments d ON e.dept_id = d.dept_id
       WHERE e.salary > 5000;

Execution Plan:
┌─────────────────────────────────────────────────┐
│ SELECT STATEMENT                                │
│ ├─ HASH JOIN                                   │
│ │  ├─ TABLE ACCESS (FULL SCAN) - EMPLOYEES    │
│ │  │  └─ Filter: salary > 5000                 │
│ │  └─ TABLE ACCESS (FULL SCAN) - DEPARTMENTS  │
│ └─ END                                          │
└─────────────────────────────────────────────────┘

Reading the Plan (Top to Bottom):
1. Start at root: SELECT STATEMENT
2. Execute HASH JOIN operation
3. First input: FULL SCAN on EMPLOYEES
   - Apply filter: salary > 5000
   - Rows returned: ~8000
4. Second input: FULL SCAN on DEPARTMENTS
   - Rows returned: 30
5. Join condition: Employee dept_id = Department dept_id
6. Return results to user
```

### Viewing the Execution Plan

```sql
-- Method 1: EXPLAIN PLAN (older)
EXPLAIN PLAN FOR
SELECT * FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > 5000;

SELECT PLAN_TABLE_OUTPUT FROM TABLE(DBMS_XPLAN.DISPLAY());

-- Method 2: AUTOTRACE (interactive)
SET AUTOTRACE ON EXPLAIN;
SELECT * FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > 5000;

-- Method 3: SQL TRACE (production)
ALTER SESSION SET SQL_TRACE = TRUE;
SELECT * FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > 5000;
ALTER SESSION SET SQL_TRACE = FALSE;

-- Analyze trace file with tkprof
```

### Cost in Execution Plan

```
Cost = Measure of resource consumption

Example Plan Output:
ID | OPERATION               | ROWS | BYTES | COST
───┼─────────────────────────┼──────┼───────┼──────
  0 | SELECT STATEMENT        |      |       |  100
  1 | HASH JOIN               | 8000 | 640K  |  100
  2 | TABLE ACCESS (FULL)     | 8000 | 400K  |   50
  3 | TABLE ACCESS (FULL)     |   30 | 1800  |   50

Interpretation:
├── Root (0): Total cost = 100 units
├── Hash join (1): Estimates 8000 rows, cost 100
├── Employees scan (2): Estimates 8000 rows, cost 50
├── Departments scan (3): Estimates 30 rows, cost 50

Cost Components:
├── CPU operations: Weighted
├── Disk I/O: Heavily weighted (slower)
├── Memory usage: Slightly weighted
└── Total: Estimate of relative resource use

Lower cost = Faster execution (typically)
Optimizer chooses plan with lowest cost
```

---

## Real-World Query Execution

### Scenario 1: Simple SELECT

```
Query:
SELECT salary FROM employees WHERE emp_id = 100;

Execution Steps:

1. Parse (5 ms)
   ├── Check syntax: Valid ✓
   └── Create parse tree

2. Validate (3 ms)
   ├── Check table employees exists: YES
   ├── Check column salary exists: YES
   ├── Check column emp_id exists: YES
   └── Check privileges: YES

3. Optimize (2 ms)
   ├── Check for index on emp_id: PRIMARY KEY index exists
   ├── Calculate selectivity: 1 row / 100,000 rows = 0.001%
   ├── Decision: Use index lookup
   └── Estimated cost: 2 units

4. Compile (1 ms)
   └── Generate bytecode for index lookup

5. Execute (10 ms)
   ├── Access primary key index on emp_id
   ├── Find entry for emp_id = 100
   ├── Get ROWID: 1.2.5 (file 1, block 2, slot 5)
   ├── Access buffer cache: Check if block 2 in memory
   ├── Hit: Block in cache (found)
   ├── Extract salary from block
   └── Return: 5000

6. Fetch (2 ms)
   ├── Format result
   └── Return to client

Total Time: 5 + 3 + 2 + 1 + 10 + 2 = 23 ms
Result: Salary = 5000
```

### Scenario 2: Complex Join Query

```
Query:
SELECT e.name, d.dept_name, COUNT(*) as emp_count
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
GROUP BY e.dept_id, d.dept_name
HAVING COUNT(*) > 10
ORDER BY emp_count DESC;

Execution Steps:

1. Parse (10 ms)
   └── Parse complex query structure

2. Validate (5 ms)
   ├── Check all tables: employees, departments ✓
   ├── Check all columns exist ✓
   ├── Check GROUP BY columns valid ✓
   └── Check aggregates valid ✓

3. Optimize (20 ms)
   ├── Find indexes: On dept_id?
   │   ├── employees.dept_id: INDEX found
   │   └── departments.dept_id: PRIMARY KEY index
   ├── Decision: Use index on employees.dept_id
   ├── Decision: Hash join with sorting
   ├── Estimated rows: 8000 (employees matching join)
   ├── Estimated cost: 85 units
   └── Estimated time: 500 ms

4. Compile (5 ms)
   └── Generate bytecode for complex plan

5. Execute (500 ms)
   ├── Table 1: Employees index scan on dept_id
   ├── Table 2: Departments full scan
   ├── Join: Hash join (build hash table from smaller table)
   ├── Aggregate: Count rows per dept_id (in memory)
   ├── Filter: WHERE COUNT(*) > 10 (remove small depts)
   ├── Sort: ORDER BY emp_count DESC
   └── Rows returned: 5 (depts with >10 employees)

6. Fetch (5 ms)
   ├── Format results
   └── Return to client

Total Time: 10 + 5 + 20 + 5 + 500 + 5 = 545 ms
Result: 5 departments with employee counts
```

---

## Bind Variables and Soft Parse

### Problem: Hard Parse Every Time

```
Without Bind Variables:

Query 1:
SELECT salary FROM employees WHERE emp_id = 100;
└── Execution: Parse + Validate + Optimize + Compile + Execute

Query 2:
SELECT salary FROM employees WHERE emp_id = 101;
└── Execution: Parse + Validate + Optimize + Compile + Execute
                (Repeated even though same query structure!)

Query 3:
SELECT salary FROM employees WHERE emp_id = 102;
└── Execution: Parse + Validate + Optimize + Compile + Execute
                (Repeated again!)

Problem:
├── Same query structure, different values
├── Full parsing done each time
├── Wasted CPU
├── Shared Pool filled with near-duplicates
└── Performance: POOR ✗
```

### Solution: Bind Variables

```
With Bind Variables:

Query Template:
SELECT salary FROM employees WHERE emp_id = :emp_id;

Execution 1: emp_id = 100
├── Parse: Full (first time)
├── Validate: Full (first time)
├── Optimize: Full (first time)
├── Compile: Full (first time)
├── Store in Shared Pool: Query + Plan
├── Execute: 100
└── Time: 20 ms

Execution 2: emp_id = 101
├── Parse: SKIPPED (found in Shared Pool)
├── Validate: SKIPPED (already done)
├── Optimize: SKIPPED (already done)
├── Compile: SKIPPED (already done)
├── Reuse: Existing plan
├── Execute: 101
└── Time: 5 ms (SOFT PARSE, 4X faster) ✓

Execution 3: emp_id = 102
├── Parse: SKIPPED (found in cache)
├── Validate: SKIPPED
├── Optimize: SKIPPED
├── Compile: SKIPPED
├── Reuse: Existing plan
├── Execute: 102
└── Time: 5 ms (SOFT PARSE, 4X faster) ✓

Benefit:
├── First query: 20 ms (hard parse)
├── Later queries: 5 ms each (soft parse)
├── Total for 1000 queries: 20 + (999 × 5) = 5015 ms
├── Without bind: 20,000 ms (1000 × 20)
└── Improvement: 4X FASTER ✓
```

---

## Caching and Performance

### Shared Pool Caching

```
Query 1: SELECT * FROM employees;
├── Parse tree created
├── Plan created
├── Bytecode generated
├── Stored in Shared Pool
└── Ready for reuse

Query 2: SELECT * FROM employees;
├── Check Shared Pool: Found!
├── Reuse: Existing parse tree
├── Reuse: Existing plan
├── Reuse: Existing bytecode
├── Execute directly
└── Time: MUCH faster

Shared Pool Hit Rate:
├── First query: Cache miss (create new)
├── Similar queries: Cache hits (reuse)
├── Hit rate target: >95%

Impact on Performance:
├── 95% hit rate: Very fast system
├── 50% hit rate: Lots of parsing overhead
└── 0% hit rate: System rebuilds cache constantly
```

### Buffer Cache Hits

```
Same Query on Same Data:

First Execution:
├── Block not in cache
├── Read from disk: 10 ms
├── Store in cache
└── Total: 10 ms

Second Execution (same block):
├── Block in cache
├── Read from cache: 0.1 ms
├── Return result
└── Total: 0.1 ms

Third Execution:
├── Block in cache
├── Read from cache: 0.1 ms
└── Total: 0.1 ms

Performance Impact:
├── Cache hits: 100X faster than disk
└── System throughput: Limited by cache size
```

---

## Query Execution Wait Events

### Where Time is Spent

```
Query Execution Breakdown:

Query 1: "SELECT * FROM big_table":
Total Time: 5000 ms (5 seconds)

Breakdown:
├── Parse: 10 ms (0.2%)
├── Optimize: 20 ms (0.4%)
├── Compile: 5 ms (0.1%)
├── Execute: 4500 ms (90%)
│   ├── Wait for I/O: 4000 ms (80%)
│   ├── CPU: 400 ms (8%)
│   └── Locks/latches: 100 ms (2%)
├── Fetch: 400 ms (8%)
└── Network transfer: 65 ms (1.3%)

Key Insight:
├── Most time: Disk I/O (80%)
├── Second most: Fetch/network (9.3%)
├── Third: Compilation (0.5%)
└── Optimization opportunity: Reduce disk reads
```

### Wait Events

```
Common Wait Events:

1. db file sequential read
   ├── Reading specific block from disk
   ├── Index access, single-block reads
   ├── Solution: Add index, cache blocks
   └── Impact: Very common

2. db file scattered read
   ├── Reading multiple blocks from disk
   ├── Full table scan, multi-block reads
   ├── Solution: Optimize query, use index
   └── Impact: Indicates full scans

3. direct path read
   ├── Reading data directly (bypass cache)
   ├── Parallel queries, large sorts
   ├── Solution: Depends on operation
   └── Impact: Performance intensive

4. log file sync
   ├── Waiting for redo logs to write
   ├── On every COMMIT
   ├── Solution: Batch commits, faster I/O
   └── Impact: Depends on disk speed

5. CPU time
   ├── Not a wait event
   ├── Actual CPU processing
   ├── Solution: Optimize code, reduce complexity
   └── Impact: High CPU = fast system (usually good)

Monitoring:
├── Check V$SESSION_WAIT
├── Check V$SYSSTAT
├── Use AWR reports
└── Find top wait events and optimize
```

---

## Key Takeaways

1. **Parse** = Check syntax
2. **Validate** = Check objects exist
3. **Optimize** = Find best execution plan
4. **Compile** = Generate executable code
5. **Execute** = Run the plan
6. **Fetch** = Return results
7. **Bind variables** = Enable soft parse (reuse)
8. **Soft parse** = Reuse existing plan (fast)
9. **Hard parse** = Build plan from scratch (slow)
10. **Execution plan** = Step-by-step roadmap
11. **Cache hits** = Reuse data in memory (fast)
12. **Wait events** = Show where time is spent
