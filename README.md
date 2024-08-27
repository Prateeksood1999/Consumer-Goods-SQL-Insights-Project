# Consumer Goods SQL Insights Project

Welcome to the repository for the Consumer Goods SQL Insights project. This project focuses on analyzing a million-row database to address 10 ad-hoc business requests and provide actionable insights for optimizing operations in the consumer goods sector.

## Overview

In this project, I utilized SQL to uncover key insights.The goal was to deliver targeted insights that support strategic decision-making and enhance operational performance.

## SQL Queries and Analysis

Below are some example SQL queries used to address the business requests for the Consumer Goods project. These queries provide insights into different areas of the business and were crucial for generating actionable recommendations.

### 1.  Provide the list of markets in which customer "Atliq Exclusive" operates its business in the APAC region.
        
   ```sql
        select
    distinct(market)
        from dim_customer
           where
           customer="Atliq Exclusive" and region="APAC" ;
```

### 2. What is the percentage of unique product increase in 2021 vs. 2020? 

       The final output contains these fields, 
              
                unique_products_2020
                
                unique_products_2021 
                
                percentage_chg 
 
   ```sql
    SELECT
    unique_products_2020,
    unique_products_2021,
    round(((unique_products_2021 - unique_products_2020) / CAST(unique_products_2020 AS DECIMAL)),4) * 100 AS percentage_chg
    FROM (
    SELECT
        (SELECT COUNT(DISTINCT product_code) FROM fact_sales_monthly WHERE fiscal_year = 2020) AS unique_products_2020,
        (SELECT COUNT(DISTINCT product_code) FROM fact_sales_monthly WHERE fiscal_year = 2021) AS unique_products_2021
     ) AS subquery;
```

## 3. Provide a report with all the unique product counts for each segment and sort them in descending order of product counts. 
                   
                   The final output contains 2 fields,
                   segment
                   product_count

 ```sql
SELECT 
    segment, COUNT(DISTINCT product_code) AS product_count
FROM
    dim_product
GROUP BY segment
ORDER BY product_count DESC
```

## 4. Follow-up: Which segment had the most increase in unique products in 2021 vs 2020? The final output contains these fields,
                     
                    segment

                    product_count_2020

                    product_count_2021

                    difference
 ```sql
with cte1 as (
  select
    segment,
    count(distinct p.product_code) as product_count_2020
  from
    dim_product as p
  join
    fact_sales_monthly as fsm
  on
    p.product_code = fsm.product_code
  where
    fiscal_year = 2020
  group by
    segment
),
cte2 as (
  select
    segment,
    count(distinct p.product_code) as product_count_2021
  from
    dim_product as p
  join
    fact_sales_monthly as fsm
  on
    p.product_code = fsm.product_code
  where
    fiscal_year = 2021
  group by
    segment
)
select
  c1.segment,
  c1.product_count_2020,
  c2.product_count_2021,
  c2.product_count_2021 - c1.product_count_2020 as difference
from
  cte1 c1
join
  cte2 c2
on
  c1.segment = c2.segment;
```
## 5. Get the products that have the highest and lowest manufacturing costs. 

             The final output should contain these fields,

                        product_code 

                        product 

                        manufacturing_cost

   ```sql
(
select
product,
p.product_code,
manufacturing_cost
from dim_product as p
join
fact_manufacturing_cost as c
on p.product_code=c.product_code
where manufacturing_cost = (SELECT MIN(manufacturing_cost) FROM fact_manufacturing_cost)
)
union 
(
select
p.product,
p.product_code,
c.manufacturing_cost
from dim_product as p
join
fact_manufacturing_cost as c
on p.product_code=c.product_code
where manufacturing_cost = (SELECT max(manufacturing_cost) FROM fact_manufacturing_cost)
)
order by manufacturing_cost desc
```
## 6. Generate a report which contains the top 5 customers who received an average high pre_invoice_discount_pct for the fiscal year 2021 and in the Indian market. 

                 The final output contains these fields,

                 customer_code

                 customer

                 average_discount_percentage

   ```sql
select c.customer_code, 
c.customer, 
round(AVG(pre_invoice_discount_pct),4) AS average_discount_percentage
FROM fact_pre_invoice_deductions d
JOIN dim_customer c ON d.customer_code = c.customer_code
WHERE c.market = "India" AND fiscal_year = "2021"
GROUP BY c.customer_code,
c.customer
ORDER BY average_discount_percentage DESC
LIMIT 5;
```

