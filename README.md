# PL/SQL Assignment One - Sunrise Supermarket

**Name:** Mihigo Wycliff
**Student ID:** 29652
**Database used:** Oracle 21c (via Oracle SQL Developer)

## Summary
This project models a supermarket database with customers, products, orders, 
and order items. It includes JOIN, CTE, and window-function queries to analyze 
customer behavior and sales trends.

## How to run
1. Open Oracle SQL Developer, connect to an Oracle 21c database (PDB service).
2. Run the CREATE TABLE statements from `schema.sql` to build the tables.
3. Run the INSERT statements from `data.sql` to populate sample data.
4. Run each query below individually in a SQL Worksheet to see results.

## Business Scenario
Sunrise Supermarket sells products to customers, who place orders containing 
one or more items. Management wants to understand who their customers are, 
what they buy, and how sales are trending over time.

## Schema

```sql
CREATE TABLE customers (
  customer_id NUMBER PRIMARY KEY,
  customer_name VARCHAR2(100),
  email VARCHAR2(100),
  city VARCHAR2(50)
);

CREATE TABLE products (
  product_id NUMBER PRIMARY KEY,
  product_name VARCHAR2(100),
  category VARCHAR2(50),
  price NUMBER(10,2)
);

CREATE TABLE orders (
  order_id NUMBER PRIMARY KEY,
  customer_id NUMBER REFERENCES customers(customer_id),
  order_date DATE
);

CREATE TABLE order_items (
  order_item_id NUMBER PRIMARY KEY,
  order_id NUMBER REFERENCES orders(order_id),
  product_id NUMBER REFERENCES products(product_id),
  quantity NUMBER
);
```

## Queries

### JOIN Query 1: Orders with customer name, city, and order date

```sql
SELECT 
  o.order_id,
  c.customer_name,
  c.city,
  o.order_date
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
ORDER BY o.order_date;
```

**Explanation:** Joins each order to its customer using the shared 
`customer_id` column, showing who placed each order, from where, and when. 
Only orders with a matching customer appear (INNER JOIN).

**Result:** 15 rows returned — one per order.

**Business interpretation:** Management can see full order history tied to 
real customer identities and locations, useful for spotting regional 
demand patterns.
<img width="1597" height="817" alt="query1" src="https://github.com/user-attachments/assets/596ab091-0955-49c4-84ce-408186d109b1" />


---

### JOIN Query 2: Order items with product name, category, price, and quantity

```sql
SELECT 
  oi.order_item_id,
  oi.order_id,
  p.product_name,
  p.category,
  p.price,
  oi.quantity
FROM order_items oi
INNER JOIN products p ON oi.product_id = p.product_id
ORDER BY oi.order_id;
```

**Explanation:** Joins each purchased line item to its product details via 
`product_id`, turning raw product IDs into readable names, categories, 
and prices alongside the quantity bought.

**Result:** 25 rows returned — one per order item.

<img width="1581" height="850" alt="query2" src="https://github.com/user-attachments/assets/0f87e0b0-1e9c-48da-9fee-44c75c7fdf6f" />

### JOIN Query 3: All customers and their orders (including customers with no orders)

```sql
SELECT 
  c.customer_name,
  c.city,
  o.order_id,
  o.order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
ORDER BY c.customer_name;
```

**Explanation:** A LEFT JOIN keeps every row from `customers` regardless of 
whether a matching order exists. Any customer with zero orders would still 
appear, with `order_id`/`order_date` shown as NULL.

**Result:** 15 rows returned. In this dataset, all 5 customers happen to 
have placed at least one order, so no NULLs appear — but the query is 
built to handle that case safely.

**Business interpretation:** This is useful for management to identify 
inactive customers (those with no purchase history) who might need 
re-engagement, e.g. through promotions.

**Business interpretation:** Shows exactly which products move in which 
categories and at what volume, helping management see which categories 
(Grains, Dairy, Household) drive the most purchases.
### CTE Query: Customers with above-average total spend
<img width="1549" height="813" alt="quey3" src="https://github.com/user-attachments/assets/8e088ed2-b3b7-4a50-ad59-cc262f25835e" />


```sql
WITH customer_totals AS (
  SELECT 
    c.customer_id,
    c.customer_name,
    SUM(oi.quantity * p.price) AS total_spend
  FROM customers c
  JOIN orders o ON c.customer_id = o.customer_id
  JOIN order_items oi ON o.order_id = oi.order_id
  JOIN products p ON oi.product_id = p.product_id
  GROUP BY c.customer_id, c.customer_name
)
SELECT *
FROM customer_totals
WHERE total_spend > (SELECT AVG(total_spend) FROM customer_totals)
ORDER BY total_spend DESC;
```

**Explanation:** The CTE (`customer_totals`) first joins all four tables 
to calculate each customer's total spend (quantity × price, summed across 
all their order items). The outer query then filters this temporary 
result set to keep only customers spending above the overall average.

**Result:** 3 customers qualified — Bob Nkurunziza (54.8), Alice Uwimana 
(39.5), Clara Mukamana (38.5).

**Business interpretation:** These are the supermarket's top-value 
customers. Management could target them with loyalty rewards, or study 
their buying patterns to attract similar high-spend customers.
<img width="1561" height="847" alt="CTE query" src="https://github.com/user-attachments/assets/072374ae-488b-45eb-b89b-3323c2a65dd6" />
---

