## Описание

Создайте MergeTree-таблицу orders с полями: 
- order_id UInt64, 
- user_id UInt32, 
- order_date Date, 
- amount Float64,
- status String.

Примените такой первичный ключ, который даст максимальную скорость для следующего запроса:

```sql
SELECT * FROM orders WHERE order_id = 12345
```

2. Наполните таблицу 1000 000 строк сгенерированных данных (см примеры из предыдущих уроков).

3. Получите план выполнения запроса с отображением статистики применения индекса 

```sql
SELECT * FROM orders WHERE order_id = 12345
```

Укажите какое количества гранул было отфильтровано благодаря индексу.

4. Создайте таблицу orders_status_order_id, которая будет выдавать высокую скорость с фильтрацией по status или с фильтрацией по order_id.
Наполните таблицу 1000 000 строк из таблицы orders.

5. С помощью утилиты clickhouse-benchmark замерьте скорость выполнения 2-х запросов:

```sql
select sum(amount) from orders_status_order_id where status = '[указать какое-нибудь значение статуса]'

select sum(amount) from orders_status_order_id where order_id = 1
```

5. Дан запрос:

```sql
SELECT status, avg(amount), count(user_id) FROM orders WHERE order_date >= ... AND order_date < ... GROUP BY status
```

Напишите 2-3 вариант скрипта создания таблицы orders с разными первичными ключами и сравните скорость выполнения запросов. Определите первичный ключ, дающий максимальную скорость запроса.
Используйте составные ключи, состоящие из нескольких полей.

### 1. Создаем MergeTree-таблицу и наполняем ее данными

```sql
DROP TABLE IF EXISTS learn_db.orders;
CREATE TABLE learn_db.orders
(
    order_id UInt64,
    user_id UInt32,
    order_date Date,
    amount Float64,
    status String,
    PRIMARY KEY (order_id)
) ENGINE = MergeTree()
ORDER BY (order_id, order_date)
AS SELECT
    number as order_id,                     
    (number % 50000) + 1 as user_id,         
    cast(now() - randUniform(0, 365*24*60*60) as date) as order_date,
    round(randUniform(1, 1000), 2) as amount,
    CASE floor(randUniform(1, 5))
        WHEN 1 THEN 'создан'
        WHEN 2 THEN 'оплачен'
        WHEN 3 THEN 'выдан'
        WHEN 4 THEN 'возвращен'
        ELSE 'отменен'
    END as status
FROM numbers(1000000);
```

### 2. Получаем план выполнения запроса с отображением статистики применения индекса 

```sql
EXPLAIN indexes = 1
SELECT * FROM orders WHERE order_id = 12345
```

Результат
```text
explain                                        |
-----------------------------------------------+
Expression ((Project names + Projection))      |
  Expression                                   |
    ReadFromMergeTree (learn_db.orders)      |
    Indexes:                                   |
      PrimaryKey                               |
        Keys:                                  |
          order_id                             |
        Condition: (order_id in [12345, 12345])|
        Parts: 1/1                             |
        Granules: 1/123                        |
```

Была отфильтрована одна гранула. Время выполнения запроса - 0.008s

### 3. Создаем таблицу orders_status_order_id, которая будет выдавать высокую скорость с фильтрацией по status или с фильтрацией по order_id.

```sql
DROP TABLE IF EXISTS learn_db.orders_status_order_id;
CREATE TABLE learn_db.orders_status_order_id
(
    order_id UInt64,
    user_id UInt32,
    order_date Date,
    amount Float64,
    status String,
    PRIMARY KEY (status)
) ENGINE = MergeTree()
ORDER BY (status)
AS SELECT
    order_id,       
    user_id,        
    order_date,
    amount,
    status
FROM orders;
```

### 4. С помощью утилиты clickhouse-benchmark замерьте скорость выполнения 2-х запросов:

