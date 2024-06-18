# sql-project-target
sql project target
Target

Target is a globally renowned brand and a prominent retailer in the United States. Target makes itself a preferred shopping destination by offering outstanding value, inspiration, innovation and an exceptional guest experience that no other retailer can deliver.
This particular business case focuses on the operations of Target in Brazil and provides insightful information about 100,000 orders placed between 2016 and 2018. The dataset offers a comprehensive view of various dimensions including the order status, price, payment and freight performance, customer location, product attributes, and customer reviews.

Dataset: https://drive.google.com/drive/folders/1TGEc66YKbD443nslRi1bWgVd238gJCnb

The data is available in 8 csv files:
1.	customers.csv
2.	sellers.csv
3.	order_items.csv
4.	geolocation.csv
5.	payments.csv
6.	reviews.csv
7.	orders.csv
8.	products.csv

1 1 Import the dataset and do usual exploratory analysis steps like checking the structure & characteristics of the dataset:
1.	Data type of all columns in the "customers" table.

SELECT * 
    FROM Target.customers;
 

1. 2 Get the time range between which the orders were placed.

SELECT MIN(order_purchase_timestamp) AS start_time,
       MAX(order_purchase_timestamp) AS end_time
      FROM Target.orders;

 

1. 3 Count the Cities & States of customers who ordered during the given period.
 

SELECT COUNT(DISTINCT customer_city) AS num_unique_cities,
       COUNT(DISTINCT customer_state) AS num_unique_states
FROM Target.customers;
(there is no given period)

 

2 1 In-depth Exploration:
Is there a growing trend in the no. of orders placed over the past years?

SELECT EXTRACT(YEAR FROM order_approved_at) AS order_year,
       COUNT(DISTINCT order_id) AS num_orders
FROM Target.orders
GROUP BY order_year
ORDER BY order_year;

 
Explanation: Based on the orders placed we can see the increasing number of orders over the year. 

2. 2 Can we see some kind of monthly seasonality in terms of the no. of orders being placed?

SELECT EXTRACT(YEAR FROM order_purchase_timestamp) AS year,
       EXTRACT(MONTH FROM order_purchase_timestamp) AS month,
       COUNT(order_id) AS no_of_orders
FROM Target.orders
GROUP BY year, month
ORDER BY no_of_orders DESC;

 
Explanation: Nov 2017 has the highest orders, maybe due to holiday season. Followed by 2018 Jan has the highest orders. In 2018, every month has orders of about 5000-6000 orders.

2. 3 During what time of the day, do the Brazilian customers mostly place their orders? (Dawn, Morning, Afternoon or Night)
o	0-6 hrs : Dawn
o	7-12 hrs : Mornings
o	13-18 hrs : Afternoon
o	19-23 hrs : Night

WITH OrderTimeCategories AS (
  SELECT order_id,
         CASE 
           WHEN EXTRACT(HOUR FROM order_purchase_timestamp) BETWEEN 0 AND 6 THEN 'Dawn'
           WHEN EXTRACT(HOUR FROM order_purchase_timestamp) BETWEEN 7 AND 12 THEN 'Morning'
           WHEN EXTRACT(HOUR FROM order_purchase_timestamp) BETWEEN 13 AND 18 THEN 'Afternoon'
           ELSE 'Night'
         END AS order_time_category
  FROM Target.orders
)

SELECT order_time_category,
       COUNT(order_id) AS no_of_orders
FROM OrderTimeCategories
GROUP BY order_time_category
ORDER BY no_of_orders DESC;

 
During Afternoon there is a higher orders placed in Brazil.

3.1  Evolution of E-commerce orders in the Brazil region: Get the month on month no. of orders placed in each state.
SELECT
    EXTRACT(YEAR FROM o.order_purchase_timestamp) AS order_year,
    EXTRACT(MONTH FROM o.order_purchase_timestamp) AS order_month,
    c.customer_state,
    COUNT(o.order_id) AS num_orders
FROM
    Target.orders o
JOIN
    Target.customers c ON o.customer_id = c.customer_id
GROUP BY
    order_year,
    order_month,
    c.customer_state
ORDER BY
    order_year,
    order_month,
    c.customer_state;

 

3.2 How are the customers distributed across all the states?
 
SELECT customer_state, COUNT(DISTINCT customer_id) AS no_of_customers
FROM Target.customers
GROUP BY customer_state
ORDER BY no_of_customers DESC;
 

