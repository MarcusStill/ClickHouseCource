## Задание

1. Добавьте к таблице проекцию, оптимизированную для запросов с фильтрацией по предмету (subject_name) и дате урока (lesson_date).
2. Сравните производительность запросов:
   * С фильтрацией по subject_name и lesson_date до добавления проекции (или с отключёнными проекциями):
   ```sql
   SELECT subject_name, count(), avg(mark)
   FROM learn_db.mart_student_lesson
   WHERE subject_name = 'Математика' AND lesson_date >= '2023-01-01' GROUP BY subject_name
   SETTINGS optimize_use_projections = 0
   ```
   * Тот же запрос с включёнными проекциями (по умолчанию):
   ```sql
   SELECT subject_name, count(), avg(mark) 
   FROM learn_db.mart_student_lesson 
   WHERE subject_name = 'Математика' AND lesson_date >= '2023-01-01' GROUP BY subject_name;
   ```

### 1. Пересоздаем таблицу и наполняем ее данными
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson; 
CREATE TABLE learn_db.mart_student_lesson
(
	`student_profile_id` Int32 , -- Идентификатор профиля обучающегося
	`person_id` String , -- GUID обучающегося
	`person_id_int` Int32 ,
	`educational_organization_id` Int16 , -- Идентификатор образовательной организации
	`parallel_id` Int16 ,
	`class_id` Int16 , -- Идентификатор класса
	`lesson_date` Date32 , -- Дата урока
	`lesson_month_digits` String ,
	`lesson_month_text` String ,
	`lesson_year` UInt16 ,
	`load_date` Date , -- Дата загрузки данных
	`t` Int16 CODEC(Delta, ZSTD),
	`teacher_id` Int32 , -- Идентификатор учителя
	`subject_id` Int16 , -- Идентификатор предмета
	`subject_name` String ,
	`mark` Nullable(UInt8) , -- Оценка
	PRIMARY KEY(lesson_date)
) ENGINE = MergeTree()
AS SELECT
	floor(randUniform(2, 10000000)) as student_profile_id,
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
    		THEN NULL
    		ELSE 
    			CASE
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 5 THEN ROUND(randUniform(4, 5))
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 9 THEN ROUND(randUniform(3, 5))
	    			ELSE ROUND(randUniform(2, 5))
    			END				
    END AS mark
FROM numbers(10000000);
```

### 2. Проверим скорость выполнения запроса

```sql
SELECT subject_name, count(), avg(mark)
FROM learn_db.mart_student_lesson
WHERE subject_name = 'Математика' AND lesson_date >= '2023-01-01' GROUP BY subject_name
SETTINGS optimize_use_projections = 0
```

Результат:
```text
Query id: c946e8e7-c5e3-4309-aac5-cec3bf9bf1a1

   ┌─subject_name─┬─count()─┬─────────avg(mark)─┐
1. │ Математика   │  666332 │ 4.350559569769713 │
   └──────────────┴─────────┴───────────────────┘

1 row in set. Elapsed: 0.133 sec. Processed 10.00 million rows, 356.68 MB (75.06 million rows/s., 2.68 GB/s.)
Peak memory usage: 3.15 MiB.
```


```sql
EXPLAIN indexes = 1
SELECT subject_name, count(), avg(mark)
FROM learn_db.mart_student_lesson
WHERE subject_name = 'Математика' AND lesson_date >= '2023-01-01' GROUP BY subject_name
SETTINGS optimize_use_projections = 0
```

Результат:
```text
explain                                                 |
--------------------------------------------------------+
Expression ((Project names + Projection))               |
  Aggregating                                           |
    Expression (Before GROUP BY)                        |
      Expression                                        |
        ReadFromMergeTree (learn_db.mart_student_lesson)|
        Indexes:                                        |
          PrimaryKey                                    |
            Keys:                                       |
              lesson_date                               |
            Condition: (lesson_date in [19358, +Inf))   |
            Parts: 4/4                                  |
            Granules: 1223/1223                         |
```

```sql
SELECT query_id, query, event_time 
FROM system.query_log
WHERE type = 'QueryStart'
ORDER BY event_time DESC;
```

Результат:
```text
query_id = 067c63bf-8893-4eaf-8c26-e88d4dde83f6
```

```sql
SELECT SUM(elapsed_us) FROM system.processors_profile_log WHERE query_id = '067c63bf-8893-4eaf-8c26-e88d4dde83f6';
```

Результат:
```text
SUM(elapsed_us)|
---------------+
        1482021|
```

### 3. Создадим проекцию

```sql
ALTER TABLE learn_db.mart_student_lesson
DROP PROJECTION IF EXISTS order_no_avg__projection;

ALTER TABLE learn_db.mart_student_lesson
ADD PROJECTION order_no_avg__projection
(
SELECT 
    	subject_name,
    	lesson_date,
    	mark
    ORDER BY subject_name, lesson_date
);

