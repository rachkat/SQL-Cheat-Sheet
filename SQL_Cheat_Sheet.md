# SQL Cheat Sheet (GitHub-Ready)

A concise, practical reference for everyday SQL. Examples are ANSI-SQL first, with notes for Postgres (PG), MySQL (MY), and SQL Server (SSMS).

---

## 1) Select, Filter, Sort

```sql
-- Return columns
SELECT col1, col2
FROM table_name;

-- Aliases
SELECT col1 AS c1, col2 AS c2
FROM table_name;

-- Distinct rows
SELECT DISTINCT col1, col2
FROM table_name;

-- Filtering
SELECT *
FROM table_name
WHERE col1 = 'A'                      -- equality
  AND col2 BETWEEN 10 AND 20          -- range
  AND col3 IN ('x', 'y', 'z')         -- set
  AND col4 LIKE 'abc%'                -- pattern
  AND col5 IS NOT NULL;               -- null check

-- Sorting
SELECT *
FROM table_name
ORDER BY col1 ASC, col2 DESC;

-- Limit (paging)
-- PG/MySQL:
SELECT * FROM table_name ORDER BY id LIMIT 10 OFFSET 20;
-- SQL Server:
SELECT * FROM table_name ORDER BY id OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

---

## 2) Aggregation (GROUP BY & HAVING)

```sql
SELECT dept_id,
       COUNT(*)        AS n_emps,
       AVG(salary)     AS avg_salary,
       SUM(bonus)      AS total_bonus,
       MIN(hire_date)  AS first_hire
FROM employees
GROUP BY dept_id
HAVING COUNT(*) >= 5;  -- filter after grouping
```

**Rules**: Every non-aggregated select column must appear in `GROUP BY` (ANSI).

---

## 3) Common Joins

```sql
-- INNER JOIN: matching rows only
SELECT e.emp_id, e.name, d.dept_name
FROM employees e
JOIN departments d ON d.dept_id = e.dept_id;

-- LEFT JOIN: keep all left rows, nulls on no match
SELECT c.customer_id, c.name, o.order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id;

-- RIGHT JOIN (less common)
SELECT o.order_id, c.customer_id, c.name
FROM orders o
RIGHT JOIN customers c ON c.customer_id = o.customer_id;

-- FULL OUTER JOIN (PG/SSMS; MySQL: emulate with UNION)
SELECT *
FROM a
FULL OUTER JOIN b ON a.key = b.key;

-- CROSS JOIN: Cartesian product
SELECT *
FROM a CROSS JOIN b;
```

**Anti/semijoins** (common interview pattern):

```sql
-- Anti-join: rows in A with no match in B
SELECT a.*
FROM A a
LEFT JOIN B b ON b.key = a.key
WHERE b.key IS NULL;

-- Semi-join: rows in A that have a match in B
SELECT DISTINCT a.*
FROM A a
JOIN B b ON b.key = a.key;
```

---

## 4) Set Operations

```sql
-- All must have same number & compatible types of columns
SELECT col1, col2 FROM t1
UNION                 -- distinct by default
SELECT col1, col2 FROM t2;

SELECT col1, col2 FROM t1
UNION ALL             -- keep duplicates
SELECT col1, col2 FROM t2;

SELECT col1, col2 FROM t1
INTERSECT             -- common rows (PG/SSMS; MySQL: use INNER JOIN)
SELECT col1, col2 FROM t2;

SELECT col1, col2 FROM t1
EXCEPT                -- t1 minus t2 (PG/SSMS; MySQL: use LEFT JOIN + NULL filter)
SELECT col1, col2 FROM t2;
```

---

## 5) Subqueries & CTEs

```sql
-- Scalar subquery
SELECT e.*,
       (SELECT AVG(salary) FROM employees) AS company_avg
FROM employees e;

-- IN subquery
SELECT *
FROM orders
WHERE customer_id IN (SELECT customer_id FROM vip_customers);

-- CTE (Common Table Expression)
WITH top_depts AS (
  SELECT dept_id, AVG(salary) AS avg_sal
  FROM employees
  GROUP BY dept_id
  HAVING AVG(salary) > 90000
)
SELECT e.name, d.dept_id, d.avg_sal
FROM employees e
JOIN top_depts d ON d.dept_id = e.dept_id;
```

---

## 6) Window Functions (Analytics)

```sql
-- Running totals, rankings, partitions
SELECT
  e.emp_id,
  e.dept_id,
  e.salary,
  SUM(e.salary) OVER (PARTITION BY e.dept_id ORDER BY e.emp_id) AS running_salary,
  AVG(e.salary) OVER (PARTITION BY e.dept_id)                    AS dept_avg,
  ROW_NUMBER() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS rn,
  RANK()       OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS rnk,
  DENSE_RANK() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS drnk
FROM employees e;
```

**Notes**:  
- `PARTITION BY` resets the window per group; omit to use whole table.  
- Use `ORDER BY` inside `OVER()` for running/ordered calculations.

---

## 7) Conditional Logic

```sql
SELECT
  order_id,
  amount,
  CASE
    WHEN amount >= 1000 THEN 'High'
    WHEN amount >= 100  THEN 'Medium'
    ELSE 'Low'
  END AS amount_band
