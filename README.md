# Business-Insights-MySQL
## Problem statement:
Atliq Hardware is struggling to handle its increasing sales data due to frequent Excel crashes. To overcome this issue, there's a need for a user-friendly MySQL tool that can efficiently manage and analyze the expanding data for better business insights and smooth operations.
## Understanding the Business Model:
So atliq hardware produce computer peripherals so they sell the hardware  to different customers from there it will go to the consumers,here cusotmer is the store who sells the products tot he consumer who uses it .
they have a ware house where they build all these parts and they sell it different countries from there and from there the material or the hard ware will go the individual stores or customers.
here we have 2 different type  of platforms one is brick and mortar which is a physical store and the other is e commerce which is online platform
when you sell the hardware to a brick and mortar store and they sell it to the consumer then it is called as retailer, in addition to this atliq has also have their own store .now this channel is called direct , and also we have another channel which is a distriutor who sells the hardware to a local store .
### The P & L Statement goes like this💰
- **Gross price:** this is the base price
- **pre-invoice sales:**  yearly discount agreements are made at the beginning  of each financial year.
- **Net invoice sales:** after deducting the pre-invoice from gross price .
- **Post-invoice deductions:** these are some additional discounts given by the atliq hardware to the store.
- **Net Sales(NS):** basically this is the revenue of the hardware.
- **Cost of Goods Sold(COGS):** these are the cost bared by the hardware like "manufacturing cost","freight cost","other costs" etc..
- **Gross Margin(GM):** Profit
- **Gross Margin % of Net Sales(GM/NS):** profit percentage.
## Approach:
here we will be exploring the database given to us
### Database Design:
- so in the database we have around 9 tables in that 2 are dimention table and the rest are fact table;
### Tool selection:
- Choosing the MySQL tool that best suits Atliq Hardware's needs, Considering factors such as scalability, ease of use, and compatibility with the existing infrastructure.

## Analytics using MySQL📊
-**Finance Analytics:** 
- Here we will be using the user-defined functions:
- **creating croma india product wise sales report for fiscal year 2021**
#### Creating a new function:
- **getting the fiscal year with the help of user defined funtion:**
```sql
CREATE DEFINER=`root`@`localhost` FUNCTION `fiscal_year`(calander_date DATE ) RETURNS int
    DETERMINISTIC
BEGIN
	declare fiscal_year INT ;
    SET fiscal_year = year(date_add(calander_date,interval 4 month));
RETURN fiscal_year ;
END
```
-**creating fiscal quarter with the help of user defined function:**
```sql
CREATE DEFINER=`root`@`localhost` FUNCTION `get_fiscal_quarter`(calander_date DATE) RETURNS char(2) CHARSET utf8mb4
    DETERMINISTIC
BEGIN
	declare fiscal_quarter int ;
    set fiscal_quarter = month(calander_date);
RETURN case when fiscal_quarter in (9,10,11) then "Q1"
when  fiscal_quarter in (12,1,2) then "Q2"
 when fiscal_quarter in (3,4,5) then "Q3"
  when fiscal_quarter in (6,7,8) then "Q4"
end;
END
```

### Stored Procedure :

- we create stored procedure cause to create function for each and every customer is so time taking so we will take the help of stored procedure.
  
-**getting the monthly sales report by customer:**
```sql
CREATE DEFINER=`root`@`localhost` PROCEDURE `get_monthly_gross_sales_for_customer`(in_customer_codes TEXT)
BEGIN
SELECT g.fiscal_year,
sum(s.sold_quantity*g.gross_price) as gross_total_price
 FROM fact_sales_monthly s
join fact_gross_price g
on g.product_code = s.product_code and g.fiscal_year = fiscal_year(s.date) 
where find_in_set(s.customer_code,in_customer_codes)>0
group by g.fiscal_year
order by g.fiscal_year;
END
```
-**Getting the market badge:**

-if total sold quantity > 5 million that market is considered as Gold else Silver.

```sql

CREATE DEFINER=`root`@`localhost` PROCEDURE `get_market_badge`
(in in_market varchar(45),
 in in_fiscal_year year,
 out out_badge varchar(45))
BEGIN
declare qty int default 0;
# set default market to be india
if in_market = "" then 
	set in_market = "india";
end if ;
```

