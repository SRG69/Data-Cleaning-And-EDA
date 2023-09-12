```SQL
CREATE SCHEMA b_e_com;
SET GLOBAL Local_infile = 1;
SET SQL_SAFE_UPDATES = 0;
```

# _Importing data_
```SQL
LOAD DATA LOCAL INFILE "file_path"
INTO TABLE table_name
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
;
```
# _Data Cleaning_
```SQL
SELECT * FROM order_items;
```
### Renaming table name 

```SQL
RENAME TABLE product_category_name_translation TO p_translation;
```

### Changing column name
```SQL
ALTER TABLE p_translation
RENAME COLUMN ï»¿product_category_name TO product_category_name;
```

-------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------------------------

# _Changing Date Datatype and Formate_

## **order_items table**

```SQL
UPDATE order_items
SET shipping_limit_date = str_to_date(shipping_limit_date,'%d-%m-%Y %H:%i:%s');

SELECT * FROM order_items;

ALTER TABLE order_items
MODIFY COLUMN shipping_limit_date DATETIME;

DESC order_items;
```

**Before** 

![desc_order_items](https://github.com/SRG69/Data-Cleaning-And-EDA/assets/131379055/c2e46b1f-5318-4d37-b74f-b3c2b0511aed)

**After**

![Desc_order_items_fixed](https://github.com/SRG69/Data-Cleaning-And-EDA/assets/131379055/fd13a462-b169-4bd0-8998-91b343ffa7bc)

------------------------------------------------------------------------------------------------------------------------------------

## **Orders Table**

**Checking for NULL and Empty Strings**
```SQL
SELECT * 
FROM 
	orders
WHERE  
	order_delivered_carrier_date = '' 
OR	
	order_delivered_carrier_date IS NULL;
``` 
 ![Blanks_orders](https://github.com/SRG69/Data-Cleaning-And-EDA/assets/131379055/06121a2d-f02d-44d8-938b-ed455aafad7e)

## **Setting NULL to all empty strings and blank sells**
```SQL
UPDATE orders
SET order_approved_at = NULL
WHERE order_approved_at  IS NULL OR order_approved_at = '';

UPDATE orders
SET order_delivered_carrier_date = NULL
WHERE order_delivered_carrier_date  IS NULL OR order_delivered_carrier_date = '';

UPDATE orders
SET order_delivered_customer_date = NULL
WHERE order_delivered_customer_date  IS NULL OR order_delivered_customer_date = '';

UPDATE orders
SET order_estimated_delivery_date = NULL
WHERE order_estimated_delivery_date  IS NULL OR order_estimated_delivery_date = '';
```
**Fixed**

![Blank_fixed_orders](https://github.com/SRG69/Data-Cleaning-And-EDA/assets/131379055/6bd7bf28-c51b-4c47-9a68-4e7f5d38d2a0)

# _Changing Date Format And Modifying Datatype_

### order_purchase_timestamp **Column**
```SQL
-- Changing the Format

UPDATE orders
SET order_purchase_timestamp = str_to_date(order_purchase_timestamp,"%Y-%m-%d %H:%i:%s");

-- Converting Text to Datetime Datatype

ALTER TABLE orders
MODIFY COLUMN order_purchase_timestamp DATETIME;
```

### order_approved_at **Column**

```SQL
-- Changing the Format
UPDATE orders
SET order_approved_at = str_to_date(order_approved_at, '%Y-%m-%d %H:%i:%s');

-- Converting Text to Datetime Datatype
ALTER TABLE orders
MODIFY COLUMN order_approved_at DATETIME;
```
### order_delivered_carrier_date **Column**
```SQL
-- Changing the Format

UPDATE orders
SET order_delivered_carrier_date = str_to_date(order_delivered_carrier_date, '%Y-%m-%d %H:%i:%s');

-- Converting Text to Datetime Datatype

ALTER TABLE orders
MODIFY COLUMN order_delivered_carrier_date DATETIME;
```

### order_delivered_customer_date **Column**
```sql
-- Changing the Format

UPDATE orders
SET order_delivered_customer_date = str_to_date(order_delivered_customer_date, '%Y-%m-%d %H:%i:%s');

-- Converting Text to Datetime Datatype

ALTER TABLE orders
MODIFY COLUMN order_delivered_customer_date DATETIME;
```
### order_estimated_delivery_date **Column**
```SQL
-- Changing the Format

UPDATE orders
SET order_estimated_delivery_date = str_to_date(order_estimated_delivery_date, '%Y-%m-%d %H:%i:%s');

-- Converting Text to Datetime Datatype

ALTER TABLE orders
MODIFY COLUMN order_estimated_delivery_date DATETIME;
```
```SQL
DESC orders;
```

# **Before**
![DESC_orders](https://github.com/SRG69/Data-Cleaning-And-EDA/assets/131379055/6bb4a90b-f9df-4fb4-aa7e-e49a7df0eeca)

# **After**
![DESC_orders Fixed](https://github.com/SRG69/Data-Cleaning-And-EDA/assets/131379055/6f5eda43-d485-439f-903f-999fd98ad0bc)


---------------------------------------------------------------------------------------------------------------------------------

#  _Join the datasets, rename some columns, and leave unnecessary columns_

```SQL
CREATE TABLE temp_t
SELECT
-- Customer Details
	  c.customer_id, c.customer_unique_id, c.customer_zip_code_prefix as customer_zip_code, c.customer_city, c.customer_state,

-- Customer Purchases Details   
    o.order_id, o.order_status, o.order_purchase_timestamp as order_date, o.order_delivered_customer_date as delivered_date, o.order_estimated_delivery_date as estimated_delivery_date,

-- org internal Information    
    i.order_item_id, i.product_id, i.seller_id, i.shipping_limit_date, i.price, i.freight_value,

-- Customer Payments    
    pay.payment_sequential, pay.payment_type, pay.payment_installments, pay.payment_value,

-- Customer Reviews    
    r.review_id, r.review_score,

-- Product Name  
    p.product_category_name

FROM customers as c
JOIN orders as o on c.customer_id = o.customer_id
JOIN order_items as i on o.order_id = i.order_id
JOIN order_payments as pay on o.order_id = pay.order_id
JOIN order_reviews as r on r.order_id = o.order_id
JOIN products as p on i.product_id = p.product_id
JOIN p_translation as pt on p.product_category_name = pt.product_category_name;
```
### Join the sellers table (didn't joined with temp_t because of unknown error)
```sql
CREATE TABLE olist_data as
SELECT
	t.*,
    s.seller_zip_code_prefix as seller_zip_code, s.seller_city, s.seller_state
FROM temp_t as t
JOIN sellers as s USING(seller_id);
```
### Doing Calculations For better Understanding and getting more Insights

```SQL
CREATE TABLE agg_data -- agg here is aggregated, since the data was aggregated
SELECT
	customer_id, customer_unique_id, customer_zip_code, customer_city, customer_state, order_id, order_status, order_date, 
    
    delivered_date,
     estimated_delivery_date,

    num_days_to_deliver, 
    delivered_on_time_or_not,

    quantity,
    product_id,
    product_name,
    
    seller_id, shipping_limit_date, price, freight_value, payment_type,

-- Finding The actual Payments made by customer using vouchers and not using vouchers
    ROUND(case when payment_type = 'voucher' then payment_value * quantity ELSE (price * quantity) + (freight_value * quantity) END, 2) as payment_value,
    
    review_id, review_score, seller_city, seller_state, seller_zip_code
FROM(
SELECT
	customer_id, customer_unique_id, customer_zip_code, customer_city, customer_state, order_id, order_status, order_date, 
    
    delivered_date, estimated_delivery_date,

-- Calculating date Difference

    DATEDIFF(delivered_date, order_date) as num_days_to_deliver, 
    DATEDIFF(estimated_delivery_date, delivered_date) as delivered_on_time_or_not,

-- Counting Total Quantity

	COUNT(order_item_id) as quantity,

    product_id, 
    
    seller_id, shipping_limit_date, price, freight_value, payment_type, payment_value, review_id, review_score, product_name,  
    seller_city, seller_state, seller_zip_code 

FROM olist_data
GROUP BY
	  customer_id, customer_unique_id, customer_zip_code, customer_city, customer_state, order_id, order_status, order_date, 
    
    delivered_date, estimated_delivery_date,

    DATEDIFF(delivered_date, order_date), 
    DATEDIFF(estimated_delivery_date, delivered_date),

    product_id, 
    
    seller_id, shipping_limit_date, price, freight_value, payment_type, payment_value, review_id, review_score, product_name,  
    seller_city, seller_state, seller_zip_code
    
    ) as sbqry;
```

### View the dataset
```SQL
DESC temp_t;
```
![Screenshot 2023-09-12 131532](https://github.com/SRG69/Data-Cleaning-And-EDA/assets/131379055/323ea066-dfad-4fed-a49b-921235326660)

```SQL
DESC agg_data;
```
![Screenshot 2023-09-12 130333](https://github.com/SRG69/Data-Cleaning-And-EDA/assets/131379055/222352f4-5318-4087-a3c0-4f6c7cfde8c5)
