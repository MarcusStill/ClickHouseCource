# Применение движка AggregatingMergeTree в Clickhouse

### 1. Пересоздаем таблицу orders
```sql
DROP TABLE IF EXISTS orders;
CREATE TABLE orders (
	order_id UInt32,
	user_id UInt32,
	product_id UInt32,
	amount Decimal(18, 2),
	order_date Date
)
ENGINE = MergeTree()
ORDER BY (product_id, order_date);
```

### 2. Вставляем в таблицу orders 4 строки

```sql
INSERT INTO learn_db.orders
(order_id, user_id, product_id, amount, order_date)
VALUES
(1, 1, 1, 10, '2025-01-01'),
(2, 2, 1, 10, '2025-01-01'),
(3, 1, 2, 5, '2025-01-01'),
(4, 2, 2, 5, '2025-01-01');
```

### 3. Считаем количество уникальных строк в orders

```sql
SELECT uniq(user_id) FROM learn_db.orders;
```

Результат
```text
uniq(user_id)|
-------------+
            2|
```

### 4. Пересоздаем таблицу orders_agg
```
DROP TABLE IF EXISTS learn_db.orders_agg;
CREATE TABLE learn_db.orders_agg (
	product_id UInt32,
	order_date Date,
	amount AggregateFunction(sum, Decimal(18, 2)),
    users AggregateFunction(uniq, UInt32)
)
ENGINE = AggregatingMergeTree()
ORDER BY (product_id, order_date);
```
       
### 5. Наполняем таблицу orders_agg агрегированными данными из таблицы orders
```sql
INSERT INTO learn_db.orders_agg
SELECT
	product_id,
	order_date,
	sumState(amount) as amount,
	uniqState(user_id) as users
FROM 
	learn_db.orders
GROUP BY 
	product_id,
	order_date;
```

### 6. Смотрим содержимое таблицы orders_agg

```sql
SELECT *, _part FROM learn_db.orders_agg;
```

Результат
```text
Query id: a6fa1599-b732-4404-bce0-2cc8896624e9

   ┌─product_id─┬─order_date─┬─amount─┬─users─┐
1. │          1 │ 2025-01-01 │        │ ,4e   │
2. │          2 │ 2025-01-01 │        │ ,4e   │
   └────────────┴────────────┴────────┴───────┘

2 rows in set. Elapsed: 0.004 sec.
```

### 7. Вставляем еще одну строку в orders
```sql
INSERT INTO learn_db.orders
(order_id, user_id, product_id, amount, order_date)
VALUES
(5, 3, 1, 10, '2025-01-01');
```

### 8. Вставляем эту же строку в orders_agg

```sql
INSERT INTO learn_db.orders_agg
SELECT
	product_id,
	order_date,
	sumState(amount) as amount,
	uniqState(user_id) as users
FROM 
	learn_db.orders
WHERE 
	order_id = 5
GROUP BY 
	product_id,
	order_date;
```

### 9. Смотрим содержимое таблицы orders_agg

```sql
SELECT *, _part FROM learn_db.orders_agg;
```

Результат
```text
Query id: 5f0f10c5-9f5d-4cfd-a423-f9e419d73ecb

   ┌─product_id─┬─order_date─┬─amount─┬─users─┐
1. │          1 │ 2025-01-01 │        │ ,4e   │
2. │          2 │ 2025-01-01 │        │ ,4e   │
   └────────────┴────────────┴────────┴───────┘

2 rows in set. Elapsed: 0.002 sec.
```

### 10. Выполняем принудительный процесс слияния частей в orders_agg и смотрим ее содержимое
```sql
OPTIMIZE TABLE learn_db.orders_agg FINAL;
SELECT *, _part FROM learn_db.orders_agg;
```

Результат
```text
Query id: 5d29ed7b-10bf-4900-85f2-1708f5b9e4be

   ┌─product_id─┬─order_date─┬─amount─┬─users─┬─_part─────┐
1. │          1 │ 2025-01-01 │        │ ,4e   │ all_1_1_2 │
2. │          2 │ 2025-01-01 │        │ ,4e   │ all_1_1_2 │
   └────────────┴────────────┴────────┴───────┴───────────┘

2 rows in set. Elapsed: 0.002 sec.
```

### 11. Выполняем аналитические запросы к orders_agg
```
SELECT 
	uniqMerge(users)
FROM 
	learn_db.orders_agg;
```

Результат
```text
QuniqMerge(users)|
----------------+
               2|
```

```sql
SELECT 
	product_id,
	uniqMerge(users)
FROM 
	learn_db.orders_agg
GROUP BY 
	product_id;
```

Результат
```text
product_id|uniqMerge(users)|
----------+----------------+
         2|               2|
         1|               2|
```

```sql
SELECT 
	sumMerge(amount),
	uniqMerge(users)
FROM 
	learn_db.orders_agg;
```

Результат
```text
sumMerge(amount)|uniqMerge(users)|
----------------+----------------+
           30.00|               2|
```

```
SELECT 
	product_id,
	sumMerge(amount),
	uniqMerge(users)
FROM 
	learn_db.orders_agg
GROUP BY 
	product_id;
```

Результат
```text
product_id|sumMerge(amount)|uniqMerge(users)|
----------+----------------+----------------+
         2|           10.00|               2|
         1|           20.00|               2|
```


# Замеряем скорость выполнения запросов к таблице с движком AggregatingMergeTree

### 12. Очищаем таблицы orders и orders_agg

