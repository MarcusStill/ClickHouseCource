# Инкрементальное материализованное представление

## Сценарий 1.

### 1.1. Создаем таблицу с заказами
```sql
DROP TABLE IF EXISTS learn_db.orders;
CREATE TABLE orders (
	order_id UInt32,
	user_id UInt32,
	product_id UInt32,
	amount Decimal(18, 2),
	order_date Date
)
ENGINE = MergeTree()
ORDER BY (order_id, user_id);
```

### 1.2. Создаем агрегированную таблицу с заказами по id продукта и дате заказа

```sql
DROP TABLE IF EXISTS learn_db.orders_sum;
CREATE TABLE learn_db.orders_sum (
	product_id UInt32,
	order_date Date,
	amount Decimal(18, 2)
)
ENGINE = SummingMergeTree()
ORDER BY (product_id, order_date);
```

### 1.3. Создаем материализованное представление, отслеживающее новые строки в orders и добавляющее строки в orders_sum_mv

```sql
DROP TABLE IF EXISTS learn_db.orders_sum_mv;
CREATE MATERIALIZED VIEW learn_db.orders_sum_mv TO learn_db.orders_sum AS
SELECT product_id,
       order_date,
       SUM(amount) as amount
FROM learn_db.orders
GROUP BY 
	product_id,
    order_date;
```
### 1.4. Вставляем новые строки в orders

```sql
INSERT INTO learn_db.orders
(order_id, user_id, product_id, amount, order_date)
VALUES
(1, 1, 1, 10, '2025-01-01'),
(2, 2, 1, 10, '2025-01-01'),
(3, 1, 2, 5, '2025-01-01'),
(4, 2, 2, 5, '2025-01-01');
```

### 1.5. Проверяем, появились ли новые строки в orders_sum

```sql
SELECT 
	*
FROM 
	learn_db.orders_sum;
```

Результат
```text
product_id|order_date|amount|
----------+----------+------+
         1|2025-01-01| 20.00|
         2|2025-01-01| 10.00|
```

### 1.6. Вставляем еще одну строку в orders

```sql
INSERT INTO learn_db.orders
(order_id, user_id, product_id, amount, order_date)
VALUES
(5, 3, 1, 10, '2025-01-01');
```

### 1.7. Смотрим, что в orders_sum

```sql
SELECT 
	*
FROM 
	learn_db.orders_sum;
```

Результат
```text
product_id|order_date|amount|
----------+----------+------+
         1|2025-01-01| 20.00|
         2|2025-01-01| 10.00|
         1|2025-01-01| 10.00|
```

```sql
SELECT 
	*
FROM 
	learn_db.orders_sum FINAL;
```

Результат
```text
product_id|order_date|amount|
----------+----------+------+
         1|2025-01-01| 30.00|
         2|2025-01-01| 10.00|
```

```sql
SELECT 
	product_id,
    order_date,
    SUM(amount) as amount
FROM
	learn_db.orders_sum
GROUP BY 
	product_id,
    order_date;
```

Результат
```text
product_id|order_date|amount|
----------+----------+------+
         2|2025-01-01| 10.00|
         1|2025-01-01| 30.00|
```

### 1.8. Вставляем в orders 100 000 000 строк

```sql
INSERT INTO learn_db.orders
SELECT
	number + 10 as order_id,
	round(randUniform(1, 100000)) as user_id,
	round(randUniform(1, 10000)) as product_id,
	randUniform(1, 5000) as amount,
	date_add(day, rand() % 366, today() - INTERVAL 1 YEAR)
FROM 
	numbers(100000000);
```

### 1.9. Смотрим, сколько строк в orders и orders_sum

```sql
SELECT COUNT(*) FROM learn_db.orders_sum;
```

Результат
```text
COUNT()|
-------+
8849531|
```

```sql
SELECT COUNT(*) FROM learn_db.orders_sum FINAL;
```

Результат
```text
COUNT()|
-------+
3660000|
```

```sql
SELECT COUNT(*) FROM learn_db.orders;
```

Результат
```text
COUNT()  |
---------+
100000005|
```

### 1.10. Получаем последние запросы из Datalens

```sql
SELECT * FROM system.query_log WHERE type = 2 and `http_user_agent` = 'DataLens' ORDER BY `event_time` DESC;
```

## Сценарий 2.

### 2.1. Создаем вспомогательную таблицу orders_user_order

