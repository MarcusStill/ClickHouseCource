## Задание

Создайте таблицу MART_STUDENT_LESSON. Выполните оптимизацию этой таблицы:
* о крайней мере у одного поля измените тип данных с целью уменьшения занимаемого места;
* примените по крайней мере к одному столбцу LowCardinality;
* по крайней мере у одного столбца измените кодировку.

### 1. Создание и наполнение таблицы MART_STUDENT_LESSON

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson;
CREATE TABLE learn_db.mart_student_lesson
(
    -- Идентификаторы студентов
    `student_profile_id` Int64,                    -- Идентификатор профиля обучающегося
    `person_id` String,                           -- GUID обучающегося
    `person_id_int` Int64,
    
    -- Образовательная организация и класс
    `educational_organization_id` Int16,          -- Идентификатор образовательной организации
    `parallel_id` Int16,
    `class_id` Int16,                             -- Идентификатор класса
    
    -- Временные метки урока
    `lesson_date` Date32,                         -- Дата урока
    `lesson_month_digits` String,
    `lesson_month_text` String,
    `lesson_year` UInt16,
    `load_date` Date,                             -- Дата загрузки данных
    
    -- Метаданные урока
    `t` Int16 CODEC(Delta, ZSTD),
    `teacher_id` Int32,                           -- Идентификатор учителя
    `subject_id` Int16,                           -- Идентификатор предмета
    `subject_name` String,
    
    -- Оценка
    `mark` Nullable(UInt8),                       -- Оценка
    
    PRIMARY KEY (lesson_date)
) 
ENGINE = MergeTree()

-- =============================================
-- НАПОЛНЕНИЕ ТАБЛИЦЫ ТЕСТОВЫМИ ДАННЫМИ
-- =============================================
AS SELECT
    -- Генерация идентификаторов студентов
    floor(randUniform(2, 10000000)) as student_profile_id,
    cast(student_profile_id as String) as person_id,
    cast(person_id as Int32) as person_id_int,
    
    -- Генерация данных об организации и классе
    student_profile_id / 365000 as educational_organization_id,
    student_profile_id / 73000 as parallel_id,
    student_profile_id / 2000 as class_id,
    
    -- Генерация дат уроков
    cast(now() - randUniform(2, 60*60*24*365) as date) as lesson_date,  -- Дата урока
    formatDateTime(lesson_date, '%Y-%m') as lesson_month_digits,
    formatDateTime(lesson_date, '%Y %M') AS lesson_month_text,
    toYear(lesson_date) as lesson_year, 
    lesson_date + rand() % 3 as load_date,                              -- Дата загрузки данных
    
    -- Генерация метаданных
    floor(randUniform(2, 137)) as t,
    educational_organization_id * 136 + t as teacher_id,
    floor(t/9) as subject_id,
    
    -- Названия предметов
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
    
    -- Генерация оценок
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

### 2. Оптимизация хранения данных

###  2.1 Сделаем замеры производительности исходной таблицы

### 2.2. Смотрим на размер столбцов

```sql
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'mart_student_lesson'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

Результат
```text
name                       |compressed_size|uncompressed_size|ratio |compressed|uncompressed|
---------------------------+---------------+-----------------+------+----------+------------+
person_id                  |72.35 MiB      |75.24 MiB        |  1.04|  75866507|    78890286|
person_id_int              |47.89 MiB      |76.29 MiB        |  1.59|  50215837|    80000000|
student_profile_id         |47.89 MiB      |76.29 MiB        |  1.59|  50215837|    80000000|
teacher_id                 |27.78 MiB      |38.15 MiB        |  1.37|  29133482|    40000000|
subject_name               |26.93 MiB      |206.66 MiB       |  7.67|  28234426|   216694346|
class_id                   |19.16 MiB      |19.07 MiB        |   1.0|  20086382|    20000000|
parallel_id                |15.82 MiB      |19.07 MiB        |  1.21|  16591105|    20000000|
educational_organization_id|14.06 MiB      |19.07 MiB        |  1.36|  14744623|    20000000|
subject_id                 |13.83 MiB      |19.07 MiB        |  1.38|  14497700|    20000000|
t                          |12.80 MiB      |19.07 MiB        |  1.49|  13423118|    20000000|
mark                       |12.21 MiB      |19.07 MiB        |  1.56|  12801039|    20000000|
load_date                  |11.45 MiB      |19.07 MiB        |  1.67|  12002923|    20000000|
lesson_month_text          |521.59 KiB     |115.85 MiB       |227.45|    534110|   121481672|
lesson_month_digits        |356.75 KiB     |76.29 MiB        |218.99|    365313|    80000000|
lesson_date                |182.38 KiB     |38.15 MiB        |214.18|    186755|    40000000|
lesson_year                |87.37 KiB      |19.07 MiB        |223.55|     89467|    20000000|
```


### 2.3. Смотрим общий размер таблицы

```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'mart_student_lesson'

