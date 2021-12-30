## Context
Danny was scrolling through his Instagram feed when something really caught his eye - “80s Retro Styling and Pizza Is The Future!”

Danny was sold on the idea, but he knew that pizza alone was not going to help him get seed funding to expand his new Pizza Empire - so he had one more genius idea to combine with it - he was going to Uberize it - and so Pizza Runner was launched!

Danny started by recruiting “runners” to deliver fresh pizza from Pizza Runner Headquarters (otherwise known as Danny’s house) and also maxed out his credit card to pay freelance developers to build a mobile app to accept orders from customers.

## Questions
### Part A. Pizza Metrics
#### Q1. How many pizzas were ordered?
```sql 
SELECT COUNT(*) FROM pizza_runner.customer_orders;
```
| count |
| ----- |
| 14    |

#### Q2. How many unique customer orders were made?
```sql
SELECT COUNT(DISTINCT order_id) FROM pizza_runner.customer_orders;
```
| count |
| ----- |
| 10    |

#### Q3. How many successful orders were delivered by each runner?
```sql
SELECT
  runner_id,
  (COUNT(DISTINCT order_id)) AS successful_orders
FROM
  new_runner_orders
WHERE
  cancellation IS NULL
GROUP BY
  runner_id;
 ```
 | runner\_id | successful\_orders |
| ---------- | ------------------ |
| 1          | 4                  |
| 2          | 3                  |
| 3          | 1                  |

#### Q4. How many of each type of pizza was delivered?
```sql
SELECT
  t2.pizza_name,
  COUNT(t1.*) AS delivered_pizza_count
FROM
  pizza_runner.customer_orders AS t1
  INNER JOIN pizza_runner.pizza_names AS t2 ON t1.pizza_id = t2.pizza_id
WHERE EXISTS (
    SELECT 1 FROM pizza_runner.runner_orders AS t3
    WHERE t1.order_id = t3.order_id
      AND t3.cancellation IS NOT NULL
      AND t3.cancellation NOT IN (
        'Restaurant Cancellation',
        'Customer Cancellation'
      )
  )
GROUP BY t2.pizza_name
ORDER BY t2.pizza_name;
```
| pizza\_name | delivered\_pizza\_count |
| ----------- | ----------------------- |
| Meatlovers  | 9                       |
| Vegetarian  | 3                       |

#### Q5. How many Vegetarian and Meatlovers were ordered by each customer?
```sql
SELECT
  customer_id,
  SUM(
    CASE
      WHEN pizza_id = 1 THEN 1
      ELSE 0
    END
  ) AS meatlovers,
  SUM(
    CASE
      WHEN pizza_id = 2 THEN 2
      ELSE 0
    END
  ) AS vegetarian
FROM
  pizza_runner.customer_orders
GROUP BY
  customer_id
ORDER BY
  customer_id;
 ```
 | customer\_id | meatlovers | vegetarian |
| ------------ | ---------- | ---------- |
| 101          | 2          | 2          |
| 102          | 2          | 2          |
| 103          | 3          | 2          |
| 104          | 3          | 0          |
| 105          | 0          | 2          |

#### Q6. What was the maximum number of pizzas delivered in a single order?
```sql
SELECT
  c.order_id,
  COUNT(*) AS pizzas_delivered
FROM
  new_customer_orders c
  JOIN new_runner_orders r ON c.order_id = r.order_id
WHERE
  cancellation IS NULL
GROUP BY
  c.order_id
ORDER BY
  pizzas_delivered DESC
LIMIT
  1;
```
| order\_id | pizzas\_delivered |
| --------- | ----------------- |
| 4         | 3                 |

