# Danny Ma Case Study 1

Access Relational Database and instructions [HERE](https://8weeksqlchallenge.com/case-study-1/)

# Questions
The Case Study seesk to answer the following questions:

1. What is the total amount each customer spent at the restaurant?
2. How many days has each customer visited the restaurant?
3. What was the first item from the menu purchased by each customer?
4. What is the most purchased item on the menu and how many times was it purchased by all customers?
5. Which item was the most popular for each customer?
6. Which item was purchased first by the customer after they became a member?
7. Which item was purchased just before the customer became a member?
8. What is the total items and amount spent for each member before they became a member?
9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points     do customer A and B have at the end of January?

# SQL Syntx 

1. What is the total amount each customer spent at the restaurant?

**Query #1**
~~~SQL
    SELECT 
    	customer_id, SUM(price)  AS Total
    FROM dannys_diner.menu AS m 
    LEFT JOIN dannys_diner.sales AS s
    USING (product_id) 
    GROUP BY customer_id
    ORDER BY Total DESC;
~~~

| customer_id | total |
| ----------- | ----- |
| A           | 76    |
| B           | 74    |
| C           | 36    |

---
2. How many days has each customer visited the restaurant?

**Query #2**
~~~ SQL
    SELECT 
    	sub.customer_id, SUM (Visit) TotalVisit
    FROM 
    (SELECT 
    	customer_id, COUNT(DISTINCT order_date) AS Visit
    FROM dannys_diner.sales
    GROUP BY customer_id, order_date
    ORDER BY Visit DESC) AS sub
    GROUP BY sub.customer_id
    ORDER BY SUM (Visit) DESC;
~~~
| customer_id | totalvisit |
| ----------- | ---------- |
| B           | 6          |
| A           | 4          |
| C           | 2          |

---
3. What was the first item from the menu purchased by each customer?
**Query #3**
~~~SQL
    SELECT customer_id, product_name
    FROM dannys_diner.sales
    LEFT JOIN dannys_diner.menu
    USING (product_id)
    ORDER BY order_date, customer_id
    LIMIT 4;
~~~

| customer_id | product_name |
| ----------- | ------------ |
| A           | sushi        |
| A           | curry        |
| B           | curry        |
| C           | ramen        |

---
4. What is the most purchased item on the menu and how many times was it purchased by all customers?
**Query #4**
~~~ SQL
    SELECT 
    	DISTINCT product_name, 
      COUNT(product_id) OVER (PARTITION BY product_name) 
    FROM dannys_diner.sales
    LEFT JOIN dannys_diner.menu
    USING (product_id);
~~~

| product_name | count |
| ------------ | ----- |
| ramen        | 8     |
| curry        | 4     |
| sushi        | 3     |

---
5. Which item was the most popular for each customer?
**Query #5**
~~~ SQL

    WITH Popular AS --CTE 
    
    (SELECT 
    	customer_id,product_name, COUNT(product_name) AS OrderCount
    FROM dannys_diner.sales
    LEFT JOIN dannys_diner.menu
    USING(product_id)
    GROUP BY customer_id,product_name
    ORDER BY customer_id, ordercount DESC) 
    
    SELECT *
    FROM 
    (SELECT Popular.customer_id, popular.product_name, popular.ordercount, 
    RANK () OVER (PARTITION BY Popular.customer_id ORDER BY popular.ordercount DESC) AS Rank --Ranking each customer's in descending order
    FROM Popular) AS sub --Subquery 
    WHERE Rank = 1 ; --Returns the most frequently ordered product of each customer
~~~

| customer_id | product_name | ordercount | rank |
| ----------- | ------------ | ---------- | ---- |
| A           | ramen        | 3          | 1    |
| B           | ramen        | 2          | 1    |
| B           | curry        | 2          | 1    |
| B           | sushi        | 2          | 1    |
| C           | ramen        | 3          | 1    |

---
6. Which item was purchased first by the customer after they became a member?
**Query #6**
~~~ SQL
    SELECT *
    FROM 
    (
      SELECT s.customer_id, s.order_date, m.product_name, mm.join_date, 
      RANK () OVER (PARTITION BY s.customer_id ORDER BY s.order_date) AS Rank
    FROM dannys_diner.sales AS s
    LEFT JOIN dannys_diner.menu AS m ON s.product_id= m.product_id
    LEFT JOIN dannys_diner.members AS mm ON s.customer_id = mm.customer_id
    WHERE mm.join_date IS NOT NULL AND 
      s.order_date >= '2021-01-07T00:00:00.000Z' --Filtering out the non-members and remaining customers became members
    ORDER BY s.order_date ASC) AS sub -- Subquery
    WHERE Rank = 1; -- Returns first order of each customer following their initial membership join date
 ~~~

| customer_id | order_date               | product_name | join_date                | rank |
| ----------- | ------------------------ | ------------ | ------------------------ | ---- |
| A           | 2021-01-07T00:00:00.000Z | curry        | 2021-01-07T00:00:00.000Z | 1    |
| B           | 2021-01-11T00:00:00.000Z | sushi        | 2021-01-09T00:00:00.000Z | 1    |

---
7. Which item was purchased just before the customer became a member?
**Query #7**
~~~ SQL

    SELECT *
    FROM (
    SELECT s.customer_id, s.order_date, m.product_name, mm.join_date, 
    RANK () OVER (PARTITION BY s.customer_id ORDER BY s.order_date DESC) AS Rank 
    FROM dannys_diner.sales AS s
    LEFT JOIN dannys_diner.menu AS m ON s.product_id= m.product_id
    LEFT JOIN dannys_diner.members AS mm ON s.customer_id = mm.customer_id
    WHERE mm.join_date IS NOT NULL AND
      s.order_date  < '2021-01-07T00:00:00.000Z' -- Filter non-members and dates prior to both customers becoming members
    ) AS sub -- Subquery
    WHERE RANK = 1; --Returns the date of a purchase prior to them becoming a member
~~~

| customer_id | order_date               | product_name | join_date                | rank |
| ----------- | ------------------------ | ------------ | ------------------------ | ---- |
| A           | 2021-01-01T00:00:00.000Z | sushi        | 2021-01-07T00:00:00.000Z | 1    |
| A           | 2021-01-01T00:00:00.000Z | curry        | 2021-01-07T00:00:00.000Z | 1    |
| B           | 2021-01-04T00:00:00.000Z | sushi        | 2021-01-09T00:00:00.000Z | 1    |

---
8. What is the total items and amount spent for each member before they became a member?
**Query #8**
~~~ SQL
    SELECT 
    	s.customer_id,COUNT(m.product_name) AS TotalItems, --Count of items purchased
        SUM(m.price) AS AmountSpent ,mm.join_date -- Sum of amount of money spent
    FROM dannys_diner.sales AS s
    LEFT JOIN dannys_diner.menu AS m ON s.product_id= m.product_id
    LEFT JOIN dannys_diner.members AS mm ON s.customer_id = mm.customer_id
    WHERE mm.join_date IS NOT NULL AND
      s.order_date  < '2021-01-07T00:00:00.000Z' -- Filter non-members and date prior to customers becoming members
    GROUP BY s.customer_id,mm.join_date
    ORDER BY SUM(m.price) DESC;
~~~

| customer_id | totalitems | amountspent | join_date                |
| ----------- | ---------- | ----------- | ------------------------ |
| B           | 3          | 40          | 2021-01-09T00:00:00.000Z |
| A           | 2          | 25          | 2021-01-07T00:00:00.000Z |

---
9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
**Query #9**
~~~ SQL
    WITH pts AS -- CTE
    (SELECT 
    	s.customer_id, m.product_name, m.price, m.price * 10 AS Pts, -- Calculating point totals of customers
        	CASE WHEN m.product_name = 'sushi' 
          THEN 2 * (M.price * 10) ELSE 0 END AS Sushi --Case statment to assign 2x point value to sushi
    FROM dannys_diner.sales AS s
    LEFT JOIN dannys_diner.menu AS m
    USING(product_id)
    GROUP BY s.customer_id, m.product_name, m.price)
    
    SELECT 
    	 distinct sub.customer_id, 
       SUM(sub.totalpts) OVER (PARTITION BY sub.customer_id) --sum of pts earned via purchases and sushi order bonuses
    FROM 
    (SELECT 
    	DISTINCT pts.customer_id,pts.pts + pts.sushi AS Totalpts
    FROM pts
    GROUP BY Pts.customer_id,pts.pts ,pts.sushi
    ORDER BY pts.pts + pts.sushi DESC) AS sub --subquery
    ORDER BY sub.customer_id;
~~~

| customer_id | sum |
| ----------- | --- |
| A           | 570 |
| B           | 570 |
| C           | 120 |

---
10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points     do customer A and B have at the end of January?
**Query #10**
~~~ SQL
    SELECT 
    	DISTINCT sub.customer_id, 
      SUM(sub.PtsTotal) OVER (PARTITION BY sub.customer_id) AS Pts_Total -- sum of total points partitioned by customer 
    FROM
    (
    SELECT s.customer_id, s.order_date, m.product_name, m.price, 
    2* (m.price * 10) AS PtsTotal -- Point calculation 
    FROM dannys_diner.sales AS s
    LEFT JOIN dannys_diner.menu AS m ON s.product_id= m.product_id
    LEFT JOIN dannys_diner.members AS mm ON s.customer_id = mm.customer_id
    WHERE mm.join_date IS NOT NULL AND 
    mm.join_date BETWEEN '2021-01-07T00:00:00.000Z' AND '2021-02-01T00:00:00.000Z'
    GROUP BY s.customer_id, s.order_date, m.product_name, m.price) AS sub; -- Subquery
~~~

| customer_id | pts_total |
| ----------- | --------- |
| B           | 1480      |
| A           | 1280      |

---

[View on DB Fiddle](https://www.db-fiddle.com/f/2rM8RAnq7h5LLDTzZiRWcd/138)

