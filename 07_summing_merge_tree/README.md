# Применение движка SummingMergeTree

### 1. Удаляем и создаем таблицу orders с сырыми данными 

```sql
DROP TABLE IF EXISTS learn_db.orders;
CREATE TABLE learn_db.orders (
	order_id UInt32,
	customer_id UInt32,
	sale_dt Date,
	product_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32
)
ENGINE = MergeTree()
ORDER BY (sale_dt, product_id, customer_id, order_id);
```

### 2. Вставляем данные в таблицу с сырыми данными

```sql
INSERT INTO learn_db.orders
(order_id, customer_id, sale_dt, product_id, status, amount, pcs)
VALUES
(1, 1, '2025-06-01', 1, 'successed', 100, 1),
(1, 1, '2025-06-01', 2, 'successed', 50, 2);

INSERT INTO learn_db.orders
(order_id, customer_id, sale_dt, product_id, status, amount, pcs)
VALUES
(2, 1, '2025-06-01', 1, 'canceled', 110, 1),
(2, 1, '2025-06-01', 2, 'canceled', 60, 2);

INSERT INTO learn_db.orders
(order_id, customer_id, sale_dt, product_id, status, amount, pcs)
VALUES
(3, 2, '2025-06-01', 1, 'successed', 100, 1);

INSERT INTO learn_db.orders
(order_id, customer_id, sale_dt, product_id, status, amount, pcs)
VALUES
(4, 3, '2025-06-01', 2, 'successed', 25, 1);

INSERT INTO learn_db.orders
(order_id, customer_id, sale_dt, product_id, status, amount, pcs)
VALUES
(5, 1, '2025-06-02', 1, 'successed', 95, 1);

INSERT INTO learn_db.orders
(order_id, customer_id, sale_dt, product_id, status, amount, pcs)
VALUES
(6, 1, '2025-06-02', 1, 'successed', 285, 3);
```

### 3. Смотрим содержимое таблиц

```
SELECT *, _part FROM learn_db.orders;
```

Результат
```text
order_id|customer_id|sale_dt   |product_id|status   |amount|pcs|_part    |
--------+-----------+----------+----------+---------+------+---+---------+
       1|          1|2025-06-01|         1|successed|100.00|  1|all_1_6_1|
       2|          1|2025-06-01|         1|canceled |110.00|  1|all_1_6_1|
       3|          2|2025-06-01|         1|successed|100.00|  1|all_1_6_1|
       1|          1|2025-06-01|         2|successed| 50.00|  2|all_1_6_1|
       2|          1|2025-06-01|         2|canceled | 60.00|  2|all_1_6_1|
       4|          3|2025-06-01|         2|successed| 25.00|  1|all_1_6_1|
       5|          1|2025-06-02|         1|successed| 95.00|  1|all_1_6_1|
       6|          1|2025-06-02|         1|successed|285.00|  3|all_1_6_1|
```
       
### 4. Создаем таблицу с движком SummingMergeTree и ключом сортировки по дате, статусу и продукту

```sql
DROP TABLE IF EXISTS learn_db.orders_summ;
CREATE TABLE learn_db.orders_summ (
	sale_dt Date,
	product_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32
)
ENGINE = SummingMergeTree()
ORDER BY (sale_dt, status, product_id);
```

Агрегация будет проходить по полям sale_dt, status, product_id.

### 5. Вставляем в таблицу с агрегированными данными данные аналогичные, вставленным в таблицу с сырыми данным

```sql
INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'successed', 100, 1),
('2025-06-01', 2, 'successed', 50, 2);

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'canceled', 110, 1),
('2025-06-01', 2, 'canceled', 60, 2);

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'successed', 100, 1);

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 2, 'successed', 25, 1);

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-02', 1, 'successed', 95, 1);
```

### 6. Смотрим содержимое таблицы orders_summ

```
SELECT *, _part FROM learn_db.orders_summ;
```

