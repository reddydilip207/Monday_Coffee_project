# Monday Coffee Expansion SQL Project

![Company Logo](https://github.com/najirh/Monday-Coffee-Expansion-Project-P8/blob/main/1.png)

## Objective
The goal of this project is to analyze the sales data of Monday Coffee, a company that has been selling its products online since January 2023, and to recommend the top three major cities in India for opening new coffee shop locations based on consumer demand and sales performance.

## Key Questions
1. **Coffee Consumers Count**  
   How many people in each city are estimated to consume coffee, given that 25% of the population does?
``` sql
SELECT 
    city_name,
	ROUND((population * 0.25)/1000000,2) AS coffee_consumers_in_millions,
	city_rank
FROM city
ORDER BY 2 DESC;
```
2. **Total Revenue from Coffee Sales**  
   What is the total revenue generated from coffee sales across all cities in the last quarter of 2023?
``` sql
SELECT 
     cit.city_name,
     SUM(s.total) AS total_revenue
FROM sales s
JOIN customers c ON c.customer_id=s.customer_id
JOIN city cit ON cit.city_id=c.city_id
WHERE EXTRACT(YEAR FROM sale_date)=2023
      AND
      EXTRACT(QUARTER FROM sale_date)=4
GROUP BY 1
ORDER BY 2 DESC;
```

3. **Sales Count for Each Product**  
   How many units of each coffee product have been sold?
``` sql
SELECT 
     p.product_name,
	 COUNT(s.sale_id) AS sold
FROM products p
LEFT JOIN sales s ON p.product_id=s.product_id
GROUP BY 1
ORDER BY 2 DESC;
```

4. **Average Sales Amount per City**  
   What is the average sales amount per customer in each city?
``` sql
SELECT 
    ci.city_name,
	SUM(s.total) AS total_sales,
	COUNT(DISTINCT s.customer_id) AS total_customers,
	ROUND(SUM(s.total)::NUMERIC/COUNT(DISTINCT s.customer_id):: NUMERIC ,2) AS avg_sale_per_customer
FROM sales s
JOIN customers c ON c.customer_id=s.customer_id
JOIN city ci ON ci.city_id=c.city_id
GROUP BY 1
ORDER BY 2 DESC;
```

5. **City Population and Coffee Consumers**  
   Provide a list of cities along with their populations and estimated coffee consumers.
``` sql
SELECT
     ci.city_name,
	 COUNT(DISTINCT c.customer_id) AS unique_customer,
	 ROUND((ci.population * 0.25)/1000000, 2) AS coffee_consumers_millions
FROM city AS ci
LEFT JOIN customers AS c ON c.city_id=ci.city_id
GROUP BY 1,3
ORDER BY 2 DESC;
```

6. **Top Selling Products by City**  
   What are the top 3 selling products in each city based on sales volume?
``` sql
SELECT *
FROM
(
SELECT
    ci.city_name,
	p.product_name,
	COUNT(s.sale_id) AS total_orders,
	DENSE_RANK() OVER(PARTITION BY ci.city_name ORDER BY COUNT(s.sale_id) DESC) AS rank
FROM sales s 
JOIN products AS p ON p.product_id=s.product_id
JOIN customers AS c ON c.customer_id= s.customer_id
JOIN city AS ci ON ci.city_id=c.city_id
GROUP BY 1,2
) AS t1
WHERE rank <=3
```

7. **Customer Segmentation by City**  
   How many unique customers are there in each city who have purchased coffee products?
``` sql
SELECT
    ci.city_name,
    COUNT(DISTINCT c.customer_id) AS unique_customers
FROM city ci
LEFT JOIN customers AS c ON c.city_id=ci.city_id
JOIN sales AS s ON c.customer_id=s.customer_id
WHERE s.product_id >=14
GROUP BY 1
```

8. **Average Sale vs Rent**  
   Find each city and their average sale per customer and avg rent per customer
``` sql
WITH city_table AS(
SELECT 
     ci.city_name,
	 SUM(s.total) AS total_sales,
	 COUNT(DISTINCT c.customer_id) AS total_customer,
	 ROUND(SUM(s.total)::NUMERIC /COUNT(DISTINCT c.customer_id)::NUMERIC,2) AS avg_sale_per_count
FROM city ci
JOIN customers AS c ON c.city_id=ci.city_id
JOIN sales AS s ON s.customer_id =c.customer_id
GROUP BY 1
ORDER BY 2 DESC
),
city_rent AS
(SELECT 
    city_name,
	estimated_rent
FROM city
)
SELECT 
     cr.city_name,
	 cr.estimated_rent,
	 ct.total_customer,
	 ct.avg_sale_per_count,
	 ROUND(cr.estimated_rent:: NUMERIC/ct.total_customer::NUMERIC,2) AS avg_rent_for_customer
FROM city_rent AS cr
JOIN city_table AS ct ON ct.city_name=cr.city_name
ORDER BY 4 DESC;
```

9. **Monthly Sales Growth**  
   Sales growth rate: Calculate the percentage growth (or decline) in sales over different time periods (monthly).
``` sql
WITH monthly_sales AS(
SELECT 
     ci.city_name,
	 EXTRACT(MONTH FROM sale_date) AS month,
	 EXTRACT(YEAR FROM sale_date) AS year,
	 SUM(s.total) AS total_sale
FROM sales s
JOIN customers AS c ON c.customer_id=s.customer_id
JOIN city AS ci ON ci.city_id=c.city_id
GROUP BY 1,2,3
ORDER BY 1,3,2
),
growth_rate AS
(
SELECT 
    city_name,
	month,
	year,
	total_sale AS current_month_sale,
	LAG(total_sale,1) OVER(PARTITION BY city_name ORDER BY year,month) AS last_month_sale
FROM monthly_sales 
)
SELECT 
     city_name,
	 month,
	 year,
	 current_month_sale,
	 last_month_sale,
	 ROUND((current_month_sale)::NUMERIC/last_month_sale ::NUMERIC * 100 , 2) AS growth_rate
FROM growth_rate
WHERE last_month_sale IS NOT NULL;
```

10. **Market Potential Analysis**  
    Identify top 3 city based on highest sales, return city name, total sale, total rent, total customers, estimated  coffee consumer
``` sql
WITH city_table AS(
SELECT 
     ci.city_name,
	 SUM(s.total) AS total_sales,
	 COUNT(DISTINCT c.customer_id) AS total_customer,
	 ROUND(SUM(s.total)::NUMERIC /COUNT(DISTINCT c.customer_id)::NUMERIC,2) AS avg_sale_per_count
FROM city ci
JOIN customers AS c ON c.city_id=ci.city_id
JOIN sales AS s ON s.customer_id =c.customer_id
GROUP BY 1
ORDER BY 2 DESC
),
city_rent AS
( SELECT 
    city_name,
	estimated_rent,
	ROUND((population * 0.25)/1000000 ,3) AS estimated_coffee_consumer_in_millions
FROM city
)
SELECT 
     cr.city_name,
	 total_sales,
	 cr.estimated_rent AS total_rent,
	  total_customer,
	 estimated_coffee_consumer_in_millions,
	 ct.avg_sale_per_count,
	 ROUND(cr.estimated_rent:: NUMERIC/ct.total_customer::NUMERIC,2) AS avg_rent_for_customer
FROM city_rent AS cr
JOIN city_table AS ct ON ct.city_name=cr.city_name
ORDER BY 2 DESC;
```    

## Recommendations
After analyzing the data, the recommended top three cities for new store openings are:

**City 1: Pune**  
1. Average rent per customer is very low.  
2. Highest total revenue.  
3. Average sales per customer is also high.

**City 2: Delhi**  
1. Highest estimated coffee consumers at 7.7 million.  
2. Highest total number of customers, which is 68.  
3. Average rent per customer is 330 (still under 500).

**City 3: Jaipur**  
1. Highest number of customers, which is 69.  
2. Average rent per customer is very low at 156.  
3. Average sales per customer is better at 11.6k.


## Technology Stack
- **Database**: PostgreSQL
- **Tools**: pgAdmin 4 (or any SQL editor)
---
