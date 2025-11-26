# Применение движка ReplacingMergeTree

### 1. Удаляем и создаем таблицу orders

```sql
DROP TABLE IF EXISTS learn_db.orders;
CREATE TABLE learn_db.orders (
	order_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32
)
ENGINE = ReplacingMergeTree()
ORDER BY (order_id);
```

Движок ReplacingMergeTree будет искать строчки с одинаковым order_id, и если он их найдет, то удалит лишние.

### 2. Вставляем 3 строки в таблицу, соответствующие одному заказу

```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs)
VALUES
(1, 'created', 100, 1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs)
VALUES
(1, 'created', 90, 1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs)
VALUES
(1, 'created', 80, 1);
```

Как будто мы вставили один заказ и дважды его отредактировали.

### 3. Получаем все строки из таблицы заказов и только актуальную строку

```sql
SELECT * FROM orders o;
```

Результат
```text
order_id|status |amount|pcs|
--------+-------+------+---+
       1|created| 80.00|  1|
       1|created|100.00|  1|
       1|created| 90.00|  1|
```

Мы получили дубли.

Выполним запрос с модификатором FINAL.

```sql
SELECT * FROM orders o FINAL;
```

Результат
```text
order_id|status |amount|pcs|
--------+-------+------+---+
       1|created| 80.00|  1|
```

В этом случае FINAL выведет последнее добавленное значение.

### 4. Добавим еще одну строку
```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs)
VALUES
(1, 'created', 110, 1);
```

Если выполним запрос, то получим актуальное состояние таблицы

```sql
SELECT * FROM orders o;
```

Результат
```text
order_id|status |amount|pcs|
--------+-------+------+---+
       1|created|100.00|  1|
       1|created|110.00|  1|
```

Выполним запрос с модификатором FINAL.

```sql
SELECT * FROM orders o FINAL;
```

Результат
```text
order_id|status |amount|pcs|
--------+-------+------+---+
       1|created|110.00|  1|
```

### 5. Пересоздаем таблицу orders, добавив колонку с номером версии строки

```
DROP TABLE IF EXISTS learn_db.orders;
CREATE TABLE learn_db.orders (
	order_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32,
	version UInt32
)
ENGINE = ReplacingMergeTree(version)
ORDER BY (order_id);
```

### 6. Вставляем 3 строки в таблицу, соответствующие одному заказу

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version)
VALUES
(1, 'created', 100, 1, 1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version)
VALUES
(1, 'created', 90, 1, 3);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version)
VALUES
(1, 'created', 80, 1, 4);
```

### 7. Получаем все строки из таблицы заказов и только актуальную строку

```sql
SELECT * FROM learn_db.orders o;
```

Результат
```text
order_id|status |amount|pcs|version|
--------+-------+------+---+-------+
       1|created|100.00|  1|      1|
       1|created| 80.00|  1|      4|
       1|created| 90.00|  1|      3|
```

### 8. Получаем все строки из таблицы заказов и только актуальную строку

```
SELECT * FROM learn_db.orders o FINAL;
```

Результат
```text
order_id|status |amount|pcs|version|
--------+-------+------+---+-------+
       1|created| 80.00|  1|      4|
```

### 9. Добавим еще одну строку

```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version)
VALUES
(1, 'created', 70, 1, 2);
```

### 10. Получаем все строки из таблицы заказов и только актуальную строку

```
SELECT * FROM learn_db.orders o;
```

Результат
```text
order_id|status |amount|pcs|version|
--------+-------+------+---+-------+
       1|created| 80.00|  1|      4|
       1|created| 90.00|  1|      3|
       1|created|100.00|  1|      1|
       1|created| 70.00|  1|      2|
```

```
SELECT * FROM learn_db.orders o FINAL;
```

Результат
```text
order_id|status |amount|pcs|version|
--------+-------+------+---+-------+
       1|created| 80.00|  1|      4|
