# RFM ANALYSIS - MySQL & PYTHON

The objective of this analysis is to segment online customers using three metrics. These are Recency, Frequency, and Monetary value.


```python
# Load Image
from IPython.display import Image
from IPython.core.display import HTML 
Image(url= "https://geoffreynel.com/wp-content/uploads/2021/08/rfm-metrics-trans.png", 
      retina=True)
```




<img src="https://geoffreynel.com/wp-content/uploads/2021/08/rfm-metrics-trans.png"/>



Import libraries and connection information


```python
# Loading libraries
import pandas as pd
import time
import sqlalchemy
from sqlalchemy import create_engine
import pymysql

# Loading the connection information
%load_ext sql
%sql mysql+pymysql://root:admin@localhost:3306/brazilecom          
```

    The sql extension is already loaded. To reload it, use:
      %reload_ext sql
    




    'Connected: root@brazilecom'



## STEP 1

This first table will collect and join the necessary data. For this, we need to access 3 tables: Orders, Customers, Payments.

The critical fields that we need are Payment Amount, Customer Unique ID, and the Order Date.


```sql

%%sql
SELECT
    CAST(ord.order_purchase_timestamp AS DATE) AS orderDate,
    opay.payment_value AS amount,
    ord.order_id AS oid,
    cur.customer_unique_id AS cid,
    (SELECT 
            CAST(MAX(order_purchase_timestamp) AS DATE)
        FROM
            orders) AS maxDate
FROM
    orders AS ord
        JOIN
    customers AS cur ON ord.customer_id = cur.customer_id
        JOIN
    order_payments AS opay ON ord.order_id = opay.order_id
GROUP BY oid
LIMIT 5
```

     * mysql+pymysql://root:***@localhost:3306/brazilecom
    5 rows affected.
    




<table>
    <thead>
        <tr>
            <th>orderDate</th>
            <th>amount</th>
            <th>oid</th>
            <th>cid</th>
            <th>maxDate</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>2018-01-14</td>
            <td>48.91</td>
            <td>dc400373e624e9b0435847a79813c658</td>
            <td>d02cbb85434c84acabb8e81bbf064d3e</td>
            <td>2018-10-17</td>
        </tr>
        <tr>
            <td>2018-05-23</td>
            <td>20.75</td>
            <td>b616f638cd86d92c7cbd06cf7484ee34</td>
            <td>23daf66dd86071f57d332fa5562afbe9</td>
            <td>2018-10-17</td>
        </tr>
        <tr>
            <td>2017-10-04</td>
            <td>76.17</td>
            <td>407408433ee2a9ed77984aa5a06e2b2e</td>
            <td>350e8172d9a1ed3cea31f50e09de726f</td>
            <td>2018-10-17</td>
        </tr>
        <tr>
            <td>2018-07-03</td>
            <td>159.07</td>
            <td>aa466a92af916f9f68e56fce7d3a21b1</td>
            <td>a4031c5f35399c228a1cd7ed7431fd02</td>
            <td>2018-10-17</td>
        </tr>
        <tr>
            <td>2017-05-12</td>
            <td>45.69</td>
            <td>a41753c6a1d8accb89732e36243432d7</td>
            <td>dc83fe8f72fb4388f079a2f9b586240c</td>
            <td>2018-10-17</td>
        </tr>
    </tbody>
</table>



## STEP 2

The next step is to create a calculated table that give us a 


```sql
%%sql
-- STEP 1
with temp as (
        SELECT 
        CAST(ord.order_purchase_timestamp AS DATE) AS orderDate,
        opay.payment_value AS amount,
        ord.order_id AS oid,
        cur.customer_unique_id AS cid,
        (SELECT 
                CAST(MAX(order_purchase_timestamp) AS DATE)
            FROM
                orders) AS maxDate
    FROM
        orders AS ord
            JOIN
        customers AS cur ON ord.customer_id = cur.customer_id
            JOIN
        order_payments AS opay ON ord.order_id = opay.order_id
    GROUP BY oid
        )

-- STEP 2
-- This 'rfm' table will run baseline r-f-m calculations per customer. 
-- Note: we only include customers with a spend above $100.
    SELECT
        datediff(maxDate, max(orderDate)) as Recency,
        count(cid) as Frequency,
        sum(amount) as Monetary,
        cid
    FROM temp
    GROUP BY cid
    HAVING monetary > 100
    ORDER BY monetary desc
limit 5
```

     * mysql+pymysql://root:***@localhost:3306/brazilecom
    5 rows affected.
    




