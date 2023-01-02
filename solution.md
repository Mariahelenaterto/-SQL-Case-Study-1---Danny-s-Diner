-- 1. What is the total amount each customer spent at the restaurant?
-- 2. How many days has each customer visited the restaurant?
-- 3. What was the first item from the menu purchased by each customer?
-- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
-- 5. Which item was the most popular for each customer?
-- 6. Which item was purchased first by the customer after they became a member?
-- 7. Which item was purchased just before the customer became a member?
-- 8. What is the total items and amount spent for each member before they became a member?
-- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
-- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?


-- 1. What is the total amount each customer spent at the restaurant? 
WITH TOTAL_AMOUNT AS (SELECT *
FROM dannys_diner.sales s
inner join dannys_diner.menu m on s.product_id = m.product_id)
SELECT customer_id, sum(price)
FROM TOTAL_AMOUNT
GROUP BY customer_id
ORDER BY customer_id;

|  customer_id  |      sum      |
| ------------- | ------------- |
|       A       |       76      |
|       B       |       74      |
|       C       |       36      |


-- 2. How many days has each customer visited the restaurant?
SELECT customer_id,
COUNT(DISTINCT(order_date)) AS DAYS_VISITED
FROM dannys_diner.sales
GROUP BY customer_id;

|  customer_id  |  days_visited |
| ------------- | ------------- |
|       A       |        4      |
|       B       |        6      |
|       C       |        2      |


-- 3. What was the first item from the menu purchased by each customer?
WITH first_item AS 
(SELECT DISTINCT order_date,customer_id, product_id
 FROM dannys_diner.sales)
 
 SELECT *
 FROM first_item
 ORDER BY order_date ASC
 LIMIT 4;
 
|         order_date        |  customer_id  |    product_id  |
| ------------------------- | ------------- |  ------------- | 
| 2021-01-01T00:00:00.000Z  |       C       |         3      |
| 2021-01-01T00:00:00.000Z  |       A       |         2      |
| 2021-01-01T00:00:00.000Z  |       B       |         2      |
| 2021-01-01T00:00:00.000Z  |       A       |         1      |


-- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
WITH top_item AS
(SELECT *
FROM dannys_diner.sales s
JOIN dannys_diner.menu m ON s.product_id = m.product_id)

SELECT COUNT (product_name) total_count,
product_name
FROM top_item
GROUP BY product_name
ORDER BY total_count DESC
LIMIT 1;

|  total_count  |  product_name |
| ------------- | ------------- |
|       8       |     ramen     |



-- 5. Which item was the most popular for each customer?
WITH top_item_customer AS
(SELECT *
FROM dannys_diner.menu m
JOIN dannys_diner.sales s ON m.product_id = s.product_id),

top_item_two AS
(SELECT COUNT (product_name) Products,
product_name, customer_id, RANK() OVER(PARTITION BY customer_id ORDER BY COUNT(product_name) DESC) AS ranking
FROM top_item_customer
GROUP BY top_item_customer.customer_id, product_name)

SELECT customer_id, product_name
FROM top_item_two
WHERE ranking = 1;

|  customer_id  |  product_name |
| ------------- | ------------- |
|       A       |     ramen     |
|       B       |     ramen     |
|       B       |     curry     |
|       B       |     sushi     |
|       C       |     ramen     |


-- 6. Which item was purchased first by the customer after they became a member?
WITH item_sales AS
(SELECT s.customer_id, s.order_date, s.product_id, m.join_date
FROM dannys_diner.sales s
INNER JOIN dannys_diner.members m ON s.customer_id = m.customer_id),

item_ranked AS
(SELECT customer_id, order_date, product_id, join_date, 
RANK() OVER(PARTITION BY customer_id ORDER BY order_date ASC) as ranker
FROM item_sales
WHERE order_date>join_date)

SELECT customer_id, order_date, product_name
FROM item_ranked r
INNER JOIN dannys_diner.menu m ON r.product_id = m.product_id
WHERE ranker=1
ORDER BY customer_id;

|   customer_id   |         order_date        |  product_name  |
| --------------- | ------------------------- |  ------------- | 
|         A       | 2021-01-10T00:00:00.000Z  |      ramen     |
|         B       | 2021-01-11T00:00:00.000Z  |      sushi     |


-- 7. Which item was purchased just before the customer became a member?
WITH item_sales AS
(SELECT s.customer_id, s.order_date, s.product_id, m.join_date
FROM dannys_diner.sales s
INNER JOIN dannys_diner.members m ON s.customer_id = m.customer_id),

item_ranked AS
(SELECT customer_id, order_date, product_id, join_date, 
RANK() OVER(PARTITION BY customer_id ORDER BY order_date DESC) as ranker
FROM item_sales
WHERE order_date<join_date)

SELECT customer_id, order_date, join_date, product_name
FROM item_ranked r
INNER JOIN dannys_diner.menu m ON r.product_id = m.product_id
WHERE ranker=1
ORDER BY customer_id;

|   customer_id   |         order_date        |        join_date         |   product_name  |
| --------------- | ------------------------- |  ----------------------  |   ------------- |
|         A       | 2021-01-01T00:00:00.000Z  | 2021-01-07T00:00:00.000Z |      sushi      |
|         A       | 2021-01-01T00:00:00.000Z  | 2021-01-07T00:00:00.000Z |      curry      |
|         B       | 2021-01-04T00:00:00.000Z  | 2021-01-09T00:00:00.000Z |      sushi      |


-- 8. What is the total items and amount spent for each member before they became a member?
WITH item_sales AS
(SELECT s.customer_id, s.order_date, s.product_id, n.price, n.product_name
FROM dannys_diner.sales s
INNER JOIN dannys_diner.members m ON s.customer_id = m.customer_id
INNER JOIN dannys_diner.menu n ON n.product_id = s.product_id
WHERE order_date<join_date)

SELECT customer_id, COUNT(customer_id), SUM(price)
FROM item_sales
GROUP BY customer_id
ORDER BY customer_id;

|  customer_id  |      count    |       sum      |
| ------------- | ------------- |  ------------- | 
|       A       |       2       |        25      |
|       B       |       3       |        40      |


-- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
WITH prices AS
(SELECT s.customer_id, s.product_id, m.price
FROM dannys_diner.sales s
JOIN dannys_diner.menu m ON s.product_id = m.product_id),

calc_points AS
(SELECT customer_id, product_id, price,
CASE
 	WHEN product_id=1 THEN 20*price
	ELSE 10*price
END AS points
FROM prices)

SELECT customer_id, SUM (points) AS total_points
FROM calc_points
GROUP BY customer_id
ORDER BY customer_id;  

|  customer_id  |  total_points |
| ------------- | ------------- |
|       A       |       860     |
|       B       |       940     |
|       C       |       360     |


-- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
WITH prices AS
(SELECT s.customer_id, s.product_id, m.price, r.join_date, s.order_date
FROM dannys_diner.sales s
JOIN dannys_diner.menu m ON s.product_id = m.product_id
JOIN dannys_diner.members r ON r.customer_id = s.customer_id),

calc_points AS
(SELECT customer_id, product_id, price,
CASE
 WHEN order_date<join_date AND product_id=1 THEN 20*price
 WHEN order_date<join_date AND product_id != 1 THEN 10*price
 ELSE
 price*20
END AS points
FROM prices)

SELECT customer_id, SUM (points) AS total_points
FROM calc_points
GROUP BY customer_id
ORDER BY customer_id;

|  customer_id  |  total_points |
| ------------- | ------------- |
|       A       |      1370     |
|       B       |      1180     |

