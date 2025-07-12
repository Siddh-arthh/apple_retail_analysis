# Apple_retail_analysis
<img width="3840" height="2160" alt="image" src="https://github.com/user-attachments/assets/0ab7cf63-49f2-46c0-98bd-7c34f97aaebc" />


### Find the number of stores in each country.
```sql
select	
	country,
	COUNT (distinct	store_id) as Total_Stores
from stores
group by Country
order by Total_Stores desc;
```
### Calculate the total no. of units sold by each store.
```sql
select 
	store_id,
	sum(quantity) as Total_Units
from sales
group by store_id
order by Total_Units desc;
```
### Identify how many sales occured in December 2023.
``` sql
select 
		sum(quantity) as Total_Units
from sales
where FORMAT(sale_date,'MM-yyyy')= '12-2023';
```

### Determine how many stores have never had a warranty claim filled.
```sql
WITH Total AS (
    SELECT COUNT(*) AS total_claims
    FROM warranty
),
Rejected AS (
    SELECT COUNT(*) AS rejected_claims
    FROM warranty
    WHERE repair_status = 'rejected'
)
SELECT 
    r.rejected_claims,
    t.total_claims,
    ROUND((CAST(r.rejected_claims AS FLOAT) / t.total_claims) * 100, 2) AS percentage_void
FROM 
    Total t
CROSS JOIN 
    Rejected r;
```
### Identify which store had the highest total units sold in the last year.
```sql
select 
	FORMAT(sale_date,'yyyy') as year,
	SUM(quantity) as Total_units_sold
from sales
where FORMAT(sale_date,'yyyy') =2024
group by FORMAT(sale_date,'yyyy');
```

###  Count the number of unique products sold in the last year.
```sql
select 
	distinct p.product_name,
	sum(s.quantity) as Total_quantity
from sales as s
left join 
	products as p on p.Product_ID=s.product_id
where FORMAT(s.sale_date,'yyyy')=2024
group by 
	p.Product_Name
order by 
	Total_quantity desc;
```
###  Find the average price of products in each category.
```sql
select 
	c.category_name,
	avg(p.price) as avg_price
from	
	category as c
left join 
	products as p on p.Category_ID=c.category_id
group by 
	c.category_name
order by 
	avg_price desc;
```

###  How many warranty claims were filed in 2020?
```sql
select COUNT (*) from warranty
where FORMAT(claim_date,'yyyy')=2024;
```

### For each store, identify the best-selling day based on highest quantity sold.
```sql
WITH c AS (
    SELECT 
        s.store_id,
        s.store_name,
        FORMAT(sa.sale_date, 'yyyy-MM-dd') AS sale_day,
        SUM(sa.quantity) AS total_quantity
    FROM 
        stores s
    LEFT JOIN 
        sales sa ON s.store_id = sa.store_id
    GROUP BY 
        s.store_id, s.store_name, FORMAT(sa.sale_date, 'yyyy-MM-dd')
),
ranked_sales AS (
    SELECT *,
        RANK() OVER (PARTITION BY store_id ORDER BY total_quantity DESC) AS rankk
    FROM c
)
SELECT 
    store_name,
    sale_day,
    total_quantity
FROM 
    ranked_sales
WHERE rankk = 1;
```



###  Identify the least selling product in each country for each year based on total units sold.
```sql
WITH product_sales AS (
    SELECT
        st.country,
        p.product_name,
        FORMAT(s.sale_date,'yyyy') AS sale_year,
        SUM(s.quantity) AS units_sold
    FROM products p
    JOIN sales s ON s.product_id = p.Product_ID
    JOIN stores st ON s.store_id = st.Store_ID
    GROUP BY p.product_name, st.country, FORMAT(s.sale_date,'yyyy')
),
ranked_sales AS (
    SELECT
        product_name,
        country,
        sale_year,
        units_sold,
        ROW_NUMBER() OVER (PARTITION BY country, sale_year ORDER BY units_sold ASC) AS rankk
    FROM product_sales
)
SELECT 
    product_name,
    country,
    sale_year,
    units_sold
FROM ranked_sales
WHERE rankk = 1;

```
###  Calculate how many warranty claims were filed within 180 days of a product sale.

```sql
WITH claim_dates AS (
    SELECT 
        MIN(claim_date) AS min_date,
        MAX(claim_date) AS max_date,
        COUNT(*) AS claims_filled
    FROM warranty
)
SELECT 
    DATEDIFF(day, min_date, max_date) AS N_days,
    claims_filled
FROM claim_dates
WHERE DATEDIFF(day, min_date, max_date) < 180;
```

###  List the months in the last three years where sales exceeded 5,000 units in the USA.
```sql
select 
	st.country,
	DATENAME(month,s.sale_date) as month,
	FORMAT(s.sale_date,'yyyy') as year,
	sum(s.quantity) as total_quantity
from sales as s
join stores as st on s.store_id=st.Store_ID
where st.Country='United States'
group by st.country,
	DATENAME(month,s.sale_date),
	FORMAT(s.sale_date,'yyyy')
having
	sum(s.quantity)>5000
order by 
	total_quantity desc;
```


