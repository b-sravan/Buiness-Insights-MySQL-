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






  