- **retriving total qty for the particular market and fiscal year**
```sql 
SELECT sum(s.sold_quantity) into qty  FROM fact_sales_monthly s
join dim_customer c 
on c.customer_code = s.customer_code
where c.market = in_market  and fiscal_year(s.date) = in_fiscal_year
group by c.market;  
-- Determining market badge
if qty>5000000 then 
	set out_badge = "gold";
else
	set out_badge = "silver";
end if;

END ;
```
#### Generating the reports for top markets top products top customers for a given financial year by net sales:
- **Creating database views for error-free and simplifying the queries:**
- **creating view for pre_invoice_discount_percentage:**
```sql
CREATE  VIEW `sales_preinv_discount` AS
    SELECT 
        `g`.`fiscal_year` AS `fiscal_year`,
        `c`.`market` AS `market`,
        `p`.`product` AS `product`,
        `p`.`variant` AS `variant`,
        (`s`.`sold_quantity` * `g`.`gross_price`) AS `gross_total_price`,
        `pre`.`pre_invoice_discount_pct` AS `pre_invoice_discount_pct`
    FROM
        ((((`fact_sales_monthly` `s`
        JOIN `dim_customer` `c` ON ((`s`.`customer_code` = `c`.`customer_code`)))
        JOIN `dim_product` `p` ON ((`p`.`product_code` = `s`.`product_code`)))
        JOIN `fact_gross_price` `g` ON (((`g`.`product_code` = `s`.`product_code`)
            AND (`g`.`fiscal_year` = FISCAL_YEAR(`s`.`date`)))))
        JOIN `fact_pre_invoice_deductions` `pre` ON (((`pre`.`customer_code` = `s`.`customer_code`)
            AND (`pre`.`fiscal_year` = FISCAL_YEAR(`s`.`date`)))))
```

- **creating view for post invoice discount percentage:**
```sql
CREATE VIEW `post_invoice_discount_pct` AS
select 
	s.date,s.fiscal_year,
    s.customer_code,s.market,
    s.product_code,s.product,s.variant,
    s.sold_quantity,s.gross_price_total,
	s.pre_invoice_discount_pct,
    (s.gross_total_price-s.pre_invoice_discount_pct*s.gross_total_price) as net_invoice_sales,
    (po.discounts_pct+po.other_deductions_pct) as post_invoice_discount_pct
    from  sales_preinv_discount s 
    join fact_post_invoice_deductions po
    on po.customer_code = s.customer_code
    and po.product_code = s.product_code
    and po.date = s.date
```

- **creating view for net sales:**
```sql
Create view net_sales as 
select 
*,
(1-post_invoice_discount_pct)*net_invoice_sales as net_sales
from sales_post_invoice_discount_pct;
```
- **Creating a Stored Procedure for Top N markets:**
```sql
DELIMITER $$
USE `gdb0041`$$
CREATE PROCEDURE `get_topN_markets_by_net_sales` (in_fiscal_year year,
in_top_n tinyint)
BEGIN
select 
	market,
    round(sum(net_sales)/1000000,2) as net_sales_mln
from net_sales 
where fiscal_year = in_fiscal_year
group by market
order by net_sales_mln desc
limit in_top_n ;
END$$

DELIMITER ;


```
- **Creating a Stored Procedure for Top N customers:**
```sql

DELIMITER $$
USE `gdb0041`$$
CREATE PROCEDURE `Get_topN_customers_by_net_sales` (
in_market varchar(45),
in_fiscal_year year,
in_topN tinyint)
BEGIN
select 
	c.customer,
    round(sum(n.net_sales)/1000000,2) as net_sales_mln
from net_sales n
join dim_customer c
on n.customer_code = c.customer_code
where fiscal_year = in_fiscal_year and n.market = in_market
group by c.customer
order by net_sales_mln desc
limit in_topN ;
END$$

DELIMITER ;
```
- **Creating a Stored Procedure for Top N products:**
```sql

DELIMITER $$
USE `gdb0041`$$
CREATE PROCEDURE `Get_topN_customers_by_net_sales` (
in_market varchar(45),
in_fiscal_year year,
in_topN tinyint)
BEGIN
select 
	p.product,
    round(sum(n.net_sales)/1000000,2) as net_sales_mln
from net_sales n
join dim_product p
on n.product_code = c.product_code
where fiscal_year = in_fiscal_year and n.market = in_market
group by p.product
order by net_sales_mln desc
limit in_topN ;
END$$
```
- **Top 10 Markets by % net sales:**
```sql

with cte1 as 
(

select 
	c.customer,
    round(sum(n.net_sales)/1000000,2) as net_sales_mln
from net_sales n
join dim_customer c 
where fiscal_year = 2021
group by c.customer)

select *,
net_sales_mln*100/sum(net_sales_mln) 
over () as pct  from cte1
order by net_sales_mln desc
limit 10


```
-**region wise net sales %:**
```sql

with cte1 as 
(

select 
	c.customer,
	c.region,
    round(sum(n.net_sales)/1000000,2) as net_sales_mln
from net_sales n
join dim_customer c 
where fiscal_year = 2021
group by c.customer,c.region)

select 
net_sales_mln*100/sum(net_sales_mln) 
over (partition by region) as pct  from cte1
order by net_sales_mln desc
limit 5 


```
#### Net Sales contribution:
**here we are using the window function to retrive the percentage share of the net sales:**
```sql
with cte1 as 
(
select 
	customer,
    round(sum(net_sales)/1000000,2) as net_sales_mln
from net_sales s
join customer c
	on c.customer_id = s.customer_id
where  s.fiscal_year = 2021
group by customer

)

select *,
	net_slaes_mln*100/sum(net_sales_mln) over() as pct
    from cte1
order by net_sales_mln desc
```