#### Q7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
```sql
SELECT
  c.customer_id,
  COUNT(
    CASE
      WHEN c.exclusions IS NOT NULL
      OR c.extras IS NOT NULL THEN 1
    END
  ) AS at_least_1_change,
  COUNT(
    CASE
      WHEN c.exclusions IS NULL
      AND c.extras IS NULL THEN 1
    END
  ) AS no_change
FROM
  new_customer_orders AS c
  INNER JOIN new_runner_orders AS r ON c.order_id = r.order_id
WHERE
  cancellation IS NULL
GROUP BY
  c.customer_id;
```
| customer\_id | at\_least\_1\_change | no\_change |
| ------------ | -------------------- | ---------- |
| 101          | 0                    | 2          |
| 102          | 0                    | 3          |
| 103          | 3                    | 0          |
| 104          | 2                    | 1          |
| 105          | 1                    | 0          |

#### Q8. How many pizzas were delivered that had both exclusions and extras?
```sql
SELECT
  COUNT(*) AS pizzas_delivered
FROM
  new_customer_orders c
  INNER JOIN new_runner_orders r ON c.order_id = r.order_id
WHERE
  cancellation IS NULL
  AND c.exclusions IS NOT NULL
  AND c.extras IS NOT NULL;
```
| pizzas\_delivered |
| ----------------- |
| 1                 |

#### Q9. What was the total volume of pizzas ordered for each hour of the day?
```sql
SELECT
  DATE_PART('hour', order_time :: TIMESTAMP) AS hour_of_day,
  COUNT(*) pizza_ordered
FROM
  pizza_runner.customer_orders
GROUP BY
  hour_of_day
ORDER BY
  hour_of_day;
```
| hour\_of\_day | pizza\_ordered |
| ------------- | -------------- |
| 11            | 1              |
| 13            | 3              |
| 18            | 3              |
| 19            | 1              |
| 21            | 3              |
| 23            | 3              |

#### Q10. What was the volume of orders for each day of the week?
```sql
SELECT
  TO_CHAR(order_time, 'Day') As day_of_week,
  COUNT(*) As pizzas_ordered
FROM
  pizza_runner.customer_orders
GROUP BY
  day_of_week,
  DATE_PART('dow', order_time)
ORDER BY
  DATE_PART('dow', order_time);
```
| day\_of\_week | pizzas\_ordered |
| ------------- | --------------- |
| Sunday        | 1               |
| Monday        | 5               |
| Friday        | 5               |
| Saturday      | 3               |

### Part B. Runner and Customer Experience
#### Q1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)
```sql
SELECT
  DATE_TRUNC('week', registration_date) :: DATE + 4 AS registration_week,
  COUNT(*) AS runners
FROM
  pizza_runner.runners
GROUP BY
  registration_week
ORDER BY
  registration_week;
```
| registration\_week| runners |
| ------------------| ------- |
| 2021-01-01        | 2       |
| 2021-01-08        | 1       |
| 2021-01-15        | 1       |

#### Q2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
```sql
WITH cte_pickup_time AS(
  SELECT DISTINCT
    r.order_id,
    (
      DATE_PART(
        'minute',
        r.pickup_time :: timestamp - c.order_time :: timestamp
      )
    ) :: INTEGER as pickup_mins
  FROM
    new_runner_orders r
    INNER JOIN new_customer_orders c ON r.order_id = c.order_id
  WHERE
    r.pickup_time IS NOT NULL
)
SELECT
  ROUND(AVG(pickup_mins), 3) AS avg_pickup_mins
FROM
  cte_pickup_time;
```
| avg\_pickup\_mins |
| ----------------- |
| 15.625            |