<table>
    <thead>
        <tr>
            <th>Recency</th>
            <th>Frequency</th>
            <th>Monetary</th>
            <th>cid</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>383</td>
            <td>1</td>
            <td>13664.08</td>
            <td>0a0a92112bd4c708ca5fde585afaa872</td>
        </tr>
        <tr>
            <td>61</td>
            <td>3</td>
            <td>9553.02</td>
            <td>46450c74a0d8c5ca9395da1daac6c120</td>
        </tr>
        <tr>
            <td>564</td>
            <td>2</td>
            <td>7571.63</td>
            <td>da122df9eeddfedc1dc1f5349a1a690c</td>
        </tr>
        <tr>
            <td>94</td>
            <td>1</td>
            <td>7274.88</td>
            <td>763c8b1c9c68a0229c42c9fc6f662b93</td>
        </tr>
        <tr>
            <td>612</td>
            <td>1</td>
            <td>6929.31</td>
            <td>dc4802a71eae9be1dd28f5d788ceb526</td>
        </tr>
    </tbody>
</table>



## STEP 3

In the next step, we take the Recency, Frequency, and Monetary values, sort them in sequntial order, then create a rank from 1-100, where a higher number is better/more valueable. We then sum that rank into a single score, which we later use to segment the customers.


```sql

%%sql
-- STEP 1
with temp as (
        SELECT 
        CAST(ord.order_purchase_timestamp AS DATE) AS orderDate,
        opay.payment_value AS amount,
        ord.order_id AS oid,
        cur.customer_unique_id AS cid,
        (SELECT 
                CAST(MAX(order_purchase_timestamp) AS DATE)
            FROM
                orders) AS maxDate
    FROM
        orders AS ord
            JOIN
        customers AS cur ON ord.customer_id = cur.customer_id
            JOIN
        order_payments AS opay ON ord.order_id = opay.order_id
    GROUP BY oid
        ),

-- STEP 2
    -- This 'rfm' table will run baseline r-f-m calculations per customer. 
    -- Note: we only include customers with a spend above $100.
        rfm as (
        SELECT
            datediff(maxDate, max(orderDate)) as Recency,
            count(cid) as Frequency,
            round(sum(amount),2) as Monetary,
            cid
        FROM temp
        GROUP BY cid
        HAVING monetary > 100
        ORDER BY monetary desc
        )

-- STEP 3
-- This 'rfmRanking' table will give us an ordered ranking per each evaluation metric (r-f-m)
-- this is one area that can be tweaked for optimized groups. (10-decile, 20-twentile, etc)
SELECT *,
    (rRank + fRank + mRank) AS score
FROM (
    SELECT 
        Recency as R,
        Frequency as F,
        Monetary as M,
        NTILE(100) over(ORDER BY Recency DESC) as rRank,
        NTILE(100) over(ORDER BY Frequency) as fRank,
        NTILE(100) over(ORDER BY Monetary) as mRank,
        cid
    FROM rfm
    ) as sub1
ORDER BY score DESC
LIMIT 5

```

     * mysql+pymysql://root:***@localhost:3306/brazilecom
    5 rows affected.
    