#### Getting top N products in each division by their quantity sold:
**With the help of the window function(dense rank):**
**creating stored procedure for  getting the topN products:**
```sql

DELIMITER $$
USE `gdb0041`$$
CREATE PROCEDURE 
`Get_top_N_products_in_each_division_by_their_quantity_sold` (
in_fiscal_year year,
in_drank int )
BEGIN
with  cte1 as 
(select 
	p.division,
    p.product,
    sum(sold_quantity) as total_qty
from fact_sales_monthly s
join dim_product p 
	on p.product_code = s.product_code
where fiscal_year = in_fiscal_year
group by p.product),

cte2 as (

select *,
	dense_rank() over(partition by division order by total_qty desc) as drank
 from cte1)
 
 select * from cte2 where drank<= in_drank;

END$$

DELIMITER ;


```
### Supply Chain Analytics:
**Here we will be forecasting the accuracy for all the customers for  a given fiscal year:**
- here we created a temporary table to find the error percentage of the forecast data:
```sql
create temporary table forecast_err_table as 

with forcast_err_table as
(
select  s.customer_code,
sum(s.sold_quantity) as total_sold_qty,
sum(s.forcast_quantity) as total_forecast_qty,
sum((forcast_quantity-sold_quantity)) as net_error,
sum((forcast_quantity-sold_quantity))*100/sum(forecast_quantity) as net_err_pct,
sum(abs(forcast_quantity-sold_quantity )) as abs_error,
sum(abs(forcast_quantity-sold_quantity ))*100/sum(forecast_quantity) as abs_err_pct
from fact_act_est
where fiscal_year = 2021
group by customer_code

)

select e.*,
c.customer,c.market,
if(abs_err_pct>100,0,100-abs_err_pct) as forecast_accuracy
from forcast_err_table e
join dim_customer c 
using (customer_code)
order by forecast_accuracy asc
```
### INSIGHTS:

### Forecast Accuracy Insights 📈

1. **Customer-Specific Accuracy:**
   - Understanding the accuracy of forecasts tailored to each customer is crucial. The analysis reveals that certain customers showcase remarkably high forecast accuracy, indicating a robust demand forecasting model tailored to their specific needs.

2. **Market Influence on Accuracy:**
   - Examining the accuracy relative to markets provides strategic insights. Some markets demonstrate consistently high accuracy levels, suggesting a potential correlation between market dynamics and forecasting precision. This correlation opens avenues for targeted improvements in forecasting methodologies based on market characteristics.

3. **Improvement Opportunities:**
   - The table showcases customers with lower forecast accuracy, pinpointing areas for improvement. Identifying and addressing factors contributing to lower accuracy, such as market volatility or product-specific challenges, can enhance overall supply chain efficiency and customer satisfaction.

### Actionable Steps 🚀

1. **Customer-Centric Adjustments:**
   - Tailor forecasting models to address the unique patterns and demands of high-accuracy customers. This can involve personalized forecasting algorithms, collaborative forecasting efforts, or regular communication channels to gather real-time insights.

2. **Market-Driven Strategies:**
   - Explore strategies to enhance accuracy in markets where performance lags. This could involve market-specific training for forecasting models, considering regional economic factors, or collaborating with local stakeholders for more precise predictions.

3. **Continuous Improvement Initiatives:**
   - Focus on customers with lower accuracy percentages to initiate continuous improvement initiatives. Regularly reassess forecasting models, incorporate feedback loops, and stay agile in adapting to evolving market conditions.

The forecast accuracy analysis not only provides a snapshot of past performance but also serves as a foundation for proactive strategies to elevate supply chain efficiency and responsiveness.