```

Получим строку с amount = 80. Если при слиянии частей в частях находятся дубли, то тогда среди них CH оставит ту строчку, 
у которой version максимальный (или если одна версия, то строку, добавленную последней).

### 11. Пересоздаем таблицу orders, добавив колонку с пометкой, что строка удалена

```
DROP TABLE IF EXISTS learn_db.orders;
CREATE TABLE learn_db.orders (
	order_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32,
	version UInt32,
	is_deleted UInt8
)
ENGINE = ReplacingMergeTree(version, is_deleted)
ORDER BY (status, order_id);
```

Атрибут is_deleted нельзя использовать без version.

### 12. Вставляем 2 строки, меняющие состояние заказа с номером 1

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version, is_deleted)
VALUES
(1, 'created', 100, 1, 1, 0);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version, is_deleted)
VALUES
(1, 'created', 90, 1, 3, 0);
```

### 13. Получаем все строки из таблицы заказов и только актуальную строку

```
SELECT * FROM learn_db.orders o;
```

Результат
```text
order_id|status |amount|pcs|version|is_deleted|
--------+-------+------+---+-------+----------+
       1|created| 90.00|  1|      3|         0|
       1|created|100.00|  1|      1|         0|
```

```
SELECT * FROM learn_db.orders o FINAL;
```

Результат
```text
order_id|status |amount|pcs|version|is_deleted|
--------+-------+------+---+-------+----------+
       1|created| 90.00|  1|      3|         0|
```

### 14. Сделаем еще вставку, удалив заказ с номером 1

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version, is_deleted)
VALUES
(1, 'created', 80, 1, 4, 1);
```

### 15. Получаем все строки из таблицы заказов и только актуальную строку

```
SELECT * FROM learn_db.orders o;
```

Результат
```text
order_id|status |amount|pcs|version|is_deleted|
--------+-------+------+---+-------+----------+
       1|created| 90.00|  1|      3|         0|
       1|created| 80.00|  1|      4|         1|
       1|created|100.00|  1|      1|         0|
```

```
SELECT * FROM learn_db.orders o FINAL;
```

Результат
```text
order_id|status|amount|pcs|version|is_deleted|
--------+------+------+---+-------+----------+
```

### 16. А если у нас асинхронная вставка и нам пришла еще одна строка

```
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version, is_deleted)
VALUES
(1, 'created', 70, 1, 2, 0);
```

В этой вставке строка не помечена удаленной, но у нее более ранняя версия. 

### 17. Получаем все строки из таблицы заказов и только актуальную строку

```
SELECT * FROM learn_db.orders o;
```

Результат
```text
order_id|status |amount|pcs|version|is_deleted|
--------+-------+------+---+-------+----------+
       1|created|100.00|  1|      1|         0|
       1|created| 90.00|  1|      3|         0|
       1|created| 80.00|  1|      4|         1|
       1|created| 70.00|  1|      2|         0|
