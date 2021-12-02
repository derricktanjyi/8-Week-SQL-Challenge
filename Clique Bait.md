## Clique Bait is not like your regular online seafood store - the founder and CEO Danny, was also a part of a digital data analytics team and wanted to expand his knowledge into the seafood industry!

### In this case study - I am required to support Danny’s vision and analyse his dataset and come up with creative solutions to calculate funnel fallout rates for the Clique Bait online store.

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


### Product Funnel Analysis

#### Using a single SQL query - create a new output table which has the following details:
* How many times was each product viewed?
* How many times was each product added to cart?
* How many times was each product added to a cart but not purchased (abandoned)?
* How many times was each product purchased?

```sql
DROP TABLE IF EXISTS product_info;
CREATE TEMP TABLE product_info AS WITH cte_product_page_events AS ( --Creating The Table
  SELECT
    events.visit_id,
    page_hierarchy.product_id,
    page_hierarchy.page_name,
    page_hierarchy.product_category,
    SUM(
      CASE
        WHEN event_type = 1 THEN 1
        ELSE 0
      END
    ) AS page_view,
    SUM(
      CASE
        WHEN event_type = 2 THEN 1
        ELSE 0
      END
    ) AS cart_add
  FROM
    clique_bait.events
    INNER JOIN clique_bait.page_hierarchy ON events.page_id = page_hierarchy.page_id
  WHERE
    page_hierarchy.product_id IS NOT NULL
  GROUP BY
    events.visit_id,
    page_hierarchy.product_id,
    page_hierarchy.page_name,
    page_hierarchy.product_category
),
cte_visit_purchase AS (
  SELECT
    DISTINCT visit_id
  FROM
    clique_bait.events
  WHERE
    event_type = 3 -- purchase event
),
cte_combined_product_events AS (
  SELECT
    t1.visit_id,
    t1.product_id,
    t1.page_name,
    t1.product_category,
    t1.page_view,
    t1.cart_add,
    CASE
      WHEN t2.visit_id IS NULL THEN 0
      ELSE 1
    END as purchase
  FROM
    cte_product_page_events AS t1
    LEFT JOIN cte_visit_purchase AS t2 ON t1.visit_id = t2.visit_id
)
SELECT --Showing the Table
  product_id,
  page_name AS product,
  product_category,
  SUM(page_view) AS page_views,
  SUM(cart_add) AS cart_adds,
  SUM(
    CASE
      WHEN cart_add = 1
      AND purchase = 0 THEN 1
      ELSE 0
    END
  ) AS abandoned,
  SUM(
    CASE
      WHEN cart_add = 1
      AND purchase = 1 THEN 1
      ELSE 0
    END
  ) AS purchases
FROM
  cte_combined_product_events
GROUP BY
  product_id,
  product_category,
  product;
SELECT
  *
FROM
  product_info
ORDER BY
  product_id;
```

| product\_id | product        | product\_category | page\_views | cart\_adds | abandoned | purchases |
| ----------- | -------------- | ----------------- | ----------- | ---------- | --------- | --------- |
| 1           | Salmon         | Fish              | 1559        | 938        | 227       | 711       |
| 2           | Kingfish       | Fish              | 1559        | 920        | 213       | 707       |
| 3           | Tuna           | Fish              | 1515        | 931        | 234       | 697       |
| 4           | Russian Caviar | Luxury            | 1563        | 946        | 249       | 697       |
| 5           | Black Truffle  | Luxury            | 1469        | 924        | 217       | 707       |
| 6           | Abalone        | Shellfish         | 1525        | 932        | 233       | 699       |
| 7           | Lobster        | Shellfish         | 1547        | 968        | 214       | 754       |
| 8           | Crab           | Shellfish         | 1564        | 949        | 230       | 719       |
| 9           | Oyster         | Shellfish         | 1568        | 943        | 217       | 726       |


#### Additionally, create another table which further aggregates the data for the above points but this time for each product category instead of individual products.

```sql
DROP TABLE IF EXISTS product_category_info;
CREATE TEMP TABLE product_category_info AS
SELECT
  product_category,
  SUM(page_views) AS page_views,
  SUM(cart_adds) AS cart_adds,
  SUM(abandoned) AS abandoned,
  SUM(purchases) AS purchases
FROM
  product_info
GROUP BY
  product_category;
SELECT
  *
FROM
  product_info
ORDER BY
  product_id;
SELECT
  *
FROM
  product_category_info
ORDER BY
  product_category;
```
| product\_category | page\_views | cart\_adds | abandoned | purchases |
| ----------------- | ----------- | ---------- | --------- | --------- |
| Fish              | 4633        | 2789       | 674       | 2115      |
| Luxury            | 3032        | 1870       | 466       | 1404      |
| Shellfish         | 6204        | 3792       | 894       | 2898      |

#### Q1. Which Product had the most views, cart adds and purchases?
```sql
-- Most Views
SELECT
  *
FROM
  product_info
ORDER BY
  page_views DESC
LIMIT
  1;
-- Most Cart Adds
SELECT
  *
FROM
  product_info
ORDER BY
  cart_adds DESC
LIMIT
  1;
-- Most Purchases
SELECT
  *
FROM
  product_info
ORDER BY
  purchases DESC
LIMIT
  1;
```
#### Most Views
|product\_id|product|product\_category|page\_views|cart\_adds|abandoned|purchases|
|-----------|-------|-----------------|-----------|----------|---------|---------|
|9          |Oyster |Shellfish        |1568       |943       |217      |726      |

