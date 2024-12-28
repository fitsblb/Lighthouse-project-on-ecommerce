Answer the following questions and provide the SQL queries used to find the answer.

    
# 1. Which cities and countries have the highest level of transaction revenues on the site?


__SQL Queries:__

__*PART 1*__

- __The purpose of this query is to rank cities by their total transaction revenue.__
  - Observed data issues, such as the city of New York incorrectly associated with Canada, have been corrected in the view '__level_of_transaction__'.
  - Additionally, __TRIM__ functions are applied to remove any leading/trailing spaces in city and country names, ensuring accurate grouping.



 ``` sql
CREATE OR REPLACE VIEW level_of_transaction AS 

    SELECT 
        TRIM(city) AS city,  -- Trim spaces from city names
        CASE
            WHEN TRIM(city) = 'New York' AND TRIM(country) = 'Canada' THEN 'New York'  -- Keep city as New York
            WHEN TRIM(city) IN ('(not set)', 'not available in demo dataset') THEN 'City in ' || TRIM(country)
            ELSE TRIM(city)
        END AS modified_city,
        CASE
            WHEN TRIM(city) = 'New York' AND TRIM(country) = 'Canada' THEN 'United States'  -- Change country to United States
            ELSE TRIM(country)
        END AS country,  -- Trim spaces from country names
        totaltransactionrevenue AS totalrevenue -- Maintain original revenue data without modifications
    FROM 
        new_sessions
    WHERE
	    totaltransactionrevenue IS NOT NULL
	    AND country NOT IN ('(not set)', 'not available in demo dataset')

		
    SELECT 
        modified_city,
    	country,
    	TO_CHAR(SUM(totalrevenue), '999,999,999,999') AS total_revenue, -- Format revenue for readability
    	RANK() OVER (ORDER BY SUM(totalrevenue) DESC) AS city_revenue_ranked -- Rank cities by total revenue in descending order
    FROM
    	level_of_transaction
    GROUP BY
    	modified_city,
    	country
    ORDER BY 
    	city_revenue_ranked;-- Sort by revenue rank to display highest-ranking cities first
```



__*PART 2:*__

- __Country-level Transaction Revenue__
  - In this query, we rank countries by their total transaction revenue.
  - Grouping by country allows us to compare revenue levels at a national level.


 ``` sql
    SELECT 
        country,
    	TO_CHAR(SUM(totalrevenue), '999,999,999,999') AS total_revenue,-- Format revenue for readability
   	    RANK() OVER (ORDER BY SUM(totalrevenue) DESC) AS country_revenue_ranked -- Rank countries by total revenue in descending order
    FROM
    	level_of_transaction
    GROUP BY
    	country
    ORDER BY 
    	country_revenue_ranked;-- Sort by revenue rank to display highest-ranking countries first
```


__Answer:__


