## Задание 1.

Имеется таблица заказов:
```sql
DROP TABLE IF EXISTS learn_db.orders;
CREATE TABLE learn_db.orders (
    order_id UInt32,
    user_id UInt32,
    product_id UInt32,
    amount Decimal(18, 2),
    order_date Date
) ENGINE = MergeTree()
ORDER BY (product_id, order_date);
```

Создайте таблицу orders_stat_agg с движком AggregatingMergeTree, которая будет хранить предварительно агрегированные данные по дням (order_date) и товарам (product_id).
Таблица должна содержать следующие агрегаты:
 * Количество уникальных заказов (uniq(order_id))
 * Минимальная сумма заказа (min(amount))
 * Максимальная сумма заказа (max(amount))
 * Медианная сумма заказа (median(amount))
 * Перцентиль суммы заказа (quantile(amount))

### 1. Создаем таблицу orders с основными данными заказов

```sql
DROP TABLE IF EXISTS learn_db.orders;
CREATE TABLE learn_db.orders (
    order_id UInt32,
    user_id UInt32,
    product_id UInt32,
    amount Decimal(18, 2),
    order_date Date
) ENGINE = MergeTree()
ORDER BY (product_id, order_date);
```

### 2. Создание агрегирующей таблицы

```sql
DROP TABLE IF EXISTS learn_db.orders_stat_agg;
CREATE TABLE learn_db.orders_stat_agg (
    product_id UInt32,  
    order_date Date,    
    uniq_orders AggregateFunction(uniq, UInt32),
    min_amount AggregateFunction(min, Decimal(18, 2)),
    max_amount AggregateFunction(max, Decimal(18, 2)),
    median_amount AggregateFunction(median, Decimal(18, 2)),
    percentile AggregateFunction(quantile(0.9), Decimal(18, 2))
)
ENGINE = AggregatingMergeTree()
ORDER BY (product_id, order_date);
```

## Задание 2.

Цель: сгенерировать 10 000 000 тестовых записей в таблице orders для последующего анализа.

Напишите запрос INSERT, который заполнит таблицу orders случайными данными со следующими характеристиками:
- order_id — уникальный номер заказа (начиная с 10)
- user_id — случайный ID пользователя (от 1 до 1000)
- product_id — случайный ID товара (от 1 до 1000)
- amount — случайная сумма заказа (от 1 до 5000)
- order_date — случайная дата за последний год.

Шаблон запроса:
```sql
INSERT INTO learn_db.orders
SELECT
    number + 10 AS order_id,
    round(randUniform(1, 1000)) AS user_id,
    round(randUniform(1, 1000)) AS product_id,
    randUniform(1, 5000) AS amount,
    date_add(DAY, rand() % 366, today() - INTERVAL 1 YEAR) AS order_date
FROM 
    numbers(10000000);
```

### 3. Наполнение таблицы тестовыми данными

```sql
INSERT INTO learn_db.orders
SELECT
    number + 10 AS order_id,
    round(randUniform(1, 1000)) AS user_id,
    round(randUniform(1, 1000)) AS product_id,
    randUniform(1, 5000) AS amount,
    date_add(DAY, rand() % 366, today() - INTERVAL 1 YEAR) AS order_date
FROM 
    numbers(10000000);
```

Делаем проверку:
```sql
SELECT count() FROM learn_db.orders;
```

Результат:
```text
count() |
--------+
10000000|
```

## Задание 3.

Цель: Научиться загружать предварительно агрегированные данные в таблицу с движком AggregatingMergeTree.

Напишите запрос INSERT, который:
- группирует данные из orders по order_date и product_id
- вычисляет требуемые агрегаты:
  - количество уникальных заказов (uniq(order_id)), 
  - минимальная, максимальная, медианная и 90-й перцентиль суммы (min(amount), max(amount), median(amount), quantile(amount))
- вставляет результат в таблицу orders_stat_agg

Примечание: для AggregatingMergeTree используются -State-функции при вставке и -Merge-функции при выборке.

Убедитесь, что структура orders_stat_agg соответствует агрегируемым полям.

### 4. Вставка агрегированных данных в таблицу orders_stat_agg

```sql
INSERT INTO learn_db.orders_stat_agg
SELECT
	product_id,
    order_date,
    uniqState(order_id) as uniq_orders,
    minState(amount) as min_amount,
    maxState(amount) as max_amount,
    medianState(amount) as median_amount,
    quantileState(0.9)(amount) as percentile
FROM 
	learn_db.orders
GROUP BY 
	product_id,
	order_date;
```

### 5. Анализ агрегированных данных

```sql
SELECT
    order_date,
    product_id,
    minMerge(min_amount) as min_amount,
    medianMerge(median_amount) as median_amount,
    quantileMerge(0.9)(percentile) as percentile
FROM learn_db.orders_stat_agg
GROUP BY order_date, product_id
ORDER BY order_date, product_id;
```

