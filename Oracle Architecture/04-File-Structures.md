# Oracle Architecture: File Structures - Physical Storage Organization

## Introduction: What Gets Stored Where?

The Oracle **Database** is made up of physical files stored on the server's hard drive. Each file serves a specific purpose:

- **Data Files** = Store actual data
- **Control Files** = Store metadata about the database
- **Redo Log Files** = Store change history
- **Parameter Files** = Store configuration
- **Password File** = Store admin credentials

Think of it like a filing cabinet where each drawer holds specific types of documents.

---

## 1. Data Files (.dbf)

### Purpose

**Data Files store the actual data** - all the tables, indexes, sequences, and everything users care about.

### Key Characteristics

```
Size: Large (can be 1 GB to 100+ GB each)
Number: Multiple per database (usually 5-100)
Location: /u01/oradata/ or similar
Format: Binary format (not readable as text)
Permanent: YES - survives shutdowns
Backup: CRITICAL - must backup
```

### Internal Structure

```
Data File Structure:

Header Section
├── File identification
├── File version
├── Creation timestamp
└── Last checkpoint info

Data Blocks (typically 8 KB each)
├── Block 1: Contains employee table data
├── Block 2: Contains employee table data
├── Block 3: Contains department table data
├── Block 4: Empty space (can be allocated)
└── Block N: Contains various data
```

### How Data is Organized

```
Real Example: USERS Tablespace

USERS_01.dbf File (10 GB)
├── Blocks 0-1000: Employee table data
├── Blocks 1001-5000: Department table data
├── Blocks 5001-8000: Salary history table data
├── Blocks 8001-10000: Customer table data
└── Blocks 10001-end: Free space for growth

All these tables exist in the SAME file
But oracle tracks which blocks belong to which table
```

### Reading Process

```
Query: SELECT salary FROM employees WHERE emp_id = 100;

Step 1: Oracle searches Data Dictionary
        └── "Employee table is in USERS tablespace"

Step 2: Oracle consults Tablespace info
        └── "USERS tablespace uses file USERS_01.dbf"

Step 3: Oracle calculates block address
        └── "Employee block #523 has this data"

Step 4: Oracle reads from USERS_01.dbf, block #523
        ├── First check Database Buffer Cache (memory)
        ├── If not cached: Read from disk
        └── Return employee record

Result: Employee salary retrieved ✓
```

### Real-World Analogy

```
Data Files = Library Books

Each book = Data file (USERS_01.dbf)
Each page = Data block (8 KB)
Each row = Individual record in the page

Book contains:
├── Pages 1-50: History section (employee table)
├── Pages 51-100: Science section (department table)
├── Pages 101-150: Fiction section (customer table)
└── Pages 151+: Empty pages (for expansion)

To find "John's salary":
├── Find which section (data dictionary lookup)
├── Find which page (block calculation)
├── Find book (tablespace location)
└── Open book, go to page, find John's record
```

---

## 2. Control Files (.ctl)

### Purpose

**Control Files store critical metadata** about the database structure and state. The database CANNOT open without them.

### Key Characteristics

```
Size: Small (1-50 MB typically)
Number: Usually 3 copies (for redundancy)
Location: Different disks for safety (/u01/, /u02/, /u03/)
Format: Binary format
Permanent: YES - critical to backup
Backup: AUTOMATIC (usually)
```

### What Control Files Contain

```
Control File Contents:

1. Database Identification
   ├── Database name (PROD, DEV, TEST)
   ├── Database ID (DBID - unique identifier)
   └── Timestamp of creation

2. File Location Information
   ├── Data file locations and names
   ├── Redo log file locations
   └── Temp file locations

3. Checkpoint Information
   ├── Last checkpoint timestamp
   ├── Last checkpoint block in redo logs
   └── System change number (SCN)

4. Redo Log Information
   ├── Which redo log is current
   ├── Redo log history
   └── Redo log sequence numbers

5. Database State Information
   ├── Is database open? YES/NO
   ├── Was last shutdown clean? YES/NO
   └── Recovery needed? YES/NO
```

### Real-World Analogy

```
Control File = Library Card Catalog (old system)

Card Catalog contains:
├── "Book A is in Aisle 5, Shelf 3"
├── "Book B is in Aisle 7, Shelf 2"
├── "Book A was last updated today at 2 PM"
└── "Someone is currently reading Book A"

If card catalog is destroyed:
├── You can't find any books
├── You can't know which books exist
├── Library is unusable ✗

That's why Oracle keeps 3 copies of control files!
```

### Control File Redundancy

```
Oracle Setup (Best Practice):

Control File 1: /u01/oradata/PROD/control01.ctl
Control File 2: /u02/oradata/PROD/control02.ctl
Control File 3: /u03/oradata/PROD/control03.ctl

Why 3 copies?
├── Disk 1 fails: Still have copies 2 & 3
├── Disk 2 fails: Still have copies 1 & 3
├── Disk 3 fails: Still have copies 1 & 2
└── If all 3 fail: Database cannot open ✗

Oracle keeps them synchronized automatically
```

