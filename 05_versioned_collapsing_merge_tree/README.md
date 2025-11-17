# Применение движка VersionedCollapsingMergeTree

### 1. Удаляем и создаем таблицу orders

```sql
DROP TABLE IF EXISTS orders;
CREATE TABLE orders (
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
(1, 'created', 100, 1, -1, 1),
(1, 'created', 90, 1, 1, 2);
```

### 4. Вставляем данные

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 90, 1, -1, 2),
(1, 'created', 80, 1, 1, 3);
```

### 5. Вставляем данные

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 80, 1, -1, 3);
```

### 6. Вставляем данные

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 70, 1, 1, 4);
```

### 7. Вставляем данные

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 70, 1, -1, 4),
(1, 'packed', 70, 1, 1, 5);
```

### 8. Вставляем данные

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'packed', 70, 1, -1, 5),
(1, 'packed', 60, 1, 1, 6);
```

### 9. Запускаем принудительное слияние всех частей таблицы

```sql
OPTIMIZE TABLE orders FINAL;
```

### 10. Смотрим содержимое таблицы

```sql
SELECT *, _part FROM orders;
```

### 11. Получаем актуальную строку заказа

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
order_id|status|amount|pcs|sign|version|_part    |
--------+------+------+---+----+-------+---------+
       1|packed| 60.00|  1|   1|      6|all_1_7_3|
```
