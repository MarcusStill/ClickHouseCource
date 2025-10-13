# Индексы пропуска данных

## Индекс пропуска данных типа minmax

### 1. Создаем таблицы с уроками и оценками учеников
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson;
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
	`mark` Nullable(UInt8), -- Оценка
	PRIMARY KEY(class_id)
) ENGINE = MergeTree();
```

### 2. Вставляем данные в таблицу со сравнением скорости вставки с индексами пропуска данных и без

```sql
INSERT INTO mart_student_lesson
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
	load_date, t, 
	teacher_id, 
	subject_id, 
	subject_name, 
	mark
)
SELECT
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

Результат
```text
Execute time: 7.465s
```

### 3. Получаем данные с фильтрацией по первичному индексу

```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	class_id = 200;
```

Результат
```text
Query id: cf804595-9e43-4f89-bc03-802c58d0ec3a

      ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬───t─┬─teacher_id─┬─subject_id─┬─subject_name────────┬─mark─┐
   1. │             400568 │ 400568    │        400568 │                           1 │           5 │      200 │  2024-10-27 │ 2024-10             │ 2024 October      │        2024 │ 2024-10-28 │ 108 │        257 │         12 │ Информатика         │    2 │
   2. │             401538 │ 401538    │        401538 │                           1 │           5 │      200 │  2025-09-13 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-14 │  35 │        184 │          3 │ Литература          │ ᴺᵁᴸᴸ │
   3. │             400022 │ 400022    │        400022 │                           1 │           5 │      200 │  2024-11-08 │ 2024-11             │ 2024 November     │        2024 │ 2024-11-10 │  83 │        232 │          9 │ Информатика         │ ᴺᵁᴸᴸ │
   4. │             400778 │ 400778    │        400778 │                           1 │           5 │      200 │  2024-10-14 │ 2024-10             │ 2024 October      │        2024 │ 2024-10-16 │  94 │        243 │         10 │ Информатика         │    4 │
   5. │             401671 │ 401671    │        401671 │                           1 │           5 │      200 │  2025-03-01 │ 2025-03             │ 2025 March        │        2025 │ 2025-03-03 │  70 │        219 │          7 │ Биология            │    4 │

August       │        2025 │ 2025-08-27 │  14 │        163 │          1 │ Математика          │    4 │
4008. │             400073 │ 400073    │        400073 │                           1 │           5 │      200 │  2025-02-17 │ 2025-02             │ 2025 February     │        2025 │ 2025-02-17 │ 116 │        265 │         12 │ Информатика         │    4 │
4009. │             400124 │ 400124    │        400124 │                           1 │           5 │      200 │  2025-10-05 │ 2025-10             │ 2025 October      │        2025 │ 2025-10-05 │ 117 │        266 │         13 │ Информатика         │ ᴺᵁᴸᴸ │
      └─student_profile_id─┴─person_id─┴─person_id_int─┴─educational_organization_id─┴─parallel_id─┴─class_id─┴─lesson_date─┴─lesson_month_digits─┴─lesson_month_text─┴─lesson_year─┴──load_date─┴───t─┴─teacher_id─┴─subject_id─┴─subject_name────────┴─mark─┘
Showed 1000 out of 4009 rows.

4009 rows in set. Elapsed: 0.017 sec. Processed 24.58 thousand rows, 2.77 MB (1.47 million rows/s., 165.29 MB/s.)
Peak memory usage: 12.16 MiB.
```

```sql
explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	class_id = 200;
```
Результат
```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Keys:                                       |
          class_id                                  |
        Condition: (class_id in [200, 200])         |
        Parts: 3/3                                  |
        Granules: 3/2446                            |
```
### 4. Получаем данные по столбцу student_profile_id без применения индекса

```sql
SELECT 	*
FROM
	learn_db.mart_student_lesson
WHERE 
	student_profile_id = 8;
```

Результат
```text
Query id: c3243449-d523-45f8-9373-9af2c48fa7bf

   ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬───t─┬─teacher_id─┬─subject_id─┬─subject_name─┬─mark─┐
1. │                  8 │ 8         │             8 │                           0 │           0 │        0 │  2025-07-07 │ 2025-07             │ 2025 July         │        2025 │ 2025-07-08 │ 136 │        136 │         15 │ Информатика  │    4 │
2. │                  8 │ 8         │             8 │                           0 │           0 │        0 │  2024-11-09 │ 2024-11             │ 2024 November     │        2024 │ 2024-11-11 │ 111 │        111 │         12 │ Информатика  │    3 │
3. │                  8 │ 8         │             8 │                           0 │           0 │        0 │  2025-07-29 │ 2025-07             │ 2025 July         │        2025 │ 2025-07-29 │ 120 │        120 │         13 │ Информатика  │    4 │
4. │                  8 │ 8         │             8 │                           0 │           0 │        0 │  2025-06-21 │ 2025-06             │ 2025 June         │        2025 │ 2025-06-21 │  59 │         59 │          6 │ География    │    3 │
5. │                  8 │ 8         │             8 │                           0 │           0 │        0 │  2024-10-28 │ 2024-10             │ 2024 October      │        2024 │ 2024-10-29 │  68 │         68 │          7 │ Биология     │ ᴺᵁᴸᴸ │
   └────────────────────┴───────────┴───────────────┴─────────────────────────────┴─────────────┴──────────┴─────────────┴─────────────────────┴───────────────────┴─────────────┴────────────┴─────┴────────────┴────────────┴──────────────┴──────┘

5 rows in set. Elapsed: 0.044 sec. Processed 20.00 million rows, 80.41 MB (455.61 million rows/s., 1.83 GB/s.)
Peak memory usage: 411.29 KiB.
```

```sql
explain indexes = 1
SELECT 	*
FROM
	learn_db.mart_student_lesson
WHERE 
	student_profile_id = 8;
```

Результат
```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
```

### 5. Создаем индекс пропуска данных по полю student_profile_id

```sql
ALTER TABLE learn_db.mart_student_lesson ADD INDEX idx_student_profile_id student_profile_id TYPE minmax GRANULARITY 2;
ALTER TABLE learn_db.mart_student_lesson MATERIALIZE INDEX idx_student_profile_id;
```

### 6. Замеряем изменения в параметрах выполнения запроса

```sql
SELECT 	*
FROM
	learn_db.mart_student_lesson
WHERE 
	student_profile_id = 8;
```

