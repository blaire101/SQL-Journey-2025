# SQL Preparation Guide

> In life, you can choose who you want to be; be very careful with that choice.

---

## 📘 SQL Basics

### 1. Query Structure & Clauses
- `FROM`, `WHERE`, `GROUP BY`, `HAVING`, `SELECT`, `ORDER BY`
- `WHERE`: `NOT IN`, `NULL`, `<>`
- Aggregate Functions: `AVG()`, `SUM()`, `COUNT()`
- Conditional: `IF`, `IFNULL(expr, val)`, `CASE WHEN`
- NULL Handling: `COALESCE()`
- `DISTINCT`

![sql](docs/sql-1.png)

### 2. Joins & Subqueries
- `INNER JOIN`, `LEFT JOIN`, `RIGHT JOIN`, `SELF JOIN`, `CROSS JOIN`
- Cartesian Product
- Subqueries in `SELECT`, `FROM`, and `WHERE`

---

## 🧮 SQL Advanced Aggregation & Window Functions

### 3. Window Function Basics
- `PARTITION BY`, `ORDER BY` in `OVER()` clause
- Ranking: `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`
- Analytics: `SUM()`, `AVG()`, `MIN()`, `MAX()` over windows
- Lag/Lead: `LAG()`, `LEAD()` to access previous/next rows
- First/Last: `FIRST_VALUE()`, `LAST_VALUE()`

#### Example:
```sql
ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date)
LAG(txn_date) OVER (PARTITION BY customer_id ORDER BY txn_date)
```

---

## 🧑‍💻 SQL Classic Use Cases

### 4. Daily Average Spend > 3
```sql
SELECT customer_id, AVG(daily_sum) AS average_spend
FROM (
    SELECT customer_id, txn_date, SUM(txn_amount) AS daily_sum
    FROM transaction
    GROUP BY customer_id, txn_date
) daily_spend
GROUP BY customer_id
HAVING AVG(daily_sum) > 3;
```

### 5. Average Days Between Transactions
```sql
WITH transaction_date AS (
    SELECT customer_id, txn_date,
        LAG(txn_date) OVER (PARTITION BY customer_id ORDER BY txn_date) AS prev_date
    FROM transaction
),
diff_days AS (
    SELECT customer_id, DATEDIFF(txn_date, prev_date) AS gap
    FROM transaction_date
    WHERE prev_date IS NOT NULL
)
SELECT customer_id, AVG(gap) AS avg_gap
FROM diff_days
GROUP BY customer_id;
```

### 6. Top 10 Spenders in Last 30 Days
```sql
SELECT user_id, SUM(amount) AS total_spend
FROM transactions
WHERE tx_time >= DATE_SUB(CURRENT_DATE(), 30)
GROUP BY user_id
ORDER BY total_spend DESC
LIMIT 10;
```

### 7. Consecutive 3-Day Logins
```sql
WITH flags AS (
    SELECT user_id, login_date,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) AS rn,
        DATE_SUB(login_date, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date)) AS flag_date
    FROM logins
),
segments AS (
    SELECT user_id, flag_date, COUNT(*) AS cnt,
        MIN(login_date) AS start_date, MAX(login_date) AS end_date
    FROM flags
    GROUP BY user_id, flag_date
    HAVING COUNT(*) >= 3
)
SELECT * FROM segments;
```

---

## 🏀 Game Analytics: Window Logic

### 8. Player Scores 3+ Consecutive Times
```sql
SELECT DISTINCT name, team
FROM (
    SELECT *,
        LAG(name, 1) OVER (PARTITION BY team ORDER BY score_time) AS lg1,
        LAG(name, 2) OVER (PARTITION BY team ORDER BY score_time) AS lg2,
        LEAD(name, 1) OVER (PARTITION BY team ORDER BY score_time) AS ld1,
        LEAD(name, 2) OVER (PARTITION BY team ORDER BY score_time) AS ld2
    FROM basketball_scores
) t
WHERE (name = lg1 AND name = lg2)
   OR (name = lg1 AND name = ld1)
   OR (name = ld1 AND name = ld2);
```