State code: SP (São Paulo )has highest number of customers and RR (Roraima) has the lowest number of customers
4.1 Impact on Economy: Analyze the money movement by e-commerce by looking at order prices, freight and others: Get the % increase in the cost of orders from year 2017 to 2018 (include months between Jan to Aug only).
You can use the "payment_value" column in the payments table to get the cost of orders.

 WITH YearlyCost AS (
    SELECT
        EXTRACT(YEAR FROM o.order_purchase_timestamp) AS order_year,
        EXTRACT(MONTH FROM o.order_purchase_timestamp) AS order_month,
        SUM(oi.price + oi.freight_value) AS total_cost
    FROM
        Target.orders o
    JOIN
        Target.order_items oi ON o.order_id = oi.order_id
    WHERE
        EXTRACT(YEAR FROM o.order_purchase_timestamp) IN (2017, 2018)
        AND EXTRACT(MONTH FROM o.order_purchase_timestamp) BETWEEN 1 AND 8
    GROUP BY
        order_year,
        order_month
)

SELECT
    2017 AS year_start,
    2018 AS year_end,
    ROUND(((yc2018.total_cost - yc2017.total_cost) / NULLIF(yc2017.total_cost, 0)) * 100, 2) AS percentage_increase
FROM
    YearlyCost yc2017
JOIN
    YearlyCost yc2018 ON yc2017.order_month = yc2018.order_month AND yc2017.order_year = 2017 AND yc2018.order_year = 2018
ORDER BY
    yc2017.order_month;

 

4 2 Calculate the Total & Average value of order price for each state.

SELECT
    c.customer_state,
    SUM(oi.price + oi.freight_value) AS total_order_price,
    AVG(oi.price + oi.freight_value) AS avg_order_price
FROM
    Target.orders o
JOIN
    Target.order_items oi ON o.order_id = oi.order_id
JOIN
    Target.customers c ON o.customer_id = c.customer_id
GROUP BY
    c.customer_state
ORDER BY
    c.customer_state;

 

4 3 SELECT
    c.customer_state,
    ROUND(SUM(oi.freight_value), 2) AS total_freight,
    ROUND(AVG(oi.freight_value), 2) AS avg_freight
FROM
    Target.orders o
JOIN
    Target.order_items oi ON o.order_id = oi.order_id
JOIN
    Target.customers c ON o.customer_id = c.customer_id
GROUP BY
    c.customer_state
ORDER BY
    c.customer_state;

 

5 1 Analysis based on sales, freight and delivery time.
Find the no. of days taken to deliver each order from the order’s purchase date as delivery time.
Also, calculate the difference (in days) between the estimated & actual delivery date of an order.
Do this in a single query.

You can calculate the delivery time and the difference between the estimated & actual delivery date using the given formula:
time_to_deliver = order_delivered_customer_date - order_purchase_timestamp
diff_estimated_delivery = order_delivered_customer_date - order_estimated_delivery_date

SELECT
    o.order_id,
    o.order_delivered_customer_date - o.order_purchase_timestamp AS time_to_deliver,
    o.order_estimated_delivery_date - o.order_delivered_customer_date AS diff_estimated_delivery
FROM
    Target.orders o;

 
Explanation: Improve logistics and shipping processes to reduce delivery times and enhance customer satisfaction.

5 2 Find out the top 5 states with the highest & lowest average freight value.

SELECT
    c.customer_state,
    ROUND(AVG(oi.freight_value), 2) AS avg_freight
FROM
    Target.customers c
JOIN
    Target.orders o ON c.customer_id = o.customer_id
JOIN
    Target.order_items oi ON o.order_id = oi.order_id
GROUP BY
    c.customer_state
ORDER BY
    avg_freight DESC
LIMIT 5;

 

Explanation Roraima has highest freight value and Piauí has the lowest value
5. 3.  Find out the top 5 states with the highest & lowest average delivery time.

SELECT
    EXTRACT(YEAR FROM o.order_purchase_timestamp) AS order_year,
    EXTRACT(MONTH FROM o.order_purchase_timestamp) AS order_month,
    p.payment_type,
    COUNT(o.order_id) AS num_orders



6. 1 Analysis based on the payments: Find the month on month no. of orders placed using different payment types.

FROM
    Target.orders o
JOIN
    Target.payments p ON o.order_id = p.order_id
GROUP BY
    order_year,
    order_month,
    payment_type
ORDER BY
    order_year,
    order_month,
    payment_type;

 
Explanation: credit card is the most favoured payment method.

6.2 Find the no. of orders placed on the basis of the payment installments that have been paid.

SELECT
    payment_installments,
    COUNT(order_id) AS num_orders
FROM
    Target.payments
GROUP BY
    payment_installments
ORDER BY
    payment_installments;

 

Most of customers paid in single instalments.
