# Применение движка VersionedCollapsingMergeTree

### 1. Удаляем и создаем таблицу orders

```sql
DROP TABLE IF EXISTS learn_db.orders;
CREATE TABLE learn_db.orders (
	order_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32,
	sign Int8,
	version UInt32
)
ENGINE = VersionedCollapsingMergeTree(sign, version)
ORDER BY (order_id);
```

### 2. Вставляем данные

```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 100, 1, 1, 1);
```

### 3. Вставляем данные

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 90, 1, -1, 2),
(1, 'created', 80, 1, 1, 3);
```

### 4. Вставляем данные

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 70, 1, 1, 4);
```

### 5. Вставляем данные

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'packed', 70, 1, -1, 5),
(1, 'packed', 60, 1, 1, 6);
```

### 6. Посмотрим на содержимое таблицы orders

```sql
SELECT 
	*, _part
FROM 
	orders;
```

Результат
```text
order_id|status |amount|pcs|sign|version|_part    |
--------+-------+------+---+----+-------+---------+
       1|packed | 70.00|  1|  -1|      5|all_4_4_1|
       1|packed | 60.00|  1|   1|      6|all_4_4_1|
       1|created| 90.00|  1|  -1|      2|all_2_2_1|
       1|created| 80.00|  1|   1|      3|all_2_2_1|
       1|created|100.00|  1|   1|      1|all_1_1_1|
       1|created| 70.00|  1|   1|      4|all_3_3_1|
```

В таблице 6 строк и они распределены по 4 партам.

### 7. Получаем актуальное состояние таблицы заказов

```sql
SELECT 
	order_id,
	status,
	version,
	SUM(amount * sign) AS amount,
	SUM(pcs * sign) AS pcs
FROM 
	orders
GROUP BY
	order_id,
	status,
	version
HAVING 
	SUM(sign) > 0;
```

В запрос был добавлен атрибут version.

Результат
```text
order_id|status |version|amount|pcs|
--------+-------+-------+------+---+
       1|created|      1|100.00|  1|
       1|created|      4| 70.00|  1|
       1|created|      3| 80.00|  1|
       1|packed |      6| 60.00|  1|
```

Получаем 4 актуальные версии строки. Для записи с order_id = 1 мы указали 4 разных состояния, но ни одно из них не отменили.

### 8. Запускаем принудительное слияние всех частей таблицы

```sql
OPTIMIZE TABLE orders FINAL;
```

### 9. Посмотрим на содержимое таблицы orders

```sql
SELECT 
	*, _part
FROM 
	orders;
```

Результат
```text
order_id|status |amount|pcs|sign|version|_part    |
--------+-------+------+---+----+-------+---------+
       1|created|100.00|  1|   1|      1|all_1_4_2|
       1|created| 90.00|  1|  -1|      2|all_1_4_2|
       1|created| 80.00|  1|   1|      3|all_1_4_2|
       1|created| 70.00|  1|   1|      4|all_1_4_2|
       1|packed | 70.00|  1|  -1|      5|all_1_4_2|
       1|packed | 60.00|  1|   1|      6|all_1_4_2|
```

В таблице 6 строк и они распределены по 4 партам.

### 10. Получаем актуальное состояние таблицы заказов

```sql
SELECT 
	order_id,
	status,
	version,
	SUM(amount * sign) AS amount,
	SUM(pcs * sign) AS pcs
FROM 
	orders
GROUP BY
	order_id,
	status,
	version
HAVING 
	SUM(sign) > 0;
```

Результат
```text
order_id|status |version|amount|pcs|
--------+-------+-------+------+---+
       1|created|      1|100.00|  1|
       1|created|      4| 70.00|  1|
       1|created|      3| 80.00|  1|
       1|packed |      6| 60.00|  1|
```

Видим что у нас остались все те же строки.

Добавим недостающие данные.

### 11. Вставляем данные

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 100, 1, -1, 1),
(1, 'created', 90, 1, 1, 2);
```

### 12. Вставляем данные

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 80, 1, -1, 3);
```

### 13. Вставляем данные

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 70, 1, -1, 4),
(1, 'packed', 70, 1, 1, 5);
```

### 14. Посмотрим на содержимое таблицы orders

```sql
SELECT 
	*, _part
FROM 
	orders;
```

Результат
```text
order_id|status |amount|pcs|sign|version|_part    |
--------+-------+------+---+----+-------+---------+
       1|created|100.00|  1|   1|      1|all_1_4_2|
       1|created| 90.00|  1|  -1|      2|all_1_4_2|
       1|created| 80.00|  1|   1|      3|all_1_4_2|
       1|created| 70.00|  1|   1|      4|all_1_4_2|
       1|packed | 70.00|  1|  -1|      5|all_1_4_2|
       1|packed | 60.00|  1|   1|      6|all_1_4_2|
       1|created| 70.00|  1|  -1|      4|all_7_7_1|
       1|packed | 70.00|  1|   1|      5|all_7_7_1|
       1|created|100.00|  1|  -1|      1|all_5_5_1|
       1|created| 90.00|  1|   1|      2|all_5_5_1|
       1|created| 80.00|  1|  -1|      3|all_6_6_1|
```

В таблице 6 строк и они распределены по 4 партам.

### 15. Получаем актуальное состояние таблицы заказов

```sql
SELECT 
	order_id,
	status,
	version,
	SUM(amount * sign) AS amount,
	SUM(pcs * sign) AS pcs
FROM 
	orders
GROUP BY
	order_id,
	status,
	version
HAVING 
	SUM(sign) > 0;
```

Результат
```text
order_id|status|version|amount|pcs|
--------+------+-------+------+---+
       1|packed|      6| 60.00|  1|
```

### 16. Запускаем принудительное слияние всех частей таблицы

```sql
OPTIMIZE TABLE orders FINAL;
```

### 17. Смотрим содержимое таблицы

```sql
SELECT *, _part FROM orders;
```

Результат
```text
order_id|status|amount|pcs|sign|version|_part    |
--------+------+------+---+----+-------+---------+
       1|packed| 60.00|  1|   1|      6|all_1_7_3|
```

### 18. Получаем актуальную строку заказа

```sql
SELECT 
	order_id,
	status,
	version, 
	SUM(amount * sign) AS amount,
	SUM(pcs * sign) AS pcs
FROM 
	orders
GROUP BY
	order_id,
	status,
	version
HAVING 
	SUM(sign) > 0;
```

Результат
```text
order_id|status|version|amount|pcs|
--------+------+-------+------+---+
       1|packed|      6| 60.00|  1|
```