Результат
```text
Query id: fd8d3ee1-b8e3-4e11-b865-d171d7ee5d20

    ┌─explain──────────────────────────────────────────────┐
 1. │ Expression ((Project names + Projection))            │
 2. │   Expression                                         │
 3. │     ReadFromMergeTree (learn_db.mart_student_lesson) │
 4. │     Indexes:                                         │
 5. │       PrimaryKey                                     │
 6. │         Condition: true                              │
 7. │         Parts: 3/3                                   │
 8. │         Granules: 2446/2446                          │
 9. │       Skip                                           │
10. │         Name: idx_student_profile_id                 │
11. │         Description: minmax GRANULARITY 2            │
12. │         Parts: 3/3                                   │
13. │         Granules: 6/2446                             │
    └──────────────────────────────────────────────────────┘

13 rows in set. Elapsed: 0.005 sec.
```

```sql
explain indexes = 1
SELECT 	*
FROM
	learn_db.mart_student_lesson
WHERE 
	student_profile_id = 8;
```


Результат
```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
      Skip                                          |
        Name: idx_student_profile_id                |
        Description: minmax GRANULARITY 2           |
        Parts: 3/3                                  |
        Granules: 6/2446                            |
```

### 7. Смотрим, в какой папке находятся части данных таблицы learn_db.mart_student_lesson

```sql
SELECT * FROM system.parts WHERE table = 'mart_student_lesson';
```

Результат
```text
partition|name          |uuid                                |part_type|active|marks|rows   |bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table              |engine   |disk_name|path                                                                              |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock|removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+--------------+------------------------------------+---------+------+-----+-------+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+-------------------+---------+---------+----------------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+----------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
tuple()  |all_1_6_1_19  |00000000-0000-0000-0000-000000000000|Wide     |     1|  817|6671718|    193402494|            193365508|              545096916|            1498|      30557|                              3303|                                3264|                          418|2025-10-08 16:24:31|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|               6|    1|          19|                       1634|                                 1762|                             6536|                                       6536|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/1e6/1e676892-ff0d-4dc2-bba2-01a7357d02ca/all_1_6_1_19/  |81534154067d50c97f58b9970b455bb1|1275caab9c28c98f6d5a31d14857c6a5|16dae5e8d50962ffc23e8be83b200352     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_7_12_1_19 |00000000-0000-0000-0000-000000000000|Wide     |     1|  816|6664141|    193196434|            193159258|              544482773|            1503|      30742|                              3303|                                3264|                          418|2025-10-08 16:24:31|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               7|              12|    1|          19|                       1632|                                 1760|                             6528|                                       6528|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/1e6/1e676892-ff0d-4dc2-bba2-01a7357d02ca/all_7_12_1_19/ |04c9b8fc8cccf0cbd99d176e20b814e1|0eac6d18a8948c275771701340e226c8|d1b132eb2a164decd9e71eb54d328f34     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_13_18_1_19|00000000-0000-0000-0000-000000000000|Wide     |     1|  816|6664141|    193191789|            193154428|              544484242|            1499|      30931|                              3303|                                3264|                          418|2025-10-08 16:24:31|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              13|              18|    1|          19|                       1632|                                 1760|                             6528|                                       6528|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/1e6/1e676892-ff0d-4dc2-bba2-01a7357d02ca/all_13_18_1_19/|76d4bac6abff0029dbc2a39485891e5b|a50006b8ecab8a69c0ad151c2283a377|7cd4424c0871eca63a2e59ad91963589     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

## Индекс пропуска данных типа set

### 8. Делаем первоначальные замеры скоростиДелаем первоначальные замеры скорости

```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	teacher_id = 2222;
```

Результат
```text
Query id: cf804595-9e43-4f89-bc03-802c58d0ec3a

      ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬───t─┬─teacher_id─┬─subject_id─┬─subject_name────────┬─mark─┐
   1. │             400568 │ 400568    │        400568 │                           1 │           5 │      200 │  2024-10-27 │ 2024-10             │ 2024 October      │        2024 │ 2024-10-28 │ 108 │        257 │         12 │ Информатика         │    2 │
   2. │             401538 │ 401538    │        401538 │                           1 │           5 │      200 │  2025-09-13 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-14 │  35 │        184 │          3 │ Литература          │ ᴺᵁᴸᴸ │
   3. │             400022 │ 400022    │        400022 │                           1 │           5 │      200 │  2024-11-08 │ 2024-11             │ 2024 November     │        2024 │ 2024-11-10 │  83 │        232 │          9 │ Информатика         │ ᴺᵁᴸᴸ │
   4. │             400778 │ 400778    │        400778 │                           1 │           5 │      200 │  2024-10-14 │ 2024-10             │ 2024 October      │        2024 │ 2024-10-16 │  94 │        243 │         10 │ Информатика         │    4 │
   5. │             401671 │ 401671    │        401671 │                           1 │           5 │      200 │  2025-03-01 │ 2025-03             │ 2025
5409. │            5899900 │ 5899900   │       5899900 │                          16 │          80 │     2949 │  2025-06-25 │ 2025-06             │ 2025 June         │        2025 │ 2025-06-26 │  24 │       2222 │          2 │ Русский язык │    5 │
      └─student_profile_id─┴─person_id─┴─person_id_int─┴─educational_organization_id─┴─parallel_id─┴─class_id─┴─lesson_date─┴─lesson_month_digits─┴─lesson_month_text─┴─lesson_year─┴──load_date─┴───t─┴─teacher_id─┴─subject_id─┴─subject_name─┴─mark─┘
Showed 1000 out of 5409 rows.

5409 rows in set. Elapsed: 0.143 sec. Processed 20.00 million rows, 160.97 MB (140.31 million rows/s., 1.13 GB/s.)
Peak memory usage: 25.45 MiB.
```

```sql
explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	teacher_id = 2222;
```

Результат
```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
```

### 9. Создаем и материализуем индекс

```sql
ALTER TABLE learn_db.mart_student_lesson
ADD INDEX teacher_id_set_index (teacher_id) TYPE set(100) GRANULARITY 1;