Результат:
```text
Query id: a36ae1eb-cb27-4c81-949c-af2a787c1f12

        ┌─order_date─┬─product_id─┬─min_amount─┬─median_amount─┬─percentile─┐
     1. │ 2024-09-24 │          1 │     879.31 │       1928.41 │    3442.54 │
     2. │ 2024-09-24 │          2 │     951.06 │        2443.3 │    4447.99 │
     3. │ 2024-09-24 │          3 │      17.27 │       1805.74 │    3631.48 │
     4. │ 2024-09-24 │          4 │     238.56 │       1940.97 │    4588.84 │
     5. │ 2024-09-24 │          5 │     535.76 │       2870.66 │    4604.15 │
365999. │ 2025-09-24 │        999 │     481.48 │        3122.7 │     4394.3 │
366000. │ 2025-09-24 │       1000 │       86.3 │       2354.72 │    3040.66 │
        └─order_date─┴─product_id─┴─min_amount─┴─median_amount─┴─percentile─┘
Showed 1000 out of 366000 rows.

366000 rows in set. Elapsed: 0.699 sec. Processed 366.00 thousand rows, 104.68 MB (523.69 thousand rows/s., 149.77 MB/s.)
Peak memory usage: 447.98 MiB.
```

EXPLAIN для проверки эффективности:
```sql
EXPLAIN indexes = 1
SELECT
    order_date,
    product_id,
    minMerge(min_amount) as min_amount,
    medianMerge(median_amount) as median_amount,
    quantileMerge(0.9)(percentile) as percentile
FROM learn_db.orders_stat_agg
WHERE order_date >= '2024-01-01'
GROUP BY order_date, product_id;
```

Результат:
```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Aggregating                                       |
    Expression (Before GROUP BY)                    |
      Expression                                    |
        ReadFromMergeTree (learn_db.orders_stat_agg)|
        Indexes:                                    |
          PrimaryKey                                |
            Keys:                                   |
              order_date                            |
            Condition: (order_date in [19723, +Inf))|
            Parts: 1/1                              |
            Granules: 45/45                         |
```

## Задание 2: Оптимизация и анализ производительности
Цель: сгенерировать 10 000 000 тестовых записей в таблице orders для последующего анализа.

Напишите запрос INSERT, который заполнит таблицу orders случайными данными со следующими характеристиками:
- order_id — уникальный номер заказа (начиная с 10)
- user_id — случайный ID пользователя (от 1 до 1000)
- product_id — случайный ID товара (от 1 до 1000)
- amount — случайная сумма заказа (от 1 до 5000)
- order_date — случайная дата за последний год

Шаблон запроса:
```sql
INSERT INTO learn_db.orders
SELECT
    number + 10 AS order_id,
    round(randUniform(1, 1000)) AS user_id,
    round(randUniform(1, 1000)) AS product_id,
    randUniform(1, 5000) AS amount,
    date_add(DAY, rand() % 366, today() - INTERVAL 1 YEAR) AS order_date
FROM 
    numbers(10000000);
```
Проверка: убедитесь, что таблица содержит 10 млн строк.

### 6. Первый запрос – вычисление агрегатов напрямую из orders.

```sql
SELECT
    order_date,
    product_id,
    median(amount) as median_amount,
    quantile(0.9)(amount) as percentile
FROM learn_db.orders
GROUP BY order_date, product_id;
```

Результат:
```text
Query id: d3d65b5a-2be3-4618-a457-c55d730456db

        ┌─order_date─┬─product_id─┬─median_amount─┬─percentile─┐
     1. │ 2025-05-19 │        401 │       3018.63 │    4489.75 │
     2. │ 2025-01-28 │        207 │       2671.59 │    4199.57 │
     3. │ 2025-04-05 │        796 │       2604.86 │    4559.78 │
     4. │ 2025-07-19 │        578 │       2606.15 │    4730.49 │
     5. │ 2024-11-20 │        976 │       2516.22 │    4447.19 │
366000. │ 2025-04-12 │        472 │       2930.16 │    4889.61 │
        └─order_date─┴─product_id─┴─median_amount─┴─percentile─┘
Showed 1000 out of 366000 rows.

366000 rows in set. Elapsed: 1.182 sec. Processed 10.00 million rows, 140.00 MB (8.46 million rows/s., 118.48 MB/s.)
Peak memory usage: 693.58 MiB.
```

### 7. Второй запрос – получение данных из orders_stat_agg.

```sql
SELECT
    medianMerge(median_amount) as median_amount,
    quantileMerge(0.9)(percentile) as p90_amount
FROM learn_db.orders_stat_agg
GROUP BY order_date, product_id
ORDER BY order_date, product_id;
```

Результат:
```text
Query id: 90dcf1cb-8c0a-47bd-88ae-1d33f10467f5

        ┌─median_amount─┬─p90_amount─┐
     1. │       1928.41 │    3442.54 │
     2. │        2443.3 │    4447.99 │
     3. │       1805.74 │    3631.48 │
     4. │       1940.97 │    4588.84 │
     5. │       2870.66 │    4604.15 │
366000. │       2354.72 │    3040.66 │
        └─median_amount─┴─p90_amount─┘
Showed 1000 out of 366000 rows.

366000 rows in set. Elapsed: 0.562 sec. Processed 366.00 thousand rows, 95.89 MB (651.82 thousand rows/s., 170.78 MB/s.)
Peak memory usage: 363.20 MiB.
```

### 8. Выводы
**Время выполнения:** 1.182 сек vs. 0.562 сек -> агрегированный запрос в 2 раза быстрее.
**Количество обработанных строк:** 10 000 000 vs. 366 000 -> в 27 раз меньше.
**Пик памяти:** 693.58 MiB vs. 363.20 MiB -> почти в 2 раза меньше.
**Скорость чтения:** 8.46 млн. строк/с vs. 651 тыс. строк/с (но данных гораздо меньше).
