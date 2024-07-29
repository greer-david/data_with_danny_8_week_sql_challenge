# Data with Danny's 8 Week SQL Challenge
This is a breakdown of my solutions to Danny Ma's 8 Week SQL Challenge

Skills tested in Week 1
- Ranking, Common Table Expressions (CTEs), Case Statements, Dates and Scalar functions

**Schema (PostgreSQL v13)**
```sql
    CREATE SCHEMA dannys_diner;
    SET search_path = dannys_diner;
    
    CREATE TABLE sales (
      "customer_id" VARCHAR(1),
      "order_date" DATE,
      "product_id" INTEGER
    );
    
    INSERT INTO sales
      ("customer_id", "order_date", "product_id")
    VALUES
      ('A', '2021-01-01', '1'),
      ('A', '2021-01-01', '2'),
      ('A', '2021-01-07', '2'),
      ('A', '2021-01-10', '3'),
      ('A', '2021-01-11', '3'),
      ('A', '2021-01-11', '3'),
      ('B', '2021-01-01', '2'),
      ('B', '2021-01-02', '2'),
      ('B', '2021-01-04', '1'),
      ('B', '2021-01-11', '1'),
      ('B', '2021-01-16', '3'),
      ('B', '2021-02-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-07', '3');
     
    
    CREATE TABLE menu (
      "product_id" INTEGER,
      "product_name" VARCHAR(5),
      "price" INTEGER
    );
    
    INSERT INTO menu
      ("product_id", "product_name", "price")
    VALUES
      ('1', 'sushi', '10'),
      ('2', 'curry', '15'),
      ('3', 'ramen', '12');
      
    
    CREATE TABLE members (
      "customer_id" VARCHAR(1),
      "join_date" DATE
    );
    
    INSERT INTO members
      ("customer_id", "join_date")
    VALUES
      ('A', '2021-01-07'),
      ('B', '2021-01-09');
```
---

### **Query #1**
What is the total amount each customer spent at the restaurant?
```sql
    SELECT
      	customer_id,
        SUM(price) tot_spend
    FROM dannys_diner.sales s
    JOIN dannys_diner.menu m
    ON s.product_id = m.product_id
    GROUP BY customer_id
    ORDER BY tot_spend DESC
    LIMIT 5;
```
| customer_id | tot_spend |
| ----------- | --------- |
| A           | 76        |
| B           | 74        |
| C           | 36        |

---
### **Query #2**
How many days has each customer visited the restaurant?
```sql
    SELECT
    	customer_id,
        COUNT(DISTINCT order_date) days
    FROM
    	dannys_diner.sales
    GROUP BY
    	customer_id
    ;
```
| customer_id | days |
| ----------- | ---- |
| A           | 4    |
| B           | 6    |
| C           | 2    |

---
### **Query #3**
What was the first item from the menu purchased by each customer?
```sql
    WITH p_rank AS (
    SELECT
    	customer_id,
        order_date,
        product_name,
        RANK() OVER(PARTITION BY customer_id ORDER BY order_date) ord_num
    FROM
    	dannys_diner.sales s
    JOIN
    	dannys_diner.menu m
    ON
    	s.product_id = m.product_id
    )
    SELECT
    	customer_id,
        product_name
    FROM
    	p_rank
    WHERE
    	ord_num = 1
    ;
```
| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |
| C           | ramen        |

---
### **Query #4**
What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
    SELECT
    	product_name,
        COUNT(*) times_purchased
    FROM 
    	dannys_diner.sales s
    JOIN
    	dannys_diner.menu m
    ON
    	s.product_id = m.product_id
    GROUP BY
    	product_name
    ORDER BY
    	COUNT(*) DESC
    LIMIT 1
    ;
```
| product_name | times_purchased |
| ------------ | --------------- |
| ramen        | 8               |

---
### **Query #5**
Which item was the most popular for each customer?
```sql
    WITH top_p AS (
    SELECT
    	customer_id,
    	product_name,
        COUNT(*) times_purchased,
        RANK() OVER(PARTITION BY customer_id ORDER BY COUNT(*) DESC) rnk
    FROM 
    	dannys_diner.sales s
    JOIN
    	dannys_diner.menu m
    ON
    	s.product_id = m.product_id
    GROUP BY
    	customer_id,
    	product_name
    )
    SELECT
    	customer_id,
        product_name
    FROM
    	top_p
    WHERE
    	rnk = 1
    ;
