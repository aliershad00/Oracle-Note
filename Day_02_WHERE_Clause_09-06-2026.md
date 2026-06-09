# Oracle SQL: WHERE Clause and Conditions

## Date: 09-06-2026

### Overview
The WHERE clause is used to filter records based on specified conditions. It allows you to retrieve only the data that meets certain criteria.

### Syntax

```sql
SELECT column1, column2, ...
FROM table_name
WHERE condition;
```

### Comparison Operators

| Operator | Description |
|----------|-------------|
| = | Equal to |
| <> or != | Not equal to |
| > | Greater than |
| < | Less than |
| >= | Greater than or equal to |
| <= | Less than or equal to |

### Logical Operators

| Operator | Description |
|----------|-------------|
| AND | All conditions must be true |
| OR | At least one condition must be true |
| NOT | Negates a condition |

### Examples

#### Simple Comparison
```sql
SELECT employee_id, first_name, salary
FROM employees
WHERE salary > 5000;
```

#### Multiple Conditions (AND)
```sql
SELECT employee_id, first_name, salary
FROM employees
WHERE salary > 5000 
  AND department_id = 30;
```

#### Multiple Conditions (OR)
```sql
SELECT employee_id, first_name, department_id
FROM employees
WHERE department_id = 30 
   OR department_id = 50;
```

#### BETWEEN Operator
```sql
SELECT employee_id, first_name, salary
FROM employees
WHERE salary BETWEEN 3000 AND 7000;
```

#### IN Operator
```sql
SELECT employee_id, first_name, department_id
FROM employees
WHERE department_id IN (10, 20, 30);
```

#### LIKE Operator (Pattern Matching)
```sql
SELECT employee_id, first_name
FROM employees
WHERE first_name LIKE 'J%';
```

#### IS NULL / IS NOT NULL
```sql
SELECT employee_id, first_name, commission_pct
FROM employees
WHERE commission_pct IS NOT NULL;
```

### Best Practices

1. Use AND for multiple required conditions
2. Use OR sparingly to avoid complex queries
3. Always check for NULL values explicitly
4. Use BETWEEN for range queries
5. Use IN for multiple exact values

