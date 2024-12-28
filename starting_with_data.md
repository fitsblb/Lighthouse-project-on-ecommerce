## Question 1: 
## EVALUATING SENTIMENT SCORE BY PRODUCTS TO FIND RELATIONSHIP BETWEEN SENTIMEN SCORE AND TOTAL ORDERED

__SQL Queries:__



``` sql
        SELECT
            ns.productsku,
            ns.v2productname AS productname,
            sr.total_ordered,
            sr.sentimentscore,
            sr.total_ordered * (ns.productprice / 100000) AS productrevenue,
            RANK() OVER (ORDER BY sr.total_ordered DESC) AS total_order_rank,
            RANK() OVER (ORDER BY sr.total_ordered * (ns.productprice / 100000) DESC) AS total_revenue_rank
        FROM
            new_sessions ns
         JOIN
            sales_report sr
        USING (productsku)
        WHERE
            ns.productprice != 0
            AND ns.productprice IS NOT NULL
            AND sr.total_ordered != 0
            AND sr.total_ordered IS NOT NULL
        GROUP BY
            ns.productsku,
            ns.v2productname,
            sr.total_ordered,
            sr.sentimentscore,
            ns.productprice
        ORDER BY
            sr.sentimentscore DESC;
``` 

__Answer:__ 
- This query ranks products by both total orders and total revenue, comparing these rankings against sentiment scores. Despite this analysis, no clear relationship emerges between sentiment scores and either total orders or revenue. This suggests that high sentiment scores don't necessarily correlate with higher sales or revenue generation for these products.



![Screen Shot 2024-11-07 at 1 01 03 PM](https://github.com/user-attachments/assets/c2d18ffa-c85b-46d3-8fbf-9ac1d89def40)



## Question 2: 
## What are the most common entry pages (pagepathlevel1) for visitors

__SQL Queries:__

``` sql
    WITH entry_visit AS (
            SELECT
                COUNT(DISTINCT fullvisitorid) AS entry_count,
                pagepathlevel1
             FROM
                new_sessions
             GROUP BY
                pagepathlevel1 
        ),
    total_entries AS (
            SELECT
                SUM(entry_count) AS total_count -- Single value expected
            FROM
                entry_visit
        )
            SELECT
                ev.pagepathlevel1,
                ev.entry_count,
                ROUND((ev.entry_count::NUMERIC / te.total_count) * 100, 2) || '%' AS percentage_entry
            FROM
                entry_visit ev, 
                total_entries te -- Calling from two CTEs
            ORDER BY
                ev.entry_count DESC;
``` 


__Answer:__
- This query reveals that the vast majority of visitors __(94.5%)__ enter the site through the page path __"/google+redesign/"__. Such a high entry percentage indicates that this page serves as a primary entry point, likely attracting significant traffic or being widely linked or promoted. Understanding this can help focus optimization efforts on this crucial entry page to enhance visitor engagement and conversion rates.



![Screen Shot 2024-11-07 at 1 03 33 PM](https://github.com/user-attachments/assets/eb8ddccf-7a1e-4349-91d1-e72649c340ee)




## Question 3: 
## Is there any relationship between the number of pages visitors view and the total number of units the buy?

__SQL Queries:__
 
``` sql
        SELECT
            full_visitor_id,
            SUM(pageviews) AS total_views,
            SUM(units_sold) AS total_units_sold,
            DENSE_RANK() OVER (ORDER BY SUM(pageviews) DESC) AS view_rank,
            DENSE_RANK() OVER (ORDER BY SUM(units_sold) DESC) AS units_sold_rank
        FROM 
            analytics
        WHERE 
            pageviews IS NOT NULL AND units_sold IS NOT NULL
        GROUP BY
            full_visitor_id
        ORDER BY
            units_sold_rank
``` 


__Answer:__
- As we can see in the results, there is no clear relationship between how many times a visitor views a page and the number of units they might actually purchase.

![Screen Shot 2024-11-07 at 1 08 29 PM](https://github.com/user-attachments/assets/b3c51a19-74e3-4075-acd8-42857f371f80)



## Question 4: 
## What does the relationship between transaction date and revenue look like? Has it grown over the years?

__SQL Queries:__



``` sql
    WITH revenue_details AS (
        SELECT
            EXTRACT(YEAR FROM ns.transaction_date) AS transaction_year,
            EXTRACT(MONTH FROM ns.transaction_date) AS transaction_month,
            SUM(a.revenue) AS revenue_total,
            RANK() OVER (ORDER BY SUM(a.revenue) DESC) AS max_revenue,
            RANK() OVER (ORDER BY SUM(a.revenue)) AS min_revenue
        FROM
            new_sessions ns
        JOIN
            analytics a ON a.full_visitor_id = ns.fullvisitorid
        WHERE
            a.revenue IS NOT NULL
        GROUP BY
            EXTRACT(YEAR FROM ns.transaction_date),
            EXTRACT(MONTH FROM ns.transaction_date)
        ORDER BY
            transaction_year,
            transaction_month,
            revenue_total
    )
        SELECT *
        FROM 
        revenue_details
        WHERE 
        max_revenue = 1 OR min_revenue = 1
```

__Answer:__

- As we can see, the website has transaction revenue data for two years. The lowest revenue was in __October 2016__, and the highest revenue was in __May 2017__.


![Screen Shot 2024-11-07 at 1 14 39 PM](https://github.com/user-attachments/assets/69bd1b6a-b479-4750-86a7-3527132774c1)


![Screen Shot 2024-11-07 at 1 13 43 PM](https://github.com/user-attachments/assets/491fcf61-bc27-4c5e-b923-40d6c3ab6929)