<table>
    <thead>
        <tr>
            <th>R</th>
            <th>F</th>
            <th>M</th>
            <th>rRank</th>
            <th>fRank</th>
            <th>mRank</th>
            <th>cid</th>
            <th>score</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>55</td>
            <td>6</td>
            <td>1094.63</td>
            <td>100</td>
            <td>100</td>
            <td>99</td>
            <td>dc813062e0fc23409cd255f7f53c7074</td>
            <td>299</td>
        </tr>
        <tr>
            <td>61</td>
            <td>3</td>
            <td>9553.02</td>
            <td>99</td>
            <td>100</td>
            <td>100</td>
            <td>46450c74a0d8c5ca9395da1daac6c120</td>
            <td>299</td>
        </tr>
        <tr>
            <td>65</td>
            <td>4</td>
            <td>1760.75</td>
            <td>98</td>
            <td>100</td>
            <td>100</td>
            <td>4facc2e6fbc2bffab2fea92d2b4aa7e4</td>
            <td>298</td>
        </tr>
        <tr>
            <td>58</td>
            <td>17</td>
            <td>927.63</td>
            <td>99</td>
            <td>100</td>
            <td>98</td>
            <td>8d50f5eadf50201ccdcedfb9e2ac8455</td>
            <td>297</td>
        </tr>
        <tr>
            <td>70</td>
            <td>4</td>
            <td>4655.91</td>
            <td>97</td>
            <td>100</td>
            <td>100</td>
            <td>c8460e4251689ba205045f3ea17884a1</td>
            <td>297</td>
        </tr>
    </tbody>
</table>



## STEP 4

In this step, we take the consolidated scores from our last table output and partition those scores into four parts. The top range of scores will distiguish our top customers, our VIPS, while the bottom part/range of scores will distinguish our lowest value customers.  


```sql

%%sql
-- STEP 1
with temp as (
        SELECT 
        CAST(ord.order_purchase_timestamp AS DATE) AS orderDate,
        opay.payment_value AS amount,
        ord.order_id AS oid,
        cur.customer_unique_id AS cid,
        (SELECT 
                CAST(MAX(order_purchase_timestamp) AS DATE)
            FROM
                orders) AS maxDate
    FROM
        orders AS ord
            JOIN
        customers AS cur ON ord.customer_id = cur.customer_id
            JOIN
        order_payments AS opay ON ord.order_id = opay.order_id
    GROUP BY oid
        ),

    -- STEP 2
    -- This 'rfm' table will run baseline r-f-m calculations per customer. 
    -- Note: we only include customers with a spend above $100.
        rfm as (
        SELECT
            maxDate,
            max(orderDate) as mostRecent,
            datediff(maxDate, max(orderDate)) as Recency,
            count(cid) as Frequency,
            round(sum(amount),2) as Monetary,
            cid
        FROM temp
        GROUP BY cid
        HAVING monetary > 100
        ORDER BY monetary desc
        ),

    -- STEP 3    
    -- This 'rfmRanking' table will give us an ordered ranking per each evaluation metric (r-f-m)
    -- this is one area that can be tweaked for optimized groups. (10-decile, 20-twentile, etc)
    rfmRanking as (
    SELECT *,
        (rRank + fRank + mRank) AS score
    FROM (
        SELECT 
            Recency as R,
            Frequency as F,
            Monetary as M,
            NTILE(100) over(ORDER BY Recency DESC) as rRank,
            NTILE(100) over(ORDER BY Frequency) as fRank,
            NTILE(100) over(ORDER BY Monetary) as mRank,
            cid
        FROM rfm
        ) as sub1
    ORDER BY score DESC
        )
    
-- STEP 4
-- This final query gives us the customer ranking/grouping by decreasing value and importance. 
-- 'GroupRank' = 1 will be our most valuable customers, while 10 is the least valuable.
-- This query gives us the average per each evaluation metric, but also a list to send to marketing.
-- The NTILE function allows us to tweaked and optimized groups (decile, twentile, etc),
-- so this query can be continually reassessed if need. It's also possible to further segment each
-- 'GroupRank' if needed, so we can split the top group to find the 'absolute' best customers.

select *,
    CASE 
    WHEN GroupRank >= 90 THEN 'Vip'
    WHEN GroupRank >= 75 THEN 'Loyal'
    WHEN GroupRank >= 50 THEN 'Insider'
    ELSE 'Starter'
    END AS Segment
FROM (SELECT *,
    NTILE(100) OVER(ORDER by score) as GroupRank
From rfmRanking) AS sub1
ORDER BY score DESC
limit 5
```

     * mysql+pymysql://root:***@localhost:3306/brazilecom
    5 rows affected.
    




