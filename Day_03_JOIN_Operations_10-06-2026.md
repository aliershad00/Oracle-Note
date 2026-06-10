# Oracle SQL: JOIN Operations

## Date: 10-06-2026

### Overview
JOINs are used to combine rows from two or more tables based on related columns. They are essential for querying data across multiple tables in a relational database.

### Types of JOINs

#### 1. INNER JOIN
Returns only rows where there is a match in both tables.

```sql
SELECT e.employee_id, e.first_name, d.department_name
FROM employees e
INNER JOIN departments d
  ON e.department_id = d.department_id;
```

#### 2. LEFT (OUTER) JOIN
Returns all rows from the left table, and the matching rows from the right table.

```sql
SELECT e.employee_id, e.first_name, d.department_name
FROM employees e
LEFT JOIN departments d
  ON e.department_id = d.department_id;
```

#### 3. RIGHT (OUTER) JOIN
Returns all rows from the right table, and the matching rows from the left table.

```sql
SELECT e.employee_id, e.first_name, d.department_name
FROM employees e
RIGHT JOIN departments d
  ON e.department_id = d.department_id;
```

#### 4. FULL OUTER JOIN
Returns all rows from both tables.

```sql
SELECT e.employee_id, e.first_name, d.department_name
FROM employees e
FULL OUTER JOIN departments d
  ON e.department_id = d.department_id;
```

#### 5. CROSS JOIN
Returns the Cartesian product of two tables.

```sql
SELECT e.employee_id, p.product_id
FROM employees e
CROSS JOIN products p;
```

### Joining Multiple Tables

```sql
SELECT 
    e.employee_id,
    e.first_name,
    d.department_name,
    l.city,
    l.state_province
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id
INNER JOIN locations l ON d.location_id = l.location_id;
```

### Self Join Example

```sql
SELECT 
    e.employee_id,
    e.first_name AS employee_name,
    m.employee_id AS manager_id,
    m.first_name AS manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id;
```

### Best Practices

1. Use explicit JOIN syntax (INNER, LEFT, RIGHT)
2. Always specify the join condition in ON clause
3. Use table aliases for better readability
4. Avoid unnecessary FULL OUTER JOINs
5. Test with sample data to verify results

