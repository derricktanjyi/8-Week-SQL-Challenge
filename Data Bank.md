## Introduction
### There is a new innovation in the financial industry called Neo-Banks: new aged digital only banks without physical branches.
### Danny thought that there should be some sort of intersection between these new age banks, cryptocurrency and the data world…so he decides to launch a new initiative - Data Bank!
### Data Bank runs just like any other digital bank - but it isn’t only for banking activities, they also have the world’s most secure distributed data storage platform!
### Customers are allocated cloud data storage limits which are directly linked to how much money they have in their accounts. There are a few interesting caveats that go with this business model, and this is where the Data Bank team need your help!
### The management team at Data Bank want to increase their total customer base - but also need some help tracking just how much data storage their customers will need.
### This case study is all about calculating metrics, growth and helping the business analyse their data in a smart way to better forecast and plan for their future developments!

### A. Customer Nodes Explorations
#### 1. How many unique nodes are there on the Data Bank system?

```sql
WITH node_combinations AS (
  SELECT
    CONCAT(region_id, node_id) AS combinations
  FROM
    data_bank.customer_nodes
)
SELECT
  COUNT(DISTINCT combinations)
FROM
  node_combinations;
 ```
 | count |
 | ----- |
 | 25    |
 
 #### 2. What is the number of nodes per region?
 
 ```sql
 SELECT
  r.region_name,
  COUNT(DISTINCT c.node_id) AS nodes
FROM
  data_bank.regions r
  INNER JOIN data_bank.customer_nodes c ON r.region_id = c.region_id
GROUP BY
  r.region_name
ORDER BY
  r.region_name;
 ```
| region\_name | nodes |
| ------------ | ----- |
| Africa       | 5     |
| America      | 5     |
| Asia         | 5     |
| Australia    | 5     |
| Europe       | 5     |

#### 3. How many customers are allocated to each region?
```sql
SELECT
  r.region_name,
  COUNT(DISTINCT c.customer_id) AS customers
FROM
  data_bank.regions r
  INNER JOIN data_bank.customer_nodes c ON r.region_id = c.region_id
GROUP BY
  r.region_name
ORDER BY
  r.region_name;
 ```
| region\_name | customers |
| ------------ | --------- |
| Africa       | 102       |
| America      | 105       |
| Asia         | 95        |
| Australia    | 110       |
| Europe       | 88        |

#### 4. How many days on average are customers reallocated to a different node?
```sql
DROP TABLE IF EXISTS ranked_customer_nodes;
CREATE TEMP TABLE ranked_customer_nodes AS
SELECT
  customer_id,
  node_id,
  region_id,
  DATE_PART('day', AGE(end_date, start_date)) :: INTEGER AS duration,
  ROW_NUMBER() OVER (
    PARTITION BY customer_id
    ORDER BY
      start_date
  ) AS rn
FROM
  data_bank.customer_nodes;
WITH RECURSIVE output_table AS (
    SELECT
      customer_id,
      node_id,
      duration,
      rn,
      1 AS run_id
    FROM
      ranked_customer_nodes
    WHERE
      rn = 1
    UNION ALL
    SELECT
      t1.customer_id,
      t2.node_id,
      t2.duration,
      t2.rn,
      -- update run_id if the node_id values do not match
      CASE
        WHEN t1.node_id != t2.node_id THEN t1.run_id + 1
        ELSE t1.run_id
      END AS run_id
    FROM
      output_table t1
      INNER JOIN ranked_customer_nodes t2 ON t1.rn + 1 = t2.rn
      AND t1.customer_id = t2.customer_id
      And t2.rn > 1
  ),
  cte_customer_nodes AS (
    SELECT
      customer_id,
      run_id,
      duration AS node_duration
    FROM
      output_table
    GROUP BY
      customer_id,
      run_id,
      duration
  )
SELECT
  ROUND(AVG(node_duration)) AS average_node_duration
FROM
  cte_customer_nodes;
 ```