Запрос 1.
```text
clickhouse-benchmark --query "select sum(amount) from orders_status_order_id where status = 'отменен'" --iterations 1

 root@19294dc11be6:/# clickhouse-benchmark --query "select sum(amount) from learn_db.orders_status_order_id where status = 'отменен'" --iterations 1
Loaded 1 queries.

Queries executed: 1.

localhost:9000, queries: 1, QPS: 8.138, RPS: 0.000, MiB/s: 0.000, result RPS: 8.138, result MiB/s: 0.000.

0%              0.004 sec.
10%             0.004 sec.
20%             0.004 sec.
30%             0.004 sec.
40%             0.004 sec.
50%             0.004 sec.
60%             0.004 sec.
70%             0.004 sec.
80%             0.004 sec.
90%             0.004 sec.
95%             0.004 sec.
99%             0.004 sec.
99.9%           0.004 sec.
99.99%          0.004 sec.
```

Запрос 2.
```text
root@19294dc11be6:/# clickhouse-benchmark --query "select sum(amount) from learn_db.orders_status_order_id where order_id = 1" --iterations 1
Loaded 1 queries.

Queries executed: 1.

localhost:9000, queries: 1, QPS: 8.337, RPS: 478065.861, MiB/s: 3.679, result RPS: 8.337, result MiB/s: 0.000.

0%              0.004 sec.
10%             0.004 sec.
20%             0.004 sec.
30%             0.004 sec.
40%             0.004 sec.
50%             0.004 sec.
60%             0.004 sec.
70%             0.004 sec.
80%             0.004 sec.
90%             0.004 sec.
95%             0.004 sec.
99%             0.004 sec.
99.9%           0.004 sec.
99.99%          0.004 sec.
```

### 5. Эксперименты с производительностью запроса

```sql
SELECT status, avg(amount), count(user_id) FROM orders WHERE order_date >= ... AND order_date < ... GROUP BY status
```

### 5.1. Создаем таблицу с первичным ключом по полю order_date

```sql
DROP TABLE IF EXISTS learn_db.orders_1;
CREATE TABLE learn_db.orders_1
(
    order_id UInt64,
    user_id UInt32,
    order_date Date,
    amount Float64,
    status String,
    PRIMARY KEY (order_date)
) ENGINE = MergeTree()
ORDER BY (order_date)
AS SELECT
    number as order_id,
    (number % 50000) + 1 as user_id,
    cast(now() - randUniform(0, 365*24*60*60) as date) as order_date,
    round(randUniform(1, 1000), 2) as amount,
    CASE floor(randUniform(1, 5))
        WHEN 1 THEN 'создан'
        WHEN 2 THEN 'оплачен'
        WHEN 3 THEN 'выдан'
        WHEN 4 THEN 'возвращен'
        ELSE 'отменен'
    END as status
FROM numbers(1000000);
```

Количество заказов за последнюю неделю сентября 2024 г. - 20284 записей

```sql
select count(*)
from learn_db.orders_1
WHERE order_date >= '2024-09-23' AND order_date < '2024-10-01'
```

Выполнение агрегирующего запроса:
```sql
SELECT status, avg(amount), count(user_id) FROM orders_1 WHERE order_date >= '2024-09-23' AND order_date < '2024-10-01' GROUP BY status -- 0.016s
```

Результат
```text
status   |avg(amount)       |count(user_id)|
---------+------------------+--------------+
оплачен  | 505.6402646129547|          5064|
выдан    | 501.0741528762796|          5076|
создан   |497.15714986269194|          5098|
возвращен| 506.4148929845428|          5046|
```

Получаем план выполнения запроса с отображением статистики применения индекса 
```sql
EXPLAIN indexes = 1
SELECT status, avg(amount), count(user_id) FROM orders_1 WHERE order_date >= '2024-09-23' AND order_date < '2024-10-01' GROUP BY status
```

Результат
```text
explain                                                                                 |
----------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                               |
  Aggregating                                                                           |
    Expression (Before GROUP BY)                                                        |
      Expression                                                                        |
        ReadFromMergeTree (learn_db.orders_1)                                           |
        Indexes:                                                                        |
          PrimaryKey                                                                    |
            Keys:                                                                       |
              order_date                                                                |
            Condition: and((order_date in (-Inf, 19996]), (order_date in [19989, +Inf)))|
            Parts: 1/1                                                                  |
            Granules: 3/123                                                             |
```

