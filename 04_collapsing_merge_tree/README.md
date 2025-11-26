# Применение движка CollapsingMergeTree

### 1. Удаляем и создаем таблицу orders

```sql
DROP TABLE IF EXISTS orders;

CREATE TABLE learn_db.orders (
	order_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32,
	sign Int8
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY (order_id);
```

### 2. Вставляем заказ с номером 1

```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', 100, 1, 1);
```

### 3. Правим сумму заказа со 100 до 90

```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', 100, 1, -1),
(1, 'created', 90, 1, 1);
```

1я строка - отмена. У нее sign = -1. 

### 4. Правим сумму заказа со 90 до 80

```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', 90, 1, -1),
(1, 'created', 80, 1, 1);
```

### 5. Получаем актуальную строку заказа

```sql
SELECT 
	order_id,
	status,
	SUM(amount * sign) AS amount,
	SUM(pcs * sign) AS pcs
FROM 
	orders
GROUP BY
	order_id,
	status
HAVING 
	SUM(sign) > 0;
```

Результат
```text
order_id|status |amount|pcs|
--------+-------+------+---+
       1|created| 80.00|  1|
```

Но при этом если мы выполним запрос ко всей таблице orders, то получим следующее:

```sql
SELECT 
	*, _part
FROM 
	orders;
```

Результат
```text
order_id|status |amount|pcs|sign|_part    |
--------+-------+------+---+----+---------+
       1|created| 90.00|  1|  -1|all_3_3_1|
       1|created| 80.00|  1|   1|all_3_3_1|
       1|created|100.00|  1|  -1|all_2_2_1|
       1|created| 90.00|  1|   1|all_2_2_1|
       1|created|100.00|  1|   1|all_1_1_1|
```

В таблице 5 строк. При этом данные распределены по трем партам.

### 6. Меняем сумму заказа с 80 до 70 с помощью двух отдельных запросов

```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', 80, 1, -1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', 70, 1, 1);
```

Хотя желательно делать отмену и вставлять новое значение одновременно. В момент между выполнением обоих запросов строка будет неактивной, а новая еще не добавится.

### 7. Смотрим строки таблицы orders

```sql
SELECT *, _part FROM orders;
```

Результат
```text
order_id|status |amount|pcs|sign|_part    |
--------+-------+------+---+----+---------+
       1|created| 90.00|  1|  -1|all_3_3_1|
       1|created| 80.00|  1|   1|all_3_3_1|
       1|created|100.00|  1|   1|all_1_1_1|
       1|created|100.00|  1|  -1|all_2_2_1|
       1|created| 90.00|  1|   1|all_2_2_1|
       1|created| 70.00|  1|   1|all_5_5_1|
       1|created| 80.00|  1|  -1|all_4_4_1|
```

Проверим содержимое таблицы orders.

```sql
SELECT 
	*, _part
FROM 
	orders;
```

Результат
```text
order_id|status |amount|pcs|sign|_part    |
--------+-------+------+---+----+---------+
       1|created| 70.00|  1|   1|all_1_5_2|
```

Видим что она схлопнулась. Произошел merge. И после этого осталась одна строка.

### 8. Меняем статус заказа

```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', 70, 1, -1),
(1, 'packed', 70, 1, 1);
```

Проверим актуальное состояние заказа

```sql
SELECT 
	order_id,
	status,
	SUM(amount * sign) AS amount,
	SUM(pcs * sign) AS pcs
FROM 
	orders
GROUP BY
	order_id,
	status
HAVING 
	SUM(sign) > 0;
```

Результат
```text
order_id|status|amount|pcs|
--------+------+------+---+
       1|packed| 70.00|  1|
```

### 9. Удаляем заказа с номером 1

```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'packed', 70, 1, -1);
```

Проверим актуальное состояние заказа

```sql
SELECT 
	order_id,
	status,
	SUM(amount * sign) AS amount,
	SUM(pcs * sign) AS pcs
FROM 
	orders
GROUP BY
	order_id,
	status
HAVING 
	SUM(sign) > 0;
```

Результат
```text
order_id|status|amount|pcs|
--------+------+------+---+
```

Посмотрим на содержимое таблицы orders

