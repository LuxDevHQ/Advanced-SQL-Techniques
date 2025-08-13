# Complete SQL Guide: From Stored Procedures to Advanced Techniques

## Part I: MySQL Stored Procedures – Beginner Guide

### What is a Stored Procedure?
A **stored procedure** in MySQL is like a *saved set of SQL steps* in the database. Instead of typing the same query every time, you **store it once** and **call it whenever you need it**.

### Basic Stored Procedure Examples

#### 1. Show All Customers
```sql
DELIMITER $$
CREATE PROCEDURE ShowAllCustomers()
BEGIN
    SELECT * FROM customer_info;
END$$
DELIMITER ;
```
**Run it:**
```sql
CALL ShowAllCustomers();
```
📌 *This simply lists all customers from the `customer_info` table.*

#### 2. Find Customer by ID (Using Input Parameter)
```sql
DELIMITER $$
CREATE PROCEDURE GetCustomerByID(IN CustID INT)
BEGIN
    SELECT * FROM customer_info WHERE customer_id = CustID;
END$$
DELIMITER ;
```
**Run it:**
```sql
CALL GetCustomerByID(1);
```
📌 *This shows details for the customer whose ID is `1`.*

#### 3. Show Sales for a Customer
```sql
DELIMITER $$
CREATE PROCEDURE GetSalesByCustomer(IN CustID INT)
BEGIN
    SELECT s.sales_id, s.total_sales, p.product_name 
    FROM sales s 
    JOIN products p ON s.product_id = p.product_id 
    WHERE s.customer_id = CustID;
END$$
DELIMITER ;
```
**Run it:**
```sql
CALL GetSalesByCustomer(2);
```
📌 *This shows all sales made by customer `2`.*

#### 4. Add a New Customer
```sql
DELIMITER $$
CREATE PROCEDURE AddCustomer(IN Name VARCHAR(120), IN Location VARCHAR(90))
BEGIN
    INSERT INTO customer_info(full_name, location) 
    VALUES(Name, Location);
END$$
DELIMITER ;
```
**Run it:**
```sql
CALL AddCustomer('John Doe', 'Nairobi');
```
📌 *This inserts a new customer into the database.*

### Advanced Stored Procedure Examples

#### 5. Procedure with Output Parameter
```sql
DELIMITER $$
CREATE PROCEDURE GetCustomerCount(OUT TotalCustomers INT)
BEGIN
    SELECT COUNT(*) INTO TotalCustomers FROM customer_info;
END$$
DELIMITER ;
```
**Run it:**
```sql
CALL GetCustomerCount(@total);
SELECT @total AS 'Total Customers';
```

#### 6. Procedure with Conditional Logic
```sql
DELIMITER $$
CREATE PROCEDURE UpdateCustomerStatus(IN CustID INT, IN NewStatus VARCHAR(20))
BEGIN
    DECLARE customer_exists INT DEFAULT 0;
    
    SELECT COUNT(*) INTO customer_exists 
    FROM customer_info 
    WHERE customer_id = CustID;
    
    IF customer_exists > 0 THEN
        UPDATE customer_info 
        SET status = NewStatus 
        WHERE customer_id = CustID;
        SELECT 'Customer status updated successfully' AS Result;
    ELSE
        SELECT 'Customer not found' AS Result;
    END IF;
END$$
DELIMITER ;
```

### Key Notes for Beginners
- `DELIMITER $$` → Tells MySQL we are writing a block of code until we say `$$`
- `IN` → Means the procedure expects an input value
- `OUT` → Means the procedure returns a value
- `CALL` → Runs the stored procedure

**Primary use cases:**
1. **View** data easily
2. **Filter** data based on a value
3. **Insert** new data
4. **Join** tables without rewriting big queries

---

## Part II: Advanced SQL Techniques

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

#### Example 3: Average Salary by Department
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