### 9. Identify Turnaround Scores
```sql
WITH running_scores AS (
    SELECT *,
        SUM(CASE WHEN team = 'A' THEN score ELSE 0 END) OVER (ORDER BY score_time) AS A_score,
        SUM(CASE WHEN team = 'B' THEN score ELSE 0 END) OVER (ORDER BY score_time) AS B_score
    FROM basketball_scores
),
score_gap AS (
    SELECT *,
        A_score - B_score AS diff,
        LAG(A_score - B_score) OVER (ORDER BY score_time) AS prev_diff
    FROM running_scores
)
SELECT team, number, score_time, name
FROM score_gap
WHERE diff * prev_diff <= 0 AND diff != 0;
```

---

## 📊 Sales & Ranking Questions

### 10. Daily Sales & City Ranking
```sql
WITH aggregated AS (
  SELECT 
      city, order_date, SUM(sales) AS daily_sales
  FROM 
      orders
  GROUP BY city, order_date
)
SELECT order_date, city, daily_sales,
    ROW_NUMBER() OVER (PARTITION BY order_date ORDER BY daily_sales DESC) AS rank,
    SUM(daily_sales) OVER (PARTITION BY order_date) AS total,
    SUM(daily_sales) OVER (PARTITION BY order_date ORDER BY daily_sales DESC) AS cumulative
FROM aggregated
ORDER BY order_date, daily_sales DESC;
```

### 11. Operator Top3 in City per Day
```sql
WITH operator_sales AS (
    SELECT city, order_date, operator_id, SUM(sales) AS total_sales
    FROM orders
    GROUP BY city, order_date, operator_id
),
ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY city, order_date ORDER BY total_sales DESC) AS rn
    FROM operator_sales
)
SELECT * FROM ranked WHERE rn <= 3;
```

---

## 🧠 Sessionize Events by Time Gap

### 12. Group Logs into Sessions (gap > 60s)
```sql
WITH logs_diff AS (
  SELECT id, ts,
      LAG(ts) OVER (PARTITION BY id ORDER BY ts) AS prev_ts,
      CASE WHEN ts - LAG(ts) OVER (PARTITION BY id ORDER BY ts) > 60 OR LAG(ts) IS NULL THEN 1 ELSE 0 END AS is_new_session
  FROM logs
),
sessions AS (
  SELECT id, ts,
      SUM(is_new_session) OVER (PARTITION BY id ORDER BY ts) AS session_id
  FROM logs_diff
)
SELECT * FROM sessions;
```

---

## 🔁 Advanced Retention & Overlap Analysis

### 13. Seller 30-Day Retention Rate
```sql
WITH prev_txn AS (
  SELECT 
      seller_id, 
      transaction_date,
      LAG(transaction_date) OVER (PARTITION BY seller_id ORDER BY transaction_date) AS prev_date
  FROM transactions
)
SELECT 
    transaction_date,
    COUNT(seller_id) AS total,
    SUM(CASE WHEN DATEDIFF(transaction_date, prev_date) <= 30 THEN 1 ELSE 0 END) AS retained,
    ROUND(SUM(CASE WHEN DATEDIFF(transaction_date, prev_date) <= 30 THEN 1 ELSE 0 END) / COUNT(seller_id), 2) AS rate
FROM 
    prev_txn
GROUP BY 
    transaction_date;
```

---

## 🎥 Live Stream Max Online Count

### 14. Maximum Concurrent Streamers
```sql
SELECT MAX(amt) AS max_online
FROM (
  SELECT dt, SUM(tag) OVER (ORDER BY dt) AS amt
  FROM (
    SELECT stt AS dt, 1 AS tag FROM streams
    UNION ALL
    SELECT edt AS dt, -1 AS tag FROM streams
  ) events
) result;
```

---

## 🧾 Bonus Techniques

- Find duplicates: `GROUP BY HAVING COUNT(*) > 1`
- Find nth highest: `ROW_NUMBER() OVER (...) = N`
- Find records in A not in B: `LEFT JOIN ... WHERE B.key IS NULL`
- Handling overlapping date ranges using `MAX(edt) OVER (...)`

---

## ✅ Tips for Interview Prep

1. Prioritize mastering **Window Functions** and **Aggregations**
2. Practice **Sessionization**, **Retention**, **Top-N**, **Rolling Sum**
3. Real-world SQL challenges: use public datasets or mock business tables
4. Understand **join types**, **execution plans**, and **data skew handling**

---

Happy practicing! 🎯