#### Most Cart Adds
|product\_id|product|product\_category|page\_views|cart\_adds|abandoned|purchases|
|-----------|-------|-----------------|-----------|----------|---------|---------|
|7          |Lobster|Shellfish        |1547       |968       |214      |754      |

#### Most Purchases
|product\_id|product|product\_category|page\_views|cart\_adds|abandoned|purchases|
|-----------|-------|-----------------|-----------|----------|---------|---------|
|7          |Lobster|Shellfish        |1547       |968       |214      |754      |

#### Q2. Which product was most likely to be abandoned?
```sql
SELECT
  product,
  ROUND((abandoned / cart_adds), 2) AS abandoned_likelihood
FROM
  product_info
ORDER BY
  abandoned_likelihood DESC
LIMIT
  1;
```
|product       |abandoned\_likelihood|
|--------------|---------------------|
|Russian Caviar|0.26                 |

#### Q3. Which product had the highest view to purchase percentage?
```sql
SELECT
  product,
  ROUND((100 * purchases / page_views), 2) AS view_purchase_percentage
FROM
  product_info
ORDER BY
  view_purchase_percentage DESC
LIMIT
  1
```
|product|view\_purchase\_percentage|
|-------|--------------------------|
|Lobster|48.74                     |

#### Q4. What is the average conversion rate from view to cart add?
```sql
SELECT
  ROUND(AVG((100 * cart_adds / page_views)), 2) AS avg_view_to_cart_add
FROM
  product_info;
```
|avg\_view\_to\_cart\_add|
|------------------------|
|60.95                   |

#### Q5. What is the average conversion rate from cart add to purchase?
```sql
SELECT
  ROUND(AVG((100 * purchases / cart_adds)), 2) AS avg_cart_adds_to_purchases
FROM
  product_info;
```
|avg\_cart\_adds\_to\_purchases|
|------------------------------|
|75.93                         |

### Campaigns Analysis

#### Generate a table that has 1 single row for every unique visit_id record and has the following columns:
* user_id
* visit_id
* visit_start_time: the earliest event_time for each visit
* page_views: count of page views for each visit
* cart_adds: count of product cart add events for each visit
* purchase: 1/0 flag if a purchase event exists for each visit
* campaign_name: map the visit to a campaign if the visit_start_time falls between the start_date and end_date
* impression: count of ad impressions for each visit
* click: count of ad clicks for each visit
* cart_products: a comma separated text value with products added to the cart sorted by the order they were added to the cart

```sql
SELECT
  u.user_id,
  e.visit_id,
  MIN(e.event_time) AS visit_start_time,
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
  ) AS cart_adds,
  SUM(
    CASE
      WHEN e.event_type = 3 THEN 1
      ELSE 0
    END
  ) AS purchase,
  c.campaign_name,
  MAX(
    CASE
      WHEN e.event_type = 4 THEN 1
      ELSE 0
    END
  ) AS impression,
  MAX(
    CASE
      WHEN e.event_type = 5 THEN 1
      ELSE 0
    END
  ) AS click,
  STRING_AGG(
    CASE
      WHEN p.product_id IS NOT NULL
      AND event_type = 2 THEN p.page_name
      ELSE NULL
    END,
    ', '
    ORDER BY
      e.sequence_number
  ) AS cart_products
FROM
  clique_bait.events e
  INNER JOIN clique_bait.users u ON e.cookie_id = u.cookie_id
  LEFT JOIN clique_bait.campaign_identifier c ON e.event_time BETWEEN c.start_date
  AND c.end_date
  LEFT JOIN clique_bait.page_hierarchy p ON e.page_id = p.page_id
GROUP BY
  u.user_id,
  e.visit_id,
  c.campaign_name
LIMIT
  5;
```
|**user\_id**|**visit\_id**|**visit\_start\_time** |**page\_views**|**cart\_adds**|**purchase**|**campaign\_name**               |**impression**|**click**|**cart\_products**                                            |
|------------|-------------|-----------------------|---------------|--------------|------------|---------------------------------|--------------|---------|--------------------------------------------------------------|
|**1**       |02a5d5       |2020-02-26 16:57:26.261|4              |0             |0           |Half Off - Treat Your Shellf(ish)|0             |0        |                                                              |
|**1**       |0826dc       |2020-02-26 05:58:37.919|1              |0             |0           |Half Off - Treat Your Shellf(ish)|0             |0        |                                                              |
|**1**       |0fc437       |2020-02-04 17:49:49.603|10             |6             |1           |Half Off - Treat Your Shellf(ish)|1             |1        |Tuna, Russian Caviar, Black Truffle, Abalone, Crab, Oyster    |
|**1**       |30b94d       |2020-03-15 13:12:54.024|9              |7             |1           |Half Off - Treat Your Shellf(ish)|1             |1        |Salmon, Kingfish, Tuna, Russian Caviar, Abalone, Lobster, Crab|
|**1**       |41355d       |2020-03-25 00:11:17.861|6              |1             |0           |Half Off - Treat Your Shellf(ish)|0             |0        |Lobster                                                       |