### Control File Information Queries

```sql
-- View control file locations
SHOW PARAMETER control_files;

-- View control file contents
SELECT * FROM V$DATABASE;
SELECT * FROM V$DATAFILE;
SELECT * FROM V$LOGFILE;
```

---

## 3. Redo Log Files (.log)

### Purpose

**Redo Log Files store complete history of all database changes** for recovery purposes.

### Key Characteristics

```
Size: Medium (100 MB - 5 GB each)
Number: Multiple groups (minimum 2)
Location: Multiple disks for safety
Format: Binary format (sequential writes)
Permanent: YES but cycled/overwritten
Backup: Needed if in ARCHIVELOG mode
```

### How Redo Logs Work

```
Redo Log Cycle:

Time 1:00 - Redo Log Group 1 (ACTIVE)
├── LGWR writes: "Update emp salary to 5000"
├── LGWR writes: "Insert new employee record"
├── LGWR writes: "Delete old customer"
└── When full: Switch to next group

Time 1:05 - Redo Log Group 2 (ACTIVE)
├── LGWR writes: "Update dept budget"
├── LGWR writes: "Create new table"
└── When full: Switch to next group

Time 1:10 - Redo Log Group 3 (ACTIVE)
├── Writing changes...
└── When full: Cycle back to Group 1

Time 1:15 - Back to Redo Log Group 1 (REUSE)
├── Old entries overwritten (no longer needed)
├── New changes written
└── Cycle continues...

Note: Redo log size and frequency affects recovery time
```

### Example Redo Log Entry

```
Redo Log Entry Format:

[14:23:45.123] Transaction ID: TX_12345
├── Operation: UPDATE
├── Table: EMPLOYEES
├── Column: SALARY
├── Old Value: 5000
├── New Value: 6000
├── Employee ID: 100
├── Timestamp: 2024-06-17 14:23:45
└── Status: COMMITTED ✓
```

### Redo Log Groups

```
Why Multiple Groups?

Scenario 1: Single Redo Log (Bad)
├── Redo Group 1 is ACTIVE
├── Changes being written to disk
├── New changes arrive while writing
└── Must WAIT for writes to finish ✗
└── Performance SLOW

Scenario 2: Multiple Redo Logs (Good)
├── Redo Group 1 is ACTIVE (writing changes)
├── Redo Group 2 is STANDBY (ready to use)
├── While Group 1 writes: new changes go to Group 2
├── No waiting ✓
└── Performance FAST
```

### Real-World Analogy

```
Redo Log = Flight Black Box

Black box records:
├── Every command from cockpit
├── Every system event
├── Every passenger action (metaphorically)
└── Complete history of flight

If plane crashes:
├── Investigators find black box
├── Read everything that happened
├── Understand cause of crash
└── Use data to prevent future crashes

Redo Log = Database black box:
├── Records everything that happened
├── If crash: Read redo log to recover
├── Restore database to exact state before crash
```

---

## 4. Parameter File (SPFILE or PFILE)

### Purpose

**Parameter File defines how the instance starts** - memory allocation, process count, etc.

### Key Characteristics

```
SPFILE: Server Parameter File
├── Binary format
├── Preferred modern approach
├── Can be modified while running
└── Changes take effect at next restart

PFILE: Plain text Parameter File
├── Text format (readable)
├── Legacy approach
├── Must be modified manually
└── Example: init.ora
```

### Common Parameters

```
# Memory Configuration
db_cache_size = 4G                    # Buffer cache size
shared_pool_size = 2G                # Shared pool size
sga_target = 8G                       # Total SGA size
pga_aggregate_target = 4G             # Total PGA allocation

# Process Configuration
processes = 500                        # Max concurrent processes
open_cursors = 1000                   # Max cursors per session

# File Locations
db_name = PROD                        # Database name
control_files = /u01/ctl1.ctl,...    # Control file locations

# Redo Log Configuration
redo_log_archive_dest_1 = 'LOCATION=/archive/'
```

### Example SPFILE Content

```
*.audit_trail='DB'
*.db_cache_size=4294967296
*.db_name='PROD'
*.open_cursors=1000
*.processes=500
*.sga_target=8589934592
*.shared_pool_size=2147483648

First asterisk (*) = parameter applies to all instances
If database has 1 instance, * is used
If database has multiple instances (RAC), can use instance names
```

### Real-World Analogy

```
SPFILE = Restaurant Operating Procedures Manual

Parameters in manual:
├── "Shift starts at 6 AM with 30 staff"
├── "Open 40 tables during lunch"
├── "Kitchen gets 4 hours prep time"
├── "Supplies ordered from Vendor ABC"
└── "Emergency procedures for power outage"

When restaurant reopens:
├── Manager reads procedures manual
├── Sets up according to parameters
├── Restaurant runs per parameters

If manual says "30 staff" but we hire 50:
├── Next day's shift still uses 30
├── Need to update manual for permanent change
```