```sql
SELECT 
	*, _part
FROM 
	orders;
```

Результат
```text
order_id|status |amount|pcs|sign|_part    |
--------+-------+------+---+----+---------+
       1|created| 70.00|  1|  -1|all_6_6_1|
       1|packed | 70.00|  1|   1|all_6_6_1|
       1|packed | 70.00|  1|  -1|all_7_7_1|
       1|created| 70.00|  1|   1|all_1_5_2|
```

Видим что merge еще не выполнился.

### 10. Удаляем и создаем таблицу orders заново. Тип поля pcs изменен на Int32

```sql
DROP TABLE IF EXISTS orders;
CREATE TABLE orders (
	order_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs Int32,
	sign Int8
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY (order_id);
```

В поле pcs выберем тип Int32 (можно хранить отрицательные числа).

### 11. Добавляем строку заказа и 2 раза меняем сумму заказа

```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', 100, 1, 1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', -100, -1, -1),
(1, 'created', 90, 1, 1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', -90, -1, -1),
(1, 'created', 80, 1, 1);
```

Когда мы хотим изменить сумму со 100 до 90 мы так же добавляем 2 строки, но у нас есть изменение в строке отмены состояния.
В поле amount мы пишем -100 (а не 100), в pcs -1 (а не 1). То есть фактически мы эти значения умножили на -1. При изменении цена на 80 делаем аналогичную вставку.

### 12. Получаем актуальную строку заказа

Запрос получения актуального состояния немного меняется (убирается умножение двух атрибутов из суммы). 

```sql
SELECT 
	order_id,
	status,
	SUM(amount) AS amount,
	SUM(pcs) AS pcs
FROM 
	orders
GROUP BY
	order_id,
	status
HAVING 
	SUM(sign) > 0;
```

Результат
```text
order_id|status |amount|pcs|
--------+-------+------+---+
       1|created| 80.00|  1|
```

Вычислений получается чуть меньше.

### 13. Смотрим на строки таблицы orders

```sql
SELECT *, _part  FROM orders;
```

Результат
```text
order_id|status |amount |pcs|sign|_part    |
--------+-------+-------+---+----+---------+
       1|created|-100.00| -1|  -1|all_2_2_1|
       1|created|  90.00|  1|   1|all_2_2_1|
       1|created| 100.00|  1|   1|all_1_1_1|
       1|created| -90.00| -1|  -1|all_3_3_1|
       1|created|  80.00|  1|   1|all_3_3_1|
```

### 14. Получаем актуальные строки таблицы orders с применением FINAL

```sql
SELECT * FROM orders FINAL;
```

Результат
```text
order_id|status |amount|pcs|sign|
--------+-------+------+---+----+
       1|created| 80.00|  1|   1|
```

### 15. Считаем количество актуальных строк в таблице orders

```sql
SELECT 
	SUM(sign)
FROM 
	orders;
```

Результат
```text
SUM(sign)|
---------+
        1|
```

### 16. Принудительно оставляем для каждого заказа только одну строку (нежелательная операция)

Смотрим на строки таблицы orders

```sql
SELECT *, _part  FROM orders;
```

Результат
```text
order_id|status |amount |pcs|sign|_part    |
--------+-------+-------+---+----+---------+
       1|created|-100.00| -1|  -1|all_2_2_1|
       1|created|  90.00|  1|   1|all_2_2_1|
       1|created| -90.00| -1|  -1|all_3_3_1|
       1|created|  80.00|  1|   1|all_3_3_1|
       1|created| 100.00|  1|   1|all_1_1_1|
```

Строки распределены по трем партам.

Выполним следующий запрос

```sql
OPTIMIZE TABLE orders FINAL;
```

Проверим содержимое таблицы orders

```sql
SELECT *, _part  FROM orders;
```

Результат
```text
order_id|status |amount|pcs|sign|_part    |
--------+-------+------+---+----+---------+
       1|created| 80.00|  1|   1|all_1_3_2|
```

Видим что осталась одна строка и один актуальный парт. Но такая операция нежелательна, т.к. на больших таблицах она может очень сильно нагрузить БД и другие операции могут пострадать из-за этого.