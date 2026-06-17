# Oracle Architecture: Logical Storage - Tablespaces and Storage Structures

## Introduction: Abstraction Between Physical and Logical

Here's a key concept that confuses many: **Physical storage is not the same as logical storage**.

- **Physical**: The actual hard drive files (data files)
- **Logical**: How Oracle organizes space inside those files

This separation lets Oracle developers think logically without worrying about physical disk details.

### Real-World Analogy: Shopping Mall

```
Physical: Actual building structure
├── Building is 100,000 square feet
├── Built on one block
└── Specific address: 123 Main Street

Logical: Departments inside mall
├── Electronics Department
├── Clothing Department
├── Food Court
└── Each occupies specific area in building

Oracle:
├── Physical = Data files on disk
├── Logical = Tablespaces containing tables
```

---

## Tablespaces: The Highest Logical Tier

### What is a Tablespace?

A **Tablespace** is a **named logical storage area** that:

- Contains one or more data files
- Holds database objects (tables, indexes, etc.)
- Can grow by adding more data files
- Has its own management settings

### Tablespace and Data File Relationship

```
Important Concept:
├── 1 Tablespace = 1 or more Data Files
├── 1 Data File = 1 Tablespace (cannot span multiple)
├── Data files are platform-specific storage
├── Tablespaces are logical containers

Example:

Tablespace: USERS
├── Data File 1: /u01/oradata/USERS_01.dbf (5 GB)
├── Data File 2: /u02/oradata/USERS_02.dbf (5 GB)
├── Data File 3: /u03/oradata/USERS_03.dbf (5 GB)
└── Total capacity: 15 GB

When USERS tablespace fills up:
├── Add new data file: /u04/oradata/USERS_04.dbf (5 GB)
├── Now capacity: 20 GB
└── From user perspective: Still just "USERS tablespace"
```

### Standard Tablespaces in Oracle

```
1. SYSTEM Tablespace
   ├── Contains: Oracle system objects
   ├── Contains: Data dictionary
   ├── Contains: Redo log buffers pointers
   ├── Mandatory: Cannot drop
   ├── Size: 500 MB - 2 GB
   └── Backup: ESSENTIAL

2. SYSAUX Tablespace
   ├── Contains: Additional system objects
   ├── Contains: Statspack/AWR data
   ├── Contains: Audit data
   ├── Mandatory: Cannot drop
   ├── Size: 500 MB - 5 GB
   └── Backup: ESSENTIAL

3. USERS Tablespace
   ├── Contains: User-created objects
   ├── Contains: Customer data, business tables
   ├── Mandatory: Usually created (good practice)
   ├── Size: Depends on data (1 GB - 500+ GB)
   └── Backup: ESSENTIAL

4. TEMP Tablespace
   ├── Contains: Temporary segments
   ├── Contains: Sort areas, hash join areas
   ├── Not persistent: Data lost on instance restart
   ├── Dedicated: Only for temporary work
   ├── Size: 1 GB - 100 GB (depends on queries)
   └── Backup: Not needed (temporary)

5. UNDO Tablespace
   ├── Contains: Undo data (old values)
   ├── Contains: Rollback capability
   ├── Not user data: Only for rollback
   ├── Automatic: Size managed automatically
   ├── Size: 1 GB - 50 GB (depends on workload)
   └── Backup: Needed only for consistency
```

### Real-World Analogy: Filing System

```
Physical: File cabinet (data files)
├── Cabinet 1: 4 drawers, each 10 GB = 40 GB capacity
├── Cabinet 2: 3 drawers, each 10 GB = 30 GB capacity
└── Total: 70 GB of physical storage

Logical: Folders (tablespaces)
├── Folder: "Finance"
   ├── Stored in Cabinet 1, Drawers 1-2
   ├── Spreadsheets, receipts, invoices
   └── Can expand into more drawers

├── Folder: "HR"
   ├── Stored in Cabinet 1, Drawer 3 & Cabinet 2, Drawer 1
   ├── Employee records, benefits, payroll
   └── Can expand if needed

├── Folder: "Operations"
   ├── Stored in Cabinet 2, Drawers 2-3
   ├── Reports, schedules, logs
   └── Can expand

When Finance folder fills:
├── Add Cabinet 3, Drawer 1 to Finance
├── Expand capacity without reorg
└── User still sees "Finance" folder
```

