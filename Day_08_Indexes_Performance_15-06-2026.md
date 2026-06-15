# Oracle SQL: Indexes and Performance Tuning

## Date: 15-06-2026

### Overview
Indexes are database objects that improve query performance by enabling faster data retrieval. Understanding indexes is crucial for writing efficient SQL queries.

### Types of Indexes

#### 1. Single Column Index
```sql
-- Create index on employee_id
CREATE INDEX idx_emp_id ON employees(employee_id);
```

#### 2. Composite Index (Multiple Columns)
```sql
-- Create index on department_id and salary
CREATE INDEX idx_dept_salary ON employees(department_id, salary);
```

#### 3. Unique Index
```sql
-- Ensures uniqueness of employee email
CREATE UNIQUE INDEX idx_emp_email ON employees(email);
```

#### 4. Function-Based Index
```sql
-- Index on uppercase last name
CREATE INDEX idx_upper_lastname ON employees(UPPER(last_name));
```

### Creating Indexes

```sql
-- Basic syntax
CREATE INDEX index_name ON table_name(column_name);

-- With multiple columns
CREATE INDEX idx_complex ON employees(department_id, salary DESC, hire_date);
```

### Dropping Indexes

```sql
DROP INDEX idx_emp_id;
```

### Index Usage Guidelines

#### Indexes Help Performance When:
- Column is frequently used in WHERE clause
- Column is used in JOIN conditions
- Column is used in ORDER BY or GROUP BY
- Large tables with selective queries

```sql
-- Good candidate for index
SELECT * FROM employees
WHERE employee_id = 105;

-- Also good (composite index)
SELECT * FROM employees
WHERE department_id = 30 AND salary > 5000;
```

#### When Indexes Hurt Performance:
- Small tables (full scan may be faster)
- Columns with low selectivity (many duplicates)
- Frequent INSERT/UPDATE/DELETE operations
- Columns rarely used in queries

### Query Performance Analysis

```sql
-- EXPLAIN PLAN helps identify index usage
EXPLAIN PLAN FOR
SELECT employee_id, first_name, salary
FROM employees
WHERE employee_id = 105;

-- View the plan
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Index Maintenance

```sql
-- Rebuild fragmented index
ALTER INDEX idx_emp_id REBUILD;

-- Disable index during bulk operations
ALTER INDEX idx_emp_id UNUSABLE;

-- Enable index after operations
ALTER INDEX idx_emp_id REBUILD;
```

### Best Practices

1. **Index Selectively**: Index columns used frequently in WHERE, JOIN, and ORDER BY
2. **Avoid Over-Indexing**: Each index increases INSERT/UPDATE/DELETE cost
3. **Use Composite Indexes**: When queries filter on multiple columns
4. **Monitor Performance**: Regularly check for unused or duplicate indexes
5. **Consider Data Volatility**: Highly volatile tables may not benefit from indexes
6. **Test Before Production**: Always test index impact on actual queries

### Common Performance Issues and Solutions

#### Problem: Slow Queries
```sql
-- Solution: Add index on frequently queried columns
CREATE INDEX idx_sales_date ON sales(sale_date);
```

#### Problem: Slow JOINs
```sql
-- Solution: Index foreign key columns
CREATE INDEX idx_emp_dept_id ON employees(department_id);
```

#### Problem: Slow GROUP BY
```sql
-- Solution: Index columns in GROUP BY clause
CREATE INDEX idx_emp_dept_job ON employees(department_id, job_id);
```

### Performance Tuning Checklist

- ✓ Add indexes to columns in WHERE clauses
- ✓ Index foreign key columns
- ✓ Avoid unnecessary subqueries
- ✓ Use JOIN instead of IN with subqueries
- ✓ Limit result set with WHERE conditions
- ✓ Monitor and remove unused indexes
- ✓ Keep table statistics up to date