```

Результат
```text
compressed_size|uncompressed_size|ratio|compressed|uncompressed|
---------------+-----------------+-----+----------+------------+
323.28 MiB     |855.51 MiB       | 2.65| 338988624|   897066304|
```

### 2.4. Выполняем диагностический запрос

```sql
SELECT
    lesson_year,
    lesson_month_digits,
    educational_organization_id,
    parallel_id,
    subject_name,
    COUNT(*) as total_lessons,
    COUNT(mark) as lessons_with_marks,
    AVG(mark) as avg_mark,
    COUNT(DISTINCT student_profile_id) as unique_students,
    COUNT(DISTINCT teacher_id) as unique_teachers
FROM learn_db.mart_student_lesson
WHERE lesson_date BETWEEN '2025-09-01' AND '2025-09-30'
	AND parallel_id BETWEEN 5 AND 11
	AND subject_id IN (1, 2, 4, 7)
GROUP BY
    lesson_year,
    lesson_month_digits,
    educational_organization_id,
    parallel_id,
    subject_name
ORDER BY
    lesson_year DESC,
    lesson_month_digits DESC,
    educational_organization_id,
    parallel_id,
    avg_mark DESC
```

Результат
```text
    Query id: e83101ab-6559-401f-87b7-84718885be9f

    ┌─lesson_year─┬─lesson_month_digits─┬─educational_organization_id─┬─parallel_id─┬─subject_name─┬─total_lessons─┬─lessons_with_marks─┬───────────avg_mark─┬─unique_students─┬─unique_teachers─┐
 1. │        2025 │ 2025-09             │                           1 │           5 │ Математика   │           386 │                172 │ 4.3604651162790695 │             386 │              36 │
 2. │        2025 │ 2025-09             │                           1 │           5 │ Русский язык │           392 │                188 │  4.281914893617022 │             392 │              35 │
 3. │        2025 │ 2025-09             │                           1 │           5 │ Физика       │           374 │                183 │   3.92896174863388 │             372 │              35 │
 4. │        2025 │ 2025-09             │                           1 │           5 │ Биология     │           411 │                184 │  3.739130434782609 │             410 │              34 │
 5. │        2025 │ 2025-09             │                           1 │           6 │ Математика   │           399 │                181 │  4.425414364640884 │             398 │              36 │
 6. │        2025 │ 2025-09             │                           1 │           6 │ Русский язык │           400 │                208 │  4.235576923076923 │             400 │              35 │
 7. │        2025 │ 2025-09             │                           1 │           6 │ Физика       │           371 │                175 │  4.005714285714285 │             370 │              34 │
 8. │        2025 │ 2025-09             │                           1 │           6 │ Биология     │           406 │                208 │              3.625 │             405 │              35 │
 9. │        2025 │ 2025-09             │                           1 │           7 │ Математика   │           407 │                201 │    4.3681592039801 │             406 │              35 │