```
| customer_id | product_name |
| ----------- | ------------ |
| A           | ramen        |
| B           | ramen        |
| B           | curry        |
| B           | sushi        |
| C           | ramen        |

---
### **Query #6**
Which item was purchased first by the customer after they became a member?
```sql
    WITH p_rank AS (
    SELECT
    	s.customer_id,
        order_date,
    	join_date,
        product_name,
        RANK() OVER(PARTITION BY s.customer_id ORDER BY order_date) ord_num
    FROM
    	dannys_diner.sales s
    JOIN
    	dannys_diner.menu m
    ON
    	s.product_id = m.product_id
    JOIN
      	dannys_diner.members me
    ON
      	s.customer_id = me.customer_id
    WHERE
    	order_date >= join_date
    )
    SELECT
    	customer_id,
        product_name
    FROM
    	p_rank
    WHERE
    	ord_num = 1
    ;
```
| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| B           | sushi        |

---
### **Query #7**
Which item was purchased just before the customer became a member?
```sql
    WITH p_rank AS (
    SELECT
    	s.customer_id,
        order_date,
    	join_date,
        product_name,
        RANK() OVER(PARTITION BY s.customer_id ORDER BY order_date DESC) ord_num
    FROM
    	dannys_diner.sales s
    JOIN
    	dannys_diner.menu m
    ON
    	s.product_id = m.product_id
    JOIN
      	dannys_diner.members me
    ON
      	s.customer_id = me.customer_id
    WHERE
    	order_date < join_date
    )
    SELECT
    	customer_id,
        product_name
    FROM
    	p_rank
    WHERE
    	ord_num = 1
    ;
```
| customer_id | product_name |
| ----------- | ------------ |
| A           | sushi        |
| A           | curry        |
| B           | sushi        |

---
### **Query #8**
What is the total items and amount spent for each member before they became a member?
```sql
    SELECT
    	s.customer_id,
        COUNT(order_date) item_count,
        SUM(price) tot_spend
    FROM
    	dannys_diner.sales s
    JOIN
      	dannys_diner.members me
    ON
      	s.customer_id = me.customer_id
    JOIN
      	dannys_diner.menu m
    ON
      	s.product_id = m.product_id
    WHERE
    	order_date < join_date
    GROUP BY
    	s.customer_id
    ;
```
| customer_id | item_count | tot_spend |
| ----------- | ---------- | --------- |
| B           | 3          | 40        |
| A           | 2          | 25        |

---
### **Query #9**
If each $1 spent equates to 10 points and sushi has a 2x points multiplier, how many points would each customer have?
```sql
    SELECT
    	SUM(CASE
                WHEN product_name = 'sushi' THEN (price * 10) * 2
                ELSE price * 10
            END) points,
    	s.customer_id
    FROM
    	dannys_diner.sales s
    JOIN
      	dannys_diner.menu m
    ON
      	s.product_id = m.product_id
    GROUP BY
    	s.customer_id
    ORDER BY
    	points DESC
    ;
```
| points | customer_id |
| ------ | ----------- |
| 940    | B           |
| 860    | A           |
| 360    | C           |

---
### **Query #10**
In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi how many points do customer A and B have at the end of January?
```sql
    SELECT
    	s.customer_id,
        SUM(CASE
        	WHEN order_date BETWEEN join_date AND join_date + 6 THEN price * 10 * 2
    		WHEN product_name = 'sushi' THEN (price * 10) * 2
    		ELSE price * 10
    	END) new_points
    FROM
    	dannys_diner.sales s
    JOIN
      	dannys_diner.menu m
    ON
      	s.product_id = m.product_id
    JOIN
      	dannys_diner.members me
    ON
      	s.customer_id = me.customer_id
    WHERE
    	order_date < '2021-02-01'
    GROUP BY
    	s.customer_id
    ;
```
| customer_id | new_points |
| ----------- | ---------- |
| A           | 1370       |
| B           | 820        |

---
