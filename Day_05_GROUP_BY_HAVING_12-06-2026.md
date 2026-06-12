# Oracle SQL: GROUP BY and HAVING Clauses

## Date: 12-06-2026

### Overview
GROUP BY groups rows that have the same values, while HAVING filters groups based on conditions. Together, they allow sophisticated data summarization and analysis.

### GROUP BY Syntax

```sql
SELECT column1, aggregate_function(column2)
FROM table_name
[WHERE condition]
GROUP BY column1
[ORDER BY column1];
```

### HAVING Syntax

```sql
SELECT column1, aggregate_function(column2)
FROM table_name
[WHERE condition]
GROUP BY column1
HAVING aggregate_function(column2) condition
[ORDER BY column1];
```

### Key Differences: WHERE vs HAVING

- **WHERE**: Filters rows BEFORE grouping
- **HAVING**: Filters groups AFTER aggregation

### Examples

#### Basic GROUP BY
```sql
-- Total salary by department
SELECT 
    department_id,
    SUM(salary) AS total_salary
FROM employees
GROUP BY department_id;
```

#### GROUP BY with Multiple Columns
```sql
-- Employee count by department and job
SELECT 
    department_id,
    job_id,
    COUNT(*) AS employee_count
FROM employees
GROUP BY department_id, job_id
ORDER BY department_id, job_id;
```

#### WHERE with GROUP BY
```sql
-- Average salary by department for employees earning > 3000
SELECT 
    department_id,
    AVG(salary) AS average_salary
FROM employees
WHERE salary > 3000
GROUP BY department_id;
```

#### HAVING Clause
```sql
-- Departments with more than 5 employees
SELECT 
    department_id,
    COUNT(*) AS employee_count
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 5;
```

#### Complex GROUP BY with HAVING
```sql
-- Departments where average salary exceeds 5000
SELECT 
    department_id,
    COUNT(*) AS emp_count,
    AVG(salary) AS avg_salary,
    SUM(salary) AS total_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 5000
ORDER BY avg_salary DESC;
```

#### GROUP BY with Multiple Aggregate Functions
```sql
-- Sales summary by product category
SELECT 
    category_id,
    COUNT(*) AS total_sales,
    SUM(sale_amount) AS total_revenue,
    AVG(sale_amount) AS avg_sale,
    MIN(sale_amount) AS min_sale,
    MAX(sale_amount) AS max_sale
FROM sales
GROUP BY category_id
HAVING SUM(sale_amount) > 10000;
```

### Best Practices

1. All non-aggregated columns must be in GROUP BY
2. Use WHERE to filter rows before grouping
3. Use HAVING to filter groups after aggregation
4. Order GROUP BY columns in meaningful sequence
5. Use meaningful aliases for aggregate columns
6. Test with edge cases (empty groups, NULL values)