---

## Storage Hierarchy: The Complete Picture

```
Database (largest container)
    ↓
Tablespaces (logical grouping)
    ├── SYSTEM (system objects)
    ├── USERS (user tables)
    ├── TEMP (temporary work)
    ├── UNDO (rollback data)
    └── CUSTOM (custom application)
        ↓
    Data Files (physical files on disk)
        ├── /u01/oradata/USERS_01.dbf
        ├── /u02/oradata/USERS_02.dbf
        └── /u03/oradata/UNDO_01.dbf
            ↓
        Segments (objects consuming space)
            ├── Segment 1: EMPLOYEES table
            ├── Segment 2: DEPARTMENTS table
            ├── Segment 3: INDEXES
            └── Segment 4: TEMP segment
                ↓
            Extents (storage allocation units)
                ├── Extent 1: 64 blocks
                ├── Extent 2: 64 blocks
                ├── Extent 3: 64 blocks
                └── Extent 4: (reserved for growth)
                    ↓
                Data Blocks (smallest unit)
                    ├── Block 1: 8 KB (standard)
                    ├── Block 2: 8 KB
                    ├── Block 3: 8 KB
                    └── 8000 blocks per extent
```

---

## Segments: Objects That Consume Space

### What is a Segment?

A **Segment** is **any database object that consumes storage space**.

### Types of Segments

```
1. Table Segment
   ├── Stores table data
   ├── Example: EMPLOYEES table segment
   ├── Grows when rows added
   └── Space: 100 MB - 100 GB+

2. Index Segment
   ├── Stores index data
   ├── Example: Index on EMPLOYEE_ID
   ├── Typically 20% of table size
   └── Space: Depends on indexed data

3. Temporary Segment
   ├── Stores temporary work
   ├── Created during: Sorts, hash joins
   ├── Deleted when: Query completes
   └── Space: Depends on query size

4. Undo Segment
   ├── Stores old values (for rollback)
   ├── Automatic creation
   ├── Keeps transactions consistent
   └── Space: Depends on transaction volume

5. Cluster Segment
   ├── Stores data from multiple tables
   ├── Physically co-locates related data
   ├── Advanced feature
   └── Space: Depends on cluster design

6. LOB Segment
   ├── Stores Large Objects (binary data, text)
   ├── Example: Image storage, document storage
   ├── Special handling for large data
   └── Space: Can be very large (GB+)
```

### Example: EMPLOYEES Table Segment

```
EMPLOYEES Table Segment = All storage space for this table

Contents:
├── Employee records (rows)
├── Row identifiers (ROWIDs)
├── Free space for new rows
└── Extent list (which extents belong to this segment)

Visual:

Extent 1          Extent 2          Extent 3
64 blocks         64 blocks         64 blocks
(512 KB)          (512 KB)          (512 KB)

├── Block 1 - 8 blocks of employee rows
├── Block 9 - 8 blocks of employee rows
├── Block 17 - 8 blocks of employee rows
└── Block 57 - 6 blocks of employee rows + 2 blocks free

All extents together = EMPLOYEES segment
```

---

## Extents: Chunks of Consecutive Blocks

### What is an Extent?

An **Extent** is a **contiguous allocation of data blocks** - consecutive space on disk.

### Why Extents?

```
Without Extents (Bad):
├── File fragmented
├── Each block scattered across disk
├── Reading one table requires reading entire disk
├── Many disk head movements
└── SLOW ✗

With Extents (Good):
├── Consecutive blocks allocated together
├── Reading table requires few disk head movements
├── Efficient sequential access
└── FAST ✓
```

