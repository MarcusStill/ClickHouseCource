## Задание

1. Удалите и создайте заново таблицу learn_db.mart_student_lesson. Вставляя за один раз в нее по 1000 строк, определите, после какой вставки произойдет первое слияние частей данных и после какой вставки произойдет второе слияние.
Напишите запрос, с помощью которого вы сможете понять, произошло ли слияние данных или нет.
Повторите вставку 1000 раз по 1000 строк в таблицу mart_student_lesson, которая была показана на уроке. Делайте вставки с помощью утилиты clickhouse-benchmark в 5 параллельных потоков.
2. Создайте таблицу learn_db.async_test, как в уроке, и воспроизведите ассинхронную вставку в нее.
Выполните запрос? который вернет 1 или более строк (что будет говорить о том, что асинхронное объединение двух или более наборов строк произошло в буфере до создания part):
```sql 
SELECT * FROM system.asynchronous_insert_log;
```
3. Объедините все части данных таблицы learn_db.async_test в один. 


#### 1. Пересоздаем таблицу learn_db.mart_student_lesson.

#### 1.1 Удаляем таблицу learn_db.mart_student_lesson

```sql
DROP TABLE learn_db.mart_student_lesson;
```

#### 1.2. Создадим пустую таблицу learn_db.mart_student_lesson

```sql
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
) ENGINE = MergeTree();
```

#### 1.3. Вставку данных по 1000 строк при помощи скрипта query.sql

```sql
INSERT INTO learn_db.mart_student_lesson
(
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
	mark)
SELECT
	floor(randUniform(2, 1300000)) AS student_profile_id,
	CAST(student_profile_id AS String) AS person_id,
	CAST(person_id AS Int32) AS person_id_int,
	student_profile_id / 365000 AS educational_organization_id,
	student_profile_id / 73000 AS parallel_id,
	student_profile_id / 2000 AS class_id,
	CAST(now() - randUniform(2,
	60 * 60 * 24 * 365) AS date) AS lesson_date,
	-- Дата урока
	formatDateTime(lesson_date,
	'%Y-%m') AS lesson_month_digits,
	formatDateTime(lesson_date,
	'%Y %M') AS lesson_month_text,
	toYear(lesson_date) AS lesson_year,
	lesson_date + rand() % 3,
	-- Дата загрузки данных
	floor(randUniform(2, 137)) AS t,
	educational_organization_id * 136 + t AS teacher_id,
	floor(t / 9) AS subject_id,
	CASE
		subject_id
    	WHEN 1 THEN 'Математика'
		WHEN 2 THEN 'Русский язык'
		WHEN 3 THEN 'Литература'
		WHEN 4 THEN 'Физика'
		WHEN 5 THEN 'Химия'
		WHEN 6 THEN 'География'
		WHEN 7 THEN 'Биология'
		WHEN 8 THEN 'Физическая культура'
		ELSE 'Информатика'
	END AS subject_name,
	CASE
		WHEN randUniform(0,
		2) > 1
    		THEN -1
		ELSE 
    			CASE
			WHEN ROUND(randUniform(0,
			5)) + subject_id < 5 THEN ROUND(randUniform(4,
			5))
			WHEN ROUND(randUniform(0,
			5)) + subject_id < 9 THEN ROUND(randUniform(3,
			5))
			ELSE ROUND(randUniform(2,
			5))
		END
	END AS mark
FROM
	numbers(1000);
```

#### 1.4. Подготавливаем скрипт для отслеживания процесса вставки

```bash
#!/bin/bash

for i in {1..50}; do
  echo "Вставка $i"
  clickhouse-client --query "$(cat query.sql)"
  
  echo "Проверка состояния кусков:"
  clickhouse-client --query "
  SELECT 
      count() as parts_count,
      sum(rows) as total_rows,
      groupArray(name) as part_names
  FROM system.parts 
  WHERE table = 'mart_student_lesson' AND database = 'learn_db' AND active"
  
  echo "---"
  
  # Небольшая пауза между вставками
  sleep 1
done
```

#### 1.5. Делаем скрипт исполняемым `chmod +x monitor_merges.sh` и выполняем его `./monitor_merges.sh`

