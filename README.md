# Домашнее задание 3. Группировка данных и оконные функции.

### 0. Загрузка и проверка набора данных
#### Создание обьектов базы данных и загрузка при помощи tsql
```sql
create database hw2;

\c hw2

CREATE TABLE customer (
    customer_id INT,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    gender VARCHAR(50),
    DOB DATE,
    job_title VARCHAR(100),
    job_industry_category VARCHAR(100),
    wealth_segment VARCHAR(100),
    deceased_indicator VARCHAR(10),
    owns_car BOOLEAN,
    address VARCHAR(200),
    postcode VARCHAR(50),
    state VARCHAR(100),
    country VARCHAR(100),
    property_valuation INT
);

\copy customer FROM '/home/george/projects/db/hw02/customer.csv' WITH (FORMAT CSV,DELIMITER ';', HEADER TRUE);

CREATE TABLE product (
    product_id INT PRIMARY KEY,
    brand VARCHAR(100),
    product_line VARCHAR(50),
    product_class VARCHAR(50),
    product_size VARCHAR(
    list_price DECIMAL(10, 2),
    standard_cost DECIMAL(10, 2)
);

-- duplictaing key in raw
CREATE TABLE product (                                                                                      
    product_id INT,
    brand VARCHAR(100),
    product_line VARCHAR(50),
    product_class VARCHAR(50),
    product_size VARCHAR(50),
    list_price DECIMAL(10, 2),
    standard_cost DECIMAL(10, 2)
);

\copy product FROM '/home/george/projects/db/hw02/product.csv' WITH (FORMAT CSV,DELIMITER ',', HEADER TRUE);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    online_order BOOLEAN,
    order_status VARCHAR(50)
);

-- orders содержит "" в поле online_order, по этому не можем использовать BOOLEAN
CREATE TABLE orders (                                                                                     
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    online_order VARCHAR(50),
    order_status VARCHAR(50)
);

\copy orders FROM '/home/george/projects/db/hw02/orders.csv' WITH (FORMAT CSV,DELIMITER ',', HEADER TRUE);

-- order_items
CREATE TABLE order_items (
    order_item_id INT,
    order_id INT,
    product_id INT,
    quantity DECIMAL(10, 2),
    item_list_price_at_sale DECIMAL(10, 2),
    item_standard_cost_at_sale DECIMAL(10, 2)
);

\copy order_items FROM '/home/george/projects/db/hw02/order_items.csv' WITH (FORMAT CSV,DELIMITER ',', HEADER TRUE);
```
#### Модель набора данных
<img width="712" height="635" alt="hw2" src="https://github.com/user-attachments/assets/d2b54d5a-db12-4541-ab93-bdb8a00a81f1" />

#### Аномальные/дублирующиеся значения полей
```sql
-- аномальные/дублирующиеся значения первичного ключа в таблице product:
WITH anomaly AS (
	SELECT count(p.product_id)
  FROM product p
	GROUP BY p.product_id
	HAVING count(p.product_id) > 1)
SELECT count(*)
FROM anomaly;

-- аномальные значения бренда
SELECT count(*)
FROM product p 
WHERE length(p.brand) = 0;
```

### 1. Вывести распределение (количество) клиентов по сферам деятельности, отсортировав результат по убыванию количества.

```sql
/*
1. Вывести распределение (количество) клиентов по сферам деятельности, 
   отсортировав результат по убыванию количества.
   
   Примечание:
   Учитываются только непустые значения
*/
SELECT c.job_title, count(c.customer_id)
FROM customer c 
GROUP BY c.job_title
HAVING c.job_title IS NOT NULL 
ORDER BY count(c.customer_id) DESC;
```
<img width="1064" height="652" alt="01" src="https://github.com/user-attachments/assets/34967610-1a00-436e-8c90-b959239b9155" />

### 2. Найти общую сумму дохода (list_price*quantity) по всем подтвержденным заказам за каждый месяц по сферам деятельности клиентов. Отсортировать результат по году, месяцу и сфере деятельности.

