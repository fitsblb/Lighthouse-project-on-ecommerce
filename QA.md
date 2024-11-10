# What are your risk areas? Identify and describe them.

1. I have found many rows in city and country in the new_sessions table where the rows were either or city was __'not set'__ or __'not available in demo dataset'__, so throughout my assignment I have adopted a filter condition and a view where I would filter them out to get concrete results

- __*FOR EXAMPLE*__, this query effectively elliminates any occurences of inconsisten data on the where condition. It also checks and eliminitates any __NULL__ values on the product price column.

  ```
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
  ```

  

2. I have also dropped some columns where the was __no values/NULL__ in the rows in the __new_session table__.



   ```
      ALTER TABLE 
            new_sessions
      DROP COLUMN 
            searchkeyword;
    ```


- The columns dropped are __searchkeyword, productrefundamount, itemquantity, itemrevenue__ ALL from __new_sessions__


3. I have changed dataype of the date column in __new_sessions__ which was in integer intially to proper date format by altering the table's format.


```
ALTER 
      TABLE new_sessions
ALTER 
      COLUMN date TYPE DATE USING TO_DATE(date::TEXT, 'YYYYMMDD');
```


4. I have found three inconsistencies in __productsku__ column in __new_sessions__ where there was unwanted space in the sku, so I trimmed out the space by replacing the space with an empty string.


   ```
   UPDATE 
      new_sessions
   SET 
      productsku = REPLACE(productsku, ' ', '')
   WHERE 
      NOT productsku ~ '^[A-Za-z0-9]+$';--checks sku's with non alphanumerical characters
   ```



5. In question number one, I have noticed the city of __New York__ in correctly associated with __Canada__ as country, so I corrected it using case method as seen below.


```
CASE
      WHEN TRIM(city) = 'New York' AND TRIM(country) = 'Canada' THEN 'New York'  -- Keep city as New York
      WHEN TRIM(city) IN ('(not set)', 'not available in demo dataset') THEN 'City in ' || TRIM(country)
      ELSE TRIM(city)
END AS modified_city,
CASE
      WHEN TRIM(city) = 'New York' AND TRIM(country) = 'Canada' THEN 'United States'  -- Change country to United States
      ELSE TRIM(country)
END AS country,  -- Trim spaces from country names
```

### There could have been alot of inconsistency like this but due to shortage of time, I could't go through all of them.



 