10. │        2025 │ 2025-09             │                           1 │           7 │ Русский язык │           441 │                236 │  4.271186440677966 │             439 │              35 │
11. │        2025 │ 2025-09             │                           1 │           7 │ Физика       │           436 │                198 │   4.05050505050505 │             434 │              34 │
12. │        2025 │ 2025-09             │                           1 │           7 │ Биология     │           404 │                205 │  3.726829268292683 │             402 │              35 │
13. │        2025 │ 2025-09             │                           1 │           8 │ Математика   │           405 │                194 │  4.293814432989691 │             405 │              35 │
14. │        2025 │ 2025-09             │                           1 │           8 │ Русский язык │           428 │                211 │ 4.2274881516587675 │             428 │              35 │
15. │        2025 │ 2025-09             │                           1 │           8 │ Физика       │           399 │                229 │  4.048034934497816 │             399 │              34 │
16. │        2025 │ 2025-09             │                           1 │           8 │ Биология     │           395 │                196 │  3.627551020408163 │             394 │              34 │
17. │        2025 │ 2025-09             │                           1 │           9 │ Математика   │           387 │                182 │  4.285714285714286 │             387 │              34 │
18. │        2025 │ 2025-09             │                           1 │           9 │ Русский язык │           439 │                221 │  4.285067873303167 │             439 │              34 │
19. │        2025 │ 2025-09             │                           1 │           9 │ Физика       │           447 │                233 │  4.017167381974249 │             446 │              35 │
20. │        2025 │ 2025-09             │                           1 │           9 │ Биология     │           407 │                203 │  3.773399014778325 │             405 │              35 │
21. │        2025 │ 2025-09             │                           2 │          10 │ Математика   │           417 │                198 │  4.343434343434343 │             415 │              36 │
22. │        2025 │ 2025-09             │                           2 │          10 │ Русский язык │           391 │                205 │  4.195121951219512 │             391 │              36 │
23. │        2025 │ 2025-09             │                           2 │          10 │ Физика       │           400 │                187 │                  4 │             400 │              35 │
24. │        2025 │ 2025-09             │                           2 │          10 │ Биология     │           412 │                203 │  3.600985221674877 │             411 │              35 │
25. │        2025 │ 2025-09             │                           2 │          11 │ Математика   │           373 │                187 │  4.342245989304812 │             373 │              36 │
26. │        2025 │ 2025-09             │                           2 │          11 │ Русский язык │           413 │                196 │  4.229591836734694 │             411 │              35 │
27. │        2025 │ 2025-09             │                           2 │          11 │ Физика       │           392 │                205 │  4.092682926829268 │             392 │              34 │
28. │        2025 │ 2025-09             │                           2 │          11 │ Биология     │           404 │                193 │ 3.6632124352331608 │             404 │              35 │
    └─────────────┴─────────────────────┴─────────────────────────────┴─────────────┴──────────────┴───────────────┴────────────────────┴────────────────────┴─────────────────┴─────────────────┘

28 rows in set. Elapsed: 0.056 sec. Processed 851.97 thousand rows, 61.05 MB (15.11 million rows/s., 1.08 GB/s.)
Peak memory usage: 36.39 MiB.
```

## Оптимизация 

### 2.5. Проверим максимальные и минимальные значения в столбцах

```sql
SELECT 
    'student_profile_id' as column_name,
    toString(min(student_profile_id)) as min_value,
    toString(max(student_profile_id)) as max_value
FROM learn_db.mart_student_lesson

UNION ALL 

SELECT 
    'person_id_int',
    toString(min(person_id_int)),
    toString(max(person_id_int))
FROM learn_db.mart_student_lesson

UNION ALL 

SELECT 
    'educational_organization_id',
    toString(min(educational_organization_id)),
    toString(max(educational_organization_id))
FROM learn_db.mart_student_lesson

UNION ALL 

SELECT 
    'parallel_id',
    toString(min(parallel_id)),
    toString(max(parallel_id))
FROM learn_db.mart_student_lesson

UNION ALL 

SELECT 
    'class_id',
    toString(min(class_id)),
    toString(max(class_id))
FROM learn_db.mart_student_lesson

UNION ALL 

SELECT 
    't',
    toString(min(t)),
    toString(max(t))
FROM learn_db.mart_student_lesson

UNION ALL 

SELECT 
    'teacher_id',
    toString(min(teacher_id)),
    toString(max(teacher_id))
FROM learn_db.mart_student_lesson

UNION ALL 

SELECT 
    'subject_id',
    toString(min(subject_id)),
    toString(max(subject_id))
FROM learn_db.mart_student_lesson

UNION ALL 

SELECT 
    'mark',
    toString(min(mark)),
    toString(max(mark))