### Identify the product category with the most warranty claims filed BETWEEN JULY 2024 TO DEC 2024.
```sql
select
	c.category_name,
	count(w.sale_id) as total_claims
from warranty as w
join sales as s on s.sale_id=w.sale_id
join products as p on p.Product_ID=s.product_id
join category as c on p.Category_ID=c.category_id
where FORMAT(claim_date,'MM-yyyy') BETWEEN '07-2024' AND '12-2024'
group by category_name
ORDER BY total_claims DESC;
```



### Determine the percentage chance of receiving warranty claims after each purchase for each country.

```sql
with a as(
select 
	st.country,
	sum(s.quantity) as total_units_sold,
	count(w.claim_id) as total_claims
from sales as s
join stores as st 
on s.store_id=st.store_id
left join warranty as w 
on w.sale_id=s.sale_id
group by st.Country
)
SELECT 
    country,
    total_units_sold,
    total_claims,
    FORMAT((total_claims * 1.0 / total_units_sold), 'P2') AS [%_claim]
FROM a;

-- Analyze the year-by-year growth ratio for each store.
with yearly_sales as(
select 
	st.store_id,
	st.store_name,
	FORMAT(s.sale_date,'yyyy') as year,
	sum(p.price*s.quantity) as current_year_sales
from stores as st
join sales as s
on s.store_id=st.store_id
join products as p
on p.product_id=s.product_id
group by 
	st.store_id,
	st.store_name,
	FORMAT(s.sale_date,'yyyy') 
),
yoy as(
select 
	Store_ID,
	store_name,
	year,
	LAG(current_year_sales) over(partition by store_id order by year) as previous_year_sales,
	current_year_sales
from yearly_sales
)
	select *,
		case 
			when previous_year_sales is null or previous_year_sales=0 then 'NA'
			else FORMAT((current_year_sales-previous_year_sales)*1.0/previous_year_sales,'P2')
			end as growth_rate
	from yoy;
```
### Calculate the correlation between product price and warranty claims for products sold in the last five years, segmented by price range.
```sql
SELECT 
	case 
		when p.price<500 then 'less expensive product'
		when p.price between 500 and 1000 then 'Mid Range  Product'
		else 'Expensive Product'
		end as price_segment,
	count(w.claim_id) as Claims_made
from 
	warranty as w
left join sales as s
on s.sale_id=w.sale_id
join products as p 
on p.Product_ID=s.product_id
group by 
	case 
		when p.price<500 then 'less expensive product'
		when p.price between 500 and 1000 then 'Mid Range  Product'
		else 'Expensive Product' end
order by Claims_made desc;
```
		


### Write a query to calculate the monthly running total of sales for each store over the past four years and compare trends during this period.
```sql
with gi as(
select
	st.store_id,
	st.store_name,
	FORMAT(s.sale_date,'yyyy-MM') as year_month,
	sum(p.price*s.quantity) as Total_sales
from stores as st
join sales as s
on st.store_id=s.store_id
join products as p
on p.Product_ID=s.product_id
group by
	st.store_id,
	st.store_name,
	FORMAT(s.sale_date,'yyyy-MM')
)
select *,
	SUM(total_sales) over (partition by store_id order by year_month rows between unbounded preceding and current row ) as Running_total
from gi;
````
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

###                                Analyze product sales trends over time, segmented into key periods: from launch to 6 months, 6–12 months, 12–18 months, and beyond 18 months.
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```sql
select
	p.product_name,
	p.launch_date,
	sum(s.quantity) as Total_Units_sold,
	case
		when s.sale_date between p.launch_date and DATEADD(MONTH,6,p.Launch_Date) then '0-6 months'
		when s.sale_date between DATEADD(MONTH,6,p.Launch_Date) and DATEADD(MONTH,12,p.Launch_Date) then '6-12 months'
		when s.sale_date between DATEADD(MONTH,12,p.Launch_Date) and DATEADD(MONTH,18,p.Launch_Date) then '12-18 months'
		else '18+ months'
	end as prod_lifecycle
from products as p
join sales as s
on p.Product_ID=s.product_id
group by p.product_name, p.launch_date, 	
	case
		when s.sale_date between p.launch_date and DATEADD(MONTH,6,p.Launch_Date) then '0-6 months'
		when s.sale_date between DATEADD(MONTH,6,p.Launch_Date) and DATEADD(MONTH,12,p.Launch_Date) then '6-12 months'
		when s.sale_date between DATEADD(MONTH,12,p.Launch_Date) and DATEADD(MONTH,18,p.Launch_Date) then '12-18 months'
		else '18+ months'
	end
order by Total_Units_sold desc;
```















