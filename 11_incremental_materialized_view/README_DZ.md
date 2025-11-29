## Задание 1: Создание исходной таблицы.

Создайте таблицу sales, в которой будут храниться продажи, с полями:
* date (Date) – дата продажи
* product_id (UInt32) – идентификатор товара
* quantity (UInt32) – количество проданных единиц
* price (Decimal(10, 2)) – цена за единицу
* total_amount (Decimal(10, 2)) – общая сумма продажи (quantity * price)

Движок таблицы: MergeTree. Первичный ключ по полю date.

#### Создаем таблицу sales

```sql
DROP TABLE IF EXISTS learn_db.sales;
CREATE TABLE learn_db.sales (
    product_id UInt32,
	quantity UInt32,
    price Decimal(10, 2),
    sales_date Date,
    total_amount Decimal(10, 2) DEFAULT quantity * price,
    PRIMARY KEY(sales_date)
) ENGINE = MergeTree()
ORDER BY (sales_date);
```

## Задание 2: Создание агрегированной таблицы

Создайте таблицу sales_daily_stats, в которой будут автоматически обновляться и хранить агрегированные данные по дням:
* date (Date)
* total_quantity (UInt32) – суммарное количество проданных товаров
* total_revenue (Decimal(10, 2)) – суммарная выручка
* avg_price (Decimal(10, 2)) – средняя цена товара за день

Движок таблицы: AggregatingMergeTree. Первичный ключ по полю date.

#### Создаем агрегирующую таблицу

```sql
DROP TABLE IF EXISTS learn_db.sales_daily_stats;
CREATE TABLE learn_db.sales_daily_stats (
	sales_date Date,
    total_quantity AggregateFunction(sum, UInt32),
    total_revenue AggregateFunction(sum, Decimal(10, 2)),
    avg_price AggregateFunction(avg, Decimal(10, 2)),
	PRIMARY KEY(sales_date)
)
ENGINE = AggregatingMergeTree()
ORDER BY (sales_date);
```

## Задание 3: Создание инкрементального материализованного представления

Создайте инкрементальное материализованное представление sales_daily_stats_mv, которое будет автоматически обновлять агрегированные данные по дням в таблице sales_daily_stats.
Используйте группировку по дням. При агрегации используйте агрегатные функции с суффиксом State.

#### Создаем ИМП

```sql
DROP TABLE learn_db.sales_daily_stats_mv;
CREATE MATERIALIZED VIEW learn_db.sales_daily_stats_mv
TO sales_daily_stats
AS SELECT
    sales_date,
    sumState(quantity) as total_quantity,
    sumState(total_amount) as total_revenue,
    avgState(price) AS avg_price
FROM sales
GROUP BY sales_date;
```

## Задание 4: Добавьте 3 строки в таблицу с продажами sales

Убедитесь, что в таблице sales_daily_stats появились данные.

```sql
INSERT INTO learn_db.sales
(product_id, quantity, price, sales_date, total_amount)
VALUES
(1, 1, 10, '2025-01-01', 10),
(2, 2, 10, '2025-01-01', 20),
(3, 1, 25, '2025-01-01', 50);
```

#### Делаем проверку в таблице sales:

```sql
select * from learn_db.sales;
```

Результат:
```text
product_id|quantity|price|sales_date|total_amount|
----------+--------+-----+----------+------------+
         1|       1|10.00|2025-01-01|       10.00|
         2|       2|10.00|2025-01-01|       20.00|
         3|       1|25.00|2025-01-01|       50.00|
```

#### Делаем проверку в таблице sales_daily_stats:

```sql
SELECT
    sales_date,
    sumMerge(total_quantity) as total_quantity,
    sumMerge(total_revenue) as total_revenue,
    avgMerge(avg_price) as avg_price
FROM learn_db.sales_daily_stats
GROUP BY sales_date;
```

Результат:
```text
sales_date|total_quantity|total_revenue|avg_price|
----------+--------------+-------------+---------+
2025-01-01|             4|        80.00|     15.0|
```

## Задание 5: Сгенерируйте 10 000 строк в таблицу sales.

Убедитесь, что в таблице sales_daily_stats появились данные.

#### Вставляем данные

```sql
INSERT INTO learn_db.sales (sales_date, product_id, quantity, price, total_amount)
SELECT
    today() - randUniform(1, 365) as sales_date,  
    randUniform(1, 100) as product_id,    
    randUniform(1, 10) as quantity,              
    round(randUniform(10, 1000), 2) as price,    
    quantity * price as total_amount
FROM numbers(10000);
```

#### Проверяем

```sql
select count(*) from learn_db.sales;
```

Результат:
```text
count()|
-------+
  10003|
```

## Задание 6: Напишите запрос получения суммы продаж за одну (любую дату)

### 1 вариант запроса: получите сумму продаж за 1 день из таблицы sales_daily_stats

Получаем данные
```sql
SELECT
    sales_date,
    sumMerge(total_quantity) as total_quantity,
    sumMerge(total_revenue) as total_revenue,
    avgMerge(avg_price) as avg_price
FROM learn_db.sales_daily_stats
WHERE sales_date = '2025-01-01'
GROUP BY sales_date;
```

Результат:
```text
Query id: 540c5d80-13e3-4568-b82b-815f2c5f5b87

   ┌─sales_date─┬─total_quantity─┬─total_revenue─┬─────────avg_price─┐
1. │ 2025-01-01 │            170 │      91284.71 │ 447.1039393939394 │
   └────────────┴────────────────┴───────────────┴───────────────────┘

1 row in set. Elapsed: 0.011 sec.
```

### 2 вариант запроса: получите сумму продаж за 1 день из таблицы sales

Получаем данные
```sql
SELECT
    sales_date,
    sum(quantity) as total_quantity,
    sum(total_amount) as total_revenue,
    avg(price) as avg_price
FROM learn_db.sales
WHERE sales_date = '2025-01-01'
GROUP BY sales_date;
```

Результат:
```text
Query id: e515a57d-583e-404d-b33d-33950e50d79a

   ┌─sales_date─┬─total_quantity─┬─total_revenue─┬─────────avg_price─┐
1. │ 2025-01-01 │            170 │      91284.71 │ 447.1039393939394 │
   └────────────┴────────────────┴───────────────┴───────────────────┘

1 row in set. Elapsed: 0.008 sec. Processed 10.00 thousand rows, 220.07 KB (1.31 million rows/s., 28.81 MB/s.)
Peak memory usage: 531.40 KiB.
```