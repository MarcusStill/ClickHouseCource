## Оптимизация запроса.

### Исходные данные
Структура таблицы mart_student_lesson

#### Создаем таблицу СТУДЕНЧЕСКИХ УРОКОВ
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson;

CREATE TABLE learn_db.mart_student_lesson
(
    `student_profile_id` Int32,      -- Идентификатор профиля обучающегося
    `person_id` String,               -- GUID обучающегося
    `person_id_int` Int32 CODEC(Delta, ZSTD),
    `educational_organization_id` Int16, -- Идентификатор образовательной организации
    `parallel_id` Int16,
    `class_id` Int16,                 -- Идентификатор класса
    `lesson_date` Date32,             -- Дата урока
    `lesson_month_digits` String,
    `lesson_month_text` String,
    `lesson_year` UInt16,
    `load_date` Date,                 -- Дата загрузки данных
    `t` Int16 CODEC(Delta, ZSTD),
    `teacher_id` Int32 CODEC(Delta, ZSTD), -- Идентификатор учителя
    `subject_id` Int16 CODEC(Delta, ZSTD), -- Идентификатор предмета
    `subject_name` String,
    `mark` Nullable(UInt8),           -- Оценка
    PRIMARY KEY(lesson_date)
) ENGINE = MergeTree()
AS SELECT
    floor(randUniform(2, 10000000)) as student_profile_id,
    cast(student_profile_id as String) as person_id,
    cast(person_id as Int32) as person_id_int,
    student_profile_id / 365000 as educational_organization_id,
    student_profile_id / 73000 as parallel_id,
    student_profile_id / 2000 as class_id,
    cast(now() - randUniform(2, 60*60*24*365) as date) as lesson_date,
    formatDateTime(lesson_date, '%Y-%m') as lesson_month_digits,
    formatDateTime(lesson_date, '%Y %M') AS lesson_month_text,
    toYear(lesson_date) as lesson_year, 
    lesson_date + rand() % 3 as load_date,
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

#### Целевой запрос для оптимизации

```sql
SELECT 
    teacher_id,
    ROUND(AVG(mark), 2) as avg_mark
FROM learn_db.mart_student_lesson
WHERE 
    subject_id = 3
    AND educational_organization_id = 10
GROUP BY teacher_id
ORDER BY avg_mark DESC;
```

## Задание:
Оптимизируйте запрос. В качестве результате напишите все диагностические запросы, которые вы применяли для поиска возможных оптимизаций. 
Также напишите выводы, которые вы сделали с помощью результатов диагностических запросов.

### 1. Выполним исходный запрос

```sql
SELECT 
    teacher_id,
    ROUND(AVG(mark), 2) as avg_mark
FROM learn_db.mart_student_lesson
WHERE 
    subject_id = 3
    AND educational_organization_id = 10
GROUP BY teacher_id
ORDER BY avg_mark DESC; -- 0.062s
```

Результат:
```text
Query id: 10d9f105-d24e-4eef-b555-6c1a02213a2a

     ┌─teacher_id─┬─avg_mark─┐
  1. │       1530 │     4.57 │
  2. │       1525 │     4.35 │
  3. │       1409 │     4.34 │
  4. │       1443 │     4.33 │
  5. │       1471 │     4.33 │
144. │       1504 │     3.91 │
     └─teacher_id─┴─avg_mark─┘

144 rows in set. Elapsed: 0.103 sec. Processed 10.00 million rows, 100.00 MB (97.32 million rows/s., 973.16 MB/s.)
Peak memory usage: 5.80 MiB.
```

### 2. Находим query_id

```sql
SELECT
	query_id,
	query,
	query_duration_ms,
	read_rows,
	read_bytes,
	memory_usage,
	ProfileEvents,
	event_time
FROM system.query_log
WHERE
	query LIKE '%mart_student_lesson%'
ORDER BY event_time DESC
```

Результат:
```text
4abc0efc-ddd4-4fb6-9d6f-d9a50ddd9cbb
```

### 3. Идем в system.query_log

```sql
SELECT * FROM system.processors_profile_log 
WHERE query_id = '4abc0efc-ddd4-4fb6-9d6f-d9a50ddd9cbb'
order by processor_uniq_id;
```

Находим самые продолжительные операции (elapsed_us):
```text
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	56575
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	66732
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	61615
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	60728
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	63927
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	58725
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	61360
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	61328
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	62288
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	53915
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	62689
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	61090
```

### 4. Анализируем использование

```sql
EXPLAIN indexes = 1
SELECT 
    teacher_id,
    ROUND(AVG(mark), 2) as avg_mark
FROM learn_db.mart_student_lesson
WHERE 
    subject_id = 3
    AND educational_organization_id = 10
GROUP BY teacher_id
ORDER BY avg_mark DESC;
```

Результат:
```text
Parts: 4/4
Granules: 1223/1223
```