ALTER TABLE learn_db.mart_student_lesson
MATERIALIZE PROJECTION order_no_avg__projection;
```

### 4. Вновь выполним запрос

```sql
SELECT subject_name, count(), avg(mark) 
FROM learn_db.mart_student_lesson 
WHERE subject_name = 'Математика' AND lesson_date >= '2023-01-01' GROUP BY subject_name;
```

Результат:
```text
Query id: 9d4ee4ea-3d52-427c-8a04-fd7fc3eddd33

   ┌─subject_name─┬─count()─┬─────────avg(mark)─┐
1. │ Математика   │  666332 │ 4.350559569769713 │
   └──────────────┴─────────┴───────────────────┘

1 row in set. Elapsed: 0.019 sec. Processed 704.51 thousand rows, 24.71 MB (36.37 million rows/s., 1.28 GB/s.)
Peak memory usage: 6.60 MiB.
```

```sql
EXPLAIN indexes = 1
SELECT subject_name, count(), avg(mark) 
FROM learn_db.mart_student_lesson 
WHERE subject_name = 'Математика' AND lesson_date >= '2023-01-01' GROUP BY subject_name;
```

Результат:
```text
explain                                                                                                 |
--------------------------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                                               |
  Aggregating                                                                                           |
    Filter                                                                                              |
      ReadFromMergeTree (order_projection)                                                              |
      Indexes:                                                                                          |
        PrimaryKey                                                                                      |
          Keys:                                                                                         |
            subject_name                                                                                |
            lesson_date                                                                                 |
          Condition: and((lesson_date in [19358, +Inf)), (subject_name in ['Математика', 'Математика']))|
          Parts: 4/4                                                                                    |
          Granules: 86/1223                                                                             |
```

### 5. Дополнительно проверим используется ли проекция

```sql
SELECT query_id, query, event_time, projections 
FROM system.query_log
WHERE type = 'QueryStart'
ORDER BY event_time DESC;

```

Результат:
```text
query_id                            |query                                                                                                                                                                                                                                                          |event_time         |projections                                                               |
------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------+--------------------------------------------------------------------------+
508960c6-80e4-4883-911c-060d485f5505|SELECT '' AS TABLE_CAT, t.database AS TABLE_SCHEM, t.name AS TABLE_NAME, CASE WHEN t.engine LIKE '%Log' THEN 'LOG TABLE' WHEN t.engine in ('Buffer', 'Memory', 'Set') THEN 'MEMORY TABLE' WHEN t.is_temporary != 0 THEN 'TEMPORARY TABLE' WHEN t.engine like '%|2025-10-16 22:43:31|[]                                                                        |
25c2c617-2184-47df-8d4b-206fee7a073c|SELECT '' AS FUNCTION_CAT, '' AS FUNCTION_SCHEM, '' AS FUNCTION_NAME, '' AS REMARKS, 0 AS FUNCTION_TYPE, '' AS SPECIFIC_NAME LIMIT 0                                                                                                                           |2025-10-16 22:43:31|[]                                                                        |
9f40af92-6daf-49a1-b195-e3f697f2b9dc|SELECT name AS TABLE_SCHEM, '' AS TABLE_CATALOG FROM system.databases WHERE name LIKE 'eaf'                                                                                                                                                                    |2025-10-16 22:43:31|[]                                                                        |
8eff3ef8-1189-43df-9f5d-6df3a34c2efd|select * from system.query_log                                                                                                                                                                                                                                 |2025-10-16 22:42:49|[]                                                                        |
76910ae4-6651-40bc-91a5-8d18c734ea2a|EXPLAIN indexes = 1¶SELECT subject_name, count(), avg(mark) ¶FROM learn_db.mart_student_lesson ¶WHERE subject_name = 'Математика' AND lesson_date >= '2023-01-01' GROUP BY subject_name                                                                        |2025-10-16 22:41:47|['learn_db.mart_student_lesson.order_projection']                         |
68dbf470-e043-4247-8145-f10081ce1c3b|SELECT subject_name, count(), avg(mark) ¶FROM learn_db.mart_student_lesson ¶WHERE subject_name = 'Математика' AND lesson_date >= '2023-01-01' GROUP BY subject_name;                                                                                           |2025-10-16 22:41:37|['learn_db.mart_student_lesson.order_projection']                         |
 
```

```sql
SELECT SUM(elapsed_us) FROM system.processors_profile_log WHERE query_id = '68dbf470-e043-4247-8145-f10081ce1c3b';
```

Результат:
```text
SUM(elapsed_us)|
---------------+
          42663|
```

**Вывод:** проекция позволила сократить объем данных прочитанных с диска и ускорить время выполнения запроса.