FROM learn_db.mart_student_lesson;
```

Результат
```text
column_name                |min_value|max_value|
---------------------------+---------+---------+
student_profile_id         |2        |9999999  |
person_id_int              |2        |9999999  |
parallel_id                |0        |136      |
educational_organization_id|0        |27       |
class_id                   |0        |4999     |
t                          |2        |136      |
teacher_id                 |2        |3861     |
subject_id                 |0        |15       |
mark                       |2        |5        |
```

### 2.6. Пересоздадим таблицу, изменив типы данных

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_optimized_1;
CREATE TABLE learn_db.mart_student_lesson_optimized_1
(
    `student_profile_id` UInt32,
    `person_id` String,
    `person_id_int` UInt32,
     `educational_organization_id` UInt8,
    `parallel_id` UInt8,
    `class_id` UInt16,
    `lesson_date` Date,
    `lesson_month_digits` LowCardinality(String),
    `lesson_month_text` LowCardinality(String),
    `lesson_year` UInt16,
    `load_date` Date,
    
    -- Метаданные урока
    `t` UInt8,
    `teacher_id` UInt16,
    `subject_id` UInt8,
    `subject_name` Enum8(
        'Математика' = 1,
        'Русский язык' = 2,
        'Литература' = 3,
        'Физика' = 4,
        'Химия' = 5,
        'География' = 6,
        'Биология' = 7,
        'Физическая культура' = 8,
        'Информатика' = 9
    ),
    `mark` Nullable(UInt8),
    
    PRIMARY KEY (lesson_date)
) 
ENGINE = MergeTree() AS 
SELECT 
	*
FROM
	learn_db.mart_student_lesson;
```

### 2.7. Проверим размер столбцов в новой таблице mart_student_lesson_optimized_1

```sql
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'mart_student_lesson_optimized_1'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

Результат
```text
name                       |compressed_size|uncompressed_size|ratio |compressed|uncompressed|
---------------------------+---------------+-----------------+------+----------+------------+
person_id                  |72.34 MiB      |75.24 MiB        |  1.04|  75856420|    78890286|
person_id_int              |38.31 MiB      |38.15 MiB        |   1.0|  40172642|    40000000|
student_profile_id         |38.31 MiB      |38.15 MiB        |   1.0|  40172642|    40000000|
class_id                   |19.16 MiB      |19.07 MiB        |   1.0|  20086362|    20000000|
teacher_id                 |19.16 MiB      |19.07 MiB        |   1.0|  20086360|    20000000|
mark                       |12.00 MiB      |19.07 MiB        |  1.59|  12586018|    20000000|
load_date                  |11.26 MiB      |19.07 MiB        |  1.69|  11809180|    20000000|
t                          |9.58 MiB       |9.54 MiB         |   1.0|  10043227|    10000000|
parallel_id                |9.58 MiB       |9.54 MiB         |   1.0|  10043226|    10000000|
educational_organization_id|9.56 MiB       |9.54 MiB         |   1.0|  10026133|    10000000|
subject_id                 |8.87 MiB       |9.54 MiB         |  1.07|   9305265|    10000000|
subject_name               |6.75 MiB       |9.54 MiB         |  1.41|   7082829|    10000000|
lesson_date                |91.02 KiB      |19.07 MiB        |214.59|     93203|    20000000|
lesson_year                |87.35 KiB      |19.07 MiB        |223.61|     89443|    20000000|
lesson_month_text          |54.03 KiB      |9.56 MiB         |181.27|     55322|    10028270|
lesson_month_digits        |53.80 KiB      |9.56 MiB         |182.01|     55096|    10028053|
```

### 2.8. Получим общий размер таблицы mart_student_lesson_optimized_1

```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'mart_student_lesson_optimized_1'
```

Результат
```text
compressed_size|uncompressed_size|ratio|compressed|uncompressed|
---------------+-----------------+-----+----------+------------+
255.17 MiB     |332.78 MiB       |  1.3| 267563368|   348946609|
```

### 2.9. Выполним диагностический запрос к таблице mart_student_lesson_optimized_1

```sql
SELECT
    lesson_year,
    lesson_month_digits,
    educational_organization_id,
    parallel_id,
    subject_name,
    COUNT(*) as total_lessons,
    COUNT(mark) as lessons_with_marks,
    AVG(mark) as avg_mark,
    COUNT(DISTINCT student_profile_id) as unique_students,
    COUNT(DISTINCT teacher_id) as unique_teachers