Неоптимальное чтение данных. В таблице первичный ключ по полю lesson_date. Высокая кардинальность.

**Варианты экспресс-оптимизации для уменьшения объема чтения данных с жесткого диска:**
- пересоздать таблицу с другим первичным индексом;
- использовать Projections с другим первичным индексом;
- поменять первичные индексы/дополнить;
- создать индекс пропуска данных.

Т.к. на данном этапе у нас нет ограничений, воспользуемся самым простым решением.

### 5. Проверим кардинальность по атрибутам, используемым в условии фильтрации

```sql
SELECT 
    uniqExact(subject_id) as subject_id, -- 16
    uniqExact(educational_organization_id) as educational_organization_id -- 28
FROM learn_db.mart_student_lesson;
```

Результат:
```text
subject_id: 16
educational_organization_id: 28
```

### 6. Пересоздадим таблицу изменив первичный ключ

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson;

CREATE TABLE learn_db.mart_student_lesson
(
    `student_profile_id` Int32,      -- Идентификатор профиля обучающегося
    `person_id` String,               -- GUID обучающегося
    `person_id_int` Int32 CODEC(Delta, ZSTD),
    `educational_organization_id` Int16, -- Идентификатор образовательной организации
    `parallel_id` Int16,
    `class_id` Int16,                 -- Идентификатор класса
    `lesson_date` Date32,             -- Дата урока
    `lesson_month_digits` String,
    `lesson_month_text` String,
    `lesson_year` UInt16,
    `load_date` Date,                 -- Дата загрузки данных
    `t` Int16 CODEC(Delta, ZSTD),
    `teacher_id` Int32 CODEC(Delta, ZSTD), -- Идентификатор учителя
    `subject_id` Int16 CODEC(Delta, ZSTD), -- Идентификатор предмета
    `subject_name` String,
    `mark` Nullable(UInt8),           -- Оценка
    PRIMARY KEY(subject_id, educational_organization_id)
) ENGINE = MergeTree()
AS SELECT
    floor(randUniform(2, 10000000)) as student_profile_id,
    cast(student_profile_id as String) as person_id,
    cast(person_id as Int32) as person_id_int,
    student_profile_id / 365000 as educational_organization_id,
    student_profile_id / 73000 as parallel_id,
    student_profile_id / 2000 as class_id,
    cast(now() - randUniform(2, 60*60*24*365) as date) as lesson_date,
    formatDateTime(lesson_date, '%Y-%m') as lesson_month_digits,
    formatDateTime(lesson_date, '%Y %M') AS lesson_month_text,
    toYear(lesson_date) as lesson_year, 
    lesson_date + rand() % 3 as load_date,
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

### 7. Перепроверим explain

```sql
EXPLAIN indexes = 1
SELECT 
    teacher_id,
    ROUND(AVG(mark), 2) as avg_mark
FROM learn_db.mart_student_lesson
WHERE 
    subject_id = 3
    AND educational_organization_id = 10
GROUP BY teacher_id
ORDER BY avg_mark DESC;
```

Результат:
```text
Parts: 4/4
Granules: 6/1223
```

### 8. Перепроверим время выполнения чтения

```sql
SELECT
	query_id,
	query,
	query_duration_ms,
	read_rows,
	read_bytes,
	memory_usage,
	ProfileEvents,
	event_time
FROM system.query_log
WHERE
query LIKE '%mart_student_lesson%'
ORDER BY event_time DESC -- 1032cb93-9339-4283-bdcd-28e8f0aaa2cd
```

### 9. Идем в system.query_log

```sql
SELECT * FROM system.processors_profile_log 
WHERE query_id = '1032cb93-9339-4283-bdcd-28e8f0aaa2cd'
order by processor_uniq_id;
```

Результат:
```text
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	1512
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	1946
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	2562
MergeTreeSelect(pool: ReadPool, algorithm: Thread)	1925
```

### 10. Выполним запрос 

```sql
SELECT 
    teacher_id,
    ROUND(AVG(mark), 2) as avg_mark
FROM learn_db.mart_student_lesson
WHERE 
    subject_id = 3
    AND educational_organization_id = 10
GROUP BY teacher_id
ORDER BY avg_mark DESC;
```

Результат:
```text
Query id: 5f915114-4dfe-41f5-b332-490cf87cb58e

     ┌─teacher_id─┬─avg_mark─┐
  1. │       1530 │      4.6 │
  2. │       1515 │     4.35 │
  3. │       1405 │     4.34 │
  4. │       1436 │     4.33 │
  5. │       1470 │     4.31 │
144. │       1440 │     3.96 │
     └─teacher_id─┴─avg_mark─┘

144 rows in set. Elapsed: 0.010 sec. Processed 49.15 thousand rows, 491.52 KB (4.90 million rows/s., 49.02 MB/s.)
Peak memory usage: 2.48 MiB.
```

**Вывод:** время выполнения сократилось в 10 раз, чтение гранул в 200 раз.