# Exercise 01 — E-Commerce Analytics

## Overview
Use the practice database from Lesson 01 to answer real analytics questions
as if you're a data analyst at an e-commerce company.

## Skills Practiced
JOINs, aggregations, window functions, CTEs, date functions

---

## Warm-Up (Lessons 02–04)

**Q1.** List all products priced between $30 and $200, sorted by price.
Show name, category, price, and stock.

**Q2.** Find all customers from 'US' or 'UK'. Sort by country then name.

**Q3.** Find orders placed in January 2024 that were delivered.
Show order id, order date, and total.

---

## Intermediate (Lessons 05–06)

**Q4.** How many orders are in each status? Show status and count, sorted by count descending.

**Q5.** What is the total revenue per category (from order_items × products)?
Join order_items → products. Show category, total units sold, and total revenue.
Order by revenue descending.

**Q6.** Which customers have placed more than 1 order?
Show customer name, email, order count, and total spent. Only include non-cancelled orders.

**Q7.** Find all products that have NEVER been ordered.
Show product name, category, price, and stock.

**Q8.** Show each customer and their most recent order date.
Include customers who have never ordered (show NULL for order date).

---

## Advanced (Lessons 07–08)

**Q9.** Using a CTE, find the top 3 customers by lifetime value (total non-cancelled spending).
Show rank, name, country, order count, and lifetime value.

**Q10.** For each order, show:
- Customer name
- Order date
- Order total
- That customer's cumulative spending up to that order (ordered by date)
- How their spending compares to the overall average

**Q11.** Rank products by revenue within each category.
Show category, product name, total revenue, and rank within category.

**Q12.** Calculate month-over-month revenue change.
Show month, revenue, previous month revenue, and % change.
(Hint: LAG window function + DATE_TRUNC)

---

## Expert (Lessons 09–12)

**Q13.** Build a daily sales calendar for the full month of January–March 2024.
For EVERY day in that range, show:
- Day
- Number of orders (0 if none)
- Revenue (0 if none)
- 7-day rolling average revenue
(Hint: GENERATE_SERIES + LEFT JOIN + window function)

**Q14.** Find "high-value" order items: items where the line total (quantity × unit_price)
is above the average line total across all orders. Show product name, order id,
quantity, line total, and how much above average.

**Q15.** Create a customer cohort analysis:
Group customers by the month they placed their first order.
For each cohort, show how many customers, their total spending, and average order value.

---

## Challenge Queries

**C1.** Find pairs of products that were ordered together in the same order.
Show product A name, product B name, and how many times they were co-ordered.
Only show each pair once (product A id < product B id). Sort by co-order count.

**C2.** Identify customers who placed orders in January 2024 but NOT in February 2024.
(The "churned" customers between those two months.)

---

## Answer Hints

<details>
<summary>Q5 Hint</summary>

```sql
SELECT
    p.category,
    SUM(oi.quantity) AS units_sold,
    ROUND(SUM(oi.quantity * oi.unit_price), 2) AS revenue
FROM order_items oi
JOIN products p ON p.id = oi.product_id
GROUP BY p.category
ORDER BY revenue DESC;
```
</details>

<details>
<summary>Q9 Hint</summary>

```sql
WITH spending AS (
    SELECT customer_id, COUNT(*) AS orders, SUM(total) AS lifetime_value
    FROM orders WHERE status != 'cancelled'
    GROUP BY customer_id
)
SELECT
    RANK() OVER (ORDER BY s.lifetime_value DESC) AS rank,
    c.name, c.country, s.orders, s.lifetime_value
FROM customers c
JOIN spending s ON s.customer_id = c.id
ORDER BY rank
LIMIT 3;
```
</details>

<details>
<summary>C1 Hint</summary>

```sql
SELECT
    p1.name AS product_a,
    p2.name AS product_b,
    COUNT(*) AS times_ordered_together
FROM order_items oi1
JOIN order_items oi2 ON oi1.order_id = oi2.order_id AND oi1.product_id < oi2.product_id
JOIN products p1 ON p1.id = oi1.product_id
JOIN products p2 ON p2.id = oi2.product_id
GROUP BY p1.name, p2.name
ORDER BY times_ordered_together DESC;
```
</details>
