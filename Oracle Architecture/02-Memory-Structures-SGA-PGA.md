# Oracle Architecture: Memory Structures - The SGA and PGA

## Introduction: Why Memory Matters

When Oracle needs data, it could:

1. Read from disk (SLOW - takes milliseconds)
2. Read from RAM (FAST - takes microseconds)

That's why Oracle allocates large amounts of memory to cache data. But memory is shared differently:

- **SGA** = Shared by everyone (like a company breakroom)
- **PGA** = Private to each person (like your personal desk)

---

## System Global Area (SGA)

### What is the SGA?

The **SGA (System Global Area)** is a **large block of shared memory** allocated when the Oracle instance starts. Every user, every process, and every background process can access this shared memory.

### Real-World Analogy: Office Building Shared Spaces

```
SGA = Shared office spaces:
├── Conference Room = Database Buffer Cache
    (Everyone can use, discussion happens here)
├── File Cabinet = Shared Pool
    (Everyone can access company documents)
├── Mailroom = Redo Log Buffer
    (All outgoing mail collected here)
└── Cafeteria = Large Pool & Java Pool
    (Special resources for special needs)
```

### Main Components of SGA

#### 1. Database Buffer Cache

**Purpose**: Stores copies of data blocks read from disk

**How It Works**:

```
Query: SELECT salary FROM employees WHERE id = 100;

Step 1: Is block with employee 100 in cache?
        → NO (first time accessing this record)

Step 2: Read from disk data file
        → Slow operation (milliseconds)

Step 3: Store in Database Buffer Cache
        → Block now in RAM

Step 4: Return data to user

Step 5: Another user queries the same employee
        → YES (found in cache)
        → Returns instantly (microseconds) ✓
```

**Real-World Analogy**:

```
Without cache:
Library user → Search entire library → Find book → 10 minutes

With cache (Database Buffer Cache):
Library → Popular books on front desk → User grabs it → 10 seconds
```

**Buffer States**:

```
1. Free Buffer
   ├── Empty space available
   └── Can be filled with new data blocks

2. Dirty Buffer
   ├── Contains modified data
   ├── Changes not yet written to disk
   └── Example: "Salary updated from 5000 to 6000"

3. Clean Buffer
   ├── Read-only data
   ├── No changes since read from disk
   └── Safe to discard or keep
```

**Performance Impact**:

```
Disk Read:        10 milliseconds (10,000 microseconds)
RAM Read:         0.1 microseconds
Speed Gain:       100X FASTER with caching
```

---

#### 2. Redo Log Buffer

**Purpose**: Records every change made to the database (for recovery)

**How It Works**:

```
User executes: UPDATE employees SET salary = 6000 WHERE id = 100;

Step 1: Change happens in Database Buffer Cache (in memory)
        ├── Employee 100 salary changed to 6000
        └── Block is now "DIRTY"

Step 2: BEFORE writing to disk, entry goes to Redo Log Buffer
        └── Entry: "At time 14:23:45, employee 100 salary = 6000"

Step 3: LGWR writes buffer to Redo Log Files (on disk)
        └── Now change is safe - even if crash happens

Step 4: User sees "Commit successful"

Step 5: Later, DBWn writes dirty block to data file
```

**Why This Order?**

```
If server crashes:
├── Change in buffer cache = LOST (in memory)
├── But entry in redo log file = SAVED (on disk)
├── On restart, SMON reads redo logs
└── Change is replayed and restored ✓

This is why redo logs are CRITICAL for safety
```

**Real-World Analogy**:

```
Redo Log Buffer = Safety net for database

Bank transaction:
├── Transfer money from account A to B
├── IMMEDIATELY write to transaction log (redo log)
├── Even if computer crashes, transaction log survived
├── When server restarts, replay transaction log
├── Money transfer is restored ✓
```

---

#### 3. Shared Pool

**Purpose**: Caches SQL statements and data dictionary information

**What It Contains**:

```
1. Library Cache
   ├── Stores parsed SQL statements
   ├── Stores execution plans
   ├── Example: SELECT * FROM employees; (already compiled)
   └── Saves time - don't need to parse again

2. Data Dictionary Cache
   ├── Table definitions
   ├── Column information
   ├── User privileges
   └── Constraints and indexes
```

**Example: SQL Parsing**