<table>
    <thead>
        <tr>
            <th>R</th>
            <th>F</th>
            <th>M</th>
            <th>rRank</th>
            <th>fRank</th>
            <th>mRank</th>
            <th>cid</th>
            <th>score</th>
            <th>GroupRank</th>
            <th>Segment</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>55</td>
            <td>6</td>
            <td>1094.63</td>
            <td>100</td>
            <td>100</td>
            <td>99</td>
            <td>dc813062e0fc23409cd255f7f53c7074</td>
            <td>299</td>
            <td>100</td>
            <td>Vip</td>
        </tr>
        <tr>
            <td>61</td>
            <td>3</td>
            <td>9553.02</td>
            <td>99</td>
            <td>100</td>
            <td>100</td>
            <td>46450c74a0d8c5ca9395da1daac6c120</td>
            <td>299</td>
            <td>100</td>
            <td>Vip</td>
        </tr>
        <tr>
            <td>65</td>
            <td>4</td>
            <td>1760.75</td>
            <td>98</td>
            <td>100</td>
            <td>100</td>
            <td>4facc2e6fbc2bffab2fea92d2b4aa7e4</td>
            <td>298</td>
            <td>100</td>
            <td>Vip</td>
        </tr>
        <tr>
            <td>58</td>
            <td>17</td>
            <td>927.63</td>
            <td>99</td>
            <td>100</td>
            <td>98</td>
            <td>8d50f5eadf50201ccdcedfb9e2ac8455</td>
            <td>297</td>
            <td>100</td>
            <td>Vip</td>
        </tr>
        <tr>
            <td>50</td>
            <td>2</td>
            <td>895.36</td>
            <td>100</td>
            <td>98</td>
            <td>98</td>
            <td>4e1554d1a2dcb38dcf22770eb7390fe1</td>
            <td>296</td>
            <td>100</td>
            <td>Vip</td>
        </tr>
    </tbody>
</table>



## STEP 5 - Review

In this final table, we can aggregate the values above to review that our calculations are ordered correctly. We see that our VIP segment orders on average more recently, more frequently, and has a higher average Monetary spend than all other segments. These metrics are also all consistent or monotonic, which is exactly what we want.


```sql

%%sql
-- STEP 1
with temp as (
        SELECT 
        CAST(ord.order_purchase_timestamp AS DATE) AS orderDate,
        opay.payment_value AS amount,
        ord.order_id AS oid,
        cur.customer_unique_id AS cid,
        (SELECT 
                CAST(MAX(order_purchase_timestamp) AS DATE)
            FROM
                orders) AS maxDate
    FROM
        orders AS ord
            JOIN
        customers AS cur ON ord.customer_id = cur.customer_id
            JOIN
        order_payments AS opay ON ord.order_id = opay.order_id
    GROUP BY oid
        ),

    -- STEP 2
    -- This 'rfm' table will run baseline r-f-m calculations per customer. 
    -- Note: we only include customers with a spend above $100.
        rfm as (
        SELECT
            maxDate,
            max(orderDate) as mostRecent,
            datediff(maxDate, max(orderDate)) as Recency,
            count(cid) as Frequency,
            sum(amount) as Monetary,
            cid
        FROM temp
        GROUP BY cid
        HAVING monetary > 100
        ORDER BY monetary desc
        ),

    -- STEP 3    
    -- This 'rfmRanking' table will give us an ordered ranking per each evaluation metric (r-f-m)
    -- this is one area that can be tweaked for optimized groups. (10-decile, 20-twentile, etc)
    rfmRanking as (
    SELECT *,
        (rRank + fRank + mRank) AS score
    FROM (
        SELECT 
            Recency as R,
            Frequency as F,
            Monetary as M,
            NTILE(100) over(ORDER BY Recency DESC) as rRank,
            NTILE(100) over(ORDER BY Frequency) as fRank,
            NTILE(100) over(ORDER BY Monetary) as mRank,
            cid
        FROM rfm
        ) as sub1
    ORDER BY score DESC
        ),
    
-- STEP 4
-- This final query gives us the customer ranking/grouping by decreasing value and importance. 
-- 'GroupRank' = 1 will be our most valuable customers, while 10 is the least valuable.
-- This query gives us the average per each evaluation metric, but also a list to send to marketing.
-- The NTILE function allows us to tweaked and optimized groups (decile, twentile, etc),
-- so this query can be continually reassessed if need. It's also possible to further segment each
-- 'GroupRank' if needed, so we can split the top group to find the 'absolute' best customers.
    RFMFinalTable as (
    select *,
        CASE 
        WHEN GroupRank >= 90 THEN 'Vip'
        WHEN GroupRank >= 75 THEN 'Loyal'
        WHEN GroupRank >= 50 THEN 'Insider'
        ELSE 'Starter'
        END AS Segment
    FROM (SELECT *,
        NTILE(100) OVER(ORDER by score) as GroupRank
    From rfmRanking) AS sub1
    ORDER BY score DESC)

SELECT Segment,
        round(avg(r)) as RecAvg,
        round(avg(r) / sum(avg(r)) over()*100,1) as RecPerc,
        round(avg(f), 2) as FreqAvg,
        round(avg(f) / sum(avg(f)) over()*100,1) as FreqPerc,
        round(avg(m)) as MonAvg,
        round(avg(m) / sum(avg(m)) over()*100,1) as MonPerc,
        count(cid) as CustCount,
        round(count(cid) / sum(count(cid)) over()*100,1) as CustPerc
--        ,group_concat(cid SEPARATOR ", ") CustomerList
FROM RFMFinalTable
GROUP BY Segment
ORDER BY GroupRank DESC
```

     * mysql+pymysql://root:***@localhost:3306/brazilecom
    4 rows affected.
    




