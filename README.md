# E-Commerce Performances Analysis with SQL, GG Data Studio

## Project Purpose
This project helps to improve the business performance of an e-commerce company by analyzing related data to answer business questions relevant to 3 aspects:
- Customer growth
- Product Quality
- Payment methods.

The biggest business questions are:
- How are Customer Growth, Product Quality, and Payment Methods situation in the company now? 
- What should be focused more to improve?

## Method and Technology

A report is made to visualize results in Google Data Studio. The data processing stage is carried out by using PostgreSQL.

From this analysis, the company could find out the best practices or solutions for their concern aspects.

## Dataset Brief

The dataset is taken from Kaggle. There are 100K orders from 2016 to 2018 about marketplaces in Brazil. 

There are 8 CSV files. Dimensions from the dataset include order, price, payment, customer location, and other product attributes.

Steps to analyze dataset:
1. Data processing.
It is to prepare raw data into structured and usable data.

2. Create a new database and its tables for the data that has been prepared by paying attention to the data type of each column.
3. Import CSV data into the database by paying attention to the dataset storage path.
4. Create Data Relationship(link) based on the schema in Figure 
5. Create Entity Relationship Diagram (ERD) (link) in PNG by setting the data type and naming the columns between interconnected tables. Adjust primary and foreign keys


## Analysis

1. _**Annual Customer Growth Analysis**_

a) Calculate average monthly active users per year

```sql
select year, round(avg(total_customer),0) as avg_active_user
from
(select date_part('year', od.order_purchase_timestamp) as year,
   date_part('month', od.order_purchase_timestamp) as month,
   count(distinct cd.customer_unique_id) as total_customer
from order_dataset as od
join customer_dataset as cd on od.customer_id = cd.customer_id
group by 1,2) a
group by 1;

```
Output: (link)
 

d. Calculate average orders per year

```sql
select year, round(avg(total_order),2) as avg_frequency_order
from
(select date_part('year', od.order_purchase_timestamp) as year,
   cd.customer_unique_id,
   count(distinct order_id) as total_order
from order_dataset as od
join customer_dataset as cd on od.customer_id = cd.customer_id
group by 1,2
) a
group by 1
order by 1;
```
Output : (link)
 

e. Create a CTE (Common Table Expression) to combine all previous results. This is the official data analysis part using SQL

```sql
with count_mau as (
select year, round(avg(total_customer),0) as avg_active_user
from
(select date_part('year', od.order_purchase_timestamp) as year,
   date_part('month', od.order_purchase_timestamp) as month,
   count(distinct cd.customer_unique_id) as total_customer
from order_dataset as od
join customer_dataset as cd on od.customer_id = cd.customer_id
group by 1,2) a
group by 1
),count_newcust as(
select 
 date_part('year', first_time_order) as year, 
 count(a.customer_unique_id) as new_customers 
from (
        select 
   c.customer_unique_id,
            min(o.order_purchase_timestamp) as first_time_order
  from order_dataset o
  inner join customer_dataset c on c.customer_id = o.customer_id
  group by 1
) as a
group by 1
order by 1
),count_repeat_order as(
select year, count(total_customer) as repeat_order
from
(select date_part('year', od.order_purchase_timestamp) as year,
   cd.customer_unique_id,
   count(cd.customer_unique_id) as total_customer,
   count(od.order_id) as total_order
from order_dataset as od
join customer_dataset as cd on od.customer_id = cd.customer_id
group by 1,2
having count(order_id) >1
) a
group by 1
order by 1
),avg_order as (
select year, round(avg(total_order),2) as avg_frequency_order
from
(select date_part('year', od.order_purchase_timestamp) as year,
   cd.customer_unique_id,
   count(distinct order_id) as total_order
from order_dataset as od
join customer_dataset as cd on od.customer_id = cd.customer_id
group by 1,2
) a
group by 1
order by 1)select 
cm.year,
cm.avg_active_user,
cn.new_customers,
cro.repeat_order,
ao.avg_frequency_order
from count_mau cm 
join count_newcust cn on cm.year=cn.year
join count_repeat_order cro on cm.year=cro.year
join avg_order ao on cm.year=ao.year;
```
Output (link)


### Visualize result using Data Studio 
- Link 1:

- Link 2:

•	A significant increase in 2017 because the data available in 2016 was only for the last 4 months, starting in September. Therefore, there could be only an increase in growth in 2017.
•	Meanwhile, the trend of Monthly Active Users (MAU) also increases every year, reaching 5,338 customers.
•	An increasing trend from 2017 to 2018 for the number of customers who made a purchase more than once. However, there was a slight decrease in the number of customers who made purchases more than once  in 2018 
•	The average number of customer orders does not change much over the year. The average customer is only 1 order.

2. _**Annual Product Category Quality Analysis**_

a. Calculate total revenue per year

```sql
create table total_revenue_per_year as
select 
 date_part('year', o.order_purchase_timestamp) as year,
 sum(revenue_per_order) as revenue
from (
 select 
  order_id, 
  sum(price+freight_value) as revenue_per_order
 from order_items_dataset
 group by 1
) subq
join order_dataset o on subq.order_id = o.order_id
where o.order_status = 'delivered'
group by 1
order by 1


```
Output: (link)
 

b. Calculate total canceled orders per year

