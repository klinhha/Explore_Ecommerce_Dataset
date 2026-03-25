# E-commerce Analytics Project (Google Analytics - BigQuery)
## I. Overview
 This project analyzes an e-commerce dataset using Google Analytics sample data on BigQuery. 
 
 The goal is to understand user behavior, conversion performance, and revenue patterns across different traffic sources and devices.

## II. Objectives
- Analyze website traffic (visits, pageviews, transactions)
- Evaluate marketing performance by traffic source
- Measure conversion rate and user behavior
- Understand purchase patterns and product relationships
- Build funnel and cohort analysis

## III. Tools & Technologies
- SQL (Google BigQuery)
- Google Analytics Sample Dataset

## IV. Dataset
 Source: ga_sessions_@BigQueryDateShardedTable

## V. Key Analyses & Resultsyses & Results
### 1. Calculate total visit, pageview, transaction for Jan, Feb and March 2017 (order by month)
```
SELECT format_date('%Y%m', parse_date('%Y%m%d', date)) month
  , sum(totals.visits) visits
  , sum(totals.pageviews) pageviews
  , sum(totals.transactions) transactions
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*` 
where _table_suffix between '0101' and '0331'
group by month
order by month;
```
#### Result: 
<img width="636" height="107" alt="Image" src="https://github.com/user-attachments/assets/fd068a9a-e940-4a96-9edb-e79d8ab86bd5" />

### 2. Bounce Rate by Traffic Source
```
SELECT trafficSource.source source
  , sum(totals.visits) total_visits
  , sum(totals.bounces) total_no_of_bounces
  , round(sum(totals.bounces) / sum(totals.visits) * 100, 3) bounce_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
where _table_suffix between '0701' and '0731'
group by source
order by total_visits desc;
```
#### Result: 
<img width="525" height="677" alt="Image" src="https://github.com/user-attachments/assets/714e7d43-bf7e-4427-8108-9ae9bd2958ce" />

### 3. Revenue by traffic source by week, by month in June 2017
```
     -- Revenue by month
SELECT 'Month' time_type
  , format_date('%Y%m', parse_date('%Y%m%d', date)) time
  , trafficSource.source source
  , round(sum(product.productRevenue) / 1000000, 4) revenue
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
UNNEST(hits) hits,
UNNEST(hits.product) product
where product.productRevenue is not NULL
group by time_type, time, source

UNION ALL

     -- Revenue by week
SELECT 'Week' time_type
  , format_date('%Y%W', parse_date('%Y%m%d', date)) time
  , trafficSource.source source
  , round(sum(product.productRevenue) / 1000000, 4) revenue
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
UNNEST(hits) hits,
UNNEST(hits.product) product
where product.productRevenue is not NULL
group by time_type, time, source
order by revenue desc;
```

#### Result: 
<img width="781" height="677" alt="Image" src="https://github.com/user-attachments/assets/ab76e8bc-191a-4b31-a4fb-c62497e9bfad" />

### 4. Conversion rate by traffic source in 2017. (order by conversion_rate desc)
```
SELECT trafficSource.source source
  , sum(totals.visits) visits
  , sum(totals.transactions) transactions
  , round(sum(totals.transactions) / sum(totals.visits) * 100) conversion_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
where _table_suffix between '0101' and '1231'
group by source
having transactions >= 50
order by conversion_rate desc;
```

#### Result:
<img width="628" height="108" alt="Image" src="https://github.com/user-attachments/assets/fcbe51d9-6509-4d11-8896-9352a30fb3f7" />

### 5. Average number of pageviews by purchaser type (purchasers vs non-purchasers) in June, July 2017.
```
     -- Avg by purchasers
with purchasers as(
  SELECT format_date('%Y%m', parse_date('%Y%m%d', date)) month
    , round(sum(totals.pageviews) / count(distinct fullVisitorId), 7) avg_pageviews_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST(hits) hits,
  UNNEST(hits.product) product
  where _table_suffix between '0601' and '0731'
    and totals.transactions >= 1
    and product.productRevenue is not NULL
  group by month)

     -- Avg by non-purchasers
, non_purchasers as(
  SELECT format_date('%Y%m', parse_date('%Y%m%d', date)) month
    , round(sum(totals.pageviews) / count(distinct fullVisitorId), 7) avg_pageviews_non_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST(hits) hits,
  UNNEST(hits.product) product
  where _table_suffix between '0601' and '0731'
    and totals.transactions is NULL
    and product.productRevenue is NULL
  group by month)

    -- Avg by all type
select *
from purchasers
left join non_purchasers
using(month)
order by month;
```

#### Result:  
<img width="603" height="80" alt="Image" src="https://github.com/user-attachments/assets/d7291696-702f-4c09-a836-c4feaa1553f7" />

### 6. Average number of transactions per user that made a purchase in July 2017
```
SELECT format_date('%Y%m', parse_date('%Y%m%d', date)) Month
  , round(sum(totals.transactions) / count(distinct fullVisitorId), 9) Avg_total_transactions_per_user
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
UNNEST(hits) hits,
UNNEST(hits.product) product
where totals.transactions >= 1
  and product.productRevenue is not NULL