<table>
    <thead>
        <tr>
            <th>Segment</th>
            <th>RecAvg</th>
            <th>RecPerc</th>
            <th>FreqAvg</th>
            <th>FreqPerc</th>
            <th>MonAvg</th>
            <th>MonPerc</th>
            <th>CustCount</th>
            <th>CustPerc</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Vip</td>
            <td>231</td>
            <td>21.4</td>
            <td>1.39</td>
            <td>31.2</td>
            <td>591.0</td>
            <td>42.4</td>
            <td>5577</td>
            <td>11.0</td>
        </tr>
        <tr>
            <td>Loyal</td>
            <td>258</td>
            <td>23.8</td>
            <td>1.05</td>
            <td>23.4</td>
            <td>414.0</td>
            <td>29.7</td>
            <td>7605</td>
            <td>15.0</td>
        </tr>
        <tr>
            <td>Insider</td>
            <td>286</td>
            <td>26.4</td>
            <td>1.02</td>
            <td>22.9</td>
            <td>248.0</td>
            <td>17.8</td>
            <td>12683</td>
            <td>25.0</td>
        </tr>
        <tr>
            <td>Starter</td>
            <td>307</td>
            <td>28.4</td>
            <td>1.01</td>
            <td>22.5</td>
            <td>139.0</td>
            <td>10.0</td>
            <td>24892</td>
            <td>49.0</td>
        </tr>
    </tbody>
</table>



## ALL SQL Code - Consolidated