---

## 5. Password File

### Purpose

**Password File allows administrative connections** even when database isn't fully open.

### Key Characteristics

```
Name: orapwd or password.ora
Size: Small (typically <10 KB)
Location: $ORACLE_HOME/dbs/
Format: Binary format (not text)
Contents: SYS and INTERNAL user credentials
Backup: Should backup
```

### When It's Used

```
Scenario 1: Normal Login
├── User connects with username/password
├── Database authenticated via data dictionary
└── Normal database login

Scenario 2: Administrative Login
├── DBA needs to connect to crashed database
├── Database not fully open yet
├── Data dictionary not accessible
├── Oracle checks password file
├── Authentication via password file ✓
└── DBA can perform recovery operations
```

### Real-World Analogy

```
Password File = Master Key Stored Safely

Normal Entry:
├── Enter building using regular key
├── Receptionist verifies in the registry
└── Allowed entry ✓

Emergency (Building broken):
├── Cannot access registry
├── But have master key in safe
├── Master key works anyway
└── Emergency personnel can enter ✓
```

---

## File Location Best Practices

### Typical File Layout

```
Production Oracle Database:

/u01/oradata/PROD/
├── datafile/
│   ├── USERS_01.dbf          (User data)
│   ├── SYSTEM_01.dbf         (System objects)
│   ├── TEMP_01.tmp           (Temporary space)
│   └── UNDO_01.dbf           (Undo data)
└── controlfile/
    └── control01.ctl

/u02/oradata/PROD/
├── control02.ctl              (Copy of control file)
└── logfile/
    ├── redo01.log            (Redo log group 1)
    └── redo02.log            (Redo log group 2)

/u03/oradata/PROD/
├── control03.ctl             (Copy of control file)
└── logfile/
    ├── redo03.log            (Redo log group 3)
    └── redo04.log            (Redo log group 4)

/archive/PROD/
├── 1_234567_89.arc           (Archived redo logs)
├── 1_234568_89.arc
└── 1_234569_89.arc

Principle: Spread files across different disks
```

### Why Spread Files?

```
Single Disk (Bad):
├── /disk1/datafile1.dbf
├── /disk1/controlfile.ctl
├── /disk1/redolog.log
├── Disk fails: EVERYTHING LOST ✗

Multiple Disks (Good):
├── /disk1/datafile1.dbf
├── /disk2/datafile2.dbf
├── /disk3/controlfile1.ctl (copy 1)
├── /disk4/controlfile2.ctl (copy 2)
├── /disk5/redolog1.log
├── /disk6/redolog2.log
├── One disk fails: Still have backups ✓
```

---

## File Size Estimation

### Data Files

```
How big will data files be?

Example: Insurance Company Database

Customers Table: 1M records × 500 bytes = 500 MB
Policies Table: 2M records × 1000 bytes = 2 GB
Claims Table: 5M records × 300 bytes = 1.5 GB
Transactions Table: 100M records × 200 bytes = 20 GB

Total data: ~24 GB

Add indexes (typically 20% of data): ~5 GB
Add growth margin (20% extra): ~6 GB

Total data files needed: ~35 GB
```

### Redo Log Files

```
How much redo log do you need?

Peak Activity: 1000 transactions/second
Average transaction size: 5 KB

Redo per second: 1000 × 5 KB = 5 MB/sec
Redo per minute: 5 MB × 60 = 300 MB/min
Redo per hour: 300 MB × 60 = 18 GB/hour

For a 1-hour recovery window:
Single redo log group: 18 GB
With 3 groups for safety: 3 × 18 GB = 54 GB total

Rule: Redo logs can be overwritten every 5-10 minutes
```

---

## File Monitoring Commands

```sql
-- View all data files
SELECT FILE#, NAME, BYTES/1024/1024 AS SIZE_MB, STATUS
FROM V$DATAFILE;

-- View control file locations
SHOW PARAMETER control_files;

-- View redo log file information
SELECT GROUP#, MEMBERS, SEQUENCE#, BYTES/1024/1024 AS SIZE_MB
FROM V$LOG;

-- Check free space in tablespaces
SELECT TABLESPACE_NAME, SUM(BYTES)/1024/1024 AS FREE_MB
FROM DBA_FREE_SPACE
GROUP BY TABLESPACE_NAME;

-- Monitor file I/O activity
SELECT PHYBLKRD, PHYBLKWRT FROM V$SYSSTAT;
```

---

## Key Takeaways

1. **Data Files** store actual data (largest files, backup critical)
2. **Control Files** store metadata (small, critical, need 3 copies)
3. **Redo Logs** store change history (for recovery, multiple groups)
4. **Parameter Files** define configuration (startup instructions)
5. **Password File** enables admin access (emergency connections)
6. **Spread files** across different disks (redundancy and performance)
7. **Monitor file growth** to prevent space issues
8. **Backup properly** (data files most critical)