```sql
/*
2. Найти общую сумму дохода (list_price*quantity)
   по всем подтвержденным заказам
   за каждый месяц
   по сферам деятельности клиентов.
   Отсортировать результат по году, месяцу и сфере деятельности.
*/
WITH approved_month_order AS (
    SELECT o.order_id,
       o.customer_id,
       EXTRACT(YEAR FROM o.order_date) AS order_year,
       EXTRACT(MONTH FROM o.order_date) AS order_month
    FROM orders o
    WHERE o.order_status = 'Approved'
),
order_month_income AS (
    SELECT oi.order_item_id,
        oi.order_id,
        amo.order_year,
        amo.order_month,
        p.list_price * oi.quantity  AS income
    FROM order_items oi
    INNER JOIN product p ON p.product_id = oi.product_id
    INNER JOIN approved_month_order amo ON amo.order_id = oi.order_id 
)
SELECT omi.order_year, omi.order_month, c.job_title,
    sum(omi.income)::NUMERIC(10, 2)
FROM customer c 
INNER JOIN orders o ON o.customer_id = c.customer_id 
INNER JOIN order_month_income omi ON omi.order_id = o.order_id 
GROUP BY omi.order_year, omi.order_month, c.job_title
ORDER BY omi.order_year, omi.order_month, c.job_title;
```
<img width="1064" height="1032" alt="02" src="https://github.com/user-attachments/assets/2ee90e08-7e0d-474c-998e-0aef91eb956f" />

### 3. Вывести количество уникальных онлайн-заказов для всех брендов в рамках подтвержденных заказов клиентов из сферы IT. Включить бренды, у которых нет онлайн-заказов от IT-клиентов, с количеством 0.

```sql
/*
3. Вывести количество уникальных онлайн-заказов для всех брендов
   в рамках подтвержденных заказов клиентов из сферы IT. 
   Включить бренды, у которых нет онлайн-заказов от IT-клиентов,
   для них должно быть указано количество 0.
   
   Примечание
   1) В таблице product присутствует запись о продукте без названия
   но с ценой. Поскольку brand не NULL то оставим в запросе
   2) В таблице product дублируются product_id, т.е. product_id
   не уникальный ключ исходя из данных.
*/
WITH it_order AS (
	SELECT o.order_id 
	FROM customer c 
	INNER JOIN orders o ON o.customer_id = c.customer_id 
	WHERE COALESCE(NULLIF(o.online_order, '')::BOOLEAN, FALSE) = TRUE 
		AND o.order_status = 'Approved'
		AND c.job_industry_category = 'IT'
)
SELECT p.brand,
	COALESCE(COUNT(DISTINCT oi.order_id), 0) AS orders_num 
FROM product p
LEFT JOIN order_items oi ON oi.product_id = p.product_id 
	AND oi.order_id IN (SELECT order_id FROM it_order)
GROUP BY p.brand;
```
<img width="1064" height="861" alt="03" src="https://github.com/user-attachments/assets/1480847b-df64-4578-b057-fcec218906aa" />

### 4. Найти по всем клиентам: сумму всех заказов (общего дохода), максимум, минимум и количество заказов, а также среднюю сумму заказа по каждому клиенту. Отсортировать результат по убыванию суммы всех заказов и количества заказов. Выполнить двумя способами: используя только GROUP BY и используя только оконные функции. Сравнить результат.

