# The issues I will address by cleaning the data?

1. I identified several rows in the city and country columns of the new_sessions table that contained values such as __empty string__ , __'not set'__, or __'not available in demo dataset'__.
2. To ensure accuracy throughout my analysis, I applied a filter condition and created a view that excludes these inconsistencies, allowing for more reliable results.
3. I have also dropped some columns where the was no __values/NULL__ in the rows.
   - The columns dropped are
      - __searchkeyword__
      - __productrefundamount__
      - __itemquantity__
      - __itemrevenue__ all of them are from new_sessions.

4. I have found three inconsistencies in __productsku__ column in __new_sessions__ where there was unwanted space in the sKu, so I trimmed out the space by replacing thee space with an empty string

5. In question number one, I have noticed the city of __New York__ in correctly associated with __Canada__ as country, so I corrected it to match it's respective country




##  SQL queries used to clean the data.

 
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
        ns.country;
```



```
	ALTER TABLE 
                new_sessions
	DROP COLUMN 
                searchkeyword;
```



```
	UPDATE 
                new_sessions
	SET 
                productsku = REPLACE(productsku, ' ', '')
	WHERE 
                NOT productsku ~ '^[A-Za-z0-9]+$';--checks sku's with non alphanumerical characters
```





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


*These are some of the queries I used when cleaning the data, some of them are sliced to capture the logic behind my code. Full queries are in the 'starting_with_questions.md' and starting_with_data.md'*