```sql
%%sql
(
-- STEP 1
with temp as (
        SELECT 
        CAST(ord.order_purchase_timestamp AS DATE) AS orderDate,
        opay.payment_value AS amount,
        ord.order_id AS oid,
        cur.customer_unique_id AS cid,
        (SELECT 
                CAST(MAX(order_purchase_timestamp) AS DATE)
            FROM
                orders) AS maxDate
    FROM
        orders AS ord
            JOIN
        customers AS cur ON ord.customer_id = cur.customer_id
            JOIN
        order_payments AS opay ON ord.order_id = opay.order_id
    GROUP BY oid
        ),

    -- STEP 2
    -- This 'rfm' table will run baseline r-f-m calculations per customer. 
    -- Note: we only include customers with a spend above $100.
        rfm as (
        SELECT
            maxDate,
            max(orderDate) as mostRecent,
            datediff(maxDate, max(orderDate)) as Recency,
            count(cid) as Frequency,
            sum(amount) as Monetary,
            cid
        FROM temp
        GROUP BY cid
        HAVING monetary > 100
        ORDER BY monetary desc
        ),

    -- STEP 3    
    -- This 'rfmRanking' table will give us an ordered ranking per each evaluation metric (r-f-m)
    -- this is one area that can be tweaked for optimized groups. (10-decile, 20-twentile, etc)
    rfmRanking as (
    SELECT *,
        (rRank + fRank + mRank) AS score
    FROM (
        SELECT 
            Recency as R,
            Frequency as F,
            Monetary as M,
            NTILE(100) over(ORDER BY Recency DESC) as rRank,
            NTILE(100) over(ORDER BY Frequency) as fRank,
            NTILE(100) over(ORDER BY Monetary) as mRank,
            cid
        FROM rfm
        ) as sub1
    ORDER BY score DESC
        ),
    
-- STEP 4
-- This final query gives us the customer ranking/grouping by decreasing value and importance. 
-- 'GroupRank' = 1 will be our most valuable customers, while 10 is the least valuable.
-- This query gives us the average per each evaluation metric, but also a list to send to marketing.
-- The NTILE function allows us to tweaked and optimized groups (decile, twentile, etc),
-- so this query can be continually reassessed if need. It's also possible to further segment each
-- 'GroupRank' if needed, so we can split the top group to find the 'absolute' best customers.
    RFMFinalTable as (
    select *,
        CASE 
        WHEN GroupRank >= 90 THEN 'Vip'
        WHEN GroupRank >= 75 THEN 'Loyal'
        WHEN GroupRank >= 50 THEN 'Insider'
        ELSE 'Starter'
        END AS Segment
    FROM (SELECT *,
        NTILE(100) OVER(ORDER by score) as GroupRank
    From rfmRanking) AS sub1
    ORDER BY score DESC)

SELECT Segment,
        round(avg(r)) as RecAvg,
        round(avg(r) / sum(avg(r)) over()*100,1) as RecPerc,
        round(avg(f), 2) as FreqAvg,
        round(avg(f) / sum(avg(f)) over()*100,1) as FreqPerc,
        round(avg(m)) as MonAvg,
        round(avg(m) / sum(avg(m)) over()*100,1) as MonPerc,
        count(cid) as CustCount,
        round(count(cid) / sum(count(cid)) over()*100,1) as CustPerc
--        ,group_concat(cid SEPARATOR ", ") CustomerList
FROM RFMFinalTable
GROUP BY Segment
ORDER BY GroupRank DESC
)
```

 

## Visualize Output