```sql
/*
/*
4. Найти по всем клиентам:
     сумму всех заказов (общего дохода),
     максимум, (заказа)
     минимум  (заказа)
     и количество заказов,
     а также среднюю сумму заказа по каждому клиенту.
   Отсортировать результат по убыванию суммы всех заказов и количества заказов.

Выполнить двумя способами:
    1) используя только GROUP BY
    2) и используя только оконные функции.
Сравнить результат.
*/

-- 1) только GROUP BY
WITH order_sum AS (
    SELECT oi.order_id,
    sum(oi.item_list_price_at_sale * oi.quantity)::NUMERIC(10, 2) AS order_total
FROM order_items oi 
GROUP BY oi.order_id
)
SELECT o.customer_id,
    SUM(os.order_total) AS total_orders_sum,
    MAX(os.order_total) total_orders_max,
    MIN(os.order_total) total_orders_min,
    COUNT(o.order_id) AS orders_num,
    ROUND(AVG(os.order_total), 2) total_orders_avg
FROM orders o
INNER JOIN order_sum os ON os.order_id = o.order_id 
GROUP BY o.customer_id 
ORDER BY total_orders_sum DESC, orders_num DESC;

-- 2) только оконные функции
WITH order_sum AS (
    SELECT oi.order_id,
        (SUM(oi.item_list_price_at_sale * oi.quantity) OVER w_order)::NUMERIC(10, 2) AS order_total
    FROM order_items oi 
    WINDOW w_order AS (PARTITION BY oi.order_id)
)
SELECT DISTINCT o.customer_id,
    SUM(os.order_total) OVER w_customer AS total_orders_sum,
    MAX(os.order_total) OVER w_customer total_orders_max,
    MIN(os.order_total) OVER w_customer total_orders_min,
    COUNT(o.order_id) OVER w_customer AS orders_num,
    ROUND(AVG(os.order_total) OVER w_customer, 2) total_orders_avg
FROM orders o
INNER JOIN order_sum os ON os.order_id = o.order_id 
WINDOW w_customer AS (PARTITION BY o.customer_id)
ORDER BY total_orders_sum DESC, orders_num DESC; 
```

<img width="1084" height="1051" alt="04-1" src="https://github.com/user-attachments/assets/397173a7-7cbc-49ee-a301-39d001303f11" />

<img width="1084" height="747" alt="04-2" src="https://github.com/user-attachments/assets/5cd4d961-93a4-4958-87ec-29527d83a218" />

### 5. Найти имена и фамилии клиентов с топ-3 минимальной и топ-3 максимальной суммой транзакций за весь период (учесть клиентов, у которых нет заказов).

```sql
/*
5. Найти имена и фамилии клиентов с топ-3 минимальной 
   и топ-3 максимальной суммой транзакций (заказа?)
   за весь период (учесть клиентов, у которых нет заказов)
*/
-- суммы по заказам
WITH order_sum AS (
    SELECT oi.order_id,
    sum(oi.item_list_price_at_sale * oi.quantity) AS order_total
    FROM order_items oi 
    GROUP BY oi.order_id
),
-- максимальные суммы заказа
max_order_rank AS (
    SELECT o.order_id,
        dense_rank() OVER w AS max_rank
    FROM orders o
    INNER JOIN order_sum os ON os.order_id = o.order_id
    WINDOW w AS (ORDER BY os.order_total DESC)
),
-- минимальные суммы заказа
min_order_rank AS (
    SELECT o.order_id,
        dense_rank() OVER w AS min_rank
    FROM orders o
    INNER JOIN order_sum os ON os.order_id = o.order_id
    WINDOW w AS (ORDER BY os.order_total ASC)
)
SELECT c.first_name, c.last_name
FROM customer c
LEFT JOIN orders o ON o.customer_id = c.customer_id 
LEFT JOIN order_sum os ON os.order_id = o.order_id
WHERE (o.order_id IN (SELECT order_id FROM max_order_rank WHERE max_rank <= 3)
        OR o.order_id IN (SELECT order_id FROM min_order_rank WHERE min_rank <= 2))
    OR o.order_id IS NULL
```
<img width="1084" height="994" alt="05" src="https://github.com/user-attachments/assets/3ccfb6bb-5468-4b7d-8bd9-ef3763d0a391" />

### 6. Вывести только вторые транзакции клиентов (если они есть). Решить с помощью оконных функций. Если у клиента меньше двух транзакций, он не должен попасть в результат.

