## Context 
Subscription based businesses are super popular and Danny realised that there was a large gap in the market - he wanted to create a new streaming service that only had food related content - something like Netflix but with only cooking shows!

## Problem
Danny finds a few smart friends to launch his new startup Foodie-Fi in 2020 and started selling monthly and annual subscriptions, giving their customers unlimited on-demand access to exclusive food videos from around the world!

Danny created Foodie-Fi with a data driven mindset and wanted to ensure all future investment decisions and new features were decided using data. This case study focuses on using subscription style digital data to answer important business questions.

### This case study is split into an initial data understanding question and thereafter data analysis questions.

### Part A. Based off the 8 sample customers provided in the sample from the `subscriptions` table, write a brief description about each customer’s onboarding journey.

```sql 
SELECT
  s.customer_id,
  s.plan_id,
  p.plan_name,
  s.start_Date,
  p.price
FROM
  foodie_fi.subscriptions s
  INNER JOIN foodie_fi.plans p ON s.plan_id = p.plan_id
WHERE customer_id IN (1, 2, 11, 13, 15, 16, 18, 19)
ORDER BY
  customer_id,
  start_date;
```
#### Result:

| customer\_id | plan\_id | plan\_name    | start\_date| price  |
| ------------ | -------- | ------------- | ---------- | ------ |
| 1            | 0        | trial         | 2020-08-01 | 0.00   |
| 1            | 1        | basic monthly | 2020-08-08 | 9.90   |
| 2            | 0        | trial         | 2020-09-20 | 0.00   |
| 2            | 3        | pro annual    | 2020-09-27 | 199.00 |
| 11           | 0        | trial         | 2020-11-19 | 0.00   |
| 11           | 4        | churn         | 2020-11-26 | null   |
| 13           | 0        | trial         | 2020-12-15 | 0.00   |
| 13           | 1        | basic monthly | 2020-12-22 | 9.90   |
| 13           | 2        | pro monthly   | 2021-03-29 | 19.90  |
| 15           | 0        | trial         | 2020-03-17 | 0.00   |
| 15           | 2        | pro monthly   | 2020-03-24 | 19.90  |
| 15           | 4        | churn         | 2020-04-29 | null   |
| 16           | 0        | trial         | 2020-05-31 | 0.00   |
| 16           | 1        | basic monthly | 2020-06-07 | 9.90   |
| 16           | 3        | pro annual    | 2020-10-21 | 199.00 |
| 18           | 0        | trial         | 2020-07-06 | 0.00   |
| 18           | 2        | pro monthly   | 2020-07-13 | 19.90  |
| 19           | 0        | trial         | 2020-06-22 | 0.00   |
| 19           | 2        | pro monthly   | 2020-06-29 | 19.90  |
| 19           | 3        | pro annual    | 2020-08-29 | 199.00 |

### Part B: Data Analysis Questions

### How many customers has Foodie-Fi ever had?

```sql
SELECT
  COUNT(DISTINCT customer_id) AS customer_count
FROM
  foodie_fi.subscriptions;
```
#### Result:

| customer\_count |
| --------------- |
| 1000            |

### What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value

```sql
SELECT
  DATE_TRUNC('month', start_date) AS month_start,
  COUNT(*) AS trial_customers
FROM
  foodie_fi.subscriptions
WHERE
  plan_id = 0
GROUP BY
  DATE_TRUNC('month', start_date)
ORDER BY
  month_start;
```
#### Result:

| month\_start | trial\_customers |
| ------------ | ---------------- |
| 2020-01-01   | 88               |
| 2020-02-01   | 68               |
| 2020-03-01   | 94               |
| 2020-04-01   | 81               |
| 2020-05-01   | 88               |
| 2020-06-01   | 79               |
| 2020-07-01   | 89               |
| 2020-08-01   | 88               |
| 2020-09-01   | 87               |
| 2020-10-01   | 79               |
| 2020-11-01   | 75               |
| 2020-12-01   | 84               |

### What plan `start_date` values occur after the year 2020 for our dataset? Show the breakdown by count of events for each `plan_name`