```
First User Executes: SELECT salary FROM employees;

Step 1: SQL Parser checks Shared Pool
        └── Not found

Step 2: Parse SQL statement (check syntax, validate)
        └── Takes CPU time

Step 3: Check data dictionary (verify table exists)
        └── Retrieve table structure

Step 4: Create execution plan
        └── Decide best way to retrieve data

Step 5: Store in Shared Pool (Library Cache)
        └── Compiled SQL + plan saved

Total Time: 50 milliseconds


Second User Executes: SELECT salary FROM employees;

Step 1: SQL Parser checks Shared Pool
        └── FOUND! ✓

Step 2: Reuse existing execution plan
        └── No parsing needed

Step 3: Execute immediately

Total Time: 1 millisecond (50X FASTER) ✓
```

**Real-World Analogy**:

```
Shared Pool = Restaurant Kitchen Prep Station

First customer orders: "Chicken Pasta"
├── Chef needs to read recipe
├── Gather ingredients
├── Prep and cook
└── Takes 30 minutes

Second customer orders: "Chicken Pasta"
├── Recipe already written on station wall
├── Ingredients already prepped
├── Chef cooks immediately
└── Takes 5 minutes
```

---

#### 4. Large Pool (Optional)

**Purpose**: Memory for large operations

**What Uses It**:

```
├── Backup operations (RMAN)
├── Parallel query execution
├── Recovery operations
└── Distributed transactions
```

**Why Separate?**

```
If large operation used Database Buffer Cache:
├── Large operation consumes tons of memory
├── Pushes useful data out of cache
└── Performance for regular queries suffers ✗

Large Pool reserved for big operations:
├── Regular queries don't interfere
├── Buffer cache stays focused on data blocks
└── Everything runs faster ✓
```

---

#### 5. Java Pool (Optional)

**Purpose**: Memory for Java code running inside database

**What It's Used For**:

```
├── Java stored procedures
├── Java functions
├── Java classes loaded in database
└── Java runtime environment
```

**Modern Usage**:

```
Less common now, as companies move Java to application servers
But some legacy systems still use it
```

---

## Program Global Area (PGA)

### What is the PGA?

The **PGA (Program Global Area)** is **private memory** allocated to each user session. Unlike the SGA, each user gets their own PGA - they cannot access other users' PGAs.

### Real-World Analogy: Personal Workspace

```
SGA = Company shared spaces (everyone can access)
PGA = Your personal desk (only you can use)

In a bank:
├── SGA = Teller lines, customer info system
    (All tellers and customers access it)
└── PGA = Each teller's workspace
    (Calculator, notepad, personal forms)
```

### Main Components of PGA

#### 1. Sort Area

**Purpose**: Memory for sorting operations

```
Query: SELECT * FROM employees ORDER BY salary DESC;

Process:
├── Database retrieves all employee records
├── Stores them in Sort Area (PGA memory)
├── Sorts by salary
└── Returns sorted results
```

**Why Not Use Disk?**

```
If sort area too small, spills to disk:
├── Sort Area in RAM (fast) = 1 second
├── Overflow to Temp Tablespace (disk) = 5 seconds
└── Total = 6 seconds (SLOW)

If sort area large enough:
├── Everything in RAM = 1 second (FAST) ✓
```

**Memory Allocation**:

```
Query: SELECT * FROM employees
       WHERE dept_id = 50
       ORDER BY salary DESC;

Without sort area: 0.5 MB (just fetch)
With sort area:    50 MB (buffer for sorting)

User with large sorts needs large sort area
User with simple queries needs small sort area
```

---

#### 2. Session Cursor Information

**Purpose**: Tracks open cursors in a session

```
What is a cursor?
├── Pointer to SQL query results
├── Keeps track of "current row"
├── Remembers WHERE you are in result set

Example:
SELECT employee_id, name FROM employees;

Result Set in Cursor:
Row 1: 100, John    ← Current position (cursor here)
Row 2: 101, Sarah
Row 3: 102, Mike
Row 4: 103, Lisa

PGA remembers:
├── Cursor is at Row 1
├── Result set has 4 rows
├── Next fetch will get Row 2
```

---

#### 3. Stack Space

**Purpose**: Memory for function calls and local variables

```
Stored Procedure Example:

CREATE PROCEDURE calculate_bonus
AS
    v_salary NUMBER;        ← Local variable (PGA)
    v_bonus NUMBER;         ← Local variable (PGA)
BEGIN
    SELECT salary INTO v_salary FROM employees WHERE id = 100;
    v_bonus := v_salary * 0.10;
    DBMS_OUTPUT.PUT_LINE('Bonus: ' || v_bonus);
END;

All these local variables stored in PGA Stack Space
```