FROM learn_db.mart_student_lesson_optimized_1
WHERE lesson_date BETWEEN '2025-09-01' AND '2025-09-30'
	AND parallel_id BETWEEN 5 AND 11
	AND subject_id IN (1, 2, 4, 7)
GROUP BY
    lesson_year,
    lesson_month_digits,
    educational_organization_id,
    parallel_id,
    subject_name
ORDER BY
    lesson_year DESC,
    lesson_month_digits DESC,
    educational_organization_id,
    parallel_id,
    avg_mark DESC
```

Результат
```text
Query id: 697346d8-456b-4cd9-9b1a-c97d0f739e29

    ┌─lesson_year─┬─lesson_month_digits─┬─educational_organization_id─┬─parallel_id─┬─subject_name─┬─total_lessons─┬─lessons_with_marks─┬───────────avg_mark─┬─unique_students─┬─unique_teachers─┐
 1. │        2025 │ 2025-09             │                           1 │           5 │ Математика   │           386 │                172 │ 4.3604651162790695 │             386 │              36 │
 2. │        2025 │ 2025-09             │                           1 │           5 │ Русский язык │           392 │                188 │  4.281914893617022 │             392 │              35 │
 3. │        2025 │ 2025-09             │                           1 │           5 │ Физика       │           374 │                183 │   3.92896174863388 │             372 │              35 │
 4. │        2025 │ 2025-09             │                           1 │           5 │ Биология     │           411 │                184 │  3.739130434782609 │             410 │              34 │
 5. │        2025 │ 2025-09             │                           1 │           6 │ Математика   │           399 │                181 │  4.425414364640884 │             398 │              36 │
 6. │        2025 │ 2025-09             │                           1 │           6 │ Русский язык │           400 │                208 │  4.235576923076923 │             400 │              35 │
 7. │        2025 │ 2025-09             │                           1 │           6 │ Физика       │           371 │                175 │  4.005714285714285 │             370 │              34 │
 8. │        2025 │ 2025-09             │                           1 │           6 │ Биология     │           406 │                208 │              3.625 │             405 │              35 │
 9. │        2025 │ 2025-09             │                           1 │           7 │ Математика   │           407 │                201 │    4.3681592039801 │             406 │              35 │
10. │        2025 │ 2025-09             │                           1 │           7 │ Русский язык │           441 │                236 │  4.271186440677966 │             439 │              35 │
11. │        2025 │ 2025-09             │                           1 │           7 │ Физика       │           436 │                198 │   4.05050505050505 │             434 │              34 │
12. │        2025 │ 2025-09             │                           1 │           7 │ Биология     │           404 │                205 │  3.726829268292683 │             402 │              35 │
13. │        2025 │ 2025-09             │                           1 │           8 │ Математика   │           405 │                194 │  4.293814432989691 │             405 │              35 │
14. │        2025 │ 2025-09             │                           1 │           8 │ Русский язык │           428 │                211 │ 4.2274881516587675 │             428 │              35 │
15. │        2025 │ 2025-09             │                           1 │           8 │ Физика       │           399 │                229 │  4.048034934497816 │             399 │              34 │
16. │        2025 │ 2025-09             │                           1 │           8 │ Биология     │           395 │                196 │  3.627551020408163 │             394 │              34 │
17. │        2025 │ 2025-09             │                           1 │           9 │ Математика   │           387 │                182 │  4.285714285714286 │             387 │              34 │
18. │        2025 │ 2025-09             │                           1 │           9 │ Русский язык │           439 │                221 │  4.285067873303167 │             439 │              34 │
19. │        2025 │ 2025-09             │                           1 │           9 │ Физика       │           447 │                233 │  4.017167381974249 │             446 │              35 │
20. │        2025 │ 2025-09             │                           1 │           9 │ Биология     │           407 │                203 │  3.773399014778325 │             405 │              35 │
21. │        2025 │ 2025-09             │                           2 │          10 │ Математика   │           417 │                198 │  4.343434343434343 │             415 │              36 │
22. │        2025 │ 2025-09             │                           2 │          10 │ Русский язык │           391 │                205 │  4.195121951219512 │             391 │              36 │
23. │        2025 │ 2025-09             │                           2 │          10 │ Физика       │           400 │                187 │                  4 │             400 │              35 │
24. │        2025 │ 2025-09             │                           2 │          10 │ Биология     │           412 │                203 │  3.600985221674877 │             411 │              35 │
25. │        2025 │ 2025-09             │                           2 │          11 │ Математика   │           373 │                187 │  4.342245989304812 │             373 │              36 │
26. │        2025 │ 2025-09             │                           2 │          11 │ Русский язык │           413 │                196 │  4.229591836734694 │             411 │              35 │
27. │        2025 │ 2025-09             │                           2 │          11 │ Физика       │           392 │                205 │  4.092682926829268 │             392 │              34 │
28. │        2025 │ 2025-09             │                           2 │          11 │ Биология     │           404 │                193 │ 3.6632124352331608 │             404 │              35 │
    └─────────────┴─────────────────────┴─────────────────────────────┴─────────────┴──────────────┴───────────────┴────────────────────┴────────────────────┴─────────────────┴─────────────────┘