## 7. Get the complete report of the Gross sales amount for the customer “Atliq Exclusive” for each month. This analysis helps to get an idea of low and high-performing months and take strategic decisions.

               The final report contains these columns:

               Month

               Year

               Gross sales Amount
 ```sql
WITH SalesData AS (
    SELECT 
         CONCAT(MONTHNAME(fs.date), ' (', YEAR(fs.date), ')') AS Month,
        YEAR(DATE_ADD(fs.date, INTERVAL 4 MONTH)) AS Fiscal_year,
        concat(round(SUM(fs.sold_quantity * gp.gross_price)/1000000,2),"M") AS Gross_sales_amount
	FROM
        fact_sales_monthly AS fs
    JOIN
        fact_gross_price AS gp ON gp.product_code = fs.product_code
    JOIN
        dim_customer AS c ON c.customer_code = fs.customer_code
    WHERE 
        c.customer = 'Atliq Exclusive'
    GROUP BY 
       Month, 
       fiscal_year 	
)
SELECT
    Month,
    Fiscal_year,
    Gross_sales_amount
FROM 
    SalesData
ORDER BY 
    fiscal_year
```

## 8.In which quarter of 2020, got the maximum total_sold_quantity? 

        The final output contains these fields sorted by the total_sold_quantity,

        Quarter

        total_sold_quantity

 ```sql
SELECT 
    CASE 
        WHEN MONTH(fsm.date) IN (9, 10, 11) THEN 'Q1'
        WHEN MONTH(fsm.date) IN (12, 1, 2) THEN 'Q2'
        WHEN MONTH(fsm.date) IN (3, 4, 5) THEN 'Q3'
        ELSE 'Q4'
    END AS Quarter,
   concat(round(sum( sold_quantity)/1000000,1),"M") as total_sold_quantity
FROM fact_sales_monthly as fsm
where fiscal_year=2020
group by Quarter
order by total_sold_quantity desc
```

## 9. Which channel helped to bring more gross sales in the fiscal year 2021 and the percentage of contribution? 

             The final output contains these fields,

             channel

             gross_sales_mln

             percentage

   ```sql
WITH channel_sales AS (
    SELECT 
        c.channel,
        ROUND(SUM(gp.gross_price * s.sold_quantity) / 1000000, 2) AS gross_sales_mln 
    FROM 
        dim_customer AS c
    JOIN 
        fact_sales_monthly AS s
    ON 
        c.customer_code = s.customer_code
    JOIN 
        fact_gross_price AS gp
    ON 
        gp.product_code = s.product_code 
    WHERE 
        s.fiscal_year = 2021
    GROUP BY 
        c.channel
),

total_sales AS (
    SELECT 
        SUM(gross_sales_mln) AS total_gross_sales_mln
    FROM 
        channel_sales
)

SELECT 
    cs.channel,
    cs.gross_sales_mln,
    concat(ROUND((cs.gross_sales_mln / ts.total_gross_sales_mln) * 100, 2),"%") AS percentage
FROM 
    channel_sales AS cs,
    total_sales AS ts
ORDER BY 
    cs.gross_sales_mln DESC;
```

## 10. Get the Top 3 products in each division that have a high total_sold_quantity in the fiscal_year 2021? 

              The final output contains these fields,

              division

              product_code

              product

              total_sold_quantity rank_order

       
```sql
WITH top_sold_products as 
(
	SELECT p.division AS division,
		   p.product_code AS product_code,
		   p.product AS product,
		   SUM(fs.sold_quantity) AS total_sold_quantity
	FROM fact_sales_monthly AS fs
	INNER JOIN dim_product AS p
	ON fs.product_code = p.product_code
	WHERE fs.fiscal_year = 2021
	GROUP BY  p.division, p.product_code, p.product 
	ORDER BY total_sold_quantity DESC
),
top_sold_per_division AS 
(
 SELECT division,
	    product_code,
        product,
        total_sold_quantity,
        DENSE_RANK() OVER(PARTITION BY division ORDER BY total_sold_quantity DESC) AS rank_order 
 FROM top_sold_products
 )
 SELECT * FROM top_sold_per_division
 WHERE rank_order <= 3;
```

## Presentation

For a detailed walkthrough of these SQL queries and insights, watch the full presentation on YouTube: https://youtu.be/P_NMae7QUHc

## Conclusion

Thank you for taking the time to review my SQL queries and analysis. These examples demonstrate how I approach solving ad-hoc business requests in the consumer goods domain, with a focus on extracting meaningful insights from data.
  




