## Clique Bait is not like your regular online seafood store - the founder and CEO Danny, was also a part of a digital data analytics team and wanted to expand his knowledge into the seafood industry!

## In this case study - you are required to support Danny’s vision and analyse his dataset and come up with creative solutions to calculate funnel fallout rates for the Clique Bait online store.

### Digital Analysis

#### Q1. How many users are there?
```sql
SELECT
  COUNT (DISTINCT user_id) AS total_users
FROM
  clique_bait.users;
```
| **total\_users** |
| ---------------- |
| 500              |

#### Q2. How many cookies does each user have on average?
```sql
WITH cte_cookies AS (
  SELECT
    user_id,
    COUNT(cookie_id) AS cookie_count
  FROM
    clique_bait.users
  GROUP BY
    user_id
)
SELECT
  ROUND(AVG(cookie_count), 2) AS avg_cookie_count
FROM
  cte_cookies;
```

| **avg\_cookie\_count** |
| ---------------------- |
| 3.56                   |

#### Q3. What is the unique number of visits by all users each month?
```sql
SELECT
  DATE_TRUNC('Month', event_time) AS month_start,
  COUNT (DISTINCT visit_id) AS unique_visits
FROM
  clique_bait.events
GROUP BY
  month_start
ORDER BY
  month_start;
```
| **month\_start**| **unique\_visits** |
| --------------  | ------------------ |
| **2020-01-01**  | 876                |
| **2020-02-01**  | 1488               |
| **2020-03-01**  | 916                |
| **2020-04-01**  | 248                |
| **2020-05-01**  | 36                 |