```text
root@fffdfdbfef12:/# /.monitor_merges.sh
bash: /.monitor_merges.sh: No such file or directory
root@fffdfdbfef12:/# ./monitor_merges.sh
Вставка 1
Проверка состояния кусков:
1       1000    ['all_1_1_0']
---
Вставка 2
Проверка состояния кусков:
2       2000    ['all_1_1_0','all_2_2_0']
---
Вставка 3
Проверка состояния кусков:
3       3000    ['all_1_1_0','all_2_2_0','all_3_3_0']
---
Вставка 4
Проверка состояния кусков:
4       4000    ['all_1_1_0','all_2_2_0','all_3_3_0','all_4_4_0']
---
Вставка 5
Проверка состояния кусков:
5       5000    ['all_1_1_0','all_2_2_0','all_3_3_0','all_4_4_0','all_5_5_0']
---
Вставка 6
Проверка состояния кусков:
1       6000    ['all_1_6_1']
---
Вставка 7
Проверка состояния кусков:
2       7000    ['all_1_6_1','all_7_7_0']
---
Вставка 8
Проверка состояния кусков:
3       8000    ['all_1_6_1','all_7_7_0','all_8_8_0']
---
Вставка 9
Проверка состояния кусков:
4       9000    ['all_1_6_1','all_7_7_0','all_8_8_0','all_9_9_0']
---
Вставка 10
Проверка состояния кусков:
5       10000   ['all_1_6_1','all_7_7_0','all_8_8_0','all_9_9_0','all_10_10_0']
---
Вставка 11
Проверка состояния кусков:
1       11000   ['all_1_11_2']
---
Вставка 12
Проверка состояния кусков:
2       12000   ['all_1_11_2','all_12_12_0']
---
Вставка 13
Проверка состояния кусков:
3       13000   ['all_1_11_2','all_12_12_0','all_13_13_0']
---
Вставка 14
Проверка состояния кусков:
4       14000   ['all_1_11_2','all_12_12_0','all_13_13_0','all_14_14_0']
---
Вставка 15
Проверка состояния кусков:
5       15000   ['all_1_11_2','all_12_12_0','all_13_13_0','all_14_14_0','all_15_15_0']
---
Вставка 16
Проверка состояния кусков:
1       16000   ['all_1_16_3']
---
Вставка 17
Проверка состояния кусков:
2       17000   ['all_1_16_3','all_17_17_0']
---
Вставка 18
Проверка состояния кусков:
3       18000   ['all_1_16_3','all_17_17_0','all_18_18_0']
---
Вставка 19
Проверка состояния кусков:
4       19000   ['all_1_16_3','all_17_17_0','all_18_18_0','all_19_19_0']
---
Вставка 20
Проверка состояния кусков:
5       20000   ['all_1_16_3','all_17_17_0','all_18_18_0','all_19_19_0','all_20_20_0']
---
```

#### 1.6. Анализируем работу merge

```text
Вставки 1-5:
all_1_1_0, all_2_2_0, all_3_3_0, all_4_4_0, all_5_5_0
Каждая вставка создаёт новый кусок с level=0. Номера блоков от 1 до 5

После 6-й вставки - первое слияние:
all_1_6_1
Все предыдущие куски (1-6) объединились в один. Уровень слияния стал 1 (последний номер в имени). Это первое слияние (merge)

Вставки 7-10:
all_1_6_1, all_7_7_0, all_8_8_0, all_9_9_0, all_10_10_0
Новые вставки создают куски с level=0. Объединённый кусок остаётся.

После 11-й вставки - второе слияние:
all_1_11_2

Все куски (объединённый 1-6 и новые 7-11) слились в один. Уровень слияния увеличился до 2. Это второе слияние.

Далее паттерн повторяется:
 - вставки 12-15 создают новые куски
 - после 16-й вставки - третье слияние (all_1_16_3)
 - и так далее
```

#### 1.7. Анализ выполнения merge можно сделать при помощи следующего запроса

```sql
SELECT 
    table,
    partition,
    name,
    active,
    rows,
    marks,
    bytes_on_disk,
    modification_time,
    min_time,
    max_time
FROM system.parts 
WHERE table = 'mart_student_lesson' AND database = 'learn_db'
ORDER BY modification_time DESC;
```