### CTE in Stored Procedures
```sql
DELIMITER $$
CREATE PROCEDURE GetTopPerformers(IN MinScore DECIMAL(3,2))
BEGIN
    WITH TopEmployees AS (
        SELECT employee_id, name, performance_score, department
        FROM employees
        WHERE performance_score >= MinScore
    )
    SELECT * FROM TopEmployees ORDER BY performance_score DESC;
END$$
DELIMITER ;
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

### Window Functions in Stored Procedures
```sql
DELIMITER $$
CREATE PROCEDURE GetSalesRanking(IN DeptName VARCHAR(50))
BEGIN
    SELECT 
        name,
        sales_amount,
        RANK() OVER (ORDER BY sales_amount DESC) as sales_rank,
        NTILE(4) OVER (ORDER BY sales_amount DESC) as quartile
    FROM employees
    WHERE department = DeptName;
END$$
DELIMITER ;
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

### Subqueries in Stored Procedures
```sql
DELIMITER $$
CREATE PROCEDURE GetAboveAverageEmployees(IN DeptID INT)
BEGIN
    SELECT name, salary
    FROM employees e1
    WHERE department_id = DeptID
    AND salary > (
        SELECT AVG(salary)
        FROM employees e2
        WHERE e2.department_id = DeptID
    );
END$$
DELIMITER ;
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

## 6. String Functions and Pattern Matching

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

### String Functions in Stored Procedures
```sql
DELIMITER $$
CREATE PROCEDURE CleanPhoneNumbers()
BEGIN
    UPDATE customers 
    SET phone = REGEXP_REPLACE(phone, '[^0-9]', '');
    
    SELECT COUNT(*) as 'Cleaned Records' FROM customers;
END$$
DELIMITER ;
```

## 7. Date and Time Functions

### Date Arithmetic
```sql
-- Date calculations
SELECT 
    order_date,
    DATE_ADD(order_date, INTERVAL 30 DAY) as due_date,
    DATEDIFF(CURRENT_DATE, order_date) as days_since_order,
    DAYOFWEEK(order_date) as day_of_week,
    DATE_FORMAT(order_date, '%Y-%m') as order_month
FROM orders;
```

### Time Series Analysis
```sql
-- Generate date series (MySQL 8.0+ with recursive CTE)
WITH RECURSIVE date_series AS (
    SELECT '2023-01-01' as date
    UNION ALL
    SELECT DATE_ADD(date, INTERVAL 1 DAY)
    FROM date_series
    WHERE date < '2023-12-31'
)
SELECT 
    ds.date,
    COALESCE(SUM(o.amount), 0) as daily_sales
FROM date_series ds
LEFT JOIN orders o ON ds.date = DATE(o.order_date)
GROUP BY ds.date
ORDER BY ds.date;
```

## 8. JSON Operations (MySQL 5.7+)

### JSON Functions
```sql
-- Working with JSON data
SELECT 
    user_id,
    JSON_EXTRACT(preferences, '$.theme') as theme_preference,
    JSON_LENGTH(JSON_EXTRACT(preferences, '$.notifications')) as notification_count,
    JSON_EXTRACT(profile, '$.address.city') as city
FROM users
WHERE JSON_EXTRACT(preferences, '$.language') = 'en';

-- JSON aggregation
SELECT 
    department,
    JSON_ARRAYAGG(
        JSON_OBJECT(
            'name', name,
            'salary', salary,
            'hire_date', hire_date
        )
    ) as employees_json
FROM employees
GROUP BY department;
```

### JSON in Stored Procedures
```sql
DELIMITER $$
CREATE PROCEDURE GetUserPreferences(IN UserID INT)
BEGIN
    SELECT 
        user_id,
        JSON_PRETTY(preferences) as formatted_preferences
    FROM users
    WHERE user_id = UserID;
