# Oracle SQL: Window Functions

## Date: 14-06-2026

### Overview
Window functions perform calculations across a set of rows related to the current row within a partition. They are powerful for analytical queries and trend analysis.

### Syntax

```sql
SELECT column_name,
       window_function() OVER (
           [PARTITION BY column_name]
           [ORDER BY column_name [ASC|DESC]]
           [ROWS BETWEEN start AND end]
       ) AS result
FROM table_name;
```

### Common Window Functions

| Function | Description |
|----------|-------------|
| ROW_NUMBER() | Assigns sequential numbers to rows |
| RANK() | Assigns rank with gaps for ties |
| DENSE_RANK() | Assigns rank without gaps |
| LAG() | Accesses data from previous row |
| LEAD() | Accesses data from next row |
| SUM() | Running total within partition |
| AVG() | Average within partition |
| COUNT() | Count within partition |
| FIRST_VALUE() | First value in window |
| LAST_VALUE() | Last value in window |

### Examples

#### ROW_NUMBER()
```sql
-- Rank employees by salary within each department
SELECT 
    employee_id,
    first_name,
    department_id,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rank
FROM employees;
```

#### RANK() vs DENSE_RANK()
```sql
-- Salary ranking (RANK allows gaps, DENSE_RANK doesn't)
SELECT 
    employee_id,
    first_name,
    salary,
    RANK() OVER (ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;
```

#### LAG() and LEAD()
```sql
-- Compare current and previous year sales
SELECT 
    month,
    sales,
    LAG(sales) OVER (ORDER BY month) AS prev_month_sales,
    LEAD(sales) OVER (ORDER BY month) AS next_month_sales,
    sales - LAG(sales) OVER (ORDER BY month) AS sales_change
FROM monthly_sales
ORDER BY month;
```

#### SUM() Window Function (Running Total)
```sql
-- Running total of salary by department, ordered by hire date
SELECT 
    employee_id,
    first_name,
    hire_date,
    salary,
    SUM(salary) OVER (
        PARTITION BY department_id 
        ORDER BY hire_date
    ) AS running_total
FROM employees;
```

#### FIRST_VALUE() and LAST_VALUE()
```sql
-- Compare each employee salary with highest in department
SELECT 
    employee_id,
    first_name,
    department_id,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department_id 
        ORDER BY salary DESC
    ) AS highest_salary,
    LAST_VALUE(salary) OVER (
        PARTITION BY department_id 
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lowest_salary
FROM employees;
```

#### AVG() Window Function
```sql
-- Each employee's salary vs department average
SELECT 
    employee_id,
    first_name,
    department_id,
    salary,
    AVG(salary) OVER (PARTITION BY department_id) AS dept_avg_salary,
    salary - AVG(salary) OVER (PARTITION BY department_id) AS variance_from_avg
FROM employees;
```

### ROWS and RANGE Frames

```sql
-- Calculate 3-month moving average
SELECT 
    sales_date,
    daily_sales,
    AVG(daily_sales) OVER (
        ORDER BY sales_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3days
FROM daily_sales;
```

### Best Practices

1. Window functions are evaluated AFTER WHERE clause
2. Use PARTITION BY to divide data into groups
3. Use ORDER BY within window for meaningful results
4. Specify ROWS frame for aggregate window functions
5. Test performance with large datasets
6. Use CTE (Common Table Expressions) for complex window queries

