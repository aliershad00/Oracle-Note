# Oracle Architecture: Backup and Recovery Concepts - Protecting Your Data

## Introduction: Why Backups Matter

Disasters happen:

- Hard drive fails
- Data center floods
- Ransomware attacks
- Accidental deletion
- Corruption

Without backups, your data is gone forever. Oracle provides multiple recovery strategies to protect against all these scenarios.

---

## Why Backups Are Critical

### Real-World Cost Analysis

```
Scenario: Database Outage Analysis

Company: Online Banking (500,000 customers)
Database Size: 2 TB
Daily Transaction Value: $10 billion

Outage Duration: 1 hour (without backups)
├── Transactions lost: $400 million
├── Customer trust: Severely damaged
├── Regulatory fines: $10-50 million
├── Total cost: $450+ million

With Backups (RPO = 15 minutes):
├── Data lost: $100 million (worst case)
├── Recovery time: 30 minutes
├── No regulatory violation
├── Total cost: $100-150 million

Backup Cost vs Recovery Cost:
├── Annual backup investment: $1-5 million
├── Offsite storage: $500K - $2 million
├── Total: $1.5-7 million per year
├── Potential loss avoided: $450+ million
└── ROI: 60X or better ✓
```

---

## Key Backup Concepts

### RPO and RTO

```
RPO = Recovery Point Objective
├── Definition: Maximum acceptable data loss
├── Question: "How much data can we afford to lose?"
├── Measured in: Minutes or hours
├── Example: "We accept 15 minutes of data loss"

If database fails at 14:30:
├── With RPO 15 minutes
├── Can recover to: 14:15 (at best)
├── Data lost: 14:15 to 14:30 (15 minutes)
├── Acceptable loss defined

RTO = Recovery Time Objective
├── Definition: Maximum acceptable downtime
├── Question: "How long can system be down?"
├── Measured in: Minutes or hours
├── Example: "System must be up in 30 minutes"

If database fails at 14:30:
├── With RTO 30 minutes
├── Must be recovered by: 15:00
├── If recovery takes 45 minutes
├── Violates RTO ✗

Business Impact:
├── Bank: RPO 1 hour, RTO 30 minutes
├── Healthcare: RPO 5 minutes, RTO 15 minutes
├── E-commerce: RPO 15 minutes, RTO 1 hour
└── Mission-critical: RPO < 1 minute, RTO < 5 minutes
```

### Two Recovery Strategies

```
Strategy 1: ARCHIVELOG Mode
├── Purpose: Protect committed transactions
├── Method: Backup + redo logs
├── Recovery: Point-in-time recovery
├── Data loss: Only uncommitted transactions
├── RPO achievable: 0 (if redo logs protected)
└── Use case: Production databases

Strategy 2: NOARCHIVELOG Mode
├── Purpose: Basic protection
├── Method: Backups only (no redo archiving)
├── Recovery: To last backup only
├── Data loss: Everything since last backup
├── RPO achievable: Whatever backup frequency
└── Use case: Development/test environments
```

---

## Backup Types

### 1. Full Database Backup

```
What: Complete copy of entire database
When: Scheduled (e.g., weekly)
Size: Very large (same as database)
Time: Hours (depending on size)

Example:
Database size: 1 TB
Backup time: 3 hours
Network bandwidth: 100 MB/sec

Use case: Foundation for recovery
├── Restore full DB from full backup
├── Apply redo logs if in ARCHIVELOG mode
└── Up-to-date database restored
```

### 2. Incremental Backup

```
What: Only changed blocks since last backup
Size: Much smaller than full backup
Time: Minutes to hours (much faster)

Example:
Full Backup: Day 1 = 1 TB
Incremental Day 2: Only 10 GB (changes only)
Incremental Day 3: Only 8 GB (changes only)
Total storage: 1 TB + 10 GB + 8 GB

Speed Comparison:
Full backup: 3 hours
Incremental: 15 minutes (12X faster) ✓

Use case:
├── Daily backups without full copy overhead
├── Combined with full backup for recovery
└── Saves time and storage
```

### 3. Differential Backup