```

```
SELECT * FROM learn_db.orders o FINAL;
```

Результат
```text
order_id|status|amount|pcs|version|is_deleted|
--------+------+------+---+-------+----------+
```

То есть если мы вставляем строчку с флагом is_deleted = 1, то ClickHouse ее оставит (если у нее максимальная версия). Для того, чтобы если позже придут другие строки с более ранними версиями по этому закажу чтобы СН не "оживил" этот заказ.

### 18. Проверим на сколько запрос замедляется при использовании FINAL. Создаем и наполняем таблицу с движком MergeTree

```sql
DROP TABLE learn_db.mart_student_lesson;
CREATE TABLE learn_db.mart_student_lesson
(
	`student_profile_id` Int32, -- Идентификатор профиля обучающегося
	`person_id` String, -- GUID обучающегося
	`person_id_int` Int32 CODEC(Delta, ZSTD),
	`educational_organization_id` Int16, -- Идентификатор образовательной организации
	`parallel_id` Int16,
	`class_id` Int16, -- Идентификатор класса
	`lesson_date` Date32, -- Дата урока
	`lesson_month_digits` String,
	`lesson_month_text` String,
	`lesson_year` UInt16,
	`load_date` Date, -- Дата загрузки данных
	`t` Int16 CODEC(Delta, ZSTD),
	`teacher_id` Int32 CODEC(Delta, ZSTD), -- Идентификатор учителя
	`subject_id` Int16 CODEC(Delta, ZSTD), -- Идентификатор предмета
	`subject_name` String,
	`mark` Int8, -- Оценка
	PRIMARY KEY(lesson_date, person_id_int, mark)
) ENGINE = MergeTree()
PARTITION BY educational_organization_id
AS SELECT
	floor(randUniform(2, 1300000)) as student_profile_id,
	cast(student_profile_id as String) as person_id,
	cast(person_id as Int32) as  person_id_int,
    student_profile_id / 365000 as educational_organization_id,
    student_profile_id / 73000 as parallel_id,
    student_profile_id / 2000 as class_id,
    cast(now() - randUniform(2, 60*60*24*365) as date) as lesson_date, -- Дата урока
    formatDateTime(lesson_date, '%Y-%m') as lesson_month_digits,
    formatDateTime(lesson_date, '%Y %M') AS lesson_month_text,
    toYear(lesson_date) as lesson_year, 
    lesson_date + rand() % 3, -- Дата загрузки данных
    floor(randUniform(2, 137)) as t,
    educational_organization_id * 136 + t as teacher_id,
    floor(t/9) as subject_id,
    CASE subject_id
    	WHEN 1 THEN 'Математика'
    	WHEN 2 THEN 'Русский язык'
    	WHEN 3 THEN 'Литература'
    	WHEN 4 THEN 'Физика'
    	WHEN 5 THEN 'Химия'
    	WHEN 6 THEN 'География'
    	WHEN 7 THEN 'Биология'
    	WHEN 8 THEN 'Физическая культура'
    	ELSE 'Информатика'
    END as subject_name,
    CASE 
    	WHEN randUniform(0, 2) > 1
    		THEN -1
    		ELSE 
    			CASE
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 5 THEN ROUND(randUniform(4, 5))
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 9 THEN ROUND(randUniform(3, 5))
	    			ELSE ROUND(randUniform(2, 5))
    			END				
    END AS mark
FROM numbers(10000000);
```

### 19. Создаем и наполняем таблицу с движком ReplacingMergeTree

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_replacing_merge_tree;
CREATE TABLE learn_db.mart_student_lesson_replacing_merge_tree
(
	`student_profile_id` Int32, -- Идентификатор профиля обучающегося
	`person_id` String, -- GUID обучающегося
	`person_id_int` Int32 CODEC(Delta, ZSTD),
	`educational_organization_id` Int16, -- Идентификатор образовательной организации
	`parallel_id` Int16,
	`class_id` Int16, -- Идентификатор класса
	`lesson_date` Date32, -- Дата урока
	`lesson_month_digits` String,
	`lesson_month_text` String,
	`lesson_year` UInt16,
	`load_date` Date, -- Дата загрузки данных
	`t` Int16 CODEC(Delta, ZSTD),
	`teacher_id` Int32 CODEC(Delta, ZSTD), -- Идентификатор учителя
	`subject_id` Int16 CODEC(Delta, ZSTD), -- Идентификатор предмета
	`subject_name` String,
	`mark` Int8, -- Оценка
	`version` UInt32,
	`is_deleted` UInt8,
	PRIMARY KEY(lesson_date, person_id_int, mark)
) ENGINE = ReplacingMergeTree(version, is_deleted)
PARTITION BY educational_organization_id
AS SELECT
	student_profile_id,
	person_id,
	person_id_int,
	educational_organization_id,
	parallel_id,
	class_id,
	lesson_date,
	lesson_month_digits,
	lesson_month_text,
	lesson_year,
	load_date,
	t,
	teacher_id,
	subject_id,
	subject_name,
	mark,
	1 as version,
	0 as is_deleted
FROM
	learn_db.mart_student_lesson;
```