28 rows in set. Elapsed: 0.017 sec. Processed 868.35 thousand rows, 14.76 MB (52.37 million rows/s., 890.22 MB/s.)
Peak memory usage: 13.39 MiB.
```

### 2.10. Пересоздадим таблицу, добавив сжатие

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_optimized_2;
CREATE TABLE learn_db.mart_student_lesson_optimized_2
(
    `student_profile_id` UInt32 CODEC(Delta, ZSTD),
    `person_id` String,
    `person_id_int` UInt32 CODEC(Delta, ZSTD),
    `educational_organization_id` UInt8 CODEC(Delta, ZSTD),
    `parallel_id` UInt8 CODEC(Delta, ZSTD),
    `class_id` UInt16 CODEC(Delta, ZSTD),
    `lesson_date` Date,
    `lesson_month_digits` LowCardinality(String),
    `lesson_month_text` LowCardinality(String),
    `lesson_year` UInt16,
    `load_date` Date,
    `t` UInt8 CODEC(Delta, ZSTD),
    `teacher_id` UInt16 CODEC(Delta, ZSTD),
    `subject_id` UInt8 CODEC(Delta, ZSTD),
    `subject_name` Enum8(
        'Математика' = 1,
        'Русский язык' = 2,
        'Литература' = 3,
        'Физика' = 4,
        'Химия' = 5,
        'География' = 6,
        'Биология' = 7,
        'Физическая культура' = 8,
        'Информатика' = 9
    ) CODEC(Delta, ZSTD),
    `mark` Nullable(UInt8) CODEC(Delta, ZSTD),
    
    PRIMARY KEY (lesson_date)
) 
ENGINE = MergeTree() AS 
SELECT 
	*
FROM
	learn_db.mart_student_lesson;
```

### 2.11. Проверим размер столбцов в новой таблице mart_student_lesson_optimized_2