ALTER TABLE learn_db.mart_student_lesson
MATERIALIZE INDEX teacher_id_set_index;
```

### 10. Проверяем, поменялось ли время выполнения запроса

```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	teacher_id = 2222;
```

Результат
```text
Query id: cf804595-9e43-4f89-bc03-802c58d0ec3a

      ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬───t─┬─teacher_id─┬─subject_id─┬─subject_name────────┬─mark─┐
   1. │             400568 │ 400568    │        400568 │                           1 │           5 │      200 │  2024-10-27 │ 2024-10             │ 2024 October      │        2024 │ 2024-10-28 │ 108 │        257 │         12 │ Информатика         │    2 │
   2. │             401538 │ 401538    │        401538 │                           1 │           5 │      200 │  2025-09-13 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-14 │  35 │        184 │          3 │ Литература          │ ᴺᵁᴸᴸ │
   3. │             400022 │ 400022    │        400022 │                           1 │           5 │      200 │  2024-11-08 │ 2024-11             │ 2024 November     │        2024 │ 2024-11-10 │  83 │        232 │          9 │ Информатика         │ ᴺᵁᴸᴸ │
   4. │             400778 │ 400778    │        400778 │                           1 │           5 │      200 │  2024-10-14 │ 2024-10             │ 2024 October      │        2024 │ 2024-10-16 │  94 │        243 │         10 │ Информатика         │    4 │
   5. │             401671 │ 401671    │        401671 │                           1 │           5 │      200 │  2025-03-01 │ 2025-03             │ 2025 March        │        2025 │ 2025-03-03 │  70 │        219 │          7 │ Биология            │    4 │
5409. │            5899900 │ 5899900   │       5899900 │                          16 │          80 │     2949 │  2025-06-25 │ 2025-06             │ 2025 June         │        2025 │ 2025-06-26 │  24 │       2222 │          2 │ Русский язык │    5 │
      └─student_profile_id─┴─person_id─┴─person_id_int─┴─educational_organization_id─┴─parallel_id─┴─class_id─┴─lesson_date─┴─lesson_month_digits─┴─lesson_month_text─┴─lesson_year─┴──load_date─┴───t─┴─teacher_id─┴─subject_id─┴─subject_name─┴─mark─┘
Showed 1000 out of 5409 rows.

5409 rows in set. Elapsed: 0.128 sec. Processed 20.00 million rows, 160.97 MB (155.94 million rows/s., 1.26 GB/s.)
Peak memory usage: 25.60 MiB.
```

```sql
explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	teacher_id = 2222;
```

Результат
```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
      Skip                                          |
        Name: teacher_id_set_index                  |
        Description: set GRANULARITY 1              |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
```

### 11. Убираем ограничение на максимальное количество уникальных значений

```sql
ALTER TABLE learn_db.mart_student_lesson DROP INDEX IF EXISTS teacher_id_set_index;

ALTER TABLE learn_db.mart_student_lesson
ADD INDEX teacher_id_set_index (teacher_id) TYPE set(0) GRANULARITY 1;

ALTER TABLE learn_db.mart_student_lesson
MATERIALIZE INDEX teacher_id_set_index;
```

### 12. Проверяем, поменялось ли время выполнения запроса и план выполнения

```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	teacher_id = 2222;
```

Результат
```text
Query id: 52f1b1fa-863f-44d9-8f58-ce3dbd79b3f9

      ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬───t─┬─teacher_id─┬─subject_id─┬─subject_name─┬─mark─┐
   1. │            5598948 │ 5598948   │       5598948 │                          15 │          76 │     2799 │  2025-05-10 │ 2025-05             │ 2025 May          │        2025 │ 2025-05-12 │ 136 │       2222 │         15 │ Информатика  │    4 │
   2. │            5599254 │ 5599254   │       5599254 │                          15 │          76 │     2799 │  2025-05-10 │ 2025-05             │ 2025 May          │        2025 │ 2025-05-12 │ 136 │       2222 │         15 │ Информатика  │ ᴺᵁᴸᴸ │
   3. │            5599317 │ 5599317   │       5599317 │                          15 │          76 │     2799 │  2025-06-14 │ 2025-06             │ 2025 June         │        2025 │ 2025-06-14 │ 136 │       2222 │         15 │ Информатика  │ ᴺᵁᴸᴸ │
   4. │            5601149 │ 5601149   │       5601149 │                          15 │          76 │     2800 │  2025-05-02 │ 2025-05             │ 2025 May          │        2025 │ 2025-05-04 │ 135 │       2222 │         15 │ Информатика  │    3 │
   5. │            5601502 │ 5601502   │       5601502 │                          15 │          76 │     2800 │  2025-06-27 │ 2025-06             │ 2025
5409. │            5792401 │ 5792401   │       5792401 │                          15 │          79 │     2896 │  2025-03-27 │ 2025-03             │ 2025 March        │        2025 │ 2025-03-29 │  64 │       2222 │          7 │ Биология            │ ᴺᵁᴸᴸ │
      └─student_profile_id─┴─person_id─┴─person_id_int─┴─educational_organization_id─┴─parallel_id─┴─class_id─┴─lesson_date─┴─lesson_month_digits─┴─lesson_month_text─┴─lesson_year─┴──load_date─┴───t─┴─teacher_id─┴─subject_id─┴─subject_name────────┴─mark─┘
Showed 1000 out of 5409 rows.

5409 rows in set. Elapsed: 0.085 sec. Processed 745.47 thousand rows, 83.95 MB (8.82 million rows/s., 993.00 MB/s.)
Peak memory usage: 47.77 MiB.
```

```sql
explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	teacher_id = 2222;
```

Результат
```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
      Skip                                          |
        Name: teacher_id_set_index                  |
        Description: set GRANULARITY 1              |
        Parts: 3/3                                  |
        Granules: 91/2446                           |
```

## Индексы пропуска данных фильтр Блума

### 13. Создаем и наполняем таблицу git.file_changes

```sql
DROP DATABASE IF EXISTS git;
CREATE DATABASE git;

CREATE TABLE git.file_changes
(
    change_type Enum('Add' = 1, 'Delete' = 2, 'Modify' = 3, 'Rename' = 4, 'Copy' = 5, 'Type' = 6),
    path LowCardinality(String),
    old_path LowCardinality(String),
    file_extension LowCardinality(String),
    lines_added UInt32,
    lines_deleted UInt32,
    hunks_added UInt32,
    hunks_removed UInt32,
    hunks_changed UInt32,
    commit_hash String,
    author LowCardinality(String),
    time DateTime,
    commit_message String,
    commit_files_added UInt32,
    commit_files_deleted UInt32,
    commit_files_renamed UInt32,
    commit_files_modified UInt32,
    commit_lines_added UInt32,
    commit_lines_deleted UInt32,
    commit_hunks_added UInt32,
    commit_hunks_removed UInt32,
    commit_hunks_changed UInt32
) ENGINE = MergeTree ORDER BY time;

INSERT INTO git.file_changes SELECT *
FROM s3('https://datasets-documentation.s3.amazonaws.com/github/commits/clickhouse/file_changes.tsv.xz', 'TSV', 'change_type Enum(\'Add\' = 1, \'Delete\' = 2, \'Modify\' = 3, \'Rename\' = 4, \'Copy\' = 5, \'Type\' = 6), path LowCardinality(String), old_path LowCardinality(String), file_extension LowCardinality(String), lines_added UInt32, lines_deleted UInt32, hunks_added UInt32, hunks_removed UInt32, hunks_changed UInt32, commit_hash String, author LowCardinality(String), time DateTime, commit_message String, commit_files_added UInt32, commit_files_deleted UInt32, commit_files_renamed UInt32, commit_files_modified UInt32, commit_lines_added UInt32, commit_lines_deleted UInt32, commit_hunks_added UInt32, commit_hunks_removed UInt32, commit_hunks_changed UInt32')

```

### 14. Замеряем время выполнения и смотрим план выполнения запроса поиск по подстроке

```sql
SELECT * FROM git.file_changes WHERE commit_message LIKE '%MYSQL%';
```

Результат
```text
Query id: 17d6ade7-5132-48b1-8274-8d526c1cca50

Row 1:
──────
change_type:           Modify
path:                  base/mysqlxx/Connection.cpp
old_path:              
file_extension:        cpp
lines_added:           7
lines_deleted:         6
hunks_added:           0
hunks_removed:         0
hunks_changed:         4
commit_hash:           4bcaed98d8bd1b138b45b21a8577a69a5d9438b2
author:                Alexander Kazakov
time:                  2021-02-26 06:49:49
commit_message:        Added "opt_reconnect" parameter to config for controlling MYSQL_OPT_RECONNECT option (#19998)
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 4
commit_lines_added:    28
commit_lines_deleted:  13
commit_hunks_added:    5
commit_hunks_removed:  0
commit_hunks_changed:  11

Row 2:
──────
change_type:           Modify
path:                  base/mysqlxx/Connection.h
old_path:              
file_extension:        h
lines_added:           9
lines_deleted:         3
hunks_added:           2
hunks_removed:         0
hunks_changed:         3
commit_hash:           4bcaed98d8bd1b138b45b21a8577a69a5d9438b2
author:                Alexander Kazakov
time:                  2021-02-26 06:49:49
commit_message:        Added "opt_reconnect" parameter to config for controlling MYSQL_OPT_RECONNECT option (#19998)
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 4
commit_lines_added:    28
commit_lines_deleted:  13
commit_hunks_added:    5
commit_hunks_removed:  0
commit_hunks_changed:  11

Row 3:
──────
change_type:           Modify
path:                  base/mysqlxx/Pool.cpp
old_path:              
file_extension:        cpp
lines_added:           6
lines_deleted:         1
hunks_added:           2
hunks_removed:         0
hunks_changed:         1
commit_hash:           4bcaed98d8bd1b138b45b21a8577a69a5d9438b2
author:                Alexander Kazakov
time:                  2021-02-26 06:49:49
commit_message:        Added "opt_reconnect" parameter to config for controlling MYSQL_OPT_RECONNECT option (#19998)
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 4
commit_lines_added:    28
commit_lines_deleted:  13
commit_hunks_added:    5
commit_hunks_removed:  0
commit_hunks_changed:  11

Row 4:
──────
change_type:           Modify
path:                  base/mysqlxx/Pool.h
old_path:              
file_extension:        h
lines_added:           6
lines_deleted:         3
hunks_added:           1
hunks_removed:         0
hunks_changed:         3
commit_hash:           4bcaed98d8bd1b138b45b21a8577a69a5d9438b2
author:                Alexander Kazakov
time:                  2021-02-26 06:49:49
commit_message:        Added "opt_reconnect" parameter to config for controlling MYSQL_OPT_RECONNECT option (#19998)
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 4
commit_lines_added:    28
commit_lines_deleted:  13
commit_hunks_added:    5
commit_hunks_removed:  0
commit_hunks_changed:  11

Row 5:
──────
change_type:           Modify
path:                  libs/libmysqlxx/src/Connection.cpp
old_path:              
file_extension:        cpp
lines_added:           1
lines_deleted:         2
hunks_added:           0
hunks_removed:         0
hunks_changed:         1
commit_hash:           d4ac9fe90416b1c302aa048c351955bae3e64fba
author:                Alexey Milovidov
time:                  2011-06-29 20:59:20
commit_message:        mysqlxx: removed MYSQL_OPT_READ_TIMEOUT [#CONV-2647].
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 1
commit_lines_added:    1
commit_lines_deleted:  2
commit_hunks_added:    0
commit_hunks_removed:  0
commit_hunks_changed:  1

Row 6:
──────
change_type:           Modify
path:                  libs/libmysqlxx/src/Connection.cpp
old_path:              
file_extension:        cpp
lines_added:           6
lines_deleted:         0
hunks_added:           1
hunks_removed:         0
hunks_changed:         0
commit_hash:           2b5d7194147dd1f542c190e8180e3e26dc975aa4
author:                Alexey Milovidov
time:                  2011-08-23 18:46:58
commit_message:        mysqlxx: added option MYSQL_OPT_LOCAL_INFILE [#CONV-3016].
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 1
commit_lines_added:    6
commit_lines_deleted:  0
commit_hunks_added:    1
commit_hunks_removed:  0
commit_hunks_changed:  0

Row 7:
──────
change_type:           Modify
path:                  src/Interpreters/InterpreterExternalDDLQuery.cpp
old_path:              
file_extension:        cpp
lines_added:           2
lines_deleted:         2
hunks_added:           0
hunks_removed:         0
hunks_changed:         2
commit_hash:           abf4fbe82f0d9bcee082f9c5c9b4a8dce4d3e6d2
author:                Yuriy Chernyshov
time:                  2021-11-10 08:38:03
commit_message:        Fix typo in USE_MYSQL check
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 2
commit_lines_added:    5
commit_lines_deleted:  5
commit_hunks_added:    0
commit_hunks_removed:  0
commit_hunks_changed:  5

Row 8:
──────
change_type:           Modify
path:                  src/Parsers/ParserExternalDDLQuery.cpp
old_path:              
file_extension:        cpp
lines_added:           3
lines_deleted:         3
hunks_added:           0
hunks_removed:         0
hunks_changed:         3
commit_hash:           abf4fbe82f0d9bcee082f9c5c9b4a8dce4d3e6d2
author:                Yuriy Chernyshov
time:                  2021-11-10 08:38:03
commit_message:        Fix typo in USE_MYSQL check
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 2
commit_lines_added:    5
commit_lines_deleted:  5
commit_hunks_added:    0
commit_hunks_removed:  0
commit_hunks_changed:  5

Row 9:
───────
change_type:           Modify
path:                  src/Storages/StorageMySQL.cpp
old_path:              
file_extension:        cpp
lines_added:           1
lines_deleted:         1
hunks_added:           0
hunks_removed:         0
hunks_changed:         1
commit_hash:           d981463b058c2100590cdb2101eb364fbd6db29c
author:                HeenaBansal2009
time:                  2022-03-10 16:58:48
commit_message:        Added RemoteHostFilter check for MYSQL and postgresSQL
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 2
commit_lines_added:    2
commit_lines_deleted:  2
commit_hunks_added:    0
commit_hunks_removed:  0
commit_hunks_changed:  2

Row 10:
───────
change_type:           Modify
path:                  src/Storages/StoragePostgreSQL.cpp
old_path:              
file_extension:        cpp
lines_added:           1
lines_deleted:         1
hunks_added:           0
hunks_removed:         0
hunks_changed:         1
commit_hash:           d981463b058c2100590cdb2101eb364fbd6db29c
author:                HeenaBansal2009
time:                  2022-03-10 16:58:48
commit_message:        Added RemoteHostFilter check for MYSQL and postgresSQL
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 2
commit_lines_added:    2
commit_lines_deleted:  2
commit_hunks_added:    0
commit_hunks_removed:  0
commit_hunks_changed:  2

10 rows in set. Elapsed: 0.031 sec. Processed 268.77 thousand rows, 14.19 MB (8.65 million rows/s., 456.81 MB/s.)
Peak memory usage: 15.49 MiB.
```

```sql
explain indexes = 1
SELECT * FROM git.file_changes WHERE commit_message LIKE '%MYSQL%';
```

Результат
```text
explain                                  |
-----------------------------------------+
Expression ((Project names + Projection))|
  Expression                             |
    ReadFromMergeTree (git.file_changes) |
    Indexes:                             |
      PrimaryKey                         |
        Condition: true                  |
        Parts: 1/1                       |
        Granules: 33/33                  |
```

### 15. Создаем n-грам индекс фильтра Блума
```sql
ALTER TABLE git.file_changes
ADD INDEX commit_message_tokenbf_v1_index commit_message TYPE ngrambf_v1(3, 10000, 3, 7) GRANULARITY 1

ALTER TABLE git.file_changes
MATERIALIZE INDEX commit_message_tokenbf_v1_index;
```

### 16. Проверяем поменялось ли время выполнения запроса и план выполнения запроса
```
SELECT * FROM git.file_changes WHERE commit_message LIKE '%MYSQL%';
```

Результат
```text
Query id: de982e9b-290c-41e1-bf31-86c270cd2918

Row 1:
──────
change_type:           Modify
path:                  libs/libmysqlxx/src/Connection.cpp
old_path:              
file_extension:        cpp
lines_added:           1
lines_deleted:         2
hunks_added:           0
hunks_removed:         0
hunks_changed:         1
commit_hash:           d4ac9fe90416b1c302aa048c351955bae3e64fba
author:                Alexey Milovidov
time:                  2011-06-29 20:59:20
commit_message:        mysqlxx: removed MYSQL_OPT_READ_TIMEOUT [#CONV-2647].
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 1
commit_lines_added:    1
commit_lines_deleted:  2
commit_hunks_added:    0
commit_hunks_removed:  0
commit_hunks_changed:  1

Row 2:
──────
change_type:           Modify
path:                  libs/libmysqlxx/src/Connection.cpp
old_path:              
file_extension:        cpp
lines_added:           6
lines_deleted:         0
hunks_added:           1
hunks_removed:         0
hunks_changed:         0
commit_hash:           2b5d7194147dd1f542c190e8180e3e26dc975aa4
author:                Alexey Milovidov
time:                  2011-08-23 18:46:58
commit_message:        mysqlxx: added option MYSQL_OPT_LOCAL_INFILE [#CONV-3016].
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 1
commit_lines_added:    6
commit_lines_deleted:  0
commit_hunks_added:    1
commit_hunks_removed:  0
commit_hunks_changed:  0

Row 3:
──────
change_type:           Modify
path:                  base/mysqlxx/Connection.cpp
old_path:              
file_extension:        cpp
lines_added:           7
lines_deleted:         6
hunks_added:           0
hunks_removed:         0
hunks_changed:         4
commit_hash:           4bcaed98d8bd1b138b45b21a8577a69a5d9438b2
author:                Alexander Kazakov
time:                  2021-02-26 06:49:49
commit_message:        Added "opt_reconnect" parameter to config for controlling MYSQL_OPT_RECONNECT option (#19998)
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 4
commit_lines_added:    28
commit_lines_deleted:  13
commit_hunks_added:    5
commit_hunks_removed:  0
commit_hunks_changed:  11

Row 4:
──────
change_type:           Modify
path:                  base/mysqlxx/Connection.h
old_path:              
file_extension:        h
lines_added:           9
lines_deleted:         3
hunks_added:           2
hunks_removed:         0
hunks_changed:         3
commit_hash:           4bcaed98d8bd1b138b45b21a8577a69a5d9438b2
author:                Alexander Kazakov
time:                  2021-02-26 06:49:49
commit_message:        Added "opt_reconnect" parameter to config for controlling MYSQL_OPT_RECONNECT option (#19998)
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 4
commit_lines_added:    28
commit_lines_deleted:  13
commit_hunks_added:    5
commit_hunks_removed:  0
commit_hunks_changed:  11

Row 5:
──────
change_type:           Modify
path:                  base/mysqlxx/Pool.cpp
old_path:              
file_extension:        cpp
lines_added:           6
lines_deleted:         1
hunks_added:           2
hunks_removed:         0
hunks_changed:         1
commit_hash:           4bcaed98d8bd1b138b45b21a8577a69a5d9438b2
author:                Alexander Kazakov
time:                  2021-02-26 06:49:49
commit_message:        Added "opt_reconnect" parameter to config for controlling MYSQL_OPT_RECONNECT option (#19998)
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 4
commit_lines_added:    28
commit_lines_deleted:  13
commit_hunks_added:    5
commit_hunks_removed:  0
commit_hunks_changed:  11

Row 6:
──────
change_type:           Modify
path:                  base/mysqlxx/Pool.h
old_path:              
file_extension:        h
lines_added:           6
lines_deleted:         3
hunks_added:           1
hunks_removed:         0
hunks_changed:         3
commit_hash:           4bcaed98d8bd1b138b45b21a8577a69a5d9438b2
author:                Alexander Kazakov
time:                  2021-02-26 06:49:49
commit_message:        Added "opt_reconnect" parameter to config for controlling MYSQL_OPT_RECONNECT option (#19998)
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 4
commit_lines_added:    28
commit_lines_deleted:  13
commit_hunks_added:    5
commit_hunks_removed:  0
commit_hunks_changed:  11

Row 7:
──────
change_type:           Modify
path:                  src/Interpreters/InterpreterExternalDDLQuery.cpp
old_path:              
file_extension:        cpp
lines_added:           2
lines_deleted:         2
hunks_added:           0
hunks_removed:         0
hunks_changed:         2
commit_hash:           abf4fbe82f0d9bcee082f9c5c9b4a8dce4d3e6d2
author:                Yuriy Chernyshov
time:                  2021-11-10 08:38:03
commit_message:        Fix typo in USE_MYSQL check
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 2
commit_lines_added:    5
commit_lines_deleted:  5
commit_hunks_added:    0
commit_hunks_removed:  0
commit_hunks_changed:  5

Row 8:
──────
change_type:           Modify
path:                  src/Parsers/ParserExternalDDLQuery.cpp
old_path:              
file_extension:        cpp
lines_added:           3
lines_deleted:         3
hunks_added:           0
hunks_removed:         0
hunks_changed:         3
commit_hash:           abf4fbe82f0d9bcee082f9c5c9b4a8dce4d3e6d2
author:                Yuriy Chernyshov
time:                  2021-11-10 08:38:03
commit_message:        Fix typo in USE_MYSQL check
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 2
commit_lines_added:    5
commit_lines_deleted:  5
commit_hunks_added:    0
commit_hunks_removed:  0
commit_hunks_changed:  5

Row 9:
───────
change_type:           Modify
path:                  src/Storages/StorageMySQL.cpp
old_path:              
file_extension:        cpp
lines_added:           1
lines_deleted:         1
hunks_added:           0
hunks_removed:         0
hunks_changed:         1
commit_hash:           d981463b058c2100590cdb2101eb364fbd6db29c
author:                HeenaBansal2009
time:                  2022-03-10 16:58:48
commit_message:        Added RemoteHostFilter check for MYSQL and postgresSQL
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 2
commit_lines_added:    2
commit_lines_deleted:  2
commit_hunks_added:    0
commit_hunks_removed:  0
commit_hunks_changed:  2

Row 10:
───────
change_type:           Modify
path:                  src/Storages/StoragePostgreSQL.cpp
old_path:              
file_extension:        cpp
lines_added:           1
lines_deleted:         1
hunks_added:           0
hunks_removed:         0
hunks_changed:         1
commit_hash:           d981463b058c2100590cdb2101eb364fbd6db29c
author:                HeenaBansal2009
time:                  2022-03-10 16:58:48
commit_message:        Added RemoteHostFilter check for MYSQL and postgresSQL
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 2
commit_lines_added:    2
commit_lines_deleted:  2
commit_hunks_added:    0
commit_hunks_removed:  0
commit_hunks_changed:  2

10 rows in set. Elapsed: 0.016 sec. Processed 32.77 thousand rows, 2.68 MB (2.03 million rows/s., 166.00 MB/s.)
Peak memory usage: 9.97 MiB.
```

```sql
explain indexes = 1
SELECT * FROM git.file_changes WHERE commit_message LIKE '%MYSQL%';
```

Результат
```text
explain                                      |
---------------------------------------------+
Expression ((Project names + Projection))    |
  Expression                                 |
    ReadFromMergeTree (git.file_changes)     |
    Indexes:                                 |
      PrimaryKey                             |
        Condition: true                      |
        Parts: 1/1                           |
        Granules: 33/33                      |
      Skip                                   |
        Name: commit_message_tokenbf_v1_index|
        Description: ngrambf_v1 GRANULARITY 1|
        Parts: 1/1                           |
        Granules: 4/33                       |
```

## Индексы пропуска данных фильтр Блума по токенам

### 17. Создаем и материализуем индекс по токенам фильтра Блума

```sql
ALTER TABLE git.file_changes
ADD INDEX commit_message_tokenbf_v1_idx commit_message TYPE tokenbf_v1(512, 3, 0) GRANULARITY 1;

ALTER TABLE git.file_changes
MATERIALIZE INDEX commit_message_tokenbf_v1_idx;
```

### 18. Замеряем скорость выполнения запроса и смотрим план выполнения запроса с поиском по токену

```sql
SELECT * FROM git.file_changes WHERE hasToken(commit_message, '9019');
```

Результат
```text
Query id: 28afd6d8-a654-46d9-b7cc-5aff28eea8ab

Row 1:
──────
change_type:           Modify
path:                  dbms/include/DB/Common/SipHash.h
old_path:              
file_extension:        h
lines_added:           15
lines_deleted:         0
hunks_added:           1
hunks_removed:         0
hunks_changed:         0
commit_hash:           156bfdc09434f30acabfb395fdec884383138198
author:                Alexey Milovidov
time:                  2013-10-21 17:32:49
commit_message:        dbms: eased use of SipHash [#METR-9019].
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 3
commit_lines_added:    17
commit_lines_deleted:  8
commit_hunks_added:    1
commit_hunks_removed:  0
commit_hunks_changed:  2

Row 2:
──────
change_type:           Modify
path:                  dbms/include/DB/Functions/FunctionsHashing.h
old_path:              
file_extension:        h
lines_added:           1
lines_deleted:         3
hunks_added:           0
hunks_removed:         0
hunks_changed:         1
commit_hash:           156bfdc09434f30acabfb395fdec884383138198
author:                Alexey Milovidov
time:                  2013-10-21 17:32:49
commit_message:        dbms: eased use of SipHash [#METR-9019].
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 3
commit_lines_added:    17
commit_lines_deleted:  8
commit_hunks_added:    1
commit_hunks_removed:  0
commit_hunks_changed:  2

Row 3:
──────
change_type:           Modify
path:                  dbms/src/Interpreters/Quota.cpp
old_path:              
file_extension:        cpp
lines_added:           1
lines_deleted:         5
hunks_added:           0
hunks_removed:         0
hunks_changed:         1
commit_hash:           156bfdc09434f30acabfb395fdec884383138198
author:                Alexey Milovidov
time:                  2013-10-21 17:32:49
commit_message:        dbms: eased use of SipHash [#METR-9019].
commit_files_added:    0
commit_files_deleted:  0
commit_files_renamed:  0
commit_files_modified: 3
commit_lines_added:    17
commit_lines_deleted:  8
commit_hunks_added:    1
commit_hunks_removed:  0
commit_hunks_changed:  2

3 rows in set. Elapsed: 0.015 sec. Processed 24.58 thousand rows, 2.20 MB (1.66 million rows/s., 148.98 MB/s.)
Peak memory usage: 7.51 MiB.
```

```sql
explain indexes = 1
SELECT * FROM git.file_changes WHERE hasToken(commit_message, '9019');
```

Результат
```text
explain                                      |
---------------------------------------------+
Expression ((Project names + Projection))    |
  Expression                                 |
    ReadFromMergeTree (git.file_changes)     |
    Indexes:                                 |
      PrimaryKey                             |
        Condition: true                      |
        Parts: 1/1                           |
        Granules: 33/33                      |
      Skip                                   |
        Name: commit_message_tokenbf_v1_index|
        Description: ngrambf_v1 GRANULARITY 1|
        Parts: 1/1                           |
        Granules: 6/33                       |
      Skip                                   |
        Name: commit_message_tokenbf_v1_idx  |
        Description: tokenbf_v1 GRANULARITY 1|
        Parts: 1/1                           |
        Granules: 3/6                        |
```

## Индексы пропуска данных фильтр Блума

### 19. Замеряем скорость выполнения запроса и смотрим план выполнения запроса с поиском по равенству строки

```sql
SELECT 
	*
FROM 
	git.file_changes
WHERE 
	commit_hash = 'b841a96c3998b7fcf5c53cfd3dd502122785d8f3'
```

Результат
```text
Query id: 9c7e553c-8c01-49ad-903e-2eaadb750666

    ┌─change_type─┬─path────────────────────────────────────┬─old_path─┬─file_extension─┬─lines_added─┬─lines_deleted─┬─hunks_added─┬─hunks_removed─┬─hunks_changed─┬─commit_hash──────────────────────────────┬─author────────┬────────────────time─┬─commit_message─┬─commit_files_added─┬─commit_files_deleted─┬─commit_files_renamed─┬─commit_files_modified─┬─commit_lines_added─┬─commit_lines_deleted─┬─commit_hunks_added─┬─commit_hunks_removed─┬─commit_hunks_changed─┐
 1. │ Modify      │ src/Functions/s2ToGeo.cpp               │          │ cpp            │           2 │             0 │           1 │             0 │             0 │ b841a96c3998b7fcf5c53cfd3dd502122785d8f3 │ Pavel Kruglov │ 2021-08-10 12:31:15 │ Refactor code  │                  2 │                    0 │                    0 │                    68 │                853 │                  318 │                 45 │                   21 │                  153 │
 2. │ Modify      │ src/Functions/stem.cpp                  │          │ cpp            │           2 │             0 │           1 │             0 │             0 │ b841a96c3998b7fcf5c53cfd3dd502122785d8f3 │ Pavel Kruglov │ 2021-08-10 12:31:15 │ Refactor code  │                  2 │                    0 │                    0 │                    68 │                853 │                  318 │                 45 │                   21 │                  153 │
 3. │ Modify      │ src/Functions/stringCutToZero.cpp       │          │ cpp            │           2 │             0 │           1 │             0 │             0 │ b841a96c3998b7fcf5c53cfd3dd502122785d8f3 │ Pavel Kruglov │ 2021-08-10 12:31:15 │ Refactor code  │                  2 │                    0 │                    0 │                    68 │                853 │                  318 │                 45 │                   21 │                  153 │
 4. │ Modify      │ src/Functions/synonyms.cpp              │          │ cpp            │           2 │             0 │           1 │             0 │             0 │ b841a96c3998b7fcf5c53cfd3dd502122785d8f3 │ Pavel Kruglov │ 2021-08-10 12:31:15 │ Refactor code  │                  2 │                    0 │                    0 │                    68 │                853 │                  318 │                 45 │                   21 │                  153 │
 5. │ Modify      │ src/Functions/toTimezone.cpp            │          │ cpp            │           2 │             2 │           1 │             1 │             0 │ b841a96c3998b7fcf5c53cfd3dd502122785d8f3 │ Pavel Kruglov │ 2021-08-10 12:31:15 │ Refactor code  │                  2 │                    0 │                    0 │                    68 │                853 │                  318 │                 45 │                   21 │
70. │ Modify      │ src/Functions/s2RectUnion.cpp           │          │ cpp            │           2 │             0 │           1 │             0 │             0 │ b841a96c3998b7fcf5c53cfd3dd502122785d8f3 │ Pavel Kruglov │ 2021-08-10 12:31:15 │ Refactor code  │                  2 │                    0 │                    0 │                    68 │                853 │                  318 │                 45 │                   21 │                  153 │
    └─change_type─┴─path────────────────────────────────────┴─old_path─┴─file_extension─┴─lines_added─┴─lines_deleted─┴─hunks_added─┴─hunks_removed─┴─hunks_changed─┴─commit_hash──────────────────────────────┴─author────────┴────────────────time─┴─commit_message─┴─commit_files_added─┴─commit_files_deleted─┴─commit_files_renamed─┴─commit_files_modified─┴─commit_lines_added─┴─commit_lines_deleted─┴─commit_hunks_added─┴─commit_hunks_removed─┴─commit_hunks_changed─┘

70 rows in set. Elapsed: 0.020 sec. Processed 268.77 thousand rows, 14.27 MB (13.71 million rows/s., 728.39 MB/s.)
Peak memory usage: 8.00 MiB.
```

```sql
explain indexes = 1
SELECT 
	*
FROM 
	git.file_changes
WHERE 
	commit_hash = 'b841a96c3998b7fcf5c53cfd3dd502122785d8f3'
```

Результат
```text
explain                                  |
-----------------------------------------+
Expression ((Project names + Projection))|
  Expression                             |
    ReadFromMergeTree (git.file_changes) |
    Indexes:                             |
      PrimaryKey                         |
        Condition: true                  |
        Parts: 1/1                       |
        Granules: 33/33                  |
```

### 20. Создаем и материализуем индекс фильтра Блума

```sql
ALTER TABLE git.file_changes
ADD INDEX commit_hash_bloom_filter_idx commit_hash TYPE bloom_filter(0.001) GRANULARITY 1

ALTER TABLE git.file_changes
MATERIALIZE INDEX commit_hash_bloom_filter_idx;
```

### 21. Проверяем поменялось ли время выполнения запроса и план выполнения запроса

```sql
SELECT 
	*
FROM 
	git.file_changes
WHERE 
	commit_hash = 'b841a96c3998b7fcf5c53cfd3dd502122785d8f3'
```

Результат
```text
Query id: 8379b3b3-d040-46e9-8620-9570e31b86f0

    ┌─change_type─┬─path────────────────────────────────────┬─old_path─┬─file_extension─┬─lines_added─┬─lines_deleted─┬─hunks_added─┬─hunks_removed─┬─hunks_changed─┬─commit_hash──────────────────────────────┬─author────────┬────────────────time─┬─commit_message─┬─commit_files_added─┬─commit_files_deleted─┬─commit_files_renamed─┬─commit_files_modified─┬─commit_lines_added─┬─commit_lines_deleted─┬─commit_hunks_added─┬─commit_hunks_removed─┬─commit_hunks_changed─┐
 1. │ Modify      │ src/Columns/ColumnAggregateFunction.cpp │          │ cpp            │           2 │             2 │           0 │             0 │             2 │ b841a96c3998b7fcf5c53cfd3dd502122785d8f3 │ Pavel Kruglov │ 2021-08-10 12:31:15 │ Refactor code  │                  2 │                    0 │                    0 │                    68 │                853 │                  318 │                 45 │                   21 │                  153 │
 2. │ Modify      │ src/Columns/ColumnAggregateFunction.h   │          │ h              │           1 │             1 │           0 │             0 │             1 │ b841a96c3998b7fcf5c53cfd3dd502122785d8f3 │ Pavel Kruglov │ 2021-08-10 12:31:15 │ Refactor code  │                  2 │                    0 │                    0 │                    68 │                853 │                  318 │                 45 │                   21 │                  153 │
 3. │ Modify      │ src/Columns/ColumnArray.cpp             │          │ cpp            │          28 │            28 │           0 │             0 │            13 │ b841a96c3998b7fcf5c53cfd3dd502122785d8f3 │ Pavel Kruglov │ 2021-08-10 12:31:15 │ Refactor code  │                  2 │                    0 │                    0 │                    68 │                853 │                  318 │                 45 │                   21 │                  153 │
 4. │ Modify      │ src/Columns/ColumnArray.h               │          │ h              │           6 │             6 │           0 │             0 │             3 │ b841a96c3998b7fcf5c53cfd3dd502122785d8f3 │ Pavel Kruglov │ 2021-08-10 12:31:15 │ Refactor code  │                  2 │                    0 │                    0 │                    68 │                853 │                  318 │                 45 │                   21 │                  153 │
 5. │ Modify      │ src/Columns/ColumnCompressed.h          │          │ h              │           1 │             1 │           0 │             0 │             1 │ b841a96c3998b7fcf5c53cfd3dd502122785d8f3 │ Pavel Kruglov │ 2021-08-10 12:31:15 │ Refactor code  │                  2 │                    0 │                    0 │                    68 │                853 │                  318 │                 45 │                   21 │                  153 │
70. │ Modify      │ src/Interpreters/ExpressionJIT.cpp      │          │ cpp            │           5 │             5 │           0 │             0 │             5 │ b841a96c3998b7fcf5c53cfd3dd502122785d8f3 │ Pavel Kruglov │ 2021-08-10 12:31:15 │ Refactor code  │                  2 │                    0 │                    0 │                    68 │                853 │                  318 │                 45 │                   21 │                  153 │
    └─change_type─┴─path────────────────────────────────────┴─old_path─┴─file_extension─┴─lines_added─┴─lines_deleted─┴─hunks_added─┴─hunks_removed─┴─hunks_changed─┴─commit_hash──────────────────────────────┴─author────────┴────────────────time─┴─commit_message─┴─commit_files_added─┴─commit_files_deleted─┴─commit_files_renamed─┴─commit_files_modified─┴─commit_lines_added─┴─commit_lines_deleted─┴─commit_hunks_added─┴─commit_hunks_removed─┴─commit_hunks_changed─┘

70 rows in set. Elapsed: 0.017 sec. Processed 16.38 thousand rows, 1.91 MB (948.31 thousand rows/s., 110.40 MB/s.)
Peak memory usage: 7.40 MiB.
```

```sql
explain indexes = 1
SELECT 
	*
FROM 
	git.file_changes
WHERE 
	commit_hash = 'b841a96c3998b7fcf5c53cfd3dd502122785d8f3'
```

Результат
```text
explain                                        |
-----------------------------------------------+
Expression ((Project names + Projection))      |
  Expression                                   |
    ReadFromMergeTree (git.file_changes)       |
    Indexes:                                   |
      PrimaryKey                               |
        Condition: true                        |
        Parts: 1/1                             |
        Granules: 33/33                        |
      Skip                                     |
        Name: commit_hash_bloom_filter_idx     |
        Description: bloom_filter GRANULARITY 1|
        Parts: 1/1                             |
        Granules: 2/33                         |
```

### Изменение скорости вставки данных

### 22. Добавляем индекс пропуска данных в таблицу learn_db.mart_student_lesson

```sql
ALTER TABLE learn_db.mart_student_lesson
ADD INDEX student_profile_id_bloom_filter_idx student_profile_id TYPE bloom_filter(0.001) GRANULARITY 1

ALTER TABLE learn_db.mart_student_lesson
MATERIALIZE INDEX student_profile_id_bloom_filter_idx;
```

### 23. Вставляем в таблицу 10 000 000 строк

```sql

INSERT INTO mart_student_lesson
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
	load_date, t, 
	teacher_id, 
	subject_id, 
	subject_name, 
	mark
)
SELECT
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

Результат
```text
Execute time: 8.728s
```