group by Month;
```

#### Result: 
<img width="343" height="53" alt="Image" src="https://github.com/user-attachments/assets/bd78da18-d26f-4d3f-b549-d6b55a10b25e" />

### 7. Revenue contribution by device (desktop,mobile...) (order by ratio desc)
```
with device_revenue as ( 
 SELECT device.deviceCategory device
    , round(sum(product.productRevenue) / 1000000, 2) revenue_by_device
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`,
  UNNEST(hits) hits,
  UNNEST(hits.product) product
  where totals.transactions is not NULL 
    and product.productRevenue is not NULL
  group by device
)

, total_rev as (
  select round(sum(revenue_by_device), 2) total_revenue
  from device_revenue
)

select d.device
  , d.revenue_by_device
  , t.total_revenue
  , round((d.revenue_by_device / t.total_revenue) * 100, 2) ratio
from device_revenue d
cross join total_rev t
order by ratio desc;
```

#### Result: 
<img width="525" height="108" alt="Image" src="https://github.com/user-attachments/assets/7e30b0b0-bd51-421a-952c-6c40ef715888" />

### 8. Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017. 
```
-- List users bought product "YouTube Men's Vintage Henley"
with buyers as (
  SELECT distinct fullVisitorId purchasers
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST(hits) hits,
  UNNEST(hits.product) product
  where product.productRevenue is not NULL
    and totals.transactions >= 1
    and product.v2ProductName = "YouTube Men's Vintage Henley"
)

SELECT product.v2ProductName other_purchased_products
  , sum(product.productQuantity) quantity
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` a,
UNNEST(hits) hits,
UNNEST(hits.product) product
inner join buyers b
  on a.fullVisitorId = b.purchasers
where product.productRevenue is not NULL
  and totals.transactions >= 1
  and product.v2ProductName != "YouTube Men's Vintage Henley"
group by other_purchased_products
order by quantity desc;
```

#### Result:
<img width="485" height="568" alt="Image" src="https://github.com/user-attachments/assets/9f6e5a69-950f-4d6c-96a6-7d5403379fc7" />

### 9. Calculate cohort map from product view to addtocart to purchase in Jan, Feb and March 2017. For example, 100% product view then 40% add_to_cart and 10% purchase.
- Option 1:
```
with view_page as (
  SELECT format_date('%Y%m', parse_date('%Y%m%d', date)) month
    , count(hits.eCommerceAction.action_type) num_product_view
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST(hits) AS hits
  where _table_suffix between '0101' and '0331'
    and hits.eCommerceAction.action_type = '2'
  group by month
)
, add_to_cart as (
  SELECT format_date('%Y%m', parse_date('%Y%m%d', date)) month
    , count(hits.eCommerceAction.action_type) num_addtocart
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST(hits) AS hits
  where _table_suffix between '0101' and '0331'
    and hits.eCommerceAction.action_type = '3'
  group by month
)
, purchase as (
  SELECT format_date('%Y%m', parse_date('%Y%m%d', date)) month
    , count(hits.eCommerceAction.action_type) num_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST(hits) AS hits,
  UNNEST(hits.product) AS product
  where _table_suffix between '0101' and '0331'
    and hits.eCommerceAction.action_type = '6'
    and product.productRevenue is not NULL
  group by month
)

select *
  , round((num_addtocart / num_product_view) * 100, 2) add_to_cart_rate
  , round((num_purchase / num_product_view) * 100, 2) purchase_rate
FROM view_page 
left join add_to_cart 
using (month)
left join purchase 
using (month)
order by month;
```

- Option 2:
```
with raw_data as (
  SELECT format_date('%Y%m', parse_date('%Y%m%d', date)) month
    , sum(case when hits.eCommerceAction.action_type = '2' then 1 else 0 end) num_product_view
    , sum(case when hits.eCommerceAction.action_type = '3' then 1 else 0 end) num_addtocart
    , sum(case when hits.eCommerceAction.action_type = '6'
            and product.productRevenue is not NULL 
              then 1 else 0 end) num_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST(hits) hits,
  UNNEST(hits.product) product
  where _table_suffix between '0101' and '0331'
  group by month
)

select *
  , round((num_addtocart / num_product_view) * 100, 2) add_to_cart_rate
  , round((num_purchase / num_product_view) * 100, 2) purchase_rate
FROM raw_data
order by month;
```

#### Result:
<img width="784" height="109" alt="Image" src="https://github.com/user-attachments/assets/79574d43-f5de-4355-ae64-38ad63b01a35" />

### 10. Calculate revenue by week from May to July 2017 and culmulative revenue.
```
with week_revenue as(
  SELECT format_date('%Y%W', parse_date('%Y%m%d', date)) Week
    , round(sum(product.productRevenue) / 1000000, 2) weekly_revenue
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST(hits) hits,
  UNNEST(hits.product) product
  where _table_suffix between '0501' and '0731'
    and product.productRevenue is not null
  group by Week
  order by Week
)

-- caculate cumulative revenue
select *
  , sum(weekly_revenue) over(order by Week) cumulative_revenue
from week_revenue 
order by Week;
```

#### Result:
<img width="417" height="406" alt="Image" src="https://github.com/user-attachments/assets/216a9a7d-3411-43ae-9d19-854be30f482f" />