```sql
SELECT
  s.plan_id,
  p.plan_name,
  COUNT(*) events
FROM
  foodie_fi.plans p
  INNER JOIN foodie_fi.subscriptions s ON p.plan_id = s.plan_id
WHERE
  start_date > '2020-12-31'
GROUP BY
  s.plan_id,
  p.plan_name
ORDER BY
  s.plan_id;
```
#### Result:

| plan\_id | plan\_name    | events |
| -------- | ------------- | ------ |
| 1        | basic monthly | 8      |
| 2        | pro monthly   | 60     |
| 3        | pro annual    | 63     |
| 4        | churn         | 71     |

### What is the customer count and percentage of customers who have churned rounded to 1 decimal place?

```sql
SELECT
  SUM(
    CASE
      WHEN plan_id = 4 THEN 1
      ELSE 0
    END
  ) AS churn_customers,
  ROUND(
    100 * SUM(
      CASE
        WHEN plan_id = 4 THEN 1
        ELSE 0
      END
    ) / COUNT(DISTINCT customer_id),
    1
  ) AS percentage
FROM
  foodie_fi.subscriptions;
```
#### Result:

| churn\_customers | percentage |
| ---------------- | ---------- |
| 307              | 30.0       |

### How many customers have churned straight after their initial free trial - what percentage is this rounded to 1 decimal place?

```sql
WITH ranked_plans AS (
  SELECT
    customer_id,
    plan_id,
    ROW_NUMBER () OVER(
      PARTITION BY customer_id
      ORDER BY
        start_date
    ) As plan_rank
  FROM
    foodie_fi.subscriptions
)
SELECT
  SUM(
    CASE
      WHEN plan_id = 4 THEN 1
      ELSE 0
    END
  ) AS churn_customers,
  ROUND(
    100 * SUM(
      CASE
        WHEN plan_id = 4 THEN 1
        ELSE 0
      END
    ) / COUNT(DISTINCT customer_id),
    1
  ) AS percentage
FROM
  ranked_plans
WHERE
  plan_rank = 2
```
#### Result:

| churn\_customers | percentage |
| ---------------- | ---------- |
| 92               | 9.0        |

### What is the number and percentage of customer plans after their initial free trial?
```sql
WITH ranked_plans AS(
  SELECT
    customer_id,
    plan_id,
    ROW_NUMBER () OVER(
      PARTITION BY customer_id
      ORDER BY
        start_date DESC
    ) As plan_rank
  FROM
    foodie_fi.subscriptions
)
SELECT
  p.plan_id,
  p.plan_name,
  COUNT(*) AS customer_count,
  ROUND(100 * COUNT(*) / SUM(COUNT(*)) OVER()) AS percentage
FROM
  ranked_plans r
  INNER JOIN foodie_fi.plans p ON r.plan_id = p.plan_id
WHERE
  plan_rank = 1
GROUP BY
  p.plan_id,
  p.plan_name
ORDER BY
  p.plan_id
```
#### Result:

| plan\_id | plan\_name    | customer\_count | percentage |
| -------- | ------------- | --------------- | ---------- |
| 1        | basic monthly | 125             | 13         |
| 2        | pro monthly   | 316             | 32         |
| 3        | pro annual    | 252             | 25         |
| 4        | churn         | 307             | 31         |

### What is the customer count and percentage breakdown of all 5 `plan_name` values at `2020-12-31`?

```sql
WITH valid_subscriptions AS (
  SELECT
    customer_id,
    plan_id,
    start_date,
    ROW_NUMBER () OVER(
      PARTITION BY customer_id
      ORDER BY
        start_date DESC
    ) AS plan_rank
  FROM
    foodie_fi.subscriptions
  WHERE
    start_date <= '2020-12-31'
),
summarised_plans AS (
  SELECT
    plan_id,
    COUNT(DISTINCT customer_id) AS customers
  FROM
    valid_subscriptions
  GROUP BY
    plan_id
)
SELECT
  p.plan_id,
  p.plan_name,
  customers,
  ROUND(100 * customers / SUM(customers) OVER(), 1) AS percentage
FROM
  summarised_plans s
  INNER JOIN foodie_fi.plans p ON s.plan_id = p.plan_id
GROUP BY
  p.plan_id,
  p.plan_name,
  customers
ORDER BY
  p.plan_id;
```
#### Result:

| plan\_id | plan\_name    | customers | percentage |
| -------- | ------------- | --------- | ---------- |
| 0        | trial         | 1000      | 40.8       |
| 1        | basic monthly | 538       | 22.0       |
| 2        | pro monthly   | 479       | 19.6       |
| 3        | pro annual    | 195       | 8.0        |
| 4        | churn         | 236       | 9.6        |

### How many customers have upgraded to an annual plan in 2020?
```sql
SELECT
  COUNT(DISTINCT customer_id) AS customers_upgrade_annual
FROM
  foodie_fi.subscriptions
WHERE
  plan_id = 3
  AND start_date BETWEEN '2020-01-01'
  AND '2020-12-31';
```
#### Result:

| customers\_upgrade\_annual |
| -------------------------- |
| 195                        |

### How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?
```sql
WITH annual_plan AS (
  SELECT
    customer_id,
    start_date
  FROM
    foodie_fi.subscriptions
  WHERE
    plan_id = 3
),
trial_plan AS (
  SELECT
    customer_id,
    start_date
  FROM
    foodie_fi.subscriptions
  WHERE
    plan_id = 0
)
SELECT
  AVG(
    DATE_PART(
      'day',
      annual_plan.start_date :: TIMESTAMP - trial_plan.start_date :: TIMESTAMP
    )
  ) AS avg_days
FROM
  annual_plan
  CROSS JOIN trial_plan;
```
#### Result:

| avg\_days        |
| ---------------- |
| 95.0566821705426 |

### Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)
```sql
WITH annual_plan AS (
  SELECT
    customer_id,
    start_date
  FROM
    foodie_fi.subscriptions
  WHERE
    plan_id = 3
),
trial_plan AS(
  SELECT
    customer_id,
    start_date
  FROM
    foodie_fi.subscriptions
  WHERE
    plan_id = 0
),
annual_days AS (
  SELECT
    DATE_PART(
      'day',
      annual_plan.start_date :: TIMESTAMP - trial_plan.start_date :: TIMESTAMP
    ) :: INTEGER As duration
  FROM
    annual_plan
    INNER JOIN trial_plan ON annual_plan.customer_id = trial_plan.customer_id
)
SELECT
  CASE
    WHEN duration < 30 THEN '0 - 30 days'
    WHEN duration >= 30
    AND duration <= 60 THEN '30 - 60 days'
    WHEN duration >= 60
    AND duration <= 90 THEN '60 - 90 days'
    WHEN duration >= 90
    AND duration <= 120 THEN '90 - 120 days'
    WHEN duration >= 120
    AND duration <= 150 THEN '120 - 150 days'
    WHEN duration >= 150
    AND duration <= 180 THEN '150 - 180 days'
    WHEN duration >= 180
    AND duration <= 210 THEN '180 - 210 days'
    WHEN duration >= 210
    AND duration <= 240 THEN '210 - 240 days'
    WHEN duration >= 240
    AND duration <= 270 THEN '240 - 270 days'
    WHEN duration >= 270
    AND duration <= 300 THEN '270 -300 days'
    WHEN duration >= 300
    AND duration <= 330 THEN '300 - 330 days'
    WHEN duration >= 330
    AND duration <= 360 THEN '330 - 360 days'
    ELSE 'More than 360 days'
  END AS breakdown_period,
  COUNT(*) AS customers
FROM
  annual_days
GROUP BY
  breakdown_period
ORDER BY
  breakdown_period,
  customers DESC;
```
#### Result:

| breakdown\_period | customers |
| ----------------- | --------- |
| 0 - 30 days       | 48        |
| 120 - 150 days    | 42        |
| 150 - 180 days    | 36        |
| 180 - 210 days    | 26        |
| 210 - 240 days    | 4         |
| 240 - 270 days    | 5         |
| 270 -300 days     | 1         |
| 300 - 330 days    | 1         |
| 30 - 60 days      | 25        |
| 330 - 360 days    | 1         |
| 60 - 90 days      | 34        |
| 90 - 120 days     | 35        |
