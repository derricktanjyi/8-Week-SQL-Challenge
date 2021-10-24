### Context
Danny seriously loves Japanese food so in the beginning of 2021, he decides to embark upon a risky venture and opens up a cute little restaurant that sells his 3 favourite foods: sushi, curry and ramen.

Danny’s Diner is in need of assistance to help the restaurant stay afloat - the restaurant has captured some very basic data from their few months of operation but have no idea how to use their data to help them run the business.

### Problem Statement
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. Having this deeper connection with his customers will help him deliver a better and more personalised experience for his loyal customers.

He plans on using these insights to help him decide whether he should expand the existing customer loyalty program - additionally he needs help to generate some basic datasets so his team can easily inspect the data without needing to use SQL.

### Example Datasets
Danny has shared 3 key datasets for this case study:

sales
menu
members


#### Q1. What is the total amount each customer spent at the restaurant?

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

|customer\_id|total\_sales|
|------------|------------|
|A           |76          |
|B           |74          |
|C           |36          |