### 5.2. Создаем таблицу с первичным ключом по полю order_date и status

```sql
DROP TABLE IF EXISTS learn_db.orders_2;
CREATE TABLE learn_db.orders_2
(
    order_id UInt64,
    user_id UInt32,
    order_date Date,
    amount Float64,
    status String,
    PRIMARY KEY (order_date, status)
) ENGINE = MergeTree()
ORDER BY (order_date, status)
AS SELECT
    number as order_id,                     
    (number % 50000) + 1 as user_id,    
    cast(now() - randUniform(0, 365*24*60*60) as date) as order_date,
    round(randUniform(1, 1000), 2) as amount,
    CASE floor(randUniform(1, 5))
        WHEN 1 THEN 'создан'
        WHEN 2 THEN 'оплачен'
        WHEN 3 THEN 'выдан'
        WHEN 4 THEN 'возвращен'
        ELSE 'отменен'
    END as status
FROM numbers(1000000);
```

Выполнение запроса - 0.013s

```sql
SELECT status, avg(amount), count(user_id) FROM learn_db.orders_2 WHERE order_date >= '2024-09-23' AND order_date < '2024-10-01' GROUP BY status
```

Получаем план выполнения запроса с отображением статистики применения индекса 

```sql
EXPLAIN indexes = 1
SELECT status, avg(amount), count(user_id) FROM orders_2 WHERE order_date >= '2024-09-23' AND order_date < '2024-10-01' GROUP BY status
```

Результат
```text
explain                                                                                 |
----------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                               |
  Aggregating                                                                           |
    Expression (Before GROUP BY)                                                        |
      Expression                                                                        |
        ReadFromMergeTree (learn_db.orders_2)                                           |
        Indexes:                                                                        |
          PrimaryKey                                                                    |
            Keys:                                                                       |
              order_date                                                                |
            Condition: and((order_date in (-Inf, 19996]), (order_date in [19989, +Inf)))|
            Parts: 1/1                                                                  |
            Granules: 3/123                                                             |                                                        |
```

Проверим производительность

```text
root@19294dc11be6:/# clickhouse-benchmark --query "SELECT status, avg(amount), count(user_id) FROM learn_db.orders_2 WHERE order_date >= '2024-09-23' AND order_date < '2024-10-01' GROUP BY status" --iterations
 10
Loaded 1 queries.

Queries executed: 10.

localhost:9000, queries: 10, QPS: 49.672, RPS: 1220729.756, MiB/s: 42.529, result RPS: 198.686, result MiB/s: 0.007.

0%              0.004 sec.
10%             0.004 sec.
20%             0.005 sec.
30%             0.006 sec.
40%             0.006 sec.
50%             0.007 sec.
60%             0.007 sec.
70%             0.007 sec.
80%             0.007 sec.
90%             0.009 sec.
95%             0.010 sec.
99%             0.010 sec.
99.9%           0.010 sec.
99.99%          0.010 sec.
```

### 5.3. Создаем таблицу с первичным ключом по полю order_date и status

```sql
DROP TABLE IF EXISTS learn_db.orders_3;
CREATE TABLE learn_db.orders_3
(
    order_id UInt64,
    user_id UInt32,
    order_date Date,
    amount Float64,
    status String,
    PRIMARY KEY (order_id, order_date, status)
) ENGINE = MergeTree()
ORDER BY (order_id, order_date, status)
AS SELECT
    number as order_id,                     
    (number % 50000) + 1 as user_id,    
    cast(now() - randUniform(0, 365*24*60*60) as date) as order_date,
    round(randUniform(1, 1000), 2) as amount,
    CASE floor(randUniform(1, 5))
        WHEN 1 THEN 'создан'
        WHEN 2 THEN 'оплачен'
        WHEN 3 THEN 'выдан'
        WHEN 4 THEN 'возвращен'
        ELSE 'отменен'
    END as status
FROM numbers(1000000);
```

Выполнение запроса - 0.034s

