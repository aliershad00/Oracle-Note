# Oracle SQL: SELECT Statement Basics

## Date: 08-06-2026

### Overview
The SELECT statement is the fundamental SQL command used to retrieve data from database tables. It forms the basis of all data queries in Oracle SQL.

### Syntax

```sql
SELECT [DISTINCT] column1, column2, ...
FROM table_name
[WHERE condition]
[ORDER BY column_name];
```

### Key Components

- **SELECT**: Specifies which columns to retrieve
- **DISTINCT**: Removes duplicate rows from results
- **FROM**: Specifies the table(s) to query
- **WHERE**: Filters records (optional)
- **ORDER BY**: Sorts results (optional)

### Examples

#### Basic SELECT
```sql
SELECT employee_id, first_name, last_name, salary
FROM employees;
```

#### SELECT with DISTINCT
```sql
SELECT DISTINCT department_id
FROM employees;
```

#### SELECT with Column Alias
```sql
SELECT 
    employee_id AS emp_id,
    first_name AS fname,
    salary * 12 AS annual_salary
FROM employees;
```

#### SELECT with Arithmetic Operations
```sql
SELECT 
    product_name,
    list_price,
    quantity_on_hand,
    (list_price * quantity_on_hand) AS total_inventory_value
FROM products;
```

### Best Practices

1. Always specify column names instead of using SELECT *
2. Use aliases for better readability
3. Use ORDER BY to arrange results meaningfully
4. Avoid unnecessary calculations in SELECT clause

### Practice Exercise
Query the EMPLOYEES table to get employee_id, first_name, and monthly salary (annual_salary/12) for all employees in department_id 50.