| average\_node\_duration |
| ----------------------- |
| 14                      |

#### 5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?
```sql
WITH RECURSIVE output_table AS (
  SELECT
    customer_id,
    node_id,
    region_id,
    duration,
    rn,
    1 AS run_id
  FROM
    ranked_customer_nodes -- temp table created in previous question
  WHERE
    rn = 1
  UNION ALL
  SELECT
    t1.customer_id,
    t2.node_id,
    t2.region_id,
    t2.duration,
    t2.rn,
    -- update run_id if the node_id values do not match
    CASE
      WHEN t1.node_id != t2.node_id THEN t1.run_id + 1
      ELSE t1.run_id
    END AS run_id
  FROM
    output_table t1
    INNER JOIN ranked_customer_nodes t2 ON t1.rn + 1 = t2.rn
    AND t1.customer_id = t2.customer_id
    And t2.rn > 1
),
cte_customer_nodes AS (
  SELECT
    region_id,
    run_id,
    duration AS node_duration
  FROM
    output_table
  GROUP BY
    region_id,
    run_id,
    duration
)
SELECT
  r.region_name,
  ROUND(
    PERCENTILE_CONT(0.5) WITHIN GROUP (
      ORDER BY
        node_duration
    )
  ) AS median_node_duration,
  ROUND(
    PERCENTILE_CONT(0.8) WITHIN GROUP (
      ORDER BY
        node_duration
    )
  ) AS pct80_node_duration,
  ROUND(
    PERCENTILE_CONT(0.95) WITHIN GROUP (
      ORDER BY
        node_duration
    )
  ) AS pct95_node_duration
FROM
  cte_customer_nodes c
  INNER JOIN data_bank.regions r ON c.region_id = r.region_id
GROUP BY
  r.region_name,
  r.region_id
ORDER BY
  r.region_id;
```
| region\_name | median\_node\_duration | pct80\_node\_duration | pct95\_node\_duration |
| ------------ | ---------------------- | --------------------- | --------------------- |
| Australia    | 14                     | 24                    | 28                    |
| America      | 15                     | 24                    | 29                    |
| Africa       | 14                     | 24                    | 28                    |
| Asia         | 14                     | 24                    | 28                    |
| Europe       | 15                     | 24                    | 28                    |

### B. Customer Transactions

#### 1. What is the unique count and total amount for each transaction type?
```sql
SELECT
  txn_type,
  COUNT(*) AS txn_count,
  SUM(txn_amount) AS total_amount
FROM
  data_bank.customer_transactions
GROUP BY
  txn_type
ORDER BY
  txn_type;
```
| txn\_type  | txn\_count | total\_amount |
| ---------- | ---------- | ------------- |
| deposit    | 2671       | 1359168       |
| purchase   | 1617       | 806537        |
| withdrawal | 1580       | 793003        |

#### 2. What is the average total historical deposit counts and amounts for all customers?
```sql
WITH total_deposits AS(
  SELECT
    customer_id,
    COUNT(*) AS txn_count,
    SUM(txn_amount) AS amount
  FROM
    data_bank.customer_transactions
  WHERE
    txn_type = 'deposit'
  GROUP BY
    customer_id
)
SELECT
  ROUND(AVG(txn_count)) AS avg_txn_count,
  ROUND(SUM(amount) / SUM(txn_count)) AS avg_deposit_amount
FROM
  total_deposits;
```
| avg\_txn\_count | avg\_deposit\_amount |
| --------------- | -------------------- |
| 5               | 509                  |