### Extent Allocation

```
Table Creation:

USERS Tablespace
├── Data File: USERS_01.dbf

Create: EMPLOYEES table

Oracle Action:
├── Step 1: Find free space in USERS_01.dbf
├── Step 2: Allocate Extent 1 (64 consecutive blocks)
    └── Blocks 100-163 (reserved)
├── Step 3: Table is created and ready
└── Extent 1 assigned to EMPLOYEES segment

Insert Rows:

Insert 1000 employees
├── Fills Extent 1 (64 blocks, 512 KB)
├── Table growing, needs more space
└── Oracle allocates Extent 2

Oracle Action:
├── Step 1: Find next free space
├── Step 2: Allocate Extent 2 (64 consecutive blocks)
    └── Blocks 200-263 (reserved)
├── Step 3: Continue inserting
└── Extent 2 assigned to EMPLOYEES segment

Result:
├── EMPLOYEES segment = Extent 1 + Extent 2 + ...
├── Can allocate more extents as needed
└── Growth is automatic (if autoextend enabled)
```

### Extent Size

```
Different Extent Sizes for Different Purposes:

Small Table (10 MB):
├── Extent size: 64 KB each
├── Number of extents: ~156
└── Memory overhead: Low

Medium Table (100 MB):
├── Extent size: 1 MB each
├── Number of extents: ~100
└── Memory overhead: Medium

Large Table (1 GB):
├── Extent size: 10 MB each
├── Number of extents: ~100
└── Memory overhead: Reasonable

Huge Table (100 GB):
├── Extent size: 100 MB each
├── Number of extents: ~1000
└── Memory overhead: Significant

Best Practice:
├── Start small extents for growth
├── Increase extent size as table grows
├── Avoid excessive fragmentation
```

---

## Data Blocks: The Smallest Unit

### What is a Data Block?

A **Data Block** is the **smallest unit of storage** that Oracle reads from disk.

### Block Characteristics

```
Standard Oracle Block Size: 8 KB

Contains:
├── Header
│   ├── Block address
│   ├── Table information
│   ├── Free space information
│   └── Transaction info
├── Data Area
│   ├── Row data
│   └── Row headers
└── Free Space
    └── Space for new rows

One Row:

+---------+---------+---------+
| Header  | Row 1   | Row 2   | Free
| (80 B)  | (500 B) | (450 B) | Space
|         |         |         | (7 KB)
+---------+---------+---------+
        8 KB Block
```

### Why 8 KB?

```
Considerations:

Too Small (1 KB):
├── Many blocks needed for one table
├── More memory to track blocks
├── More disk I/O
└── SLOW ✗

Optimal (8 KB):
├── Good balance
├── Not too much memory overhead
├── Efficient disk I/O
├── Industry standard
└── Good performance ✓

Too Large (64 KB):
├── Waste space on small rows
├── Example: 1 KB row in 64 KB block = 63 KB wasted
├── More blocks needed per query
└── Less efficient
```

### Block Reading Example

```
Query: SELECT name FROM employees WHERE emp_id = 100;

Step 1: Oracle finds employee in block 523
Step 2: Oracle reads ENTIRE block 523 (8 KB) from disk
Step 3: Searches within block for emp_id = 100
Step 4: Returns name to user

Note: Even if row is 500 bytes, entire 8 KB block read
      That's the smallest read size Oracle can do
      Reason: Block is the atomic unit of storage
```

---

## Real-World Example: E-commerce Database

### Structure