```
What: Changes since last FULL backup only
Difference vs Incremental:

Incremental:
├── Backup 1 (Full): Blocks A, B, C, D
├── Backup 2 (Incremental): Blocks B, E (changes only)
├── Backup 3 (Incremental): Blocks C, F (changes only)
└── Total: 3 backups needed for recovery

Differential:
├── Backup 1 (Full): Blocks A, B, C, D
├── Backup 2 (Differential): Blocks B, C, E (changed since full)
├── Backup 3 (Differential): Blocks B, C, E, F (changed since full)
└── Need: Full + last Differential for recovery

Advantage:
└── Faster recovery (fewer backups to restore)

Disadvantage:
└── Later backups larger (cumulative)
```

### 4. Transaction Log Backup

```
What: Only redo log files since last backup
When: Frequent (every 5-30 minutes)
Size: Very small (50 MB - 1 GB)
Purpose: Point-in-time recovery

How It Works:
├── Full backup at 00:00
├── Log backup at 06:00 (archives logs 00:00-06:00)
├── Log backup at 12:00 (archives logs 06:00-12:00)
├── Log backup at 18:00 (archives logs 12:00-18:00)
├── Database fails at 15:45

Recovery:
├── Restore full backup from 00:00
├── Apply log backup from 06:00 (now at 06:00)
├── Apply log backup from 12:00 (now at 12:00)
├── Apply log backup from 18:00 (now at 18:00)
├── Apply individual redo entries until 15:45
└── Restored to exact moment before failure ✓
```

---

## Backup Location Strategies

### Local Backup

```
Backup stored on same data center:

Location: Different disk than production
├── Disk 1: Production database
├── Disk 2: Backup (same location)
├── Disk 3: Backup copy (same location)

Advantages:
├── Fast restore (local disk)
├── Simple management
└── Good for non-critical systems

Disadvantages:
├── Fire/flood destroys both production and backup ✗
├── Data center disaster = total loss ✗
├── Not recommended for critical systems

Use case:
└── Development/test databases only
```

### Remote Backup

```
Backup stored in different location:

Setup:
├── Disk 1: Production (New York data center)
├── Disk 2: Backup (same data center)
├── Network: Backup transmitted to remote site
├── Disk 3: Backup (Los Angeles data center)

Advantages:
├── Protected against regional disasters ✓
├── Fire/flood in NY doesn't affect LA backup ✓
├── Redundancy across geography ✓

Disadvantages:
├── Slower restore (network transfer)
├── More complex setup
├── Higher costs (multiple sites)

Use case:
└── Critical production databases ✓
```

### Cloud Backup

```
Modern Approach:

Setup:
├── Production: On-premises or cloud
├── Backup: Cloud storage (AWS, Azure, OCI)
├── Redundancy: Cloud provider handles
├── Access: Over internet/VPN

Advantages:
├── Infinite scalability
├── Automatic redundancy
├── Cost-effective (pay per GB)
├── Managed by cloud provider
├── Geo-distributed automatically

Disadvantages:
├── Network bandwidth requirements
├── Internet dependency
├── Potential latency

Use case:
└── Modern enterprise databases ✓
```

---

## RMAN - Oracle Recovery Manager

### What is RMAN?

**RMAN** is Oracle's **enterprise backup solution** that:

- Automates backups
- Manages backup catalogs
- Performs point-in-time recovery
- Verifies backup integrity
- Provides compression

### RMAN vs Cold Backup

```
Cold Backup (Manual):
├── Shutdown database
├── Copy data files to backup location
├── Startup database
├── Restart: Full downtime during backup

Time Requirement:
├── Database size: 1 TB
├── Backup time: 3 hours (full shutdown)
├── Users waiting: 3 hours ✗

RMAN Backup (Online):
├── Database stays running
├── RMAN reads data files while online
├── Users continue working
├── No downtime required ✓

Time Requirement:
├── Database size: 1 TB
├── Backup time: 3 hours (parallel execution)
├── User impact: None (background process)
└── Downtime: 0 hours ✓
```

### RMAN Features

```
1. Incremental Backups
   └── Only backup changed blocks

2. Compression
   ├── Reduce backup size by 50-80%
   └── Faster network transfer

3. Parallel Backup
   ├── Use multiple processes
   ├── 10 processes = 10X faster (approximately)

4. Backup Verification
   ├── Test backup integrity
   ├── Ensure restorable

5. Catalog Management
   ├── Track all backups
   ├── Know what backed up when

6. Automated Backup Script
   ├── Schedule daily backups
   ├── No manual intervention
```

### RMAN Example Commands

```sql
-- Configure backup settings
CONFIGURE BACKUP OPTIMIZATION ON;
CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT '/backup/rman_%d_%U';
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;

-- Take full backup
BACKUP DATABASE PLUS ARCHIVELOG;

-- Take incremental backup
BACKUP INCREMENTAL LEVEL 1 DATABASE;

-- Backup with compression
BACKUP AS COMPRESSED BACKUPSET DATABASE;

-- Parallel backup (4 processes)
CONFIGURE CHANNELS DEVICE TYPE DISK PARALLELISM 4;
BACKUP DATABASE;

-- List backups
LIST BACKUP;

-- Verify backup integrity
BACKUP VALIDATE CHECK LOGICAL DATABASE;

-- Restore and recover
RESTORE DATABASE;
RECOVER DATABASE;
ALTER DATABASE OPEN RESETLOGS;
```

---

## Recovery Scenarios

### Scenario 1: Single File Loss

```
Situation: Data file USERS_01.dbf corrupted

Error:
SELECT * FROM employees;
ORA-01578: ORACLE data block corrupted
ORA-01110: data file 4: '/u01/users_01.dbf'

Recovery Steps:

Step 1: Take offline corrupted file
ALTER DATABASE DATAFILE '/u01/users_01.dbf' OFFLINE;

Step 2: Restore from backup
RESTORE DATAFILE '/u01/users_01.dbf';

Step 3: Apply redo logs
RECOVER DATAFILE '/u01/users_01.dbf';

Step 4: Bring online
ALTER DATABASE DATAFILE '/u01/users_01.dbf' ONLINE;

Step 5: Verify
SELECT * FROM employees;  ✓

Result:
├── Downtime: 5-10 minutes
├── Data loss: 0 (protected by redo logs)
├── Users affected: Only that tablespace initially
└── Recovery complete ✓
```

### Scenario 2: Complete Database Loss

```
Situation: Hardware failure destroys all data

Symptoms:
├── No data files accessible
├── Database won't start
├── All tables gone

Recovery Steps:

Step 1: Replace hardware / restore storage

Step 2: Restore all data files
RMAN> RESTORE DATABASE;

Step 3: Recover entire database
RMAN> RECOVER DATABASE;

Step 4: Open database (resetlogs)
ALTER DATABASE OPEN RESETLOGS;

Step 5: Verify recovery
SELECT COUNT(*) FROM employees;  ✓

Time Requirement:
├── Database size: 1 TB
├── Restore time: 2 hours
├── Recovery time: 1 hour
├── Total: 3 hours
├── Downtime: 3 hours

Data Loss:
├── All committed transactions: RECOVERED ✓
├── Uncommitted transactions: LOST ✗
└── Acceptable for most businesses
```

### Scenario 3: Accidental Deletion

```
Situation: User accidentally deletes important table

Command executed:
DROP TABLE customers;  ← Oops!

Problem:
├── Table gone
├── Application broken
├── Need to recover urgently

Solution: Flashback

Step 1: Identify drop time
SELECT DROPTIME FROM RECYCLEBIN WHERE ORIGINAL_NAME = 'CUSTOMERS';
└── Result: 14:30:00

Step 2: Check redo logs
SHOW PARAMETER log_archive_dest_1;
├── Redo logs available: 14:00 to 14:45
└── Covers the drop time ✓

Step 3: Recover using flashback
FLASHBACK DATABASE TO TIMESTAMP (SYSDATE - INTERVAL '2' HOUR);
ALTER DATABASE OPEN RESETLOGS;

Step 4: Export the table
EXPDP ... TABLE=CUSTOMERS ...;

Step 5: Restore database to current time
RECOVER DATABASE USING BACKUP CONTROLFILE UNTIL CANCEL;
ALTER DATABASE OPEN RESETLOGS;

Step 6: Import recovered data
IMPDP ... TABLE=CUSTOMERS ...;

Result:
├── Table recovered from before delete ✓
├── Application restored ✓
├── Downtime: 30-60 minutes
└── Crisis averted ✓
```

---

## Backup Best Practices

### The 3-2-1 Rule

```
Backup Rule: 3-2-1 Strategy

Meaning:
├── 3 = Keep 3 copies of data
├── 2 = Use 2 different storage media
├── 1 = Keep 1 offsite copy

Implementation:

Copy 1: Production database (live)
├── Location: Primary data center
├── Media: SAN storage

Copy 2: Backup (local)
├── Location: Same data center
├── Media: Different disk/array

Copy 3: Backup (remote)
├── Location: Different geographic area
├── Media: Cloud or offsite vault

Example Disaster Scenario:
├── Scenario 1: Disk fails
    └── Still have: Backup + Remote backup ✓
├── Scenario 2: Data center fire
    └── Still have: Remote backup ✓
├── Scenario 3: Corrupted data
    └── Backup from yesterday: Available ✓
└── Result: Always protected ✓
```

### Backup Frequency

```
General Guidelines:

Development Database:
├── Backup: Weekly
├── RPO: Acceptable to lose one week
├── RTO: Not critical
└── Low priority

Test Database:
├── Backup: Weekly or never
├── RPO: Can recreate from code
├── RTO: Not critical
└── Low priority

Production Database:
├── Full backup: Weekly or daily
├── Incremental: Daily
├── Redo logs: Every 5-30 minutes
├── RPO: < 1 hour required
├── RTO: < 1 hour required
└── High priority

Mission-Critical Database:
├── Full backup: Daily
├── Incremental: Every 6 hours
├── Redo logs: Every 5 minutes
├── Standby database: Real-time sync
├── RPO: < 5 minutes required
├── RTO: < 5 minutes required
└── Critical priority
```

### Testing Backups

```
Never Assume Backups Work!

Common Mistake:
├── Backup runs every night
├── Logs say "Backup successful"
├── Assume it's good
├── First true test: When needed ✗
├── Backup corrupted! ✗
└── Total data loss ✗

Best Practice:

Monthly Backup Verification:
├── Step 1: Select random backup
├── Step 2: Restore to test system
├── Step 3: Verify data integrity
├── Step 4: Run DBCC / consistency checks
├── Step 5: Document results
└── Result: Know backups work ✓

Scenario Testing (Annual):
├── Simulate different failure scenarios
├── Test recovery procedures
├── Time the recovery
├── Document lessons learned
└── Update recovery procedures if needed
```

---

## Disaster Recovery Planning

### RTO/RPO Matrix

```
Determine your requirements:

Business Function    | RPO     | RTO
─────────────────────┼─────────┼──────────
E-commerce site      | 1 min   | 5 min
Batch processing     | 1 hour  | 4 hours
Reporting DB         | 1 day   | 8 hours
Archival data        | 1 week  | 1 day

Implementation:

Fast RPO/RTO = Expensive
├── Real-time replication
├── Standby database
├── Automatic failover
└── Cost: High

Moderate RPO/RTO = Balanced
├── Daily backups
├── RMAN incremental
├── Warm standby
└── Cost: Medium

Slow RPO/RTO = Budget
├── Weekly backups
├── Manual recovery
├── Cold standby
└── Cost: Low
```

---

## Real-World Example: Bank Recovery

```
Scenario: Production database corruption (16:00)

Setup:
├── Database: 500 GB
├── RPO requirement: 15 minutes
├── RTO requirement: 30 minutes
├── Backup: Hourly redo logs + daily full backup

Timeline:

16:00 - Corruption detected
├── Admin notified immediately

16:02 - Assessment
├── Determine: How much data lost?
├── Check: Latest backup timestamp
├── Latest backup: 15:45 (15 minutes old)

16:05 - Start recovery (using RMAN)
├── Restore latest backup (15:45)
├── Apply redo logs: 15:45 to 16:00
├── Estimated time: 20 minutes

16:25 - Recovery complete
├── Database opened
├── Consistency check passed

16:28 - Verification
├── Test queries
├── Verify balances
├── Application restarted

16:30 - Users reconnected
├── Total downtime: 30 minutes
├── RPO achieved: 15 minutes (acceptable)
├── RTO achieved: 30 minutes (on target)

Result:
├── Transactions 15:45-16:00 might be missing (15 min window)
├── Acceptable loss vs alternative (total loss)
├── Customer impact: Minimal
└── Business continuity achieved ✓
```

---

## Key Takeaways

1. **RPO** = Maximum acceptable data loss
2. **RTO** = Maximum acceptable downtime
3. **ARCHIVELOG** = Protection for committed transactions
4. **RMAN** = Enterprise backup solution
5. **3-2-1 Rule** = 3 copies, 2 media, 1 offsite
6. **Incremental** = Only backup changed blocks
7. **Full backup** = Foundation for recovery
8. **Test backups** = Verify they actually work
9. **Disaster plan** = Documented procedures
10. **Regular testing** = Practice before disaster strikes