FROM orders;
```

---

## 8) Date & Time Essentials

```sql
-- Current timestamp
-- PG:    NOW()
-- MY:    NOW()
-- SSMS:  SYSDATETIME()

-- Date arithmetic (PG style shown)
SELECT NOW() - INTERVAL '7 days' AS a_week_ago;

-- Extract parts
SELECT EXTRACT(YEAR FROM order_date) AS yr,
       EXTRACT(MONTH FROM order_date) AS mo
FROM orders;

-- Truncation
-- PG: date_trunc('month', order_date)
-- SSMS: DATEFROMPARTS(YEAR(order_date), MONTH(order_date), 1)
-- MY:   DATE_FORMAT(order_date, '%Y-%m-01')
```

---

## 9) Strings & Regex (quick hits)

```sql
-- Concatenation
-- PG: col1 || ' ' || col2
-- MY: CONCAT(col1, ' ', col2)
-- SSMS: CONCAT(col1, ' ', col2)

-- Substring
-- PG/SSMS: SUBSTRING(col FROM 1 FOR 5) / SUBSTRING(col, 1, 5)
-- MY: SUBSTRING(col, 1, 5)

-- Replace
-- PG/MY: REPLACE(col, 'old', 'new')
-- SSMS:  REPLACE(col, 'old', 'new')

-- Regex (PG)
SELECT *
FROM t
WHERE col ~ '^[A-Z]{3}[0-9]{2}$';  -- matches e.g. ABC12
```

---

## 10) NULLs & Coalescing

```sql
-- NULL-safe logic
WHERE col IS NULL         -- test for null
WHERE col IS NOT NULL

-- Substitute NULL with default
SELECT COALESCE(middle_name, '') AS mid
FROM people;
```

---

## 11) DDL Basics (Tables, Keys, Indexes)

```sql
-- Create table
CREATE TABLE employees (
  emp_id      INT PRIMARY KEY,
  name        VARCHAR(100) NOT NULL,
  dept_id     INT,
  salary      DECIMAL(12,2),
  hire_date   DATE,
  CONSTRAINT fk_emp_dept
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
);

-- Indexes (simple)
CREATE INDEX idx_employees_dept ON employees(dept_id);

-- Insert / Update / Delete
INSERT INTO employees (emp_id, name, dept_id, salary, hire_date)
VALUES (1, 'Ava', 10, 95000, '2024-01-01');

UPDATE employees
SET salary = salary * 1.05
WHERE dept_id = 10;

DELETE FROM employees
WHERE emp_id = 1;
```

---

## 12) Transactions (ACID)

```sql
BEGIN;                            -- or START TRANSACTION
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;                           -- or ROLLBACK;
```

**Note**: Some environments autocommit; wrap multi-step ops in a transaction.

---

## 13) Performance Tips

- Filter early; avoid `SELECT *` in production.  
- Index columns used in joins, filters, and sorts (measure with `EXPLAIN`).  
- Prefer `UNION ALL` when duplicates don’t matter.  
- Beware functions on indexed columns in `WHERE` (may prevent index use).  
- Use proper data types (e.g., `DATE` vs. `VARCHAR`).  
- Batch inserts/updates for large loads.

---

## 14) Common Interview Patterns

```sql
-- 2nd highest salary (per dept)
SELECT dept_id, MAX(salary) AS second_highest
FROM employees e
WHERE salary < (
  SELECT MAX(salary)
  FROM employees e2
  WHERE e2.dept_id = e.dept_id
)
GROUP BY dept_id;

-- Customers who purchased in consecutive months
WITH months AS (
  SELECT customer_id,
         DATE_TRUNC('month', order_date) AS m
  FROM orders
  GROUP BY customer_id, DATE_TRUNC('month', order_date)
),
pairs AS (
  SELECT a.customer_id, a.m AS m1, b.m AS m2
  FROM months a
  JOIN months b
    ON a.customer_id = b.customer_id
   AND b.m = a.m + INTERVAL '1 month'
)
SELECT DISTINCT customer_id FROM pairs;
```

---

## 15) Admin & Introspection (quick)

```sql
-- What tables exist?
-- PG:  SELECT * FROM information_schema.tables WHERE table_schema='public';
-- MY:  SHOW TABLES;
-- SSMS: SELECT * FROM INFORMATION_SCHEMA.TABLES;

-- Inspect columns
-- PG/SSMS: SELECT * FROM INFORMATION_SCHEMA.COLUMNS WHERE table_name='employees';
-- MY:      SHOW COLUMNS FROM employees;
```

---

## 16) Safe Deletes & Updates

```sql
-- Always preview first!
SELECT * FROM orders WHERE status = 'CANCELLED';

-- Then delete
DELETE FROM orders WHERE status = 'CANCELLED';
```

---

### License
MIT — free to use in your repos.

---

### Credits
Authored by Rachel Gainer.