```sql
/*
6. Вывести только вторые транзакции клиентов (если они есть).
   Решить с помощью оконных функций. 
   Если у клиента меньше двух транзакций, он не должен попасть в результат
*/
WITH customer_order AS (
    SELECT o.customer_id,
        o.order_id,
        dense_rank() over w as "order_number"
    FROM orders o 
    WINDOW w AS (
        PARTITION BY o.customer_id 
        ORDER BY o.order_date ASC
    )
    ORDER BY o.customer_id
)
SELECT co.customer_id, co.order_id
FROM customer_order co
WHERE co.order_number = 2
```
<img width="1084" height="804" alt="06" src="https://github.com/user-attachments/assets/c629c3b1-00cc-43f1-ba15-7cb4548ef083" />

### 7. Вывести имена, фамилии и профессии клиентов, а также длительность максимального интервала (в днях) между двумя последовательными заказами. Исключить клиентов, у которых только один или меньше заказов.

```sql
/*
Вывести имена, фамилии и профессии клиентов, 
а также длительность максимального интервала (в днях) 
между двумя последовательными заказами.
Исключить клиентов, у которых только один или меньше заказов

*/
WITH order_date AS (
    SELECT o.customer_id,
        o.order_id,
        o.order_date,
        lead(o.order_date) OVER w AS next_order
    FROM orders o 
    WINDOW w AS (
        PARTITION BY o.customer_id
        ORDER BY o.order_id
    )
    ORDER BY o.customer_id, o.order_id
),
customer_max_interval AS (
    SELECT od.customer_id,
        max(EXTRACT(EPOCH FROM age(od.next_order, od.order_date)) / (24*60*60)) AS max_interval
    FROM order_date od
    GROUP BY od.customer_id
)
SELECT c.first_name, c.last_name, c.job_title, cmi.max_interval::NUMERIC(10)
FROM customer c 
INNER JOIN customer_max_interval cmi ON cmi.customer_id = c.customer_id
WHERE cmi.max_interval IS NOT NULL;
```
<img width="1084" height="861" alt="07" src="https://github.com/user-attachments/assets/5f545a01-cf58-41ec-be19-609374f0d43b" />

### 8. Найти топ-5 клиентов (по общему доходу) в каждом сегменте благосостояния (wealth_segment). Вывести имя, фамилию, сегмент и общий доход. Если в сегменте менее 5 клиентов, вывести всех.

```sql
/*
8. Найти топ-5 клиентов (по общему доходу) (orders amount)
   в каждом сегменте благосостояния (wealth_segment). 
   Вывести имя, фамилию, сегмент и общий доход. 
   Если в сегменте менее 5 клиентов, вывести всех
*/
WITH order_sum AS (
    SELECT oi.order_id,
        sum(oi.item_list_price_at_sale * oi.quantity) AS order_total
    FROM order_items oi 
    GROUP BY oi.order_id
),
customer_sum AS (
    SELECT c.customer_id,
        sum(os.order_total) AS customer_total
    FROM customer c 
    INNER JOIN orders o ON o.customer_id = c.customer_id 
    INNER JOIN order_sum os ON os.order_id = o.order_id
    GROUP BY c.customer_id 
),
customer_rank AS (
    SELECT c.customer_id,
        c.first_name,
        c.last_name,
        c.wealth_segment,
        cs.customer_total,
        dense_rank() OVER w AS "order_number"
    FROM customer c
    INNER JOIN customer_sum cs ON cs.customer_id = c.customer_id
    WINDOW w AS (
        PARTITION BY c.wealth_segment
        ORDER BY cs.customer_total DESC
    )
)
SELECT cr.first_name,
    cr.last_name,
    cr.wealth_segment,
    cr.customer_total,
    cr.order_number
FROM customer_rank cr
WHERE  cr.order_number <=5;
```
<img width="1084" height="1279" alt="08" src="https://github.com/user-attachments/assets/292fb71a-e367-4a6b-bd9a-44b98dcbf1ff" />
