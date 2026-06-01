# Window Functions in SQL

Window functions perform calculations across a set of rows related to the current row — without collapsing them into a single result like GROUP BY does.

This is slightly advanced but comes up in interviews for backend roles that involve complex reporting or analytics queries.

---

## The difference from GROUP BY

With GROUP BY you lose individual rows — everything gets collapsed into one row per group.

```sql
SELECT department, AVG(salary)
FROM employees
GROUP BY department;
```

You get one row per department. Individual employee rows are gone.

With window functions you keep all rows and add the calculation alongside:

```sql
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
```

Every employee row stays. The department average is just added as an extra column. That's the key difference.

---

## Sample data

```sql
+----+--------+------------+--------+
| id | name   | department | salary |
+----+--------+------------+--------+
| 1  | Rohit  | Backend    | 80000  |
| 2  | Sneha  | Backend    | 90000  |
| 3  | Arjun  | Frontend   | 70000  |
| 4  | Priya  | Frontend   | 75000  |
| 5  | Karan  | Backend    | 85000  |
+----+--------+------------+--------+
```

---

## ROW_NUMBER

Assigns a unique number to each row within a partition.

```sql
SELECT name, department, salary,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num
FROM employees;
```

Result:

```
+--------+------------+--------+---------+
| name   | department | salary | row_num |
+--------+------------+--------+---------+
| Sneha  | Backend    | 90000  | 1       |
| Karan  | Backend    | 85000  | 2       |
| Rohit  | Backend    | 80000  | 3       |
| Priya  | Frontend   | 75000  | 1       |
| Arjun  | Frontend   | 70000  | 2       |
+--------+------------+--------+---------+
```

Row numbering resets for each department. Useful for getting the top N records per group.

**Get the highest paid employee per department:**

```sql
SELECT name, department, salary
FROM (
    SELECT name, department, salary,
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num
    FROM employees
) ranked
WHERE row_num = 1;
```

---

## RANK and DENSE_RANK

Similar to ROW_NUMBER but handles ties differently.

```sql
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) AS rnk,
       DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rnk
FROM employees;
```

If two people have the same salary —

- `RANK` skips the next number (1, 2, 2, 4)
- `DENSE_RANK` doesn't skip (1, 2, 2, 3)
- `ROW_NUMBER` always unique, no ties (1, 2, 3, 4)

---

## LAG and LEAD

Access the previous or next row's value without a self join.

```sql
SELECT name, salary,
       LAG(salary) OVER (ORDER BY salary) AS previous_salary,
       LEAD(salary) OVER (ORDER BY salary) AS next_salary
FROM employees;
```

Result:

```
+--------+--------+-----------------+-------------+
| name   | salary | previous_salary | next_salary |
+--------+--------+-----------------+-------------+
| Arjun  | 70000  | NULL            | 75000       |
| Priya  | 75000  | 70000           | 80000       |
| Rohit  | 80000  | 75000           | 85000       |
| Karan  | 85000  | 80000           | 90000       |
| Sneha  | 90000  | 85000           | NULL        |
+--------+--------+-----------------+-------------+
```

LAG and LEAD are useful for comparing a row with the previous or next one — like month over month sales comparison.

---

## SUM and AVG as window functions

Running total:

```sql
SELECT name, salary,
       SUM(salary) OVER (ORDER BY id) AS running_total
FROM employees;
```

Each row shows the cumulative sum up to that point. Without `ORDER BY` inside OVER it would just show the total for every row.

---

## PARTITION BY vs ORDER BY inside OVER

- `PARTITION BY` — divides rows into groups, like GROUP BY but keeps all rows
- `ORDER BY` inside OVER — defines the order within each partition for functions like ROW_NUMBER, LAG, LEAD, running totals

You can use both together:

```sql
ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC)
```

Partition by department, rank by salary within each department.

---

## Stuff I want to remember

**What is the difference between ROW_NUMBER, RANK and DENSE_RANK?**

All three rank rows but handle ties differently. ROW_NUMBER always gives a unique number even for ties. RANK gives the same number for ties but skips the next rank — so after two 2nd places the next is 4th. DENSE_RANK gives the same number for ties but doesn't skip — after two 2nd places the next is 3rd.

**What is the difference between window functions and GROUP BY?**

GROUP BY collapses rows into groups and you lose individual row data. Window functions keep all rows and add an extra calculated column. If I want the average salary per department alongside each employee's individual data, GROUP BY can't do that — window functions can.

**When would you use LAG?**

When you need to compare a value with the previous row without writing a self join. Common use case is month over month comparison — current month sales vs last month sales in the same query.

---

*Window functions looked scary at first but once I understood PARTITION BY is just "GROUP BY that doesn't collapse rows" it clicked.*