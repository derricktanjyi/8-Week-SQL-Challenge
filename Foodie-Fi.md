### Subscription based businesses are super popular and Danny realised that there was a large gap in the market - he wanted to create a new streaming service that only had food related content - something like Netflix but with only cooking shows!

### Danny finds a few smart friends to launch his new startup Foodie-Fi in 2020 and started selling monthly and annual subscriptions, giving their customers unlimited on-demand access to exclusive food videos from around the world!

### Danny created Foodie-Fi with a data driven mindset and wanted to ensure all future investment decisions and new features were decided using data. This case study focuses on using subscription style digital data to answer important business questions.

### This case study is split into an initial data understanding question and thereafter data analysis questions.

#### Part A. Based off the 8 sample customers provided in the sample from the `subscriptions` table, write a brief description about each customer’s onboarding journey.

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

#### How many customers has Foodie-Fi ever had?

```sql
SELECT
  COUNT(DISTINCT customer_id) AS customer_count
FROM
  foodie_fi.subscriptions;
```

| customer\_count |
| --------------- |
| 1000            |