#### 3. For each month - how many Data Bank customers make more than 1 deposit and at least either 1 purchase or 1 withdrawal in a single month?
```sql
WITH cte_customer_months AS (
  SELECT
    DATE_TRUNC('month', txn_date) :: DATE AS month,
    customer_id,
    SUM(
      CASE
        WHEN txn_type = 'deposit' THEN 1
        ELSE 0
      END
    ) AS deposit_count,
    SUM(
      CASE
        WHEN txn_type = 'withdrawal' THEN 1
        ELSE 0
      END
    ) AS withdrawal_count,
    SUM(
      CASE
        WHEN txn_type = 'purchase' THEN 1
        ELSE 0
      END
    ) AS purchase_count
  FROM
    data_bank.customer_transactions
  GROUP BY
    month,
    customer_id
)
SELECT
  month,
  COUNT(DISTINCT customer_id) AS customer_count
FROM
  cte_customer_months
WHERE
  deposit_count > 1
  AND (
    purchase_count >= 1
    OR withdrawal_count >= 1
  )
GROUP BY
  month
ORDER BY
  month;
```
| month      | customer\_count |
| ---------- | --------------- |
| 2020-01-01 | 168             |
| 2020-02-01 | 181             |
| 2020-03-01 | 192             |
| 2020-04-01 | 70              |

#### 4. What is the closing balance for each customer at the end of the month? Also show the change in balance each month in the same table output.
#### Check the total range of months available in our analysis
```sql
SELECT
  DATE_TRUNC('month', txn_date) :: DATE AS month,
  COUNT(customer_id) AS record_count
FROM
  data_bank.customer_transactions
GROUP BY
  month
ORDER BY
  month;
```
| month      | record\_count |
| -----------| ------------- |
| 2020-01-01 | 1497          |
| 2020-02-01 | 1715          |
| 2020-03-01 | 1869          |
| 2020-04-01 | 787           |

#### We don’t know if customers have transactions in any set month, we will need to make sure that we have something in case they don’t have transactions.
#### We’ll need to generate those months to use in a cross join in our analysis with some logic to take the previous month’s balance if there are no records in the current month.
```sql
WITH cte_monthly_balances AS (
  SELECT
    customer_id,
    DATE_TRUNC('month', txn_date) :: DATE as month,
    SUM(
      CASE
        WHEN txn_type = 'deposit' THEN txn_amount
        ELSE (- txn_amount)
      END
    ) AS balance
  FROM
    data_bank.customer_transactions
  GROUP BY
    customer_id,
    month
  ORDER BY
    customer_id,
    month
),
cte_generated_months AS (
  SELECT
    DISTINCT customer_id,
    (
      '2020-01-01' :: DATE + GENERATE_SERIES(0, 3) * INTERVAL '1 MONTH'
    ) :: DATE AS month
  FROM
    data_bank.customer_transactions
)
SELECT
  m.customer_id,
  m.month,
  COALESCE(b.balance, 0) AS balance_contribution,
  SUM(b.balance) OVER (
    PARTITION BY m.customer_id
    ORDER BY
      m.month ROWS BETWEEN UNBOUNDED PRECEDING
      AND CURRENT ROW
  ) AS ending_balance
FROM
  cte_generated_months m
  LEFT JOIN cte_monthly_balances b ON m.month = b.month
  AND m.customer_id = b.customer_id
WHERE
  m.customer_id BETWEEN 1
  and 3;
```
| customer\_id | month      | balance\_contribution | ending\_balance |
| ------------ | -----------| --------------------- | --------------- |
| 1            | 2020-01-01 | 312                   | 312             |
| 1            | 2020-02-01 | 0                     | 312             |
| 1            | 2020-03-01 | \-952                 | \-640           |
| 1            | 2020-04-01 | 0                     | \-640           |
| 2            | 2020-01-01 | 549                   | 549             |
| 2            | 2020-02-01 | 0                     | 549             |
| 2            | 2020-03-01 | 61                    | 610             |
| 2            | 2020-04-01 | 0                     | 610             |
| 3            | 2020-01-01 | 144                   | 144             |
| 3            | 2020-02-01 | \-965                 | \-821           |
| 3            | 2020-03-01 | \-401                 | \-1222          |
| 3            | 2020-04-01 | 493                   | \-729           |