```sql
SELECT status, avg(amount), count(user_id) FROM learn_db.orders_3 WHERE order_date >= '2024-09-23' AND order_date < '2024-10-01' GROUP BY status
```

Получаем план выполнения запроса с отображением статистики применения индекса 

```sql
EXPLAIN indexes = 1
SELECT status, avg(amount), count(user_id) FROM orders_3 WHERE order_date >= '2024-09-23' AND order_date < '2024-10-01' GROUP BY status
```

Результат
```text
explain                                                                                 |
----------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                               |
  Aggregating                                                                           |
    Expression (Before GROUP BY)                                                        |
      Expression                                                                        |
        ReadFromMergeTree (learn_db.orders_3)                                           |
        Indexes:                                                                        |
          PrimaryKey                                                                    |
            Keys:                                                                       |
              order_date                                                                |
            Condition: and((order_date in (-Inf, 19996]), (order_date in [19989, +Inf)))|
            Parts: 1/1                                                                  |
            Granules: 123/123                                                           |
```

### 5.4. Создаем таблицу с первичным ключом по полю order_date и status и группировкой order_date, status, user_id

```sql
DROP TABLE IF EXISTS learn_db.orders_4;
CREATE TABLE learn_db.orders_4
(
    order_id UInt64,
    user_id UInt32,
    order_date Date,
    amount Float64,
    status String,
    PRIMARY KEY (order_date, status)
) ENGINE = MergeTree()
ORDER BY (order_date, status, user_id)
AS SELECT
    number as order_id,                     
    (number % 50000) + 1 as user_id,    
    cast(now() - randUniform(0, 365*24*60*60) as date) as order_date,
    round(randUniform(1, 1000), 2) as amount,
    CASE floor(randUniform(1, 5))
        WHEN 1 THEN 'создан'
        WHEN 2 THEN 'оплачен'
        WHEN 3 THEN 'выдан'
        WHEN 4 THEN 'возвращен'
        ELSE 'отменен'
    END as status
FROM numbers(1000000);
```

Выполнение запроса - 0.015s

```sql
SELECT status, avg(amount), count(user_id) FROM learn_db.orders_4 WHERE order_date >= '2024-09-23' AND order_date < '2024-10-01' GROUP BY status 
```

Получаем план выполнения запроса с отображением статистики применения индекса 

```sql
EXPLAIN indexes = 1
SELECT status, avg(amount), count(user_id) FROM orders_4 WHERE order_date >= '2024-09-23' AND order_date < '2024-10-01' GROUP BY status
```

Результат
```text
explain                                                                                 |
----------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                               |
  Aggregating                                                                           |
    Expression (Before GROUP BY)                                                        |
      Expression                                                                        |
        ReadFromMergeTree (learn_db.orders_4)                                           |
        Indexes:                                                                        |
          PrimaryKey                                                                    |
            Keys:                                                                       |
              order_date                                                                |
            Condition: and((order_date in (-Inf, 19996]), (order_date in [19989, +Inf)))|
            Parts: 1/1                                                                  |
            Granules: 3/123                                                             |                                                        |
```

Проверим производительность

```text
root@19294dc11be6:/# clickhouse-benchmark --query "SELECT status, avg(amount), count(user_id) FROM learn_db.orders_4 WHERE order_date >= '2024-09-23' AND order_date < '2024-10-01' GROUP BY status" --iterations
 10
Loaded 1 queries.

Queries executed: 10.

localhost:9000, queries: 10, QPS: 52.438, RPS: 1288719.665, MiB/s: 44.905, result RPS: 209.753, result MiB/s: 0.008.

0%              0.004 sec.
10%             0.005 sec.
20%             0.005 sec.
30%             0.005 sec.
40%             0.005 sec.
50%             0.005 sec.
60%             0.005 sec.
70%             0.006 sec.
80%             0.006 sec.
90%             0.007 sec.
95%             0.008 sec.
99%             0.008 sec.
99.9%           0.008 sec.
99.99%          0.008 sec.
```