Результат
```text 
sale_dt   |product_id|status   |amount|pcs|_part    |
----------+----------+---------+------+---+---------+
2025-06-02|         1|successed| 95.00|  1|all_5_5_1|
2025-06-01|         1|canceled |110.00|  1|all_2_2_1|
2025-06-01|         2|canceled | 60.00|  2|all_2_2_1|
2025-06-01|         2|successed| 25.00|  1|all_4_4_1|
2025-06-01|         1|successed|100.00|  1|all_3_3_1|
2025-06-01|         1|successed|100.00|  1|all_1_1_1|
2025-06-01|         2|successed| 50.00|  2|all_1_1_1|
```

Видим что данные находятся в разных партах.

### 7. Добавим еще одну строку

```sql
INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-02', 1, 'successed', 285, 3);
```

### 8. Смотрим содержимое таблицы orders_summ

```
SELECT *, _part FROM learn_db.orders_summ;
```

Результат
```text 
sale_dt   |product_id|status   |amount|pcs|_part    |
----------+----------+---------+------+---+---------+
2025-06-02|         1|successed|285.00|  3|all_6_6_1|
2025-06-01|         1|canceled |110.00|  1|all_1_5_2|
2025-06-01|         2|canceled | 60.00|  2|all_1_5_2|
2025-06-01|         1|successed|200.00|  2|all_1_5_2|
2025-06-01|         2|successed| 75.00|  3|all_1_5_2|
2025-06-02|         1|successed| 95.00|  1|all_1_5_2|
```

Видим что количество строк сократилось. Почти все данные попали в один парт. На 2025-06-01 по товару 1 две строки: одна со статусом successed, другая со статусом canceled.
Но агрегация прошла не полностью.на 2025-06-02 у товара 1 две строки со статусом successed (с одинаковым ключом сортировки). Т.к. строки не попали в один парт, то они не просуммировались.


### 9. Окончательный запрос получения сгруппированных, просуммированных данных

```
SELECT 
	sale_dt,
	product_id,
	status,
	SUM(amount) as amount,
	SUM(pcs) as pcs
FROM
	learn_db.orders_summ
GROUP BY
	sale_dt,
	product_id,
	status;
```

Результат
```text
sale_dt   |product_id|status   |amount|pcs|
----------+----------+---------+------+---+
2025-06-01|         1|successed|200.00|  2|
2025-06-01|         1|canceled |110.00|  1|
2025-06-01|         2|successed| 75.00|  3|
2025-06-01|         2|canceled | 60.00|  2|
2025-06-02|         1|successed|380.00|  4|
```

Так же мы можем применить модификатор FINAL для принудительного слияния данных.

```sql
SELECT *, _part FROM orders_summ FINAL;
```

Результат
```text
sale_dt   |product_id|status   |amount|pcs|_part    |
----------+----------+---------+------+---+---------+
2025-06-01|         1|canceled |110.00|  1|all_1_5_2|
2025-06-01|         2|canceled | 60.00|  2|all_1_5_2|
2025-06-01|         1|successed|200.00|  2|all_1_5_2|
2025-06-01|         2|successed| 75.00|  3|all_1_5_2|
2025-06-02|         1|successed|380.00|  4|all_1_5_2|
```

Получим такие же данные, что и в запросе с агрегацией по трем полям. Строки находятся в одном парте.

### 10. Пересоздаем таблицу с агрегированными данными с ключом сортировки по дате и продукту (без статуса)

```
DROP TABLE IF EXISTS learn_db.orders_summ;
CREATE TABLE learn_db.orders_summ (
	sale_dt Date,
	product_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32
)
ENGINE = SummingMergeTree()
ORDER BY (sale_dt, product_id);
```

### 11. Вставляем данные

```sql
INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'successed', 100, 1),
('2025-06-01', 2, 'successed', 50, 2);

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'canceled', 110, 1),
('2025-06-01', 2, 'canceled', 60, 2);
```

### 12. Смотрим содержимое таблиц

