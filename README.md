# Advanced SQL Techniques

## 1. Common Table Expressions (CTEs)

### Definition
A **CTE (Common Table Expression)** is a temporary named result set used to simplify complex queries, especially those with subqueries.

### Syntax
```sql
WITH cte_name (optional_column_names) AS (
    SELECT ...
    FROM ...
    WHERE ...
)
SELECT * FROM cte_name;
```

### Benefits
- Simplifies nested queries
- Improves readability and reusability
- Can be recursive
- Better performance than subqueries in some cases
- Allows for modular query construction

### Examples

#### Example 1: Students Scoring Above 85
```sql
WITH TopStudents AS (
    SELECT name, score FROM students WHERE score > 85
)
SELECT * FROM TopStudents ORDER BY score DESC;
```

#### Example 2: High Earners
```sql
WITH HighEarners AS (
    SELECT name, salary FROM employees WHERE salary > 50000
)
SELECT * FROM HighEarners ORDER BY salary DESC;
```

#### Example 3: Total Sales Over 3000
```sql
WITH BigSales AS (
    SELECT * FROM sales WHERE amount > 3000
)
SELECT employee_id, sale_date, amount FROM BigSales;
```

#### Example 4: Average Salary by Department
```sql
WITH AvgDeptSalary AS (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
)
SELECT d.department_name, ads.avg_salary
FROM AvgDeptSalary ads
JOIN departments d ON ads.department_id = d.department_id;
```

### Recursive CTEs
```sql
WITH RECURSIVE employee_hierarchy AS (
    -- Base case: top-level managers
    SELECT employee_id, name, manager_id, 1 as level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: employees reporting to previous level
    SELECT e.employee_id, e.name, e.manager_id, eh.level + 1
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM employee_hierarchy ORDER BY level, name;
```

### Multiple CTEs
```sql
WITH 
high_performers AS (
    SELECT employee_id, name, performance_score
    FROM employees
    WHERE performance_score > 4.5
),
recent_hires AS (
    SELECT employee_id, name, hire_date
    FROM employees
    WHERE hire_date > '2023-01-01'
)
SELECT hp.name, hp.performance_score, rh.hire_date
FROM high_performers hp
JOIN recent_hires rh ON hp.employee_id = rh.employee_id;
```

## 2. Window Functions

### Definition
Window functions perform calculations across a set of rows related to the current row, without grouping the result set.

### Syntax
```sql
function_name() OVER (
    [PARTITION BY column1, column2, ...]
    [ORDER BY column1, column2, ...]
    [ROWS/RANGE BETWEEN ... AND ...]
)
```

### Types of Window Functions

#### Ranking Functions
```sql
-- ROW_NUMBER(): Assigns unique sequential integers
SELECT name, salary,
       ROW_NUMBER() OVER (ORDER BY salary DESC) as row_num
FROM employees;

-- RANK(): Assigns ranks with gaps for ties
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) as rank
FROM employees;

-- DENSE_RANK(): Assigns ranks without gaps
SELECT name, salary,
       DENSE_RANK() OVER (ORDER BY salary DESC) as dense_rank
FROM employees;

-- NTILE(): Divides rows into specified number of groups
SELECT name, salary,
       NTILE(4) OVER (ORDER BY salary DESC) as quartile
FROM employees;
```

#### Aggregate Window Functions
```sql
-- Running totals
SELECT order_date, amount,
       SUM(amount) OVER (ORDER BY order_date) as running_total
FROM orders;

-- Moving averages
SELECT order_date, amount,
       AVG(amount) OVER (
           ORDER BY order_date 
           ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ) as moving_avg_3_days
FROM orders;

-- Percentage of total
SELECT department, salary,
       salary / SUM(salary) OVER () * 100 as pct_of_total
FROM employees;
```

#### Value Functions
```sql
-- LAG/LEAD: Access previous/next row values
SELECT order_date, amount,
       LAG(amount, 1) OVER (ORDER BY order_date) as prev_amount,
       LEAD(amount, 1) OVER (ORDER BY order_date) as next_amount
FROM orders;

-- FIRST_VALUE/LAST_VALUE: First/last value in window
SELECT name, salary, department,
       FIRST_VALUE(name) OVER (
           PARTITION BY department 
           ORDER BY salary DESC
       ) as highest_paid_in_dept
FROM employees;
```

## 3. Advanced Joins

### Self Joins
```sql
-- Find employees and their managers
SELECT e1.name AS employee, e2.name AS manager
FROM employees e1
LEFT JOIN employees e2 ON e1.manager_id = e2.employee_id;
```

### Cross Joins
```sql
-- Generate all possible combinations
SELECT p.product_name, c.category_name
FROM products p
CROSS JOIN categories c;
```

### Lateral Joins (PostgreSQL, SQL Server)
```sql
-- Get top 3 orders for each customer
SELECT c.customer_name, o.order_date, o.amount
FROM customers c
CROSS JOIN LATERAL (
    SELECT order_date, amount
    FROM orders
    WHERE customer_id = c.customer_id
    ORDER BY order_date DESC
    LIMIT 3
) o;
```

