## Context
Danny seriously loves Japanese food so in the beginning of 2021, he decides to embark upon a risky venture and opens up a cute little restaurant that sells his 3 favourite foods: sushi, curry and ramen.

Danny’s Diner is in need of assistance to help the restaurant stay afloat - the restaurant has captured some very basic data from their few months of operation but have no idea how to use their data to help them run the business.

## Problem Statement
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. Having this deeper connection with his customers will help him deliver a better and more personalised experience for his loyal customers.

He plans on using these insights to help him decide whether he should expand the existing customer loyalty program - additionally he needs help to generate some basic datasets so his team can easily inspect the data without needing to use SQL.

### Example Datasets
Danny has shared 3 key datasets for this case study:

* sales
* menu
* members


### What is the total amount each customer spent at the restaurant?

``` sql
SELECT
  s.customer_id,
  SUM(m.price) AS total_sales
FROM
  dannys_diner.sales s
  INNER JOIN dannys_diner.menu m ON s.product_id = m.product_id
GROUP BY
  s.customer_id
ORDER BY
  s.customer_id;
```
#### Result:

|customer\_id|total\_sales|
|------------|------------|
|A           |76          |
|B           |74          |
|C           |36          |

### What was the first item(s) from the menu purchased by each customer?

``` sql
WITH product_sales AS (
  SELECT
    s.customer_id,
    DENSE_RANK() OVER (
      PARTITION BY s.customer_id
      ORDER BY
        s.order_date
    ) AS order_rank,
    m.product_name
  FROM
    dannys_diner.sales s
    INNER JOIN dannys_diner.menu m ON s.product_id = m.product_id
)
SELECT
  DISTINCT customer_id,
  product_name
FROM
  product_sales
WHERE
  order_rank = 1;
```
#### Result:

|customer\_id|product\_name|
|------------|-------------|
|A           |curry        |
|A           |sushi        |
|B           |curry        |
|C           |ramen        |

### What is the most purchased item on the menu and how many times was it purchased by all customers?

``` sql
SELECT
  m.product_name,
  COUNT(s.*) AS total_purchases
FROM
  dannys_diner.sales s
  INNER JOIN dannys_diner.menu m ON s.product_id = m.product_id
GROUP BY
  m.product_name
ORDER BY
  total_purchases DESC
LIMIT
  1;
```
#### Result:

|product\_name|total\_purchases|
|-------------|----------------|
|ramen        |8               |

#### Which item(s) was the most popular for each customer?

``` sql
WITH customer_orders AS (
  SELECT
    s.customer_id,
    m.product_name,
    COUNT(s.*) AS order_quantity,
    DENSE_RANK() OVER (
      PARTITION BY s.customer_id
      ORDER BY
        COUNT(s.*) DESC
    ) AS orders_rank
  FROM
    dannys_diner.sales s
    INNER JOIN dannys_diner.menu m ON s.product_id = m.product_id
  GROUP BY
    s.customer_id,
    m.product_name
)
SELECT
  customer_id,
  product_name,
  order_quantity
FROM
  customer_orders
WHERE
  orders_rank = 1
ORDER BY
  customer_id ASC;
```
#### Result:

|customer\_id|product\_name|order\_quantity|
|------------|-------------|---------------|
|A           |ramen        |3              |
|B           |sushi        |2              |
|B           |curry        |2              |
|B           |ramen        |2              |
|C           |ramen        |3              |

### Which item was purchased first by the customer after they became a member and what date was it? (including the date they joined)

``` sql
WITH customer_purchases AS (
  SELECT
    s.customer_id,
    s.order_date,
    mu.product_name,
    RANK() OVER(
      PARTITION BY s.customer_id
      ORDER BY
        s.order_date,
        m.join_date
    ) AS purchases_rank
  FROM
    dannys_diner.menu mu
    INNER JOIN dannys_diner.sales s ON mu.product_id = s.product_id
    INNER JOIN dannys_diner.members m ON s.customer_id = m.customer_id
  WHERE
    s.order_date >= m.join_date
)
SELECT
  customer_id,
  order_date,
  product_name
FROM
  customer_purchases
WHERE
  purchases_rank = 1
ORDER BY
  customer_id;
```
#### Result:

|customer\_id|order\_date   |product\_name|
|------------|--------------|-------------|
|A           |2021-01-07    |curry        |
|B           |2021-01-11    |sushi        |

### Which menu item(s) was purchased just before the customer became a member and when?

``` sql
WITH customer_purchases AS(
  SELECT
    s.customer_id,
    s.order_date,
    mu.product_name,
    RANK() OVER(
      PARTITION BY s.customer_id
      ORDER BY
        s.order_date DESC
    ) AS purchases_rank
  FROM
    dannys_diner.sales s
    INNER JOIN dannys_diner.menu mu ON s.product_id = mu.product_id
    INNER JOIN dannys_diner.members m ON s.customer_id = m.customer_id
  WHERE
    s.order_date < m.join_date :: DATE
)
SELECT
  customer_id,
  order_date,
  product_name
FROM
  customer_purchases
WHERE
  purchases_rank = 1
ORDER BY
  customer_id,
  product_name;
```
#### Result:

|customer\_id|order\_date   |product\_name|
|------------|--------------|-------------|
|A           |2021-01-01    |curry        |
|A           |2021-01-01    |sushi        |
|B           |2021-01-04    |sushi        |

### What is the number of unique menu items and total amount spent for each member before they became a member?

``` sql
SELECT
  s.customer_id,
  COUNT(DISTINCT s.product_id) AS unique_menu_items,
  SUM(mu.price) AS total_spend
FROM
  dannys_diner.sales s
  INNER JOIN dannys_diner.menu mu ON s.product_id = mu.product_id
  INNER JOIN dannys_diner.members m ON s.customer_id = m.customer_id
WHERE
  s.order_date < m.join_date :: DATE
GROUP BY
  s.customer_id;
```
#### Result:

|customer\_id|unique\_menu\_items|total\_spend|
|------------|-------------------|------------|
|A           |2                  |25          |
|B           |2                  |40          |

#### If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

``` sql
SELECT
  s.customer_id,
  SUM (
    CASE
      WHEN mu.product_name = 'sushi' THEN 2 * 10 * mu.price
      ELSE 10 * mu.price
    END
  ) AS points
FROM
  dannys_diner.sales s
  LEFT JOIN dannys_diner.menu mu ON s.product_id = mu.product_id
GROUP BY
  s.customer_id
ORDER BY
  points DESC;
```
#### Result:

|customer\_id|points|
|------------|------|
|B           |940   |
|A           |860   |
|C           |360   |

### In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

``` sql
SELECT
  s.customer_id,
  SUM (
    CASE
      WHEN s.order_date BETWEEN (m.join_date :: DATE)
      AND m.join_date :: DATE + 6 THEN 2 * 10 * mu.price
    END
  ) AS points
FROM
  dannys_diner.sales s
  INNER JOIN dannys_diner.menu mu ON s.product_id = mu.product_id
  INNER JOIN dannys_diner.members m ON s.customer_id = m.customer_id
GROUP BY
  s.customer_id
ORDER BY
  points DESC;
```
#### Result:

|customer\_id|points|
|------------|------|
|A           |1020  |
|B           |200   |