```
CUSTOMERS Tablespace:
├── Data Files:
│   ├── /u01/customers_01.dbf (10 GB)
│   ├── /u02/customers_02.dbf (10 GB)
│   └── /u03/customers_03.dbf (5 GB)
│   └── Total Capacity: 25 GB

├── Segments:
│   ├── CUSTOMERS table segment (8 GB)
│   │   ├── Extent 1: blocks 0-63
│   │   ├── Extent 2: blocks 64-127
│   │   └── ... (many more extents)
│   │
│   ├── ORDERS table segment (6 GB)
│   │   ├── Extent 1: blocks 1000-1063
│   │   └── ... (many more extents)
│   │
│   └── idx_customer_id segment (2 GB)
│       ├── Extent 1: blocks 5000-5063
│       └── ... (more extents)
```

### Query Execution

```
Query: SELECT * FROM customers WHERE id = 12345;

Step 1: Find customer ID 12345
        └── Check index idx_customer_id (in idx segment)

Step 2: Index returns: "Customer is in block 1050"

Step 3: Oracle reads block 1050 (8 KB) from disk
        ├── Block is part of Extent 16 of CUSTOMERS segment
        ├── Extent 16 is in data file CUSTOMERS_01.dbf
        └── File located at: /u01/customers_01.dbf

Step 4: Find customer 12345 in block 1050
        └── Returns all columns for that customer

Total I/O: 2 disk reads
├── Read 1: Index block (8 KB)
└── Read 2: Data block (8 KB)
```

---

## Automatic Extent Allocation

### AUTOEXTEND Feature

```
Without AUTOEXTEND:
├── Tablespace fills to capacity
├── INSERT fails with: "ORA-01653: unable to extend table"
├── DBA must manually add data files
├── Application down until DBA responds ✗

With AUTOEXTEND:
├── Tablespace nearly full
├── Oracle automatically allocates next extent
├── Table continues growing
├── No error, no downtime ✓
```

### Setting AUTOEXTEND

```
Create Tablespace:
CREATE TABLESPACE USERS
  DATAFILE '/u01/users_01.dbf' SIZE 10G
  AUTOEXTEND ON NEXT 100M MAXSIZE 50G;

Meaning:
├── Initial size: 10 GB
├── Auto extend: Enabled
├── Growth increment: 100 MB at a time
├── Max size: 50 GB (prevents runaway growth)

Add Data File:
ALTER TABLESPACE USERS ADD DATAFILE
  '/u02/users_02.dbf' SIZE 10G
  AUTOEXTEND ON NEXT 100M MAXSIZE 50G;
```

---

## Monitoring Tablespace Usage

```sql
-- View all tablespaces
SELECT TABLESPACE_NAME, STATUS, EXTENT_MANAGEMENT
FROM DBA_TABLESPACES;

-- View datafiles in tablespace
SELECT FILE#, FILE_NAME, BYTES/1024/1024 AS SIZE_MB, STATUS
FROM DBA_DATA_FILES
WHERE TABLESPACE_NAME = 'USERS';

-- View free space
SELECT TABLESPACE_NAME, SUM(BYTES)/1024/1024 AS FREE_MB
FROM DBA_FREE_SPACE
GROUP BY TABLESPACE_NAME;

-- View segment sizes
SELECT OWNER, SEGMENT_NAME, SEGMENT_TYPE, BYTES/1024/1024 AS SIZE_MB
FROM DBA_SEGMENTS
ORDER BY BYTES DESC;

-- Check autoextend status
SELECT FILE_NAME, AUTOEXTENSIBLE, MAXBYTES/1024/1024 AS MAX_MB
FROM DBA_DATA_FILES;
```

---

## Key Takeaways

1. **Tablespace** = Logical container for related objects
2. **Data File** = Physical file holding tablespace data
3. **Segment** = Any object consuming storage (table, index, etc.)
4. **Extent** = Chunk of consecutive blocks allocated together
5. **Data Block** = Smallest unit (typically 8 KB)
6. **One-to-many**: 1 tablespace → many data files
7. **Autogrowth** = Automatic extent allocation
8. **Separation of concerns** = Physical files separate from logical objects
9. **Monitoring important** = Prevent "out of space" errors
10. **Planning needed** = Size tablespaces appropriately
