# Oracle SQL: Aggregate Functions

## Date: 11-06-2026

### Overview
Aggregate functions perform calculations on a set of values and return a single result. They are commonly used to summarize data from multiple rows.

### Common Aggregate Functions

| Function | Description |
|----------|-------------|
| COUNT() | Counts the number of rows |
| SUM() | Calculates the sum of values |
| AVG() | Calculates the average of values |
| MIN() | Returns the minimum value |
| MAX() | Returns the maximum value |
| STDDEV() | Calculates standard deviation |
| VARIANCE() | Calculates variance |

### Syntax

```sql
SELECT aggregate_function(column_name)
FROM table_name
[WHERE condition];
```

### Examples

#### COUNT Function
```sql
-- Count total employees
SELECT COUNT(*) AS total_employees
FROM employees;

-- Count employees with commission
SELECT COUNT(commission_pct) AS employees_with_commission
FROM employees;
```

#### SUM Function
```sql
-- Calculate total salary expense
SELECT SUM(salary) AS total_salary_expense
FROM employees;

-- Sum sales for a specific product
SELECT SUM(quantity_sold) AS total_quantity
FROM sales
WHERE product_id = 101;
```

#### AVG Function
```sql
-- Calculate average salary
SELECT AVG(salary) AS average_salary
FROM employees;

-- Average salary by department
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id;
```

#### MIN and MAX Functions
```sql
-- Find salary range
SELECT 
    MIN(salary) AS minimum_salary,
    MAX(salary) AS maximum_salary
FROM employees;

-- Latest and earliest hire dates
SELECT 
    MIN(hire_date) AS earliest_hire,
    MAX(hire_date) AS latest_hire
FROM employees;
```

#### Multiple Aggregate Functions
```sql
SELECT 
    COUNT(*) AS total_employees,
    SUM(salary) AS total_salary,
    AVG(salary) AS average_salary,
    MIN(salary) AS minimum_salary,
    MAX(salary) AS maximum_salary
FROM employees
WHERE department_id = 50;
```

### Important Notes

- Aggregate functions ignore NULL values (except COUNT(*))
- You cannot use non-aggregated columns in SELECT with aggregates unless using GROUP BY
- Aggregate functions work with numeric, date, and text data types

### Best Practices

1. Use COUNT(*) to count all rows including NULLs
2. Use COUNT(column) to count non-NULL values
3. Always provide meaningful aliases for aggregate results
4. Test with NULL values to ensure expected behavior

