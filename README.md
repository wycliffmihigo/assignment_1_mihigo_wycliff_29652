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

**Business interpretation:** Shows exactly which products move in which 
categories and at what volume, helping management see which categories 
(Grains, Dairy, Household) drive the most purchases.

---