### 20. Считаем количество оценок в таблице с движком MergeTree

Для удобства замера времени выполнения запросов воспользуемся встроенным клиентом запросов. Перейдем в контейнер с БД на вкладку Exec и выполним команды

```bash
bash
clickhouse-client
```

Увидим приветствие
```text
root@19294dc11be6:/# clickhouse-client
ClickHouse client version 25.4.13.22 (official build).
Connecting to localhost:9000 as user username.
Connected to ClickHouse server version 25.4.13.
```

Далее вставим текст запроса и выполним его

```sql
SELECT 
	mark, 
	count(*) 
FROM learn_db.mart_student_lesson
GROUP BY
	mark;
```

Результат
```text
Query id: b21c19a9-1c9b-4c33-b0c8-f563f39ceedd

   ┌─mark─┬─count()─┐
1. │    2 │  484785 │
2. │    3 │ 1300779 │
3. │    4 │ 2013440 │
4. │    5 │ 1198561 │
5. │   -1 │ 5002435 │
   └──────┴─────────┘

5 rows in set. Elapsed: 0.032 sec. Processed 10.00 million rows, 10.00 MB (316.73 million rows/s., 316.73 MB/s.)
Peak memory usage: 77.84 KiB.
```

### 21. Считаем количество оценок в таблице с движком ReplacingMergeTree без применения FINAL

```sql
SELECT 
	mark, 
	count(*) 
FROM learn_db.mart_student_lesson_replacing_merge_tree
GROUP BY
	mark;
```
Результат
```text
Query id: f1c259ee-2546-4a4f-9e87-8e7363330b7c

   ┌─mark─┬─count()─┐
1. │    2 │  484590 │
2. │    3 │ 1299434 │
3. │    4 │ 2010357 │
4. │    5 │ 1197455 │
5. │   -1 │ 4983641 │
   └──────┴─────────┘

5 rows in set. Elapsed: 0.012 sec. Processed 9.98 million rows, 9.98 MB (811.60 million rows/s., 811.60 MB/s.)
Peak memory usage: 254.91 KiB.
```

### 22. Считаем количество оценок в таблице с движком ReplacingMergeTree без применения FINAL

```sql
SELECT 
	mark, 
	count(*) 
FROM learn_db.mart_student_lesson_replacing_merge_tree
GROUP BY
	mark;
```

Результат:
```text
Query id: e36f25d8-9e29-44bd-b993-3c6765fc0cc8

   ┌─mark─┬─count()─┐
1. │    2 │  484590 │
2. │    3 │ 1299434 │
3. │    4 │ 2010357 │
4. │    5 │ 1197455 │
5. │   -1 │ 4983641 │
   └──────┴─────────┘

5 rows in set. Elapsed: 0.010 sec. Processed 9.98 million rows, 9.98 MB (956.91 million rows/s., 956.91 MB/s.)
Peak memory usage: 79.21 KiB.
```

### 18. Считаем количество оценок в таблице с движком ReplacingMergeTree c применением FINAL

```sql
SELECT 
	mark, 
	count(*) 
FROM learn_db.mart_student_lesson_replacing_merge_tree FINAL
GROUP BY
	mark;
```

Результат:
```text
Query id: ffed7eee-26fe-407b-9337-3d3fd2081d85

   ┌─mark─┬─count()─┐
1. │    2 │  484524 │
2. │    3 │ 1298993 │
3. │    4 │ 2009196 │
4. │    5 │ 1197054 │
5. │   -1 │ 4976417 │
   └──────┴─────────┘

5 rows in set. Elapsed: 0.201 sec. Processed 11.10 million rows, 144.27 MB (55.32 million rows/s., 719.14 MB/s.)
Peak memory usage: 157.00 MiB.
```

Замедление весьма существенное!