```sql
DROP TABLE IF EXISTS learn_db.orders_user_order;
CREATE TABLE learn_db.orders_user_order (
	order_id UInt32,
	user_id UInt32
) ENGINE = MergeTree ORDER BY user_id;
```

### 2.2. Создаем материализованное представление orders_user_order_mv для наполнения orders_user_order

```sql
DROP TABLE IF EXISTS learn_db.orders_user_order_mv;
CREATE MATERIALIZED VIEW learn_db.orders_user_order_mv TO learn_db.orders_user_order AS
SELECT 
	order_id,
    user_id
FROM learn_db.orders;
```

### 2.3. Получаем количество строк из orders_user_order

```sql
SELECT COUNT(*) FROM learn_db.orders_user_order;
```

Результат
```text
COUNT()|
-------+
      0|
```

```sql
SELECT COUNT(*) FROM learn_db.orders_user_order_mv;
```

Результат
```text
COUNT()|
-------+
      0|
```

### 2.4. Вставляем 4 строки в orders и смотрим содержимое orders_user_order

```sql
INSERT INTO learn_db.orders
(order_id, user_id, product_id, amount, order_date)
VALUES
(100000011, 1, 1, 10, '2025-01-01'),
(100000012, 2, 1, 10, '2025-01-01'),
(100000013, 1, 2, 5, '2025-01-01'),
(100000014, 2, 2, 5, '2025-01-01');

SELECT COUNT(*) FROM learn_db.orders_user_order;
SELECT COUNT(*) FROM learn_db.orders_user_order_mv;
```

Результат
```text
COUNT()|
-------+
      4|
```

Результат
```text
COUNT()|
-------+
      4|
```

### 2.5. Вставляем в orders_user_order все строки из orders, за исключением последних 4

```
INSERT INTO `orders_user_order`
(
	`order_id`, 
	`user_id`
)
SELECT
	order_id,
	user_id
FROM 	
	learn_db.orders
WHERE 
	order_id <= 100000010;
```

```sql
SELECT * FROM learn_db.orders;
```

### 2.6. Проверяем количество записей в таблице

Результат
```text
Query id: 4035e9ce-44c3-4189-818f-63c517cc697e

         ┌─order_id─┬─user_id─┬─product_id─┬──amount─┬─order_date─┐
      1. │ 65605237 │   59370 │       4223 │ 1328.87 │ 2024-10-11 │
      2. │ 65605238 │   68082 │        343 │ 4706.15 │ 2025-01-25 │
      3. │ 65605239 │   88097 │       9897 │ 3092.66 │ 2024-12-26 │
      4. │ 65605240 │   20023 │        906 │ 2797.58 │ 2025-08-10 │
      5. │ 65605241 │   37280 │        317 │ 3173.94 │ 2025-09-24 │
1064960. │ 33481479 │   77126 │       8579 │ 3543.09 │ 2025-09-07 │
         └─order_id─┴─user_id─┴─product_id─┴──amount─┴─order_date─┘
Showed 1000 out of 100000009 rows.

100000009 rows in set. Elapsed: 9.132 sec. Processed 100.00 million rows, 2.20 GB (10.95 million rows/s., 240.90 MB/s.)
Peak memory usage: 28.26 MiB.
```

### 2.7. Замеряем время поиска одного заказа по order_id

```sql
SELECT 
	*
FROM
	learn_db.orders
WHERE 
	order_id = 1000;
```

Результат
```text
Query id: c23ee8be-057d-4d76-9948-2ce4ca3f4222

   ┌─order_id─┬─user_id─┬─product_id─┬──amount─┬─order_date─┐
1. │     1000 │   35291 │       9472 │ 3206.67 │ 2025-01-10 │
   └──────────┴─────────┴────────────┴─────────┴────────────┘

1 row in set. Elapsed: 0.418 sec. Processed 8.19 thousand rows, 50.70 KB (19.61 thousand rows/s., 121.36 KB/s.)
Peak memory usage: 24.66 KiB.
```

### 2.8. Выполним запрос к вспомогательной таблице

```sql
SELECT
    order_id
FROM
    learn_db.orders_user_order
WHERE
    user_id = 1000;
```

Результат
```text
Query id: b60ac94f-6564-4d84-9980-a910fc3d5aa5

      ┌─order_id─┐
   1. │ 90647875 │
   2. │ 49001801 │
   3. │ 49011179 │
   4. │ 65715435 │
   5. │ 74058893 │
1012. │ 64401098 │
      └─order_id─┘
Showed 1000 out of 1012 rows.

1012 rows in set. Elapsed: 0.007 sec. Processed 57.34 thousand rows, 432.36 KB (8.80 million rows/s., 66.32 MB/s.)
Peak memory usage: 542.84 KiB.
```

### 2.9. Замеряем время поиска всех заказов одного покупателя с применением вспомогательной таблицы

```sql
SELECT 
	*
FROM
	learn_db.orders
WHERE 
	order_id in (
		SELECT
			order_id
		FROM
			learn_db.orders_user_order
		WHERE
			user_id = 1000
	);
```

Результат
```text
Query id: 7bfffe74-bbfe-43a8-9668-1d22397d9759

         ┌─order_id─┬─user_id─┬─product_id─┬──amount─┬─order_date─┐
      1. │ 81916177 │    1000 │       9188 │ 1030.76 │ 2025-08-28 │
      2. │ 81976145 │    1000 │       7967 │ 2258.33 │ 2025-01-06 │
      3. │ 81978671 │    1000 │       4429 │ 4738.32 │ 2024-11-13 │
      4. │ 82006493 │    1000 │       7124 │ 1611.69 │ 2025-08-09 │
      5. │ 82032337 │    1000 │       5230 │ 2718.56 │ 2025-08-05 │
   1000. │  3246223 │    1000 │       7282 │ 2803.87 │ 2025-08-27 │
         └─order_id─┴─user_id─┴─product_id─┴──amount─┴─order_date─┘
Showed 1000 out of 1012 rows.

1012 rows in set. Elapsed: 0.346 sec. Processed 8.04 million rows, 135.20 MB (23.24 million rows/s., 391.05 MB/s.)
Peak memory usage: 1.69 MiB.
```

### 2.8. Создаем материализованное представление, сохраняющее данные в себе

```
DROP TABLE IF EXISTS learn_db.orders_user_order_mv_v2;
CREATE MATERIALIZED VIEW learn_db.orders_user_order_mv_v2 
ENGINE = MergeTree()
ORDER BY (user_id)
AS SELECT 
	order_id,
    user_id
FROM learn_db.orders;
```

### 2.9. Смотрим содержимое orders_user_order_mv_v2

```sql
SELECT * FROM learn_db.orders_user_order_mv_v2;
```

Результат
```text
order_id|user_id|
--------+-------+
```

```sql
SELECT COUNT(*) FROM learn_db.orders_user_order_mv_v2;
```

Результат
```text
COUNT()|
-------+
      0|
```

### 2.10. Вставляем еще один заказ в orders и смотрим содержимое orders_user_order_mv_v2

```sql
INSERT INTO learn_db.orders
(order_id, user_id, product_id, amount, order_date)
VALUES
(100000015, 1, 1, 10, '2025-01-01');
```

```sql
SELECT * FROM learn_db.orders_user_order_mv_v2;
```

Результат
```text
order_id |user_id|
---------+-------+
100000015|      1|
```

```sql
SELECT COUNT(*) FROM learn_db.orders_user_order_mv_v2;
```

Результат
```text
COUNT()|
-------+
      1|
```

### 2.11. Вставляем в orders_user_order_mv_v2 все заказы кроме последнего

```sql
INSERT INTO learn_db.orders_user_order_mv_v2
SELECT 
	order_id,
    user_id
FROM learn_db.orders
WHERE order_id <> 100000015;
```

### 2.12. Проверяем реагирование материализованного представления на редактирование данных

```sql
SELECT COUNT(*) FROM learn_db.orders_user_order;
```

Результат
```text
COUNT()  |
---------+
100000010|
```

```sql
SELECT COUNT(*) FROM learn_db.orders;
```

Результат
```text
COUNT()  |
---------+
100000010|
```

### 2.13. Удаляем запись.

```sql
ALTER TABLE learn_db.orders DELETE WHERE order_id = 1000;
```

```sql
SELECT COUNT(*) FROM learn_db.orders_user_order;
```

Результат
```text
COUNT()  |
---------+
100000010|
```

```sql
SELECT COUNT(*) FROM learn_db.orders;
```

Результат
```text
COUNT()  |
---------+
100000010|
```