```sql
CREATE TABLE total_cancel_order_per_year AS
SELECT
 date_part('year',order_purchase_timestamp) as year,
 COUNT(o.order_id) AS total_cancel
FROM order_dataset as o
WHERE order_status = 'canceled'
GROUP BY 1
ORDER BY 1

```
Output : (link)
 

c. Calculate the highest total revenues per product category per year

```sql
create table top_product_category_by_revenue_per_year as 
select 
 year, 
 product_category_name, 
 revenue 
from (
 select 
  date_part('year', o.order_purchase_timestamp) as year,
  p.product_category_name,
  sum(oi.price + oi.freight_value) as revenue,
  rank() over(
   partition by date_part('year', o.order_purchase_timestamp) 
  order by 
 sum(oi.price + oi.freight_value) desc) as rk
 from order_items_dataset oi
 join order_dataset o on o.order_id = oi.order_id
 join product_dataset p on p.product_id = oi.product_id
 where o.order_status = 'delivered'
 group by 1,2
) sq
where rk = 1;


```
Output (link)

d. Calculate highest canceled order per product category per year

```sql
create table top_product_category_by_cancel_per_year as 
select 
 year, 
 product_category_name, 
 total_cancel 
from (
 select 
  date_part('year', o.order_purchase_timestamp) as year,
  p.product_category_name,
  count(o.order_id) as total_cancel,
  rank() over(
   partition by date_part('year', o.order_purchase_timestamp) 
  order by 
 count(o.order_id) desc) as rk
 from order_items_dataset oi
 join order_dataset o on o.order_id = oi.order_id
 join product_dataset p on p.product_id = oi.product_id
 where o.order_status = 'canceled'
 group by 1,2
) sq
where rk = 1;

```

Output (link)

e. Create a new table including all analyses above

```sql
select 
        tpy.year,
  tpy.product_category_name AS top_product_category_by_revenue,
  tpy.revenue AS category_revenue,
  tr.revenue AS year_total_revenue,
  tcy.product_category_name AS most_canceled_product_category,
  tcy.total_cancel AS category_num_canceled,
  tco.total_cancel AS year_total_num_canceled
from top_product_category_by_revenue_per_year tpy
join total_revenue_per_year tr on tpy.year = tr.year
join top_product_category_by_cancel_per_year tcy on tpy.year = tcy.year
join total_cancel_order_per_year tco on tpy.year = tco.year


```

Output (link)


### 2.f. Visualize result using Data Studio 
- Link 1: Best-sell Categor product: Furniture_decor, Bed_bath_table, Heath_beauty with its revenue

- Link 2: Most Cancelled Product ad Best Selling Product

From this visualization, we can see:

•	Most canceled product category is always changing over the year. However, in 2018, the interesting thing is both the most canceled product category and best-selling product categories were health & beauty.
-> The reason could be health & beauty category was dominating overall transactions made in 2018.

3. _**Annual Payment Type Usage Analysis**_

a. Calculate the number of usage per payment method, sorted by most frequently use

```sql
select 
 op.payment_type,
 count(op.order_id) as num_used
from order_payments_dataset op 
join order_dataset o on o.order_id = op.order_id
group by 1
order by 2 desc

```
Output: (link)
 

b. Calculate annual payment type usage

```sql
with 
tmp as (
select 
 date_part('year', o.order_purchase_timestamp) as year,
 op.payment_type,
 count(1) as num_used
from order_payments_dataset op 
join order_dataset o on o.order_id = op.order_id
group by 1, 2
)select *,
 case when year_2017 = 0 then NULL
  else round((year_2018 - year_2017) / year_2017, 2)
 end as pct_change_2017_2018
from (
select 
  payment_type,
  sum(case when year = '2016' then num_used else 0 end) as year_2016,
  sum(case when year = '2017' then num_used else 0 end) as year_2017,
  sum(case when year = '2018' then num_used else 0 end) as year_2018
from tmp 
group by 1) subq
order by 2;


```
Output : (link)
 

### 3.c. Visualize results using Google Data Studio 
- Link 1: Payment type usage per year

From this visualization, we can see:

•	Overall, the most preferred payment method is the credit card.
 -> Further analysis could be done on Customer Behavior in using credit cards including the selected tenor that product categories are usually purchased with a credit card, etc.
•	In term of debit card, we can see that there was a significant increase from 2017 to 2018. On the other hand, voucher usage was decreased from 2017 to 2018.
=> The reason could be collaborations among certain debit cards as well as a reduction in promotional methods using vouchers

-> **Further analysis** can be done by confirming with other departments( Marketing, Business Development) about the Credit Cards problems above.


## Conclusion

1.	In terms of _Customer Growth_ or _Annual Customer Activity Growth_, It is shown that the growth of new subscribers in 2017 was indicated by **increasing the number of new customers and the number of Monthly Active Users (MAU)**.

2.	However, there was a slight **decrease in the number of customers** who made **purchases more than _**once**_ in 2018**.

3.	The average number of orders by customers is _**only once**_.

4.	Meanwhile, in terms of _Product Quality_ or _Annual Product Category Quality Analysis_, there is an **increase in total revenue every year**.

5.	The _product categories_ that get the _most canceled_ and _the Top Sell Orders_ changes each year. Surprisingly, in 2018, both most canceled product category and the Top Sell Product Category were _**health & beauty**_.

6.	In addition, the most popular type of payment from year to year is _**Credit Card_**.

## Further Information
Explain more details about the project:

- Blog ( link)
- Video (link)



