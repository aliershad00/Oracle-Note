# Oracle SQL: Subqueries

## Date: 13-06-2026

### Overview
A subquery (also called inner query) is a query within another SQL query. Subqueries are useful for breaking complex problems into smaller, manageable pieces.

### Types of Subqueries

#### 1. Scalar Subquery
Returns a single row and single column.

```sql
-- Find employees earning more than average salary
SELECT employee_id, first_name, salary
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
);
```

#### 2. Row Subquery
Returns a single row with multiple columns.

```sql
-- Find employees with the same salary and department as employee 101
SELECT employee_id, first_name, salary, department_id
FROM employees
WHERE (salary, department_id) = (
    SELECT salary, department_id
    FROM employees
    WHERE employee_id = 101
);
```

#### 3. Table Subquery
Returns multiple rows and columns.

```sql
-- Find employees in departments located in a specific city
SELECT employee_id, first_name, department_id
FROM employees
WHERE department_id IN (
    SELECT department_id
    FROM departments
    WHERE location_id IN (
        SELECT location_id
        FROM locations
        WHERE city = 'New York'
    )
);
```

### Subquery in WHERE Clause

```sql
-- Employees from departments with average salary > 5000
SELECT employee_id, first_name, salary
FROM employees
WHERE department_id IN (
    SELECT department_id
    FROM employees
    GROUP BY department_id
    HAVING AVG(salary) > 5000
);
```

### Subquery in FROM Clause (Derived Table)

```sql
-- Get departments with employee counts
SELECT 
    department_id,
    employee_count,
    avg_salary
FROM (
    SELECT 
        department_id,
        COUNT(*) AS employee_count,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
) dept_stats
WHERE employee_count > 10;
```

### Correlated Subquery

A subquery that references columns from the outer query.

```sql
-- Find employees earning more than their department average
SELECT e.employee_id, e.first_name, e.salary
FROM employees e
WHERE e.salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE department_id = e.department_id
);
```

### EXISTS Operator

```sql
-- Find departments that have employees
SELECT department_id, department_name
FROM departments d
WHERE EXISTS (
    SELECT 1
    FROM employees e
    WHERE e.department_id = d.department_id
);
```

### Best Practices

1. Use subqueries to improve readability
2. Prefer JOINs over subqueries when possible (usually faster)
3. Keep subqueries simple and focused
4. Use meaningful aliases for derived tables
5. Test performance with large datasets
6. Use EXISTS instead of IN for better performance with large result sets

