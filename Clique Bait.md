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

#### Q4. What is the number of events for each event type?
```sql
SELECT
  e.event_type,
  i.event_name,
  COUNT(cookie_id) AS event_count
FROM
  clique_bait.events e
  INNER JOIN clique_bait.event_identifier i ON e.event_type = i.event_type
GROUP BY
  i.event_name,
  e.event_type
ORDER BY
  event_count DESC;
```

| event\_type | event\_name   | event\_count |
| ----------- | ------------- | ------------ |
| 1           | Page View     | 20928        |
| 2           | Add to Cart   | 8451         |
| 3           | Purchase      | 1777         |
| 4           | Ad Impression | 876          |
| 5           | Ad Click      | 702          |

#### Q5. What is the percentage of visits which have a purchase event?
```sql
WITH cte_visits_with_purchase AS(
  SELECT
    visit_id,
    MAX(
      CASE
        WHEN event_type = 3 THEN 1
        ELSE 0
      END
    ) AS purchase_flag
  FROM
    clique_bait.events
  GROUP BY
    visit_id
)
SELECT
  ROUND(100 * SUM(purchase_flag) :: NUMERIC / COUNT(*), 2) AS purchase_percentage
FROM
  cte_visits_with_purchase;
```
|purchase\_percentage|
|--------------------|
|49.86               |

#### Q6. What is the percentage of visits which view the checkout page but do not have a purchase event?
```sql
WITH cte_visits_with_checkout_no_purchase AS(
  SELECT
    visit_id,
    MAX(
      CASE
        WHEN event_type = 1
        AND page_id = 12 THEN 1
        ELSE 0
      END
    ) AS checkout_flag,
    MAX(
      CASE
        WHEN event_type = 3 THEN 1
        ELSE 0
      END
    ) AS purchase_flag
  FROM
    clique_bait.events
  GROUP BY
    visit_id
)
SELECT
  ROUND(
    100 * SUM(
      CASE
        WHEN purchase_flag = 0 THEN 1
        ELSE 0
      END
    ) :: NUMERIC / COUNT(*),
    2
  )
FROM
  cte_visits_with_checkout_no_purchase
WHERE
  checkout_flag = 1;
```
|round|
|-----|
|15.50|

#### Q7. What are the top 3 pages by number of views?
```sql
SELECT
  p.page_name,
  COUNT(e.visit_id) AS page_views
FROM
  clique_bait.events e
  INNER JOIN clique_bait.page_hierarchy p ON e.page_id = p.page_id
WHERE
  event_type = 1 --page view event type
GROUP BY
  p.page_name
ORDER BY
  page_views DESC
LIMIT
  3;
```
|page\_name  |page\_views|
|------------|-----------|
|All Products|3174       |
|Checkout    |2103       |
|Home Page   |1782       |

#### Q8. What is the number of views and cart adds for each product category?
```sql
SELECT
  p.product_category,
  SUM(
    CASE
      WHEN e.event_type = 1 THEN 1
      ELSE 0
    END
  ) AS page_views,
  SUM(
    CASE
      WHEN e.event_type = 2 THEN 1
      ELSE 0
    END
  ) AS cart_adds
FROM
  clique_bait.events e
  INNER JOIN clique_bait.page_hierarchy p ON e.page_id = p.page_id
WHERE
  p.product_category IS NOT NULL
GROUP BY
  p.product_category
ORDER BY
  page_views DESC;
```
|product\_category|page\_views|cart\_adds|
|-----------------|-----------|----------|
|Shellfish        |6204       |3792      |
|Fish             |4633       |2789      |
|Luxury           |3032       |1870      |

#### Q9. What are the top 3 products by purchases?
```sql
WITH cte_purchase_visits AS (
  SELECT
    visit_id
  FROM
    clique_bait.events
  WHERE
    event_type = 3
)
SELECT
  p.product_id,
  p.page_name AS product_name,
  SUM(
    CASE
      WHEN event_type = 2 THEN 1
      ELSE 0
    END
  ) AS purchases
FROM
  clique_bait.events e
  INNER JOIN clique_bait.page_hierarchy p ON e.page_id = p.page_id
WHERE
  EXISTS(
    SELECT
      visit_id
    FROM
      cte_purchase_visits
    WHERE
      e.visit_id = cte_purchase_visits.visit_id
  )
  AND p.product_id IS NOT NULL
GROUP BY
  p.product_id,
  product_name
ORDER BY
  p.product_id;
```
|product\_id|product\_name |purchases|
|-----------|--------------|---------|
|1          |Salmon        |711      |
|2          |Kingfish      |707      |
|3          |Tuna          |697      |
|4          |Russian Caviar|697      |
|5          |Black Truffle |707      |
|6          |Abalone       |699      |
|7          |Lobster       |754      |
|8          |Crab          |719      |
|9          |Oyster        |726      |