![Screen Shot 2024-11-07 at 10 16 22 AM](https://github.com/user-attachments/assets/8b98c672-eada-4346-acf7-68960fc254ed)


![Screen Shot 2024-11-07 at 12 06 52 PM](https://github.com/user-attachments/assets/a9905daa-e2cc-4f38-ab58-055c33ff0347)



# 2. What is the average number of products ordered from visitors in each city and country?


__SQL Queries:__

 ``` sql
/* Step 1: Do some data cleaning */


WITH cleaned_sessions AS (
    SELECT 
        CASE 
            WHEN ns.city IN ('(not set)', 'not available in demo dataset') OR ns.city IS NULL 
            THEN NULL  -- -- Set to NULL for unknown cities to ensure they are not included in further analysis
            ELSE TRIM(ns.city)  -- Trim spaces for consistency
        END AS city,
        CASE 
            WHEN ns.country IN ('(not set)', 'not available in demo dataset') OR ns.country IS NULL 
            THEN NULL  -- Set to NULL for unknown countries
            ELSE TRIM(ns.country)  -- Trim spaces for consistency
        END AS country,
        ns.fullvisitorid,
        ns.productsku
    FROM 
        new_sessions ns
    WHERE 
        ns.city NOT LIKE '%not set%' 
        AND ns.country NOT LIKE '%not set%' 
        AND ns.city NOT LIKE 'not available in demo dataset' 
        AND ns.fullvisitorid IS NOT NULL 
        AND ns.country IS NOT NULL 
),

/*Step 2: Count products ordered by city and country, filtering out NULL cities*/

product_counts AS (
    SELECT 
 	    city,
 	    country,
 	    COUNT(productsku) AS total_orders,-- Count the number of products ordered per visitor
 	    fullvisitorid -- Associate the count with individual visitors for calculating average orders per visitor
    FROM 
 	    cleaned_sessions
    WHERE 
 	    city IS NOT NULL
	    AND country IS NOT NULL-- Exclude unknown cities and countries from this count
    GROUP BY 
 	    city, 
        country, 
        fullvisitorid
)

/*Step 3: Calculate average orders per visitor, by city and country*/

    SELECT 
 	    city,
 	    country,
 	    ROUND(AVG(total_orders),2) AS avg_orders_per_visitor-- Calculate the average number of orders per visitor and round to 2 decimal places
    FROM 
 	    product_counts
    GROUP BY 
 	    city, 
        country -- Group by city and country to calculate the average per region
    ORDER BY 
 	    avg_orders_per_visitor DESC; -- Order by the average number of orders per visitor in descending order
```



__Answer:__

![Screen Shot 2024-11-07 at 12 12 25 PM](https://github.com/user-attachments/assets/8abf9e26-dab0-4201-879a-31f115174539)





# 3. Is there any pattern in the types (product categories) of products ordered from visitors in each city and country?


__SQL Queries:__
- **Sub-question 3.1:** Top categories with highest revenue  

``` sql
WITH cleaned_sessions AS (
    SELECT 
        CASE 
            WHEN ns.country IN ('(not set)', 'not available in demo dataset') OR ns.country IS NULL 
            THEN NULL  -- Set to NULL for unknown cities
            ELSE TRIM(ns.country)  -- Trim spaces for consistency
        END AS country,
        ns.v2productcategory AS productcategory,
        ns.productsku AS productid,
        ns.productprice / 1000000 AS productprice
    FROM 
        new_sessions ns
    WHERE 
        ns.v2productcategory NOT LIKE '%not set%'
        AND ns.country NOT LIKE '%not set%'
	AND ns.city NOT IN ('(not set)', 'not available in demo dataset')
        AND ns.fullvisitorid IS NOT NULL 
        AND ns.country IS NOT NULL 
        AND ns.productprice > 0  -- Remove zero prices for accurate info
    GROUP BY 
        ns.country,
        ns.v2productcategory,
        ns.productsku,
        ns.productprice
    ORDER BY 1
),

joined_sales AS (
    SELECT 
        cs.country,
        cs.productcategory,
        (sk.total_ordered * cs.productprice) AS total_revenue
    FROM 
        cleaned_sessions cs
    JOIN
        sales_by_sku sk
    ON
        cs.productid = sk.productsku
    WHERE 
        sk.total_ordered > 0
),

ranked_sales AS (
    SELECT 
        country,
        productcategory,
        total_revenue,
        DENSE_RANK() OVER (PARTITION BY country ORDER BY total_revenue DESC) AS country_category_rank
    FROM 
        joined_sales
)

SELECT 
    country,
    productcategory,
    total_revenue
FROM 
    ranked_sales
WHERE 
    country_category_rank = 1
	
ORDER BY 
    total_revenue DESC;
```


![Screen Shot 2024-11-07 at 12 15 55 PM](https://github.com/user-attachments/assets/115c7ddf-624f-4426-b8fe-ec5d61fa4a16)


- **Sub-question 3.2:** top  categories with the highest total ordered sold in __2017__ per city and country, where total ordered is greater than __200__.

``` sql
WITH cleaned_sessions AS (
    SELECT 
        TRIM(ns.country) AS country,
        TRIM(ns.city) AS city,
        ns.v2productcategory AS productcategory,
        ns.productsku AS productid,
        ns.productprice / 1000000 AS productprice,
        ns.transaction_date AS transaction_date
    FROM 
        new_sessions ns
    WHERE 
        ns.v2productcategory NOT LIKE '%not set%'
        AND ns.country NOT LIKE '%not set%' 
        AND ns.city NOT IN ('(not set)', 'not available in demo dataset') 
        AND ns.fullvisitorid IS NOT NULL 
        AND ns.country IS NOT NULL 
        AND ns.productprice > 0  -- Filter out zero prices
        AND EXTRACT(YEAR FROM ns.transaction_date )= '2017'  -- Only 2017 transactions
    GROUP BY 
        ns.country,
        ns.city,
        ns.v2productcategory,
        ns.productsku,
        ns.productprice,
        ns.transaction_date
),

joined_sales AS (
    SELECT 
        cs.country,
        cs.city,
        cs.productcategory,
        sk.total_ordered AS total_ordered,
        cs.transaction_date
    FROM 
        cleaned_sessions cs
    JOIN
        sales_by_sku sk
    ON
        cs.productid = sk.productsku
    WHERE 
        sk.total_ordered > 0  -- Exclude zero quantity orders

),
region_ranked AS (
    SELECT 
        country,
    	city,
    	productcategory,
    	total_ordered,
    	transaction_date,
    	DENSE_RANK() OVER (PARTITION BY country, city ORDER BY total_ordered DESC) AS category_rank
    FROM 
    	joined_sales
)

SELECT 
	country,
    city,
    productcategory,
    total_ordered,
    transaction_date

FROM
	region_ranked
WHERE 
    transaction_date >= '2017-01-01' AND transaction_date <= '2017-12-31'  -- Ensure it's within 2017
    AND category_rank <= 5  -- Top 5 categories per city and country
	AND total_ordered >200
ORDER BY 
    category_rank,
	total_ordered DESC;
```

![Screen Shot 2024-11-07 at 12 17 21 PM](https://github.com/user-attachments/assets/3248fa71-6e3f-4a92-b600-8d9dbe9f1a01)


- **Sub-question 3.3:** Analyzing the Relationship Between Order-to-Stock Ratio and Total Revenue by Product Category Across Regions


``` sql
WITH cleaned_sessions AS (
    SELECT 
        TRIM(ns.country) AS country,
        TRIM(ns.city) AS city,
        ns.v2productcategory AS productcategory,
        ns.productsku AS productid,
        ns.productprice / 1000000 AS productprice
    FROM 
        new_sessions ns
    WHERE 
        ns.v2productcategory NOT LIKE '%not set%'
        AND ns.country NOT LIKE '%not set%' 
        AND ns.city NOT IN ('(not set)', 'not available in demo dataset') 
        AND ns.fullvisitorid IS NOT NULL 
        AND ns.country IS NOT NULL 
        AND ns.productprice > 0  -- Filter out zero prices
    GROUP BY 
        ns.country,
        ns.city,
        ns.v2productcategory,
        ns.productsku,
        ns.productprice
),
joined_sales AS (
    SELECT 
        cs.country,
        cs.city,
        cs.productcategory,
        (sr.total_ordered * cs.productprice) AS total_revenue,
        sr.order_to_stock_ratio
    FROM 
        cleaned_sessions cs
    JOIN
        sales_report sr ON cs.productid = sr.productsku
    WHERE 
        sr.total_ordered > 0  -- Exclude zero quantity orders
)

SELECT 
    country,
    city,
    productcategory,
    total_revenue,
    order_to_stock_ratio,
    DENSE_RANK() OVER (PARTITION BY order_to_stock_ratio ORDER BY total_revenue DESC) AS category_rank
FROM 
    joined_sales
WHERE 
    order_to_stock_ratio > (SELECT AVG(order_to_stock_ratio) FROM joined_sales)--Sub-query to find average of order_to_stock_ratio  
ORDER BY 
    order_to_stock_ratio, 
    total_revenue DESC;
```  
	

![Screen Shot 2024-11-07 at 12 19 09 PM](https://github.com/user-attachments/assets/08c65d3f-eb35-4b08-a79e-d334aee471e7)




# 4. What is the top-selling product from each city/country? Can we find any pattern worthy of noting in the products sold?


__SQL Queries:__

``` sql
WITH product_details AS (
    SELECT 
        ns.productsku AS product_id,
        p.product_name AS product_name,
        ns.city AS city,
        ns.country AS country,
        COUNT(ns.productsku) AS total_orders,
	    RANK() OVER (PARTITION BY country ORDER BY COUNT(ns.productsku) DESC) AS country_product_rank,
	    RANK() OVER (PARTITION BY city ORDER BY COUNT(ns.productsku) DESC) AS city_product_rank,
    	DENSE_RANK() OVER (ORDER BY COUNT(ns.productsku) DESC) AS global_rank  -- Rank products globally by popularity
    FROM 
        new_sessions ns
    JOIN 
        products p 
    ON 
        p.sku = ns.productsku
    WHERE
        ns.city IS NOT NULL
        AND ns.country IS NOT NULL
        AND ns.productsku IS NOT NULL
        AND ns.city NOT IN ('(not set)', 'not available in demo dataset')
        AND ns.country NOT IN ('(not set)', 'not available in demo dataset')
        AND ns.productprice IS NOT NULL
    GROUP BY
        ns.productsku,
        p.product_name,
        ns.city,
        ns.country
)

SELECT
    product_id,
    product_name,
    city,
    country,
    total_orders,
	country_product_rank,
	city_product_rank,
	global_rank

FROM
    product_details
WHERE
    country_product_rank = 1 OR
	city_product_rank  = 1
ORDER BY
   total_orders DESC;
```

__Answer:__


- We can observe how cities are ranked within their respective countries and how countries are ranked globally.
- Additionally, we see that certain products, such as the 'Cam Indoor Security Camera - USA' and the 'Men's 100% Cotton Short Sleeve Hero Tee White,' are top performers.
- These products not only rank #1 and #3 globally by total orders but also consistently rank as best-sellers within multiple cities and countries."



  ![Screen Shot 2024-11-07 at 12 22 26 PM](https://github.com/user-attachments/assets/d6f2d1da-f8bf-418d-8af8-5375f46ff60c)





``` sql
WITH product_details AS (
    SELECT 
        ns.productsku AS product_id,
        ns.city AS city,
        ns.country AS country,
        COUNT(ns.productsku) AS total_orders,
        SUM(ns.productprice) / 1000000 AS productprice,
        RANK() OVER (PARTITION BY ns.country ORDER BY COUNT(ns.productsku) DESC) AS country_product_rank,
        RANK() OVER (PARTITION BY ns.city ORDER BY COUNT(ns.productsku) DESC) AS city_product_rank
    FROM new_sessions ns
    WHERE
        ns.city IS NOT NULL
        AND ns.country IS NOT NULL
        AND ns.productsku IS NOT NULL
        AND ns.city NOT IN ('(not set)', 'not available in demo dataset')
        AND ns.country NOT IN ('(not set)', 'not available in demo dataset')
        AND ns.productprice IS NOT NULL
    GROUP BY
        ns.productsku,
        ns.city,
        ns.country
)

SELECT
    pd.product_id,
    p.product_name,
    pd.city,
    pd.country,
    pd.total_orders,
    pd.country_product_rank,
    pd.city_product_rank,
    SUM(pd.productprice * pd.total_orders) AS total_revenue,
    DENSE_RANK() OVER (ORDER BY pd.total_orders DESC) AS global_order_rank,
    DENSE_RANK() OVER (ORDER BY SUM(pd.productprice * pd.total_orders) DESC) AS global_revenue_rank
FROM
    product_details pd
JOIN 
    products p ON pd.product_id = p.sku
WHERE
    pd.country_product_rank = 1 OR pd.city_product_rank = 1
GROUP BY
    pd.product_id,
    p.product_name,
    pd.city,
    pd.country,
    pd.total_orders,
    pd.country_product_rank,
    pd.city_product_rank
ORDER BY
    global_order_rank,global_revenue_rank DESC;
```


__Answer:__
- I've observed a pattern where products such as the __Cam Indoor Security Camera - USA__ in __Mountain View__ and the __Men's 100% Cotton Short Sleeve Hero Tee White__ in __New York__ are not only top sellers in their respective cities but also major revenue generators globally."



![Screen Shot 2024-11-07 at 12 23 44 PM](https://github.com/user-attachments/assets/997fa96f-6048-4316-8986-667b7ef3bb8f)




# 5. Can we summarize the impact of revenue generated from each city/country?

__SQL Queries:__

- Just to make things cleaner I have created a view called __region_product_revenue__



``` sql
CREATE OR REPLACE VIEW region_product_revenue_view AS

WITH region_product_revenue AS (
    SELECT 
        ns.city AS city,
        ns.country AS country,
        COUNT(ns.productsku) AS total_orders,
        SUM(ns.productprice) / 1000000 AS productprice
    FROM new_sessions ns
    WHERE
        ns.city IS NOT NULL
        AND ns.country IS NOT NULL
        AND ns.productsku IS NOT NULL
        AND ns.city NOT IN ('(not set)', 'not available in demo dataset')
        AND ns.country NOT IN ('(not set)', 'not available in demo dataset')
        AND ns.productprice IS NOT NULL
    GROUP BY
        ns.productsku,
        ns.city,
        ns.country
)

SELECT
    
    pd.city,
    pd.country,
	ROUND(AVG(pd.productprice * pd.total_orders),2) AS avg_revenue,
 	SUM(pd.productprice * pd.total_orders) AS total_revenue
	
FROM
    region_product_revenue pd

GROUP BY
    pd.city,
    pd.country
```

- The purpose of the view is to do some Data cleaning and also aggregate product revenue based on region



![Screen Shot 2024-11-07 at 12 27 06 PM](https://github.com/user-attachments/assets/0b03ba88-cae6-4f54-ab0e-90a6023810ab)



- __PART 1__ : Regional Revenue Contribution Analysis — Evaluating Each City's Impact on Total Global Revenue

``` sql
WITH revenue_totals AS (
    SELECT 
        country,
        city,
        SUM(total_revenue) AS city_revenue---because we're grouping by city and country
    FROM 
        region_product_revenue_view
    GROUP BY 
        country, city
), 

total_global_revenue AS (
    SELECT 
        SUM(city_revenue) AS global_revenue---to get all cities revenue and think of it like country revenue
    FROM 
        revenue_totals
)
SELECT 
    rt.country,
    rt.city,
    TO_CHAR(city_revenue, 'FM999,999,999') AS city_revenue,
    ROUND((rt.city_revenue / tgr.global_revenue) * 100,4) AS revenue_percentage
FROM 
    revenue_totals rt,
    total_global_revenue tgr
ORDER BY 
    revenue_percentage DESC;
```

__Answer:__

- More than half of our revenue was generated from the city of __Mountain view__ in the __USA__



  ![Screen Shot 2024-11-07 at 12 28 27 PM](https://github.com/user-attachments/assets/5dff6421-15e6-4e9b-9e63-3212605b0e8d)



- __PART 2__ : Country-Level Revenue Contribution Analysis — Assessing Each Country's Share of Global Revenue
``` sql
WITH country_revenue AS (
    SELECT
        country,
        SUM(total_revenue) AS country_total_revenue
    FROM
        region_product_revenue_view
    GROUP BY
        country
),
global_revenue AS (
    SELECT
        SUM(country_total_revenue) AS global_total_revenue
    FROM
        country_revenue
)

SELECT
    cr.country,
    TO_CHAR(cr.country_total_revenue, 'FM999,999,999') AS formatted_country_revenue,
    ROUND((cr.country_total_revenue * 100.0 / gr.global_total_revenue), 2) AS country_global_impact_percentage
FROM
    country_revenue cr,
    global_revenue gr
ORDER BY
    country_global_impact_percentage DESC;
```


__Answer__:

In __Part 2__: Country-Level Revenue Contribution Analysis — Assessing Each Country's Share of Global Revenue we can see the __USA__ leading with __91.75__% of the total revenue



![Screen Shot 2024-11-07 at 12 29 38 PM](https://github.com/user-attachments/assets/f741e5bf-9b99-4fdc-aeba-dded913122e0)



# Schema for the ecommerce Database



![schema](https://github.com/user-attachments/assets/9f96e0e5-cef0-47df-8eea-badca7161817)