```sql
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'mart_student_lesson_optimized_2'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

Результат
```text
name                       |compressed_size|uncompressed_size|ratio |compressed|uncompressed|
---------------------------+---------------+-----------------+------+----------+------------+
person_id                  |72.33 MiB      |75.24 MiB        |  1.04|  75842226|    78890286|
person_id_int              |33.60 MiB      |38.15 MiB        |  1.14|  35229131|    40000000|
student_profile_id         |33.60 MiB      |38.15 MiB        |  1.14|  35229131|    40000000|
class_id                   |17.25 MiB      |19.07 MiB        |  1.11|  18091908|    20000000|
teacher_id                 |16.86 MiB      |19.07 MiB        |  1.13|  17679226|    20000000|
load_date                  |11.20 MiB      |19.07 MiB        |   1.7|  11747919|    20000000|
parallel_id                |9.50 MiB       |9.54 MiB         |   1.0|   9963834|    10000000|
t                          |9.46 MiB       |9.54 MiB         |  1.01|   9918345|    10000000|
educational_organization_id|6.60 MiB       |9.54 MiB         |  1.45|   6919698|    10000000|
subject_id                 |5.57 MiB       |9.54 MiB         |  1.71|   5842593|    10000000|
mark                       |5.06 MiB       |19.07 MiB        |  3.77|   5304993|    20000000|
subject_name               |4.50 MiB       |9.54 MiB         |  2.12|   4713777|    10000000|
lesson_date                |90.87 KiB      |19.07 MiB        |214.94|     93049|    20000000|
lesson_year                |87.38 KiB      |19.07 MiB        |223.53|     89472|    20000000|
lesson_month_text          |54.11 KiB      |9.56 MiB         |180.99|     55407|    10028255|
lesson_month_digits        |53.89 KiB      |9.56 MiB         |181.71|     55187|    10028045|
```

### 2.12. Получим общий размер таблицы mart_student_lesson_optimized_2

```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'mart_student_lesson_optimized_2'
```

Результат
```text
compressed_size|uncompressed_size|ratio|compressed|uncompressed|
---------------+-----------------+-----+----------+------------+
225.81 MiB     |332.78 MiB       | 1.47| 236775896|   348946586|
```

### 2.13. Выполним диагностический запрос к таблице mart_student_lesson_optimized_2

```sql
SELECT
    lesson_year,
    lesson_month_digits,
    educational_organization_id,
    parallel_id,
    subject_name,
    COUNT(*) as total_lessons,
    COUNT(mark) as lessons_with_marks,
    AVG(mark) as avg_mark,
    COUNT(DISTINCT student_profile_id) as unique_students,
    COUNT(DISTINCT teacher_id) as unique_teachers
FROM learn_db.mart_student_lesson_optimized_2
WHERE lesson_date BETWEEN '2025-09-01' AND '2025-09-30'
	AND parallel_id BETWEEN 5 AND 11
	AND subject_id IN (1, 2, 4, 7)
GROUP BY
    lesson_year,
    lesson_month_digits,
    educational_organization_id,
    parallel_id,
    subject_name
ORDER BY
    lesson_year DESC,
    lesson_month_digits DESC,
    educational_organization_id,
    parallel_id,
    avg_mark DESC
```

Результат
```text
Query id: 75a6a1a1-1475-4415-8411-4aebe810588c

    ┌─lesson_year─┬─lesson_month_digits─┬─educational_organization_id─┬─parallel_id─┬─subject_name─┬─total_lessons─┬─lessons_with_marks─┬───────────avg_mark─┬─unique_students─┬─unique_teachers─┐
 1. │        2025 │ 2025-09             │                           1 │           5 │ Математика   │           386 │                172 │ 4.3604651162790695 │             386 │              36 │
 2. │        2025 │ 2025-09             │                           1 │           5 │ Русский язык │           392 │                188 │  4.281914893617022 │             392 │              35 │
 3. │        2025 │ 2025-09             │                           1 │           5 │ Физика       │           374 │                183 │   3.92896174863388 │             372 │              35 │
 4. │        2025 │ 2025-09             │                           1 │           5 │ Биология     │           411 │                184 │  3.739130434782609 │             410 │              34 │
 5. │        2025 │ 2025-09             │                           1 │           6 │ Математика   │           399 │                181 │  4.425414364640884 │             398 │              36 │
 6. │        2025 │ 2025-09             │                           1 │           6 │ Русский язык │           400 │                208 │  4.235576923076923 │             400 │              35 │
 7. │        2025 │ 2025-09             │                           1 │           6 │ Физика       │           371 │                175 │  4.005714285714285 │             370 │              34 │
 8. │        2025 │ 2025-09             │                           1 │           6 │ Биология     │           406 │                208 │              3.625 │             405 │              35 │
 9. │        2025 │ 2025-09             │                           1 │           7 │ Математика   │           407 │                201 │    4.3681592039801 │             406 │              35 │