```sql
TRUNCATE learn_db.orders;
TRUNCATE learn_db.orders_agg;

select count(*) from learn_db.orders_agg;
select count(*) from learn_db.orders;
```

### 13. Наполняем таблицу orders 100 000 000 строк

```sql
INSERT INTO learn_db.orders
SELECT
	number + 10 as order_id,
	round(randUniform(1, 1000)) as user_id,
	round(randUniform(1, 1000)) as product_id,
	randUniform(1, 5000) as amount,
	date_add(day, rand() % 366, today() - INTERVAL 1 YEAR)
FROM 
	numbers(100000000);
```

### 14. Наполняем orders_agg 366 000 строк
```sql
INSERT INTO learn_db.orders_agg
SELECT
	product_id,
	order_date,
	sumState(amount) as amount,
	uniqState(user_id) as users
FROM 
	learn_db.orders
GROUP BY 
	product_id,
	order_date;
```

### 15. Сравниваем скорость выполнения запросов

```sql
SELECT 
	product_id,
	sum(amount),
	uniq(user_id)
FROM 
	learn_db.orders
GROUP BY 
	product_id;
```

Результат
```text
Query id: 0bf6a224-089e-4705-b3a7-51d9b5202666

      ┌─product_id─┬──sum(amount)─┬─uniq(user_id)─┐
   1. │        610 │    250656791 │             1 │
   2. │        720 │ 251251603.35 │             1 │
   3. │        948 │ 250588640.81 │             1 │
   4. │        774 │ 250733371.69 │             1 │
   5. │        462 │ 251488210.12 │             1 │
   ...
   995. │        554 │ 250041638.59 │             1 │
 996. │        664 │ 249114462.72 │             1 │
 997. │         80 │ 249459787.58 │             1 │
 998. │        226 │ 249070020.61 │             1 │
 999. │        390 │ 249732951.27 │             1 │
1000. │        308 │ 251113055.34 │             1 │
      └─product_id─┴──sum(amount)─┴─uniq(user_id)─┘
Showed 1000 out of 1000 rows.

1000 rows in set. Elapsed: 1.594 sec. Processed 100.00 million rows, 1.60 GB (62.74 million rows/s., 1.00 GB/s.)
Peak memory usage: 474.73 KiB.
```

```sql
SELECT 
	product_id,
	sumMerge(amount),
	uniqMerge(users)
FROM 
	learn_db.orders_agg
GROUP BY 
	product_id;
```
Результат
```text
Query id: fde00f02-c35e-4583-a403-135e442bd1ed

      ┌─product_id─┬─sumMerge(amount)─┬─uniqMerge(users)─┐
   1. │        610 │        250656791 │                1 │
   2. │        720 │     251251603.35 │                1 │
   3. │        948 │     250588640.81 │                1 │
   4. │        774 │     250733371.69 │                1 │
   5. │        462 │     251488210.12 │                1 │
   ...
  995. │        554 │     250041638.59 │                1 │
 996. │        664 │     249114462.72 │                1 │
 997. │         80 │     249459787.58 │                1 │
 998. │        226 │     249070020.61 │                1 │
 999. │        390 │     249732951.27 │                1 │
1000. │        308 │     251113055.34 │                1 │
      └─product_id─┴─sumMerge(amount)─┴─uniqMerge(users)─┘
Showed 1000 out of 1000 rows.

1000 rows in set. Elapsed: 0.017 sec. Processed 366.00 thousand rows, 42.46 MB (21.14 million rows/s., 2.45 GB/s.)
Peak memory usage: 28.02 MiB. 
```

### 16. Очищаем таблицы orders и orders_agg
```sql
TRUNCATE learn_db.orders;
TRUNCATE learn_db.orders_agg;

select count(*) from learn_db.orders_agg;
select count(*) from learn_db.orders;
```

### 17. Наполняем таблицу orders 100 000 000 строк, но количество уникальных товаров увеличено на порядок

```sql
INSERT INTO learn_db.orders
SELECT
	number + 10 as order_id,
	round(randUniform(1, 1000)) as user_id,
	round(randUniform(1, 10000)) as product_id,
	randUniform(1, 5000) as amount,
	date_add(day, rand() % 366, today() - INTERVAL 1 YEAR)
FROM 
	numbers(100000000);
```

### 18. Наполняем orders_agg 3 660 000 строк

```sql
INSERT INTO learn_db.orders_agg
SELECT
	product_id,
	order_date,
	sumState(amount) as amount,
	uniqState(user_id) as users
FROM 
	learn_db.orders
GROUP BY 
	product_id,
	order_date;
```

### 19. Сравниваем скорость выполнения запросов
```sql
SELECT 
	product_id,
	sum(amount),
	uniq(user_id)
FROM 
	learn_db.orders
GROUP BY 
	product_id;
```

Результат
```text
10000 rows in set. Elapsed: 0.961 sec. Processed 100.00 million rows, 1.60 GB (104.02 million rows/s., 1.66 GB/s.)
Peak memory usage: 343.63 MiB.
```

```sql
SELECT 
	product_id,
	sumMerge(amount),
	uniqMerge(users)
FROM 
	learn_db.orders_agg
GROUP BY 
	product_id;
```

Результат
```text
10000 rows in set. Elapsed: 1.038 sec. Processed 3.66 million rows, 424.56 MB (3.53 million rows/s., 409.09 MB/s.)
Peak memory usage: 334.52 MiB.
```

В документации указано: Использование AggregatingMergeTree оправдано, если оно уменьшает количество строк на порядок.