```
SELECT *, _part FROM learn_db.orders_summ;
```

Результат
```text 
sale_dt   |product_id|status   |amount|pcs|_part    |
----------+----------+---------+------+---+---------+
2025-06-01|         1|successed|100.00|  1|all_1_1_1|
2025-06-01|         2|successed| 50.00|  2|all_1_1_1|
2025-06-01|         1|canceled |110.00|  1|all_2_2_1|
2025-06-01|         2|canceled | 60.00|  2|all_2_2_1|
```

### 13. Делаем принудительное слияние частей и смотрим содержимое

```
OPTIMIZE TABLE learn_db.orders_summ FINAL; 
SELECT *, _part FROM learn_db.orders_summ;
```

Результат
```text 
sale_dt   |product_id|status   |amount|pcs|_part    |
----------+----------+---------+------+---+---------+
2025-06-01|         1|successed|210.00|  2|all_1_2_2|
2025-06-01|         2|successed|110.00|  4|all_1_2_2|
```

Все товары по товарам товару объединились (несмотря на статус, в этом поле СН мог взять случайное значение).

### 14. Пересоздаем таблицу с агрегированными данными, указав суммирование только по полю amount

```
DROP TABLE IF EXISTS learn_db.orders_summ;
CREATE TABLE learn_db.orders_summ (
	sale_dt Date,
	product_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32
)
ENGINE = SummingMergeTree(amount)
ORDER BY (sale_dt, status, product_id);
```

Суммироваться будет только поле amount.

### 15. Вставляем данные

```
INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'successed', 100, 1),
('2025-06-01', 2, 'successed', 50, 2);

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'successed', 100, 1);
```

### 16. Делаем принудительное слияние частей и смотрим содержимое

```sql
SELECT *, _part FROM learn_db.orders_summ;
```

Результат
```text 
sale_dt   |product_id|status   |amount|pcs|_part    |
----------+----------+---------+------+---+---------+
2025-06-01|         1|successed|100.00|  1|all_1_1_1|
2025-06-01|         2|successed| 50.00|  2|all_1_1_1|
2025-06-01|         1|successed|100.00|  1|all_2_2_1|
```

```sql
OPTIMIZE TABLE learn_db.orders_summ FINAL; 
SELECT *, _part FROM learn_db.orders_summ;
```

Результат
```text 
sale_dt   |product_id|status   |amount|pcs|_part    |
----------+----------+---------+------+---+---------+
2025-06-01|         1|successed|200.00|  1|all_1_2_2|
2025-06-01|         2|successed| 50.00|  2|all_1_2_2|
```

В pcs случайное значение.

### 17. Пересоздаем таблицу с агрегированными данными (у поля pcs тип меняем на Int32)

```sql
DROP TABLE IF EXISTS learn_db.orders_summ;
CREATE TABLE learn_db.orders_summ (
	sale_dt Date,
	product_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs Int32
)
ENGINE = SummingMergeTree()
ORDER BY (sale_dt, status, product_id);
```

Изменим тип данных у pcs.

### 18. Вставляем данные

```sql
INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'successed', 100, 1),

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'successed', -100, -1),
```

### 19. Делаем принудительное слияние частей и смотрим содержимое

```sql
SELECT *, _part FROM learn_db.orders_summ;
```

Результат
```text 
sale_dt   |product_id|status   |amount |pcs|_part    |
----------+----------+---------+-------+---+---------+
2025-06-01|         1|successed| 100.00|  1|all_1_1_1|
2025-06-01|         1|successed|-100.00| -1|all_2_2_1|
```

```sql
OPTIMIZE TABLE learn_db.orders_summ FINAL; 
SELECT *, _part FROM learn_db.orders_summ;
```

Результат
```text 
sale_dt|product_id|status|amount|pcs|_part|
-------+----------+------+------+---+-----+
```

Таблица пустая.  Если после группировке в парте по какой-то строке сумма по всем числовым полям получается 0, то эта строчка удаляется (как не имеющая смысла).