10. │        2025 │ 2025-09             │                           1 │           7 │ Русский язык │           441 │                236 │  4.271186440677966 │             439 │              35 │
11. │        2025 │ 2025-09             │                           1 │           7 │ Физика       │           436 │                198 │   4.05050505050505 │             434 │              34 │
12. │        2025 │ 2025-09             │                           1 │           7 │ Биология     │           404 │                205 │  3.726829268292683 │             402 │              35 │
13. │        2025 │ 2025-09             │                           1 │           8 │ Математика   │           405 │                194 │  4.293814432989691 │             405 │              35 │
14. │        2025 │ 2025-09             │                           1 │           8 │ Русский язык │           428 │                211 │ 4.2274881516587675 │             428 │              35 │
15. │        2025 │ 2025-09             │                           1 │           8 │ Физика       │           399 │                229 │  4.048034934497816 │             399 │              34 │
16. │        2025 │ 2025-09             │                           1 │           8 │ Биология     │           395 │                196 │  3.627551020408163 │             394 │              34 │
17. │        2025 │ 2025-09             │                           1 │           9 │ Математика   │           387 │                182 │  4.285714285714286 │             387 │              34 │
18. │        2025 │ 2025-09             │                           1 │           9 │ Русский язык │           439 │                221 │  4.285067873303167 │             439 │              34 │
19. │        2025 │ 2025-09             │                           1 │           9 │ Физика       │           447 │                233 │  4.017167381974249 │             446 │              35 │
20. │        2025 │ 2025-09             │                           1 │           9 │ Биология     │           407 │                203 │  3.773399014778325 │             405 │              35 │
21. │        2025 │ 2025-09             │                           2 │          10 │ Математика   │           417 │                198 │  4.343434343434343 │             415 │              36 │
22. │        2025 │ 2025-09             │                           2 │          10 │ Русский язык │           391 │                205 │  4.195121951219512 │             391 │              36 │
23. │        2025 │ 2025-09             │                           2 │          10 │ Физика       │           400 │                187 │                  4 │             400 │              35 │
24. │        2025 │ 2025-09             │                           2 │          10 │ Биология     │           412 │                203 │  3.600985221674877 │             411 │              35 │
25. │        2025 │ 2025-09             │                           2 │          11 │ Математика   │           373 │                187 │  4.342245989304812 │             373 │              36 │
26. │        2025 │ 2025-09             │                           2 │          11 │ Русский язык │           413 │                196 │  4.229591836734694 │             411 │              35 │
27. │        2025 │ 2025-09             │                           2 │          11 │ Физика       │           392 │                205 │  4.092682926829268 │             392 │              34 │
28. │        2025 │ 2025-09             │                           2 │          11 │ Биология     │           404 │                193 │ 3.6632124352331608 │             404 │              35 │
    └─────────────┴─────────────────────┴─────────────────────────────┴─────────────┴──────────────┴───────────────┴────────────────────┴────────────────────┴─────────────────┴─────────────────┘

28 rows in set. Elapsed: 0.020 sec. Processed 868.35 thousand rows, 14.76 MB (44.46 million rows/s., 755.88 MB/s.)
Peak memory usage: 11.60 MiB.
```

## Сравнительный анализ результатов 

### 2.14. Эффективность сжатия данных

Результат
```text
Таблица	        Сжатый размер	Исходный размер	    Коэффициент сжатия
Исходная	    323.28 MiB	    855.51 MiB	        2.65x
Оптимизация 1	255.17 MiB	    332.78 MiB	        1.30x
Оптимизация 2	225.81 MiB	    332.78 MiB	        1.47x
```

### 2.15. Анализ по столбцам

Результат
```text

Столбец	                    Исходный	    Оптимизация 1	    Оптимизация 2
subject_name	            26.93 MiB   →   6.75 MiB        →   4.50 MiB
mark	                    12.21 MiB   →   12.00 MiB       →   5.06 MiB
teacher_id	                27.78 MiB   →   19.16 MiB       →   16.86 MiB
person_id_int	            47.89 MiB   →   38.31 MiB       →   33.60 MiB
educational_organization_id	14.06 MiB   →   9.56 MiB        →   6.60 MiB

```

### 2.16. Производительность запросов

Результат
```text

Метрика	                Исходная таблица	    Оптимизация 1	        Оптимизация 2
Время выполнения	    0.056 sec	            0.017 sec	            0.020 sec
Обработано строк	    851.97 thousand	rows    868.35 thousand	rows    868.35 thousand rows
Пропускная способность	15.11 million rows/s.	52.37 million rows/s.	44.46 million rows/s.
Память	                36.39 MiB	            13.39 MiB	            11.60 MiB
```

Применение кодеков для сжатия позволило существенно сократить объем данных на жестком диске. Однако, таблица без сжатия показала наилучшую производительность при выполнении тестового запроса.  