## 4. Subqueries and Correlated Subqueries

### Scalar Subqueries
```sql
-- Employee with highest salary
SELECT name, salary
FROM employees
WHERE salary = (SELECT MAX(salary) FROM employees);
```

### Correlated Subqueries
```sql
-- Employees earning above department average
SELECT name, salary, department_id
FROM employees e1
WHERE salary > (
    SELECT AVG(salary)
    FROM employees e2
    WHERE e2.department_id = e1.department_id
);
```

### EXISTS and NOT EXISTS
```sql
-- Customers with at least one order
SELECT customer_name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);

-- Products never ordered
SELECT product_name
FROM products p
WHERE NOT EXISTS (
    SELECT 1 FROM order_items oi WHERE oi.product_id = p.product_id
);
```

## 5. Advanced Aggregation

### GROUPING SETS
```sql
-- Multiple grouping levels in one query
SELECT department, job_title, COUNT(*) as employee_count
FROM employees
GROUP BY GROUPING SETS (
    (department),
    (job_title),
    (department, job_title),
    ()  -- Grand total
);
```

### ROLLUP
```sql
-- Hierarchical totals
SELECT department, job_title, COUNT(*) as employee_count
FROM employees
GROUP BY ROLLUP (department, job_title);
```

### CUBE
```sql
-- All possible combinations of grouping columns
SELECT department, job_title, COUNT(*) as employee_count
FROM employees
GROUP BY CUBE (department, job_title);
```

### FILTER Clause
```sql
-- Conditional aggregation
SELECT department,
       COUNT(*) as total_employees,
       COUNT(*) FILTER (WHERE salary > 50000) as high_earners,
       AVG(salary) FILTER (WHERE hire_date > '2020-01-01') as avg_salary_recent_hires
FROM employees
GROUP BY department;
```

## 6. Pivot and Unpivot Operations

### PIVOT (SQL Server, Oracle)
```sql
-- Transform rows to columns
SELECT *
FROM (
    SELECT department, job_title, salary
    FROM employees
) AS source
PIVOT (
    AVG(salary)
    FOR job_title IN ([Manager], [Developer], [Analyst])
) AS pivot_table;
```

### Manual Pivot (PostgreSQL, MySQL)
```sql
SELECT department,
       AVG(CASE WHEN job_title = 'Manager' THEN salary END) as manager_avg,
       AVG(CASE WHEN job_title = 'Developer' THEN salary END) as developer_avg,
       AVG(CASE WHEN job_title = 'Analyst' THEN salary END) as analyst_avg
FROM employees
GROUP BY department;
```

## 7. String Functions and Pattern Matching

### Advanced String Functions
```sql
-- String manipulation
SELECT 
    CONCAT(first_name, ' ', last_name) as full_name,
    SUBSTRING(email, 1, POSITION('@' IN email) - 1) as username,
    REGEXP_REPLACE(phone, '[^0-9]', '', 'g') as clean_phone
FROM users;
```

### Regular Expressions
```sql
-- PostgreSQL regex examples
SELECT *
FROM products
WHERE product_name ~ '^[A-Z][a-z]+ \d+$';  -- Starts with capital, ends with number

-- Extract patterns
SELECT 
    product_code,
    SUBSTRING(product_code FROM '[A-Z]{2,3}') as category_code,
    SUBSTRING(product_code FROM '\d+') as product_number
FROM products;
```

## 8. Date and Time Functions

### Date Arithmetic
```sql
-- Date calculations
SELECT 
    order_date,
    order_date + INTERVAL '30 days' as due_date,
    AGE(CURRENT_DATE, order_date) as order_age,
    EXTRACT(DOW FROM order_date) as day_of_week,
    DATE_TRUNC('month', order_date) as order_month
FROM orders;
```

### Time Series Analysis
```sql
-- Generate date series
WITH date_series AS (
    SELECT generate_series('2023-01-01'::date, '2023-12-31'::date, '1 day'::interval)::date as date
)
SELECT 
    ds.date,
    COALESCE(SUM(o.amount), 0) as daily_sales
FROM date_series ds
LEFT JOIN orders o ON ds.date = o.order_date
GROUP BY ds.date
ORDER BY ds.date;
```

## 9. JSON and XML Operations

### JSON Functions (PostgreSQL, MySQL 8.0+)
```sql
-- Working with JSON data
SELECT 
    user_id,
    preferences->>'theme' as theme_preference,
    JSON_ARRAY_LENGTH(preferences->'notifications') as notification_count,
    JSON_EXTRACT_PATH_TEXT(profile, 'address', 'city') as city
FROM users
WHERE preferences->>'language' = 'en';

-- JSON aggregation
SELECT 
    department,
    JSON_AGG(
        JSON_BUILD_OBJECT(
            'name', name,
            'salary', salary,
            'hire_date