---

#### 4. Private SQL Area

**Purpose**: Memory for SQL statement execution

```
Each user's SQL execution gets separate memory:

User1 executes: SELECT * FROM BIG_TABLE;
├── Execution context: 10 MB of PGA
├── Parses SQL
├── Creates execution plan
└── Stores in User1's Private SQL Area

User2 executes: SELECT * FROM BIG_TABLE;
├── Same SQL, but separate execution
├── Execution context: 10 MB of User2's PGA
├── Parses SQL
├── Creates execution plan
└── Stores in User2's Private SQL Area

Total PGA usage: 20 MB (separate for each user)
Total SGA usage: 0 MB (SGA isn't used here)
```

---

## SGA vs PGA Comparison

| Feature             | SGA                    | PGA                         |
| ------------------- | ---------------------- | --------------------------- |
| **Shared?**         | YES - all users        | NO - each user private      |
| **Size**            | Large (1-100 GB)       | Smaller (1-500 MB per user) |
| **When Allocated?** | Instance startup       | User login                  |
| **When Freed?**     | Instance shutdown      | User logout                 |
| **Accessible By**   | All processes          | Only that user's session    |
| **Used For**        | Data cache, shared SQL | Sorting, temp calculations  |
| **Backup Needed**   | NO                     | NO                          |
| **Example**         | Restaurant kitchen     | Chef's personal station     |

---

## Real-World Example: Bank Database

### System Setup

```
SGA Allocation:
├── Database Buffer Cache: 40 GB (hold customer data)
├── Redo Log Buffer: 100 MB (transaction log)
├── Shared Pool: 10 GB (SQL caching)
└── Total SGA: ~50 GB (shared by all users)

PGA Allocation:
├── Per user: 200 MB
├── 1000 concurrent users
└── Total PGA: ~200 GB (separate for each user)
```

### Customer Service Scenario

```
Time: 2:00 PM - Peak Hours

Customer 1: "What's my account balance?"
├── Connection created
├── PGA allocated (200 MB private workspace)
├── Query: SELECT balance FROM accounts WHERE id = 'C001'
├── Check SGA Buffer Cache
    └── Account data found (loaded hours ago)
    └── Returns FAST ✓
└── Result: $5,000

Customer 2: "Show me my last 100 transactions sorted by date"
├── Connection created
├── PGA allocated (200 MB private workspace)
├── Query: SELECT * FROM transactions WHERE id = 'C002'
          ORDER BY date DESC LIMIT 100
├── Uses PGA Sort Area (200 MB per user)
    └── Sorts 100 transactions
    └── Returns FAST ✓
└── Result: List of recent transactions

Customer 3: "Transfer $1000 to another account"
├── Connection created
├── PGA allocated (200 MB)
├── Update transactions in Database Buffer Cache (SGA)
├── Entry written to Redo Log Buffer (SGA)
    └── Immediately persisted to disk
├── Commit successful ✓
└── Money transferred
```

---

## Memory Tuning Tips

### How Much SGA?

```
Rule of thumb:
├── Small system:     2-4 GB SGA
├── Medium system:   10-20 GB SGA
├── Large system:    50-100 GB SGA
└── Enterprise:      100-500 GB SGA
```

### How Much PGA?

```
Rule of thumb:
├── Simple queries:     100 MB per user
├── Complex reports:    500 MB per user
├── Data warehouse:     1000 MB per user

Total = PGA per user × number of concurrent users
```

### Signs of Memory Problems

```
Symptom 1: "Queries are slow"
Cause: Too many buffer cache misses
Solution: Increase Database Buffer Cache in SGA

Symptom 2: "Sorting operations are slow"
Cause: Sort spilling to disk from PGA
Solution: Increase PGA_AGGREGATE_TARGET

Symptom 3: "SQL keeps re-parsing"
Cause: Not enough Shared Pool
Solution: Increase Shared Pool size
```

---

## Key Takeaways

1. **SGA is shared** - Everyone benefits from larger caches
2. **PGA is private** - Each user needs their own workspace
3. **Buffer cache speeds up reads** - Most important SGA component
4. **Redo log ensures safety** - Most critical for recovery
5. **Shared pool saves CPU** - SQL parsing is expensive
6. **Memory > Disk** - Always cache when possible
7. **Tuning memory** - Can improve performance 10-100X