Результат
```text
table              |partition|name       |active|rows |marks|bytes_on_disk|modification_time  |min_time           |max_time           |
-------------------+---------+-----------+------+-----+-----+-------------+-------------------+-------------------+-------------------+
mart_student_lesson|tuple()  |all_20_20_0|     1| 1000|    2|        31513|2025-06-29 15:43:02|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_19_19_0|     1| 1000|    2|        31728|2025-06-29 15:43:01|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_18_18_0|     1| 1000|    2|        31496|2025-06-29 15:43:00|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_17_17_0|     1| 1000|    2|        31722|2025-06-29 15:42:59|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_1_16_3 |     1|16000|    3|       390992|2025-06-29 15:42:58|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_16_16_0|     0| 1000|    2|        31673|2025-06-29 15:42:58|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_15_15_0|     0| 1000|    2|        31731|2025-06-29 15:42:57|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_14_14_0|     0| 1000|    2|        31653|2025-06-29 15:42:56|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_13_13_0|     0| 1000|    2|        31534|2025-06-29 15:42:55|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_12_12_0|     0| 1000|    2|        31576|2025-06-29 15:42:54|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_1_11_2 |     0|11000|    3|       275980|2025-06-29 15:42:53|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_11_11_0|     0| 1000|    2|        31661|2025-06-29 15:42:53|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_10_10_0|     0| 1000|    2|        31625|2025-06-29 15:42:52|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_9_9_0  |     0| 1000|    2|        31610|2025-06-29 15:42:50|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_8_8_0  |     0| 1000|    2|        31659|2025-06-29 15:42:49|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_7_7_0  |     0| 1000|    2|        31597|2025-06-29 15:42:48|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_1_6_1  |     0| 6000|    2|       155821|2025-06-29 15:42:47|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_6_6_0  |     0| 1000|    2|        31550|2025-06-29 15:42:47|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_5_5_0  |     0| 1000|    2|        31630|2025-06-29 15:42:46|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_4_4_0  |     0| 1000|    2|        31620|2025-06-29 15:42:45|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_3_3_0  |     0| 1000|    2|        31710|2025-06-29 15:42:44|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_2_2_0  |     0| 1000|    2|        31697|2025-06-29 15:42:43|1970-01-01 03:00:00|1970-01-01 03:00:00|
mart_student_lesson|tuple()  |all_1_1_0  |     0| 1000|    2|        31681|2025-06-29 15:42:42|1970-01-01 03:00:00|1970-01-01 03:00:00|
```

#### 1.8. Визуализация процесса

```text
Вставки 1-6:   [1][2][3][4][5][6] → merge → [1-6]
Вставки 7-11:  [1-6]+[7][8][9][10][11] → merge → [1-11]
Вставки 12-16: [1-11]+[12][13][14][15][16] → merge → [1-16]
Вставки 17-20: [1-16]+[17][18][19][20] (ожидается merge после 21-й)
```

#### 2. Повторим асинхронную вставку в таблицу `learn_db.async_test` и посмотрим на результат выполнения запроса

```sql
SELECT *
FROM system.asynchronous_insert_log;
```

#### 2.1. Создадим таблицу

```sql
CREATE TABLE learn_db.async_test
(
    id UInt32,
    timestamp DateTime,
    data String
)
ENGINE = MergeTree
ORDER BY (id, timestamp);
```

#### 2.2. Отредактируем скрипт для вставки данных

```bash
#!/bin/bash

INSERT INTO learn_db.async_test SETTINGS async_insert=1 VALUES (1, now(), 'async data');
```

#### 2.3. Включаем асинхронную вставку для сессии

`SET async_insert = 1;`

#### 2.4. Запускаем 1000 000 запросов на вставку по 1 строк в 10 параллельных потоков. То есть всего вставляем 1000 000 строк.

```text
clickhouse-benchmark -i 1000000 -c 10 --query "`cat query.sql`"
```

#### 2.5. Форсируем объединение частей в одну

```sql
OPTIMIZE TABLE learn_db.mart_student_lesson FINAL;
SELECT * FROM system.parts where table = 'mart_student_lesson';
```