### Window Query 1: Rank customers by total amount spent

```sql
WITH customer_totals AS (
  SELECT 
    c.customer_id,
    c.customer_name,
    SUM(oi.quantity * p.price) AS total_spend
  FROM customers c
  JOIN orders o ON c.customer_id = o.customer_id
  JOIN order_items oi ON o.order_id = oi.order_id
  JOIN products p ON oi.product_id = p.product_id
  GROUP BY c.customer_id, c.customer_name
)
SELECT 
  customer_name,
  total_spend,
  RANK() OVER (ORDER BY total_spend DESC) AS spend_rank
FROM customer_totals;
```

**Explanation:** `RANK() OVER (ORDER BY total_spend DESC)` assigns each 
customer a rank based on total spend, highest first, without collapsing 
rows the way GROUP BY would. Ties would share a rank and skip the next 
number (e.g. 1, 1, 3).

**Result:** Bob Nkurunziza ranks #1 (54.8), down to Eva Ingabire at #5 (16.5).

**Business interpretation:** Gives management an instant leaderboard of 
customer value, useful for prioritizing loyalty outreach or VIP treatment.
<img width="1362" height="815" alt="window function 1" src="https://github.com/user-attachments/assets/e327de02-b34f-4614-bc1b-32c874ae0c4e" />

---
### Window Query 2: Number each customer's orders in the order placed

```sql
SELECT 
  c.customer_name,
  o.order_id,
  o.order_date,
  ROW_NUMBER() OVER (PARTITION BY c.customer_id ORDER BY o.order_date) AS order_sequence
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
ORDER BY c.customer_name, order_sequence;
```

**Explanation:** `PARTITION BY c.customer_id` splits the rows into groups 
per customer, and `ROW_NUMBER()` restarts counting from 1 within each 
group, ordered by `order_date`. This numbers each customer's own order 
history independently of everyone else's.

**Result:** 15 rows. E.g. Alice's 4 orders are numbered 1-4 chronologically, 
then numbering resets to 1 for Bob's orders, and so on.

**Business interpretation:** Useful for identifying a customer's first 
order (order_sequence = 1, good for tracking new customer acquisition) 
vs. repeat/loyal purchase behavior.
<img width="1305" height="815" alt="window fuction 2" src="https://github.com/user-attachments/assets/3bb77a29-513b-4505-a608-728f9adbd038" />

---
### Window Query 3: Running total of revenue over time

```sql
WITH order_revenue AS (
  SELECT 
    o.order_id,
    o.order_date,
    SUM(oi.quantity * p.price) AS order_total
  FROM orders o
  JOIN order_items oi ON o.order_id = oi.order_id
  JOIN products p ON oi.product_id = p.product_id
  GROUP BY o.order_id, o.order_date
)
SELECT 
  order_id,
  order_date,
  order_total,
  SUM(order_total) OVER (ORDER BY order_date) AS running_total
FROM order_revenue
ORDER BY order_date;
```

**Explanation:** The CTE first calculates each order's total revenue. The 
outer query then uses `SUM(order_total) OVER (ORDER BY order_date)` with 
no `PARTITION BY`, so it treats all orders as one continuous timeline — 
each row's running total accumulates everything up to and including 
that date.

**Result:** 15 rows, running total climbs from 26.5 to a final 173.8 
(total revenue across the whole dataset).

**Business interpretation:** Lets management visualize revenue growth 
over time at a glance — useful for spotting trends, seasonal spikes, 
or slow periods.
<img width="703" height="798" alt="windo function 3" src="https://github.com/user-attachments/assets/a8f47497-5db4-49d3-829e-0f7dd284f569" />

---
### Window Query 4: Days between current and previous order per customer

```sql
SELECT 
  customer_name,
  order_id,
  order_date,
  order_date - LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS days_since_previous
FROM (
  SELECT c.customer_id, c.customer_name, o.order_id, o.order_date
  FROM customers c
  JOIN orders o ON c.customer_id = o.customer_id
) sub
ORDER BY customer_name, order_date;
```

**Explanation:** `LAG(order_date)` looks back to the previous row's date 
within each customer's partition. Subtracting two DATE values in Oracle 
returns the difference in days directly. The first order for each 
customer shows NULL since there's no earlier order to compare against.

**Result:** 15 rows. E.g. Alice's orders are 7, 20, and 19 days apart 
respectively; her first order shows NULL as expected.

**Business interpretation:** Reveals purchase frequency/cadence per 
customer — useful for spotting who buys regularly vs. sporadically, and 
could inform when to send re-engagement reminders (e.g., if someone's 
gap is unusually long).
<img width="1547" height="777" alt="window function 4" src="https://github.com/user-attachments/assets/b6d8c79e-cfaa-43a3-9c9b-fc70c68da132" />

---
   ## Challenges & Resolutions
   - Initially couldn't connect to the database because the Oracle Listener 
     service wasn't running/registered — resolved by using Net Configuration 
     Assistant (netca) to create a new listener.
   - Forgot the SYS password from installation — resolved by connecting via 
     OS authentication (`sqlplus / as sysdba`) and resetting it with 
     `ALTER USER`.
   - Hit `ORA-01109: database not open` — the pluggable database wasn't 
     open after a fresh restart; resolved with `ALTER PLUGGABLE DATABASE 
     ALL OPEN`, and added a startup trigger to auto-open it going forward.