#### Q3. Is there any relationship between the number of pizzas and how long the order takes to prepare?
```sql
SELECT
  DISTINCT r.order_id,
  DATE_PART(
    'minute',
    AGE(
      r.pickup_time :: TIMESTAMP,
      c.order_time :: TIMESTAMP
    )
  ) :: INTEGER AS pickup_mins,
  COUNT(c.order_id) AS pizza_count
FROM
  new_runner_orders r
  INNER JOIN new_customer_orders c ON r.order_id = c.order_id
WHERE
  r.pickup_time IS NOT NULL
GROUP BY
  r.order_id,
  pickup_mins
ORDER BY
  pizza_count;
```
| order\_id | pickup\_mins | pizza\_count |
| --------- | ------------ | ------------ |
| 1         | 10           | 1            |
| 2         | 10           | 1            |
| 5         | 10           | 1            |
| 7         | 10           | 1            |
| 8         | 20           | 1            |
| 3         | 21           | 2            |
| 10        | 15           | 2            |
| 4         | 29           | 3            |

#### Q4. What was the average distance travelled for each customer?
```sql
WITH cte_customer_order_distance AS (
  SELECT
    DISTINCT c.customer_id,
    c.order_id,
    r.distance :: NUMERIC AS distance_travel
  FROM
    new_customer_orders c
    INNER JOIN new_runner_orders r ON c.order_id = r.order_id
  WHERE
    r.pickup_time IS NOT NULL
)
SELECT
  customer_id,
  ROUND(AVG(distance_travel), 1) AS avg_distance
FROM
  cte_customer_order_distance
GROUP BY
  customer_id
ORDER BY
  customer_id;
```
| customer\_id | avg\_distance |
| ------------ | ------------- |
| 101          | 20.0          |
| 102          | 18.4          |
| 103          | 23.4          |
| 104          | 10.0          |
| 105          | 25.0          |

#### Q5. What was the difference between the longest and shortest delivery times for all orders?
```sql
WITH cte_duration AS (
  SELECT
    duration :: NUMERIC AS duration
  FROM
    new_runner_orders
  WHERe
    pickup_time IS NOT NULL
)
SELECT
  MAX(duration) - MIN(duration) AS max_difference
FROM
  cte_duration;
```
| max\_difference |
| --------------- |
| 30              |

#### Q6. What was the average speed for each runner for each delivery and do you notice any trend for these values?
```sql
SELECT
  runner_id,
  order_id,
  DATE_PART('hour', pickup_time :: TIMESTAMP) As hour_of_day,
  distance,
  duration,
  ROUND((distance / duration), 1) AS avg_speed
FROM
  new_runner_orders
WHERE
  pickup_time IS NOT NULL
ORDER BY
  runner_id;
```
| runner\_id | order\_id | hour\_of\_day | distance | duration | avg\_speed |
| ---------- | --------- | ------------- | -------- | -------- | ---------- |
| 1          | 1         | 18            | 20       | 32       | 0.6        |
| 1          | 2         | 19            | 20       | 27       | 0.7        |
| 1          | 10        | 18            | 10       | 10       | 1.0        |
| 1          | 3         | 0             | 13.4     | 20       | 0.7        |
| 2          | 4         | 13            | 23.4     | 40       | 0.6        |
| 2          | 7         | 21            | 25       | 25       | 1.0        |
| 2          | 8         | 0             | 23.4     | 15       | 1.6        |
| 3          | 5         | 21            | 10       | 15       | 0.7        |

#### Q7. What is the successful delivery percentage for each runner?
```sql
SELECT
  runner_id,
  ROUND(
    (
      100 * SUM(
        CASE
          WHEN pickup_time IS NOT NULL THEN 1
          ELSE 0
        END
      ) / COUNT(*)
    ),
    1
  ) AS success_percentage
FROM
  new_runner_orders
GROUP BY
  runner_id
ORDER BY
  runner_id,
  success_percentage;
```
| runner\_id | success\_percentage |
| ---------- | ------------------- |
| 1          | 100.0               |
| 2          | 75.0                |
| 3          | 50.0                |

### Part C. Ingredient Optimisation
#### Q1. What are the standard ingredients for each pizza?
```sql
WITH cte_split_pizza_names AS (
  SELECT
    pizza_id,
    REGEXP_SPLIT_TO_TABLE(toppings, '[,\s]+') :: INTEGER AS topping_id
  FROM
    pizza_runner.pizza_recipes
)
SELECT
  pizza_id,
  STRING_AGG(t.topping_name :: TEXT, ',') AS standard_ingredients
FROM
  cte_split_pizza_names p
  INNER JOIN pizza_runner.pizza_toppings t ON p.topping_id = t.topping_id
GROUP BY
  pizza_id
ORDER BY
  pizza_id;
```
| pizza\_id | standard\_ingredients                                          |
| --------- | -------------------------------------------------------------- |
| 1         | Bacon,BBQ Sauce,Beef,Cheese,Chicken,Mushrooms,Pepperoni,Salami |
| 2         | Cheese,Mushrooms,Onions,Peppers,Tomatoes,Tomato Sauce          |

#### Q2. What was the most commonly added extra?
```sql
WITH cte_extras AS (
  SELECT
    REGEXP_SPLIT_TO_TABLE(extras, '[,\s]+') :: INTEGER AS topping_id
  FROM
    new_customer_orders
)
SELECT
  topping_name,
  COUNT(*) AS extras_count
FROM
  cte_extras e
  INNER JOIN pizza_runner.pizza_toppings t ON e.topping_id = t.topping_id
GROUP BY
  topping_name
ORDER BY
  topping_name;
```
| topping\_name | extras\_count |
| ------------- | ------------- |
| Bacon         | 4             |
| Cheese        | 1             |
| Chicken       | 1             |

#### Q3. What was the most common exclusion?
```sql
WITH cte_exclusions AS (
  SELECT
    REGEXP_SPLIT_TO_TABLE(exclusions, '[,\s]+') :: INTEGER AS topping_id
  FROM
    new_customer_orders
)
SELECT
  topping_name,
  COUNT(*) AS exclusions_count
FROM
  cte_exclusions e
  INNER JOIN pizza_runner.pizza_toppings t ON e.topping_id = t.topping_id
GROUP BY
  topping_name
ORDER BY
  exclusions_count DESC;
```
| topping\_name | exclusions\_count |
| ------------- | ----------------- |
| Cheese        | 4                 |
| Mushrooms     | 1                 |
| BBQ Sauce     | 1                 |

### Part D. Pricing and Ratings
#### Q1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?
```sql
SELECT
  SUM(
    CASE
      WHEN pizza_id = 1 THEN 12
      WHEN pizza_id = 2 THEN 10
    END
  ) AS revenue
FROM
  new_customer_orders;
```
| revenue |
| ------- |
| 160     |

#### Q2. What if there was an additional $1 charge for any pizza extras? + Add cheese is $1 extra
```sql
SELECT
  SUM(
    CASE
      WHEN pizza_id = 1 THEN 12
      WHEN pizza_id = 2 THEN 10
    END - COALESCE(
      CARDINALITY(REGEXP_SPLIT_TO_ARRAY(extras, '[,\s]+')),
      1
    )
  ) AS cost
FROM
  new_customer_orders;
```
| cost |
| ---- |
| 144  |

#### Q3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.
```sql
SELECT
  SETSEED(1);
DROP TABLE IF EXISTS pizza_runner.ratings;
CREATE TABLE pizza_runner.ratings ("order_id" INTEGER, "rating" INTEGER);
INSERT INTO
  pizza_runner.ratings
SELECT
  order_id,
  FLOOR(1 + 5 * RANDOM()) AS rating
FROM
  pizza_runner.runner_orders
WHERE
  pickup_time IS NOT NULL;
SELECT
  *
FROM
  pizza_runner.ratings;
```
| order\_id | rating |
| --------- | ------ |
| 1         | 3      |
| 2         | 4      |
| 6         | 4      |
| 7         | 3      |
| 8         | 3      |
| 9         | 2      |
| 10        | 2      |
| 3         | 3      |
| 4         | 2      |
| 5         | 5      |