Результат выполнения запроса
```text
partition|name      |uuid                                |part_type|active|marks|rows |bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table              |engine   |disk_name|path                                                                          |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                |removal_tid_lock    |removal_tid                                 |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+----------+------------------------------------+---------+------+-----+-----+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+-------------------+---------+---------+------------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------+--------------------+--------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
tuple()  |all_1_20_4|00000000-0000-0000-0000-000000000000|Compact  |     0|    4|20000|       485038|               483482|                1599934|              70|        248|                                 0|                                   0|                            0|2025-06-29 15:49:33|2025-06-29 16:27:28|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|              20|    4|           1|                         36|                                  419|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/99f/99f44313-71de-436c-8dcb-141d8e7821e7/all_1_20_4/|a3d314bb3b765861a818a6928750c9a0|40908017bdd64d1388ece029d7f4dca2|680c5c64e2f3eb68752afa1e0e494ed5     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|[1, 1, 00000000-0000-0000-0000-000000000000]|15317705874040209379|[1, 1, 00000000-0000-0000-0000-000000000000]|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_1_20_5|00000000-0000-0000-0000-000000000000|Compact  |     1|    4|20000|       485038|               483482|                1599934|              70|        248|                                 0|                                   0|                            0|2025-06-29 16:27:28|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|              20|    5|           1|                         36|                                  419|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/99f/99f44313-71de-436c-8dcb-141d8e7821e7/all_1_20_5/|a3d314bb3b765861a818a6928750c9a0|40908017bdd64d1388ece029d7f4dca2|680c5c64e2f3eb68752afa1e0e494ed5     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|[1, 1, 00000000-0000-0000-0000-000000000000]|                   0|[0, 0, 00000000-0000-0000-0000-000000000000]|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

#### 2.6. Выполним следующий запрос
```sql
SELECT 
    flush_query_id,
    min(event_time) AS first_event_time,
    max(event_time) AS last_event_time,
    count() AS merged_inserts,
    sum(rows) AS total_rows,
    max(flush_time) AS flush_time
FROM system.asynchronous_insert_log
WHERE `table` = 'async_test'
GROUP BY flush_query_id
HAVING count() > 1  -- Показываем только объединенные вставки
ORDER BY first_event_time DESC
LIMIT 5;
```

Результат:
```text
flush_query_id                      |first_event_time   |last_event_time    |merged_inserts|total_rows|flush_time         |
------------------------------------+-------------------+-------------------+--------------+----------+-------------------+
462aa77c-370c-41b3-8bbc-6bcb776e3b95|2025-06-29 16:31:51|2025-06-29 16:31:51|            10|        10|2025-06-29 16:31:51|
9b83ded6-5621-4dc4-aa09-b1bf6033069e|2025-06-29 16:31:51|2025-06-29 16:31:51|            10|        10|2025-06-29 16:31:51|
c35a7feb-ec19-4aa8-8080-c9d24b8eb367|2025-06-29 16:31:51|2025-06-29 16:31:51|            10|        10|2025-06-29 16:31:51|
614b72b6-f2e2-4769-92e4-c22ddffdc6fb|2025-06-29 16:31:50|2025-06-29 16:31:50|            10|        10|2025-06-29 16:31:50|
d54aa15c-38a5-48c0-b4c3-f5cf346bf4a4|2025-06-29 16:31:50|2025-06-29 16:31:50|            10|        10|2025-06-29 16:31:50|
```

Получается что:
- каждая строка представляет пакет из 10 INSERT-запросов, объединённых в один батч (merged_inserts=10)
- в каждом батче обработано ровно 10 строк (total_rows=10)
- временные метки: first_event_time и last_event_time совпадают с точностью до секунды, что означает все 10 запросов были получены сервером в пределах одной секунды
- батчи формируются каждую секунду (см. flush_time): 16:31:50 - 2 батча, 16:31:51 - 3 батча.

#### 2.7. Выполним следующий запрос

```sql
SELECT 
    name, 
    rows,
    active,
    formatDateTime(modification_time, '%T') AS flush_time
FROM system.parts
WHERE table = 'async_test'
ORDER BY modification_time DESC
LIMIT 5;
```

Результат выполнения:
```text
name           |rows |active|flush_time|
---------------+-----+------+----------+
all_4245_4245_0|   10|     1|13:33:34  |
all_1_4241_800 |42374|     1|13:33:33  |
all_4239_4239_0|   10|     0|13:33:33  |
all_4240_4240_0|   10|     0|13:33:33  |
all_4242_4242_0|   10|     1|13:33:33  |
```

Получаем подтверждение асинхронного объединения.
- Батчи по 10 строк: наличие кусков с rows=10 (например, all_4245_4245_0) соответствует логу, где merged_inserts=10 и total_rows=10. Это доказывает, что 10 отдельных INSERT-запросов были объединены в буфере и записаны как один кусок данных (part)
- Крупные куски (например, all_1_4241_800 с 42,374 строками): это результат фоновых мержей ClickHouse, которые объединяют мелкие куски для оптимизации.