END$$
DELIMITER ;
```

## 9. Combining Techniques: Advanced Stored Procedures

### Comprehensive Sales Report Procedure
```sql
DELIMITER $$
CREATE PROCEDURE GenerateSalesReport(
    IN StartDate DATE,
    IN EndDate DATE,
    IN DepartmentFilter VARCHAR(50)
)
BEGIN
    -- Declare variables
    DECLARE total_sales DECIMAL(15,2) DEFAULT 0;
    DECLARE avg_sale DECIMAL(10,2) DEFAULT 0;
    
    -- Main report with multiple advanced techniques
    WITH 
    sales_data AS (
        SELECT 
            e.department,
            e.name as employee_name,
            s.sale_date,
            s.amount,
            ROW_NUMBER() OVER (PARTITION BY e.department ORDER BY s.amount DESC) as dept_rank
        FROM sales s
        JOIN employees e ON s.employee_id = e.employee_id
        WHERE s.sale_date BETWEEN StartDate AND EndDate
        AND (DepartmentFilter IS NULL OR e.department = DepartmentFilter)
    ),
    department_totals AS (
        SELECT 
            department,
            SUM(amount) as dept_total,
            AVG(amount) as dept_avg,
            COUNT(*) as sale_count
        FROM sales_data
        GROUP BY department
    )
    SELECT 
        sd.department,
        sd.employee_name,
        sd.amount,
        sd.dept_rank,
        dt.dept_total,
        dt.dept_avg,
        ROUND((sd.amount / dt.dept_total) * 100, 2) as pct_of_dept_sales
    FROM sales_data sd
    JOIN department_totals dt ON sd.department = dt.department
    ORDER BY sd.department, sd.dept_rank;
    
    -- Summary statistics
    SELECT 
        SUM(amount) INTO total_sales
    FROM sales s
    JOIN employees e ON s.employee_id = e.employee_id
    WHERE s.sale_date BETWEEN StartDate AND EndDate
    AND (DepartmentFilter IS NULL OR e.department = DepartmentFilter);
    
    SELECT CONCAT('Total Sales: $', FORMAT(total_sales, 2)) as Summary;
    
END$$
DELIMITER ;
```

### Dynamic Query Building Procedure
```sql
DELIMITER $$
CREATE PROCEDURE DynamicEmployeeSearch(
    IN SearchName VARCHAR(100),
    IN MinSalary DECIMAL(10,2),
    IN Department VARCHAR(50),
    IN SortBy VARCHAR(20)
)
BEGIN
    SET @sql = 'SELECT employee_id, name, salary, department, hire_date FROM employees WHERE 1=1';
    
    IF SearchName IS NOT NULL THEN
        SET @sql = CONCAT(@sql, ' AND name LIKE ''%', SearchName, '%''');
    END IF;
    
    IF MinSalary IS NOT NULL THEN
        SET @sql = CONCAT(@sql, ' AND salary >= ', MinSalary);
    END IF;
    
    IF Department IS NOT NULL THEN
        SET @sql = CONCAT(@sql, ' AND department = ''', Department, '''');
    END IF;
    
    CASE SortBy
        WHEN 'name' THEN SET @sql = CONCAT(@sql, ' ORDER BY name');
        WHEN 'salary' THEN SET @sql = CONCAT(@sql, ' ORDER BY salary DESC');
        WHEN 'hire_date' THEN SET @sql = CONCAT(@sql, ' ORDER BY hire_date DESC');
        ELSE SET @sql = CONCAT(@sql, ' ORDER BY employee_id');
    END CASE;
    
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END$$
DELIMITER ;
```

## Best Practices Summary

### For Stored Procedures:
1. Use meaningful parameter names with proper prefixes (IN/OUT/INOUT)
2. Include error handling with DECLARE ... HANDLER
3. Add comments to explain complex logic
4. Use transactions for data modifications
5. Validate input parameters

### For Advanced SQL:
1. Use CTEs for complex queries to improve readability
2. Choose appropriate window functions for analytical queries
3. Optimize joins by understanding execution plans
4. Use EXISTS instead of IN for better performance
5. Consider indexing strategies for complex queries

### General Tips:
1. Test procedures thoroughly with edge cases
2. Use consistent naming conventions