Между выполнениями теста в clickhouse-benchmark запросов на таблиц learn_db.orders_2 и learn_db.orders_4 я запускал другие запросы, чтобы не было кэширования.
У orders_4 QPS выше.


### 5.5. Создаем таблицу с первичным ключом по полю order_date и status и группировкой order_date, status, user_id

```sql
DROP TABLE IF EXISTS learn_db.orders_5;
CREATE TABLE learn_db.orders_5
(
    order_id UInt64,
    user_id UInt32,
    order_date Date,
    amount Float64,
    status String,
    PRIMARY KEY (order_date, status)
) ENGINE = MergeTree()
ORDER BY (order_date, status, user_id)
PARTITION BY toYYYYMM(order_date)
AS SELECT
    number as order_id,                     
    (number % 50000) + 1 as user_id,    
    cast(now() - randUniform(0, 365*24*60*60) as date) as order_date,
    round(randUniform(1, 1000), 2) as amount,
    CASE floor(randUniform(1, 5))
        WHEN 1 THEN 'создан'
        WHEN 2 THEN 'оплачен'
        WHEN 3 THEN 'выдан'
        WHEN 4 THEN 'возвращен'
        ELSE 'отменен'
    END as status
FROM numbers(1000000);
```

Выполнение запроса - 0.013s

```sql
SELECT status, avg(amount), count(user_id) FROM learn_db.orders_5 WHERE order_date >= '2024-09-23' AND order_date < '2024-10-01' GROUP BY status 
```

Получаем план выполнения запроса с отображением статистики применения индекса 
```sql
EXPLAIN indexes = 1
SELECT status, avg(amount), count(user_id) FROM orders_5 WHERE order_date >= '2024-09-23' AND order_date < '2024-10-01' GROUP BY status
```

Результат
```text
explain                                                                                                       |
--------------------------------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                                                     |
  Aggregating                                                                                                 |
    Expression (Before GROUP BY)                                                                              |
      Expression                                                                                              |
        ReadFromMergeTree (learn_db.orders_5)                                                                 |
        Indexes:                                                                                              |
          MinMax                                                                                              |
            Keys:                                                                                             |
              order_date                                                                                      |
            Condition: and((order_date in (-Inf, 19996]), (order_date in [19989, +Inf)))                      |
            Parts: 1/13                                                                                       |
            Granules: 2/119                                                                                   |
          Partition                                                                                           |
            Keys:                                                                                             |
              toYYYYMM(order_date)                                                                            |
            Condition: and((toYYYYMM(order_date) in (-Inf, 202410]), (toYYYYMM(order_date) in [202409, +Inf)))|
            Parts: 1/1                                                                                        |
            Granules: 2/2                                                                                     |
          PrimaryKey                                                                                          |
            Keys:                                                                                             |
              order_date                                                                                      |
            Condition: and((order_date in (-Inf, 19996]), (order_date in [19989, +Inf)))                      |
            Parts: 1/1                                                                                        |
            Granules: 2/2                                                                                     |                                                         |                                                        |
```

Проверим производительность

```text
root@19294dc11be6:/# clickhouse-benchmark --query "SELECT status, avg(amount), count(user_id) FROM learn_db.orders_5 WHERE order_date >= '2024-09-23' AND order_date < '2024-10-01' GROUP BY status" --iterations
 10
Loaded 1 queries.

Queries executed: 10.

localhost:9000, queries: 10, QPS: 53.958, RPS: 1073380.298, MiB/s: 37.369, result RPS: 215.831, result MiB/s: 0.008.

0%              0.004 sec.
10%             0.004 sec.
20%             0.004 sec.
30%             0.004 sec.
40%             0.005 sec.
50%             0.006 sec.
60%             0.006 sec.
70%             0.006 sec.
80%             0.007 sec.
90%             0.008 sec.
95%             0.008 sec.
99%             0.008 sec.
99.9%           0.008 sec.
99.99%          0.008 sec.
```

С применением партиций запрос к learn_db.orders_5 получается эффективнее: 2 гранулы из 2 плюс QPS: 53.958 (выше чем у learn_db.orders_4).