```python
# Load data into a dataframe
import mysql.connector
import pandas as pd

#Function to Pull Data
def sq(query):
    conn = mysql.connector.connect(
        user="root", database="brazilecom", password="admin", host="localhost", port=3306
    )
    cursor = conn.cursor()
    query = query
    df = pd.read_sql(query, con=conn)
    df = pd.DataFrame(df)
    conn.close()
    return df

df = sq("""

(
-- STEP 1
with temp as (
            SELECT 
            CAST(ord.order_purchase_timestamp AS DATE) AS orderDate,
            opay.payment_value AS amount,
            ord.order_id AS oid,
            cur.customer_unique_id AS cid,
            (SELECT 
                    CAST(MAX(order_purchase_timestamp) AS DATE)
                FROM
                    orders) AS maxDate
        FROM
            orders AS ord
                JOIN
            customers AS cur ON ord.customer_id = cur.customer_id
                JOIN
            order_payments AS opay ON ord.order_id = opay.order_id
        GROUP BY oid
            ),

        -- STEP 2
        -- This 'rfm' table will run baseline r-f-m calculations per customer. 
        -- Note: we only include customers with a spend above $100.
            rfm as (
            SELECT
                maxDate,
                max(orderDate) as mostRecent,
                datediff(maxDate, max(orderDate)) as Recency,
                count(cid) as Frequency,
                sum(amount) as Monetary,
                cid
            FROM temp
            GROUP BY cid
            HAVING monetary > 100
            ORDER BY monetary desc
            ),

        -- STEP 3    
        -- This 'rfmRanking' table will give us an ordered ranking per each evaluation metric (r-f-m)
        -- this is one area that can be tweaked for optimized groups. (10-decile, 20-twentile, etc)
        rfmRanking as (
        SELECT *,
            (rRank + fRank + mRank) AS score
        FROM (
            SELECT 
                Recency as R,
                Frequency as F,
                Monetary as M,
                NTILE(100) over(ORDER BY Recency DESC) as rRank,
                NTILE(100) over(ORDER BY Frequency) as fRank,
                NTILE(100) over(ORDER BY Monetary) as mRank,
                cid
            FROM rfm
            ) as sub1
        ORDER BY score DESC
            ),

    -- STEP 4
    -- This final query gives us the customer ranking/grouping by decreasing value and importance. 
    -- 'GroupRank' = 1 will be our most valuable customers, while 10 is the least valuable.
    -- This query gives us the average per each evaluation metric, but also a list to send to marketing.
    -- The NTILE function allows us to tweaked and optimized groups (decile, twentile, etc),
    -- so this query can be continually reassessed if need. It's also possible to further segment each
    -- 'GroupRank' if needed, so we can split the top group to find the 'absolute' best customers.
        RFMFinalTable as (
        select *,
            CASE 
            WHEN GroupRank >= 90 THEN 'Vip'
            WHEN GroupRank >= 75 THEN 'Loyal'
            WHEN GroupRank >= 50 THEN 'Insider'
            ELSE 'Starter'
            END AS Segment
        FROM (SELECT *,
            NTILE(100) OVER(ORDER by score) as GroupRank
        From rfmRanking) AS sub1
        ORDER BY score DESC)

    SELECT Segment,
            round(avg(r)) as RecAvg,
            round(avg(r) / sum(avg(r)) over()*100,1) as RecPerc,
            round(avg(f), 2) as FreqAvg,
            round(avg(f) / sum(avg(f)) over()*100,1) as FreqPerc,
            round(avg(m)) as MonAvg,
            round(avg(m) / sum(avg(m)) over()*100,1) as MonPerc,
            count(cid) as CustCount,
            round(count(cid) / sum(count(cid)) over()*100,1) as CustPerc
    --        ,group_concat(cid SEPARATOR ", ") CustomerList
    FROM RFMFinalTable
    GROUP BY Segment
    ORDER BY GroupRank DESC
    )

""")

```

#### Reset Dataframe


```python
# Restructure dataframe for plotting
plotData = df.set_index('Segment').copy()
plotData = plotData[['RecPerc', 'FreqPerc', 'MonPerc']]
plotData.columns = ['Recency', 'Frequency', 'Monetary']
plotData
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Recency</th>
      <th>Frequency</th>
      <th>Monetary</th>
    </tr>
    <tr>
      <th>Segment</th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Vip</th>
      <td>21.4</td>
      <td>31.2</td>
      <td>42.4</td>
    </tr>
    <tr>
      <th>Loyal</th>
      <td>23.8</td>
      <td>23.4</td>
      <td>29.7</td>
    </tr>
    <tr>
      <th>Insider</th>
      <td>26.4</td>
      <td>22.9</td>
      <td>17.8</td>
    </tr>
    <tr>
      <th>Starter</th>
      <td>28.4</td>
      <td>22.5</td>
      <td>10.0</td>
    </tr>
  </tbody>
</table>
</div>



#### Plot Data

Below we can see a visualization of each segment by our evaluation metrics. Although it is possible to further refine, we can be happy with the consistency as well as the monotonicity of the results. Each group is clearly "Better" than the next by each evaluation metric. 


```python
# Import libraries and plot
import matplotlib.pyplot as plt
import seaborn as sns
plt.style.use('ggplot')
sns.set_palette("Blues_d")

# Plot Visuals
fig, axes = plt.subplots(1,3, sharey=True, figsize=(12,8))
# fig.suptitle('Plot')

sns.barplot(x=plotData.index, y=plotData['Recency'], ax=axes[0])
axes[0].set_title('Recency - Lower is better')

sns.barplot(x=plotData.index, y=plotData['Frequency'], ax=axes[1])
axes[1].set_title('Frequency - Higher is better')

sns.barplot(x=plotData.index, y=plotData['Monetary'], ax=axes[2])
axes[2].set_title('Monetary - Higher is better')

plt.show()

```


    
![png](output_29_0.png)
    

