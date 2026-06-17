# Oracle SQL: Transactions and ACID Properties

## Date: 16-06-2026

### Overview

Transactions are fundamental units of work in Oracle SQL that ensure data integrity through ACID properties. Understanding transactions is critical for building reliable database applications that maintain consistency even during failures or concurrent access.

### ACID Properties

#### A - Atomicity

All operations in a transaction succeed together, or all fail together. No partial updates.

#### C - Consistency

Database transitions from one valid state to another. All rules and constraints are maintained.

#### I - Isolation

Concurrent transactions don't interfere with each other. Different isolation levels control interaction.

#### D - Durability

Once committed, changes are permanent even if the system fails.

### Transaction Control Statements

#### COMMIT

```sql
-- Permanently saves all changes made in the transaction
COMMIT;
```

#### ROLLBACK

```sql
-- Undoes all changes made since the start of the transaction or a savepoint
ROLLBACK;

-- Rollback to a specific savepoint
ROLLBACK TO SAVEPOINT savepoint_name;
```

#### SAVEPOINT

```sql
-- Creates a named rollback point within a transaction
SAVEPOINT savepoint_name;
```

### Basic Transaction Syntax

```sql
START TRANSACTION;  -- Or just begin first statement
  -- DML statements (INSERT, UPDATE, DELETE)
COMMIT;             -- Or ROLLBACK;
```

### Examples

#### Simple Transaction with Commit

```sql
UPDATE employees
SET salary = salary * 1.10
WHERE department_id = 50;

UPDATE employees
SET commission_pct = 0.15
WHERE department_id = 50;

COMMIT;  -- All changes are saved
```

#### Transaction with Rollback

```sql
DELETE FROM employees
WHERE employee_id = 100;

SELECT COUNT(*) FROM employees;  -- Shows deleted row not counted

ROLLBACK;  -- Undo the delete

SELECT COUNT(*) FROM employees;  -- Employee 100 is back
```

#### Using SAVEPOINTS

```sql
UPDATE employees SET salary = 50000 WHERE department_id = 10;
SAVEPOINT sp1;

UPDATE employees SET salary = 60000 WHERE department_id = 20;
SAVEPOINT sp2;

UPDATE employees SET salary = 70000 WHERE department_id = 30;

-- If something goes wrong:
ROLLBACK TO SAVEPOINT sp2;  -- Undo only the last update
COMMIT;  -- Save first two updates
```

#### Transfer Transaction (Real-world Example)

```sql
-- Debit one account
UPDATE accounts
SET balance = balance - 500
WHERE account_id = 'ACC001';

-- Credit another account
UPDATE accounts
SET balance = balance + 500
WHERE account_id = 'ACC002';

-- Both succeed or both fail
COMMIT;
```

### Isolation Levels

#### Read Uncommitted (Dirty Reads)

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- Allows reading uncommitted data from other transactions
```

#### Read Committed (Default in Oracle)

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Allows reading only committed data
-- Prevents dirty reads but allows non-repeatable reads
```

#### Repeatable Read

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- Prevents dirty and non-repeatable reads
-- Phantom reads are possible
```

#### Serializable

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Highest isolation level
-- Prevents all anomalies (dirty reads, non-repeatable reads, phantom reads)
```

### Implicit vs Explicit Transactions

#### Implicit (Autocommit - Default)

```sql
UPDATE employees SET salary = 5000;
-- Automatically committed (if autocommit is ON)
```

#### Explicit

```sql
-- Disable autocommit for explicit transaction control
SET AUTOCOMMIT OFF;

UPDATE employees SET salary = 5000;
COMMIT;  -- Explicit commit required

SET AUTOCOMMIT ON;  -- Re-enable autocommit
```

### Locking Mechanisms

#### Row-Level Lock

```sql
SELECT * FROM employees
WHERE employee_id = 100
FOR UPDATE;  -- Locks the row

UPDATE employees
SET salary = 5000
WHERE employee_id = 100;

COMMIT;  -- Release lock
```

#### Deadlock Prevention

```sql
-- Always lock resources in the same order to prevent deadlocks
-- Transaction 1: Lock A then B
-- Transaction 2: Lock A then B (not B then A)
```

### Best Practices

1. Keep transactions short and focused
2. Always handle both COMMIT and ROLLBACK scenarios
3. Use SAVEPOINTS for complex multi-step operations
4. Choose appropriate isolation levels based on requirements
5. Avoid locking tables during user interaction
6. Use exception handling to catch and handle errors
7. Monitor for deadlocks and optimize queries accordingly
8. Always ensure data consistency after rollbacks
9. Document complex transaction logic
10. Test transaction behavior under concurrent load

### Common Pitfalls

- Long-running transactions lock resources
- High isolation levels can reduce concurrency
- Forgetting COMMIT causes data loss
- Nested transactions without proper SAVEPOINT management
- Not handling rollback scenarios in application code
- Inconsistent transaction scope across similar operations

### Practice Exercise

Create a transaction that:

1. Decreases the salary of an employee in department_id 50 by 5%
2. Increases the commission for the same department by 1%
3. Use a SAVEPOINT after the first update
4. Verify the changes and COMMIT
5. Test by rolling back to the SAVEPOINT
