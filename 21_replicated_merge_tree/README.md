# Применение движка ReplicatedMergeTree

## Код, выполняемый на 1ой реплике

### 1. Создаем реплицируемую таблицу на кластере
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson ON CLUSTER cluster_2S_2R; 
CREATE TABLE learn_db.mart_student_lesson ON CLUSTER cluster_2S_2R
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
	PRIMARY KEY(lesson_date)
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/mart_student_lesson', '{replica}');
```

### 2. Удаляем таблицу на кластере

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson ON CLUSTER cluster_2S_2R;
```

### 3. Создаем реплицируемую таблицу на одной ноде кластера, после чего создаем такую же таблицу на третьей ноде

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
	PRIMARY KEY(lesson_date)
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/mart_student_lesson', '{replica}');
```

Если таблица была создана ранее, то удаляем его по-другому:

```sql
SYSTEM DROP REPLICA '01' FROM ZKPATH '/clickhouse/tables/01/mart_student_lesson';
```

### 4. Вставляем данные на первой ноде

```sql
INSERT INTO learn_db.mart_student_lesson 
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
FROM numbers(1);
```
       
### 5. Проверяем, что данные появились в таблице, после чего делаем такую же проверку на третьей ноде

```sql
SELECT * FROM learn_db.mart_student_lesson;
```

Результат
```text
student_profile_id|person_id|person_id_int|educational_organization_id|parallel_id|class_id|lesson_date|lesson_month_digits|lesson_month_text|lesson_year|load_date |t |teacher_id|subject_id|subject_name|mark|
------------------+---------+-------------+---------------------------+-----------+--------+-----------+-------------------+-----------------+-----------+----------+--+----------+----------+------------+----+
           9661619|9661619  |      9661619|                         26|        132|    4830| 2025-01-09|2025-01            |2025 January     |       2025|2025-01-11|89|      3688|         9|Информатика |5   |
```

### 6. Вставляем данные на третьей ноде

```sql
INSERT INTO learn_db.mart_student_lesson 
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
FROM numbers(1);
```

### 7. Проверяем, что данные появились в таблице на первой ноде

```sql
SELECT * FROM learn_db.mart_student_lesson;
```

Результат
```text
student_profile_id|person_id|person_id_int|educational_organization_id|parallel_id|class_id|lesson_date|lesson_month_digits|lesson_month_text|lesson_year|load_date |t  |teacher_id|subject_id|subject_name|mark|
------------------+---------+-------------+---------------------------+-----------+--------+-----------+-------------------+-----------------+-----------+----------+---+----------+----------+------------+----+
           9661619|9661619  |      9661619|                         26|        132|    4830| 2025-01-09|2025-01            |2025 January     |       2025|2025-01-11| 89|      3688|         9|Информатика |5   |
            370084|370084   |       370084|                          1|          5|     185| 2025-06-29|2025-06            |2025 June        |       2025|2025-06-30|133|       270|        14|Информатика |    |
```

### 8. Удаляем одну строку на первой ноде 

```sql
ALTER TABLE learn_db.mart_student_lesson DELETE WHERE student_profile_id = 9661619;
```

### 9. Проверяем, что строка удалена. Выполняем скрипт на первой и третьей нодах

```sql
SELECT * FROM learn_db.mart_student_lesson;
```

Результат - нода 1
```text
student_profile_id|person_id|person_id_int|educational_organization_id|parallel_id|class_id|lesson_date|lesson_month_digits|lesson_month_text|lesson_year|load_date |t  |teacher_id|subject_id|subject_name|mark|
------------------+---------+-------------+---------------------------+-----------+--------+-----------+-------------------+-----------------+-----------+----------+---+----------+----------+------------+----+
            370084|370084   |       370084|                          1|          5|     185| 2025-06-29|2025-06            |2025 June        |       2025|2025-06-30|133|       270|        14|Информатика |    |
```

Результат - нода 3
```text
student_profile_id|person_id|person_id_int|educational_organization_id|parallel_id|class_id|lesson_date|lesson_month_digits|lesson_month_text|lesson_year|load_date |t  |teacher_id|subject_id|subject_name|mark|
------------------+---------+-------------+---------------------------+-----------+--------+-----------+-------------------+-----------------+-----------+----------+---+----------+----------+------------+----+
            370084|370084   |       370084|                          1|          5|     185| 2025-06-29|2025-06            |2025 June        |       2025|2025-06-30|133|       270|        14|Информатика |    |
```

### 10. Выполним мутацию строки через UPDATE на третьей ноде

```sql
ALTER TABLE learn_db.mart_student_lesson UPDATE person_id_int = 1 WHERE person_id_int > 1;
```

### 11. Проверим результат

```sql
SELECT * FROM learn_db.mart_student_lesson;
```

Результат - нода 3
```text
student_profile_id|person_id|person_id_int|educational_organization_id|parallel_id|class_id|lesson_date|lesson_month_digits|lesson_month_text|lesson_year|load_date |t  |teacher_id|subject_id|subject_name|mark|
------------------+---------+-------------+---------------------------+-----------+--------+-----------+-------------------+-----------------+-----------+----------+---+----------+----------+------------+----+
            370084|370084   |            1|                          1|          5|     185| 2025-06-29|2025-06            |2025 June        |       2025|2025-06-30|133|       270|        14|Информатика |    |
```

Результат - нода 1
```text
student_profile_id|person_id|person_id_int|educational_organization_id|parallel_id|class_id|lesson_date|lesson_month_digits|lesson_month_text|lesson_year|load_date |t  |teacher_id|subject_id|subject_name|mark|
------------------+---------+-------------+---------------------------+-----------+--------+-----------+-------------------+-----------------+-----------+----------+---+----------+----------+------------+----+
            370084|370084   |            1|                          1|          5|     185| 2025-06-29|2025-06            |2025 June        |       2025|2025-06-30|133|       270|        14|Информатика |    |
```

### 12. После выполнения запроса на третьей ноде, меняющего person_id_int, выполняем легковесное удаление строки на первой ноде

```sql
DELETE FROM learn_db.mart_student_lesson WHERE person_id_int = 1;
```

### 13. Проверяем, что строка удалена. Выполняем скрипт на первой и третьей нодах

```sql
SELECT * FROM learn_db.mart_student_lesson;
```

Результат - нода 1
```text
student_profile_id|person_id|person_id_int|educational_organization_id|parallel_id|class_id|lesson_date|lesson_month_digits|lesson_month_text|lesson_year|load_date|t|teacher_id|subject_id|subject_name|mark|
------------------+---------+-------------+---------------------------+-----------+--------+-----------+-------------------+-----------------+-----------+---------+-+----------+----------+------------+----+
```

Результат - нода 3
```text
student_profile_id|person_id|person_id_int|educational_organization_id|parallel_id|class_id|lesson_date|lesson_month_digits|lesson_month_text|lesson_year|load_date|t|teacher_id|subject_id|subject_name|mark|
------------------+---------+-------------+---------------------------+-----------+--------+-----------+-------------------+-----------------+-----------+---------+-+----------+----------+------------+----+
```

### 14. Вставим 10 млн. записей на первой ноде

```
INSERT INTO learn_db.mart_student_lesson 
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

### 15. Сравниваем части данных таблицы на первой и второй нодах

```sql
select * from system.parts where table = 'mart_student_lesson' order by part_type;
```

Результат
```text
partition|name       |uuid                                |part_type|active|marks|rows   |bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table              |engine             |disk_name|path                                                                           |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock|removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+-----------+------------------------------------+---------+------+-----+-------+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+-------------------+-------------------+---------+-------------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+----------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
tuple()  |all_1_9_1_4|00000000-0000-0000-0000-000000000000|Wide     |     1|  681|5559765|    167883715|            167856098|              454258540|             588|      25820|                                 0|                                   0|                            0|2025-10-30 19:36:56|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|               9|    1|           4|                       2724|                                 2852|                             5448|                                       5448|        0|learn_db|mart_student_lesson|ReplicatedMergeTree|default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_1_9_1_4/|0da9049ebe3b5a22f43b66dd499e883a|c323bf9df9e892fabb8f66783a41dbac|c3d20d19ba64621f0b065eeb77a0aaf4     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_10_10_0|00000000-0000-0000-0000-000000000000|Wide     |     1|  137|1111953|     33576228|             33568422|               90848529|             331|       6269|                                 0|                                   0|                            0|2025-10-30 19:36:54|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              10|              10|    0|          10|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|ReplicatedMergeTree|default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_10_10_0/|551cf417e34f10d33a62b6b73b3ff4fd|8f1477f167a6312ab28cb01beb8613ea|3f26b40b4d15def5e4f23dc3f119dac5     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_11_11_0|00000000-0000-0000-0000-000000000000|Wide     |     1|  137|1111953|     33582733|             33574932|               90843985|             332|       6263|                                 0|                                   0|                            0|2025-10-30 19:36:55|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              11|              11|    0|          11|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|ReplicatedMergeTree|default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_11_11_0/|f3a759235e48ccd8c8a1f58e8e5df1bd|36bad86f005a30f11e879db80158c5ac|82182268248725a4b93d922953fa1e9f     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_12_12_0|00000000-0000-0000-0000-000000000000|Wide     |     1|  137|1111953|     33586440|             33578641|               90849525|             333|       6260|                                 0|                                   0|                            0|2025-10-30 19:36:57|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              12|              12|    0|          12|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|ReplicatedMergeTree|default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_12_12_0/|4fb25454b1af4f3e1f3c2188ee364771|48e4573b1ecddb6b3c72be9162db0507|a958236ba3155aba6fe027cd337789f0     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_13_13_0|00000000-0000-0000-0000-000000000000|Wide     |     1|  136|1104376|     33348869|             33341021|               90225514|             344|       6298|                                 0|                                   0|                            0|2025-10-30 19:36:57|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              13|              13|    0|          13|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|ReplicatedMergeTree|default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_13_13_0/|faed736d6b033a3f3bd49e8aad5b5763|574809af568ccacd15cb1878a7678545|b4b9d8965b8804f4d6ac1e0c438b7aeb     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

### 16. Сравниваем историю изменения частей данных таблицы на первой и третьей нодах

```
select * from system.part_log order by event_time desc;
```

Результат - нода 1
```text
hostname     |query_id                            |event_type     |merge_reason|merge_algorithm|event_date|event_time         |event_time_microseconds      |duration_ms|database|table              |table_uuid                          |part_name  |partition_id|partition|part_type|disk_name|path_on_disk                                                                            |rows   |size_in_bytes|merged_from                                                                |bytes_uncompressed|read_rows|read_bytes|peak_memory_usage|error|exception                                                                                                                                                                                                                                                      |ProfileEvents                                                                                                                                                                                                                                                  |
-------------+------------------------------------+---------------+------------+---------------+----------+-------------------+-----------------------------+-----------+--------+-------------------+------------------------------------+-----------+------------+---------+---------+---------+----------------------------------------------------------------------------------------+-------+-------------+---------------------------------------------------------------------------+------------------+---------+----------+-----------------+-----+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:46:17|2025-10-30 19:46:17.878347000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_9_9_0  |all         |tuple()  |Wide     |         |                                                                                        |1111953|     33587653|[]                                                                         |          90905331|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:46:17|2025-10-30 19:46:17.878347000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_8_8_0  |all         |tuple()  |Wide     |         |                                                                                        |1111953|     33578699|[]                                                                         |          90913121|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:46:17|2025-10-30 19:46:17.878347000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_7_7_0  |all         |tuple()  |Wide     |         |                                                                                        |1111953|     33583561|[]                                                                         |          90902449|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:46:17|2025-10-30 19:46:17.878347000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_6_6_0  |all         |tuple()  |Wide     |         |                                                                                        |1111953|     33585231|[]                                                                         |          90914034|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:46:17|2025-10-30 19:46:17.878347000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_5_5_0  |all         |tuple()  |Wide     |         |                                                                                        |1111953|     33586844|[]                                                                         |          90905825|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:46:17|2025-10-30 19:46:17.878347000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_1_1_0_4|all         |tuple()  |Compact  |         |                                                                                        |      1|         1973|[]                                                                         |               681|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:41:17|2025-10-30 19:41:17.465729000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_1_1_0_3|all         |tuple()  |Compact  |         |                                                                                        |      1|         1874|[]                                                                         |               648|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:41:17|2025-10-30 19:41:17.465729000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_1_1_0_2|all         |tuple()  |Compact  |         |                                                                                        |      1|         1874|[]                                                                         |               648|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|2241c21e-d959-491c-9df6-27062f54ab24|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:56|2025-10-30 19:36:56.922199000|        715|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_13_13_0|all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_13_13_0/         |1104376|     33348869|[]                                                                         |          90281546|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=66, WriteBufferFromFileDescriptorWriteBytes=33352005, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=19568, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, M|
clickhouse-01|2241c21e-d959-491c-9df6-27062f54ab24|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:56|2025-10-30 19:36:56.912402000|        847|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_12_12_0|all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_12_12_0/         |1111953|     33586440|[]                                                                         |          90905969|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33589575, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=36343, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, M|
clickhouse-01|                                    |MergeParts     |RegularMerge|Vertical       |2025-10-30|2025-10-30 19:36:56|2025-10-30 19:36:56.588345000|       3133|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_1_9_1_4|all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_1_9_1_4/         |5559765|    167883715|['all_1_1_0_4','all_5_5_0','all_6_6_0','all_7_7_0','all_8_8_0','all_9_9_0']|         454539112|  5559766| 615491834|         12047661|    0|                                                                                                                                                                                                                                                               |{FileOpen=145, Seek=16, ReadBufferFromFileDescriptorRead=1382, ReadBufferFromFileDescriptorReadBytes=168397832, WriteBufferFromFileDescriptorWrite=196, WriteBufferFromFileDescriptorWriteBytes=167921118, ReadCompressedBytes=168397483, CompressedReadBufferB|
clickhouse-01|2241c21e-d959-491c-9df6-27062f54ab24|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:55|2025-10-30 19:36:55.858431000|        893|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_11_11_0|all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_11_11_0/         |1111953|     33582733|[]                                                                         |          90900429|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33585868, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=26204, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, M|
clickhouse-01|2241c21e-d959-491c-9df6-27062f54ab24|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:54|2025-10-30 19:36:54.687274000|        613|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_10_10_0|all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_10_10_0/         |1111953|     33576228|[]                                                                         |          90904973|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33579361, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=16282, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, M|
clickhouse-01|                                    |MergePartsStart|RegularMerge|Undecided      |2025-10-30|2025-10-30 19:36:53|2025-10-30 19:36:53.628755000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_1_9_1_4|all         |tuple()  |Unknown  |         |                                                                                        |      0|            0|['all_1_1_0_4','all_5_5_0','all_6_6_0','all_7_7_0','all_8_8_0','all_9_9_0']|                 0|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|2241c21e-d959-491c-9df6-27062f54ab24|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:53|2025-10-30 19:36:53.537122000|        529|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_9_9_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_9_9_0/           |1111953|     33587653|[]                                                                         |          90905331|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33590788, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=14327, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, M|
clickhouse-01|2241c21e-d959-491c-9df6-27062f54ab24|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:52|2025-10-30 19:36:52.710890000|        548|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_8_8_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_8_8_0/           |1111953|     33578699|[]                                                                         |          90913121|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33581833, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=16304, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, M|
clickhouse-01|2241c21e-d959-491c-9df6-27062f54ab24|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:51|2025-10-30 19:36:51.931408000|        515|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_7_7_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_7_7_0/           |1111953|     33583561|[]                                                                         |          90902449|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33586696, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=13927, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, M|
clickhouse-01|2241c21e-d959-491c-9df6-27062f54ab24|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:51|2025-10-30 19:36:51.219703000|        494|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_6_6_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_6_6_0/           |1111953|     33585231|[]                                                                         |          90914034|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33588366, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=17515, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, M|
clickhouse-01|2241c21e-d959-491c-9df6-27062f54ab24|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:50|2025-10-30 19:36:50.492997000|        560|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_5_5_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_5_5_0/           |1111953|     33586844|[]                                                                         |          90905825|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33589979, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=21068, ZooKeeperTransactions=3, ZooKeeperExists=1, ZooKeeperMulti=2, ZooKe|
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:13|2025-10-30 19:36:13.612340000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_1_1_0  |all         |tuple()  |Compact  |         |                                                                                        |      1|         1874|[]                                                                         |               648|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:13|2025-10-30 19:36:13.612340000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_0_0_1_2|all         |tuple()  |Compact  |         |                                                                                        |      0|            1|[]                                                                         |                 0|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |MutatePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:33:15|2025-10-30 19:33:15.628791000|         19|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_1_1_0_4|all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_1_1_0_4/         |      1|         1973|['all_1_1_0_3']                                                            |               681|        1|       109|          5863065|    0|                                                                                                                                                                                                                                                               |{QueriesWithSubqueries=1, SelectQueriesWithSubqueries=1, FileOpen=11, ReadBufferFromFileDescriptorRead=11, ReadBufferFromFileDescriptorReadBytes=621, WriteBufferFromFileDescriptorWrite=10, WriteBufferFromFileDescriptorWriteBytes=3617, ReadCompressedBytes=|
clickhouse-01|                                    |MutatePartStart|NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:33:15|2025-10-30 19:33:15.610082000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_1_1_0_4|all         |tuple()  |Unknown  |         |                                                                                        |      0|            0|['all_1_1_0_3']                                                            |                 0|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:31:08|2025-10-30 19:31:08.241858000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_0_0_0_2|all         |tuple()  |Compact  |         |                                                                                        |      0|            1|[]                                                                         |                 0|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |MutatePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:30:43|2025-10-30 19:30:43.572323000|         10|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_1_1_0_3|all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_1_1_0_3/         |      1|         1874|['all_1_1_0_2']                                                            |               648|        1|       108|          5823132|    0|                                                                                                                                                                                                                                                               |{QueriesWithSubqueries=1, SelectQueriesWithSubqueries=1, FileOpen=11, ReadBufferFromFileDescriptorRead=11, ReadBufferFromFileDescriptorReadBytes=621, WriteBufferFromFileDescriptorWrite=10, WriteBufferFromFileDescriptorWriteBytes=3446, ReadCompressedBytes=|
clickhouse-01|                                    |MutatePartStart|NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:30:43|2025-10-30 19:30:43.562045000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_1_1_0_3|all         |tuple()  |Unknown  |         |                                                                                        |      0|            0|['all_1_1_0_2']                                                            |                 0|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:26:07|2025-10-30 19:26:07.268060000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_0_0_0  |all         |tuple()  |Compact  |         |                                                                                        |      1|         1878|[]                                                                         |               652|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |MutatePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:25:59|2025-10-30 19:25:59.124259000|         56|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_1_1_0_2|all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_1_1_0_2/         |      0|         1874|['all_1_1_0']                                                              |               648|        0|         0|           199328|    0|                                                                                                                                                                                                                                                               |{QueriesWithSubqueries=1, SelectQueriesWithSubqueries=1, FileOpen=8, ReadBufferFromFileDescriptorRead=16, ReadBufferFromFileDescriptorReadBytes=2783, ReadCompressedBytes=347, CompressedReadBufferBlocks=2, CompressedReadBufferBytes=767, OpenedFileCacheMiss|
clickhouse-01|                                    |MutatePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:25:59|2025-10-30 19:25:59.108260000|         65|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_0_0_0_2|all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_0_0_0_2/         |      0|            1|['all_0_0_0']                                                              |                 0|        0|         0|          4412163|    0|                                                                                                                                                                                                                                                               |{QueriesWithSubqueries=1, SelectQueriesWithSubqueries=1, FileOpen=9, WriteBufferFromFileDescriptorWrite=6, WriteBufferFromFileDescriptorWriteBytes=1469, IOBufferAllocs=24, IOBufferAllocBytes=5488104, FunctionExecute=1, QueryConditionCacheMisses=1, DiskWri|
clickhouse-01|                                    |MutatePartStart|NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:25:59|2025-10-30 19:25:59.071215000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_1_1_0_2|all         |tuple()  |Unknown  |         |                                                                                        |      0|            0|['all_1_1_0']                                                              |                 0|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |MutatePartStart|NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:25:59|2025-10-30 19:25:59.046451000|          0|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_0_0_0_2|all         |tuple()  |Unknown  |         |                                                                                        |      0|            0|['all_0_0_0']                                                              |                 0|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:23:04|2025-10-30 19:23:04.726794000|         13|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_1_1_0  |all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_1_1_0/           |      1|         1874|[]                                                                         |               648|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=18, ReadBufferFromFileDescriptorRead=16, ReadBufferFromFileDescriptorReadBytes=2783, WriteBufferFromFileDescriptorWrite=10, WriteBufferFromFileDescriptorWriteBytes=3446, ReadCompressedBytes=347, CompressedReadBufferBlocks=2, CompressedReadBuffer|
clickhouse-01|59324cfb-637f-477b-aa77-ddf7f882c8fb|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:19:17|2025-10-30 19:19:17.726103000|          1|learn_db|mart_student_lesson|09c91824-f4d4-451a-a445-eee11fefb702|all_0_0_0  |all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/09c/09c91824-f4d4-451a-a445-eee11fefb702/all_0_0_0/           |      1|         1878|[]                                                                         |               652|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=10, WriteBufferFromFileDescriptorWrite=10, WriteBufferFromFileDescriptorWriteBytes=3450, IOBufferAllocs=26, IOBufferAllocBytes=5494374, DiskWriteElapsedMicroseconds=73, ZooKeeperTransactions=4, ZooKeeperExists=1, ZooKeeperMulti=3, ZooKeeperMulti|
clickhouse-01|19330066-1e9a-4c3d-bfb2-389e15e7480b|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:07:58|2025-10-30 19:07:58.877772000|          1|learn_db|mart_student_lesson|f67f3766-b80f-46c3-a5ad-41d4b3d1cf23|all_5_5_0  |all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/f67/f67f3766-b80f-46c3-a5ad-41d4b3d1cf23/tmp_insert_all_5_5_0/|      1|         1864|[]                                                                         |               639|        0|         0|                0|  999|Code: 999. Coordination::Exception: Transaction failed (No node): Op #0, path: /clickhouse/tables/01/mart_student_lesson/replicas/01/host. (KEEPER_EXCEPTION), Stack trace (when copying this message, always include the lines below):¶¶0. DB::Exception::Exce|{FileOpen=10, WriteBufferFromFileDescriptorWrite=10, WriteBufferFromFileDescriptorWriteBytes=3436, IOBufferAllocs=26, IOBufferAllocBytes=5494374, DiskWriteElapsedMicroseconds=48, ZooKeeperTransactions=2, ZooKeeperExists=1, ZooKeeperMulti=1, ZooKeeperMulti|
clickhouse-01|7b288d3b-f923-4463-bb55-ef30031627a2|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:05:47|2025-10-30 19:05:47.856156000|          0|learn_db|mart_student_lesson|f67f3766-b80f-46c3-a5ad-41d4b3d1cf23|all_4_4_0  |all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/f67/f67f3766-b80f-46c3-a5ad-41d4b3d1cf23/tmp_insert_all_4_4_0/|      1|         1894|[]                                                                         |               668|        0|         0|                0|  999|Code: 999. Coordination::Exception: Transaction failed (No node): Op #0, path: /clickhouse/tables/01/mart_student_lesson/replicas/01/host. (KEEPER_EXCEPTION), Stack trace (when copying this message, always include the lines below):¶¶0. DB::Exception::Exce|{FileOpen=10, WriteBufferFromFileDescriptorWrite=10, WriteBufferFromFileDescriptorWriteBytes=3466, IOBufferAllocs=26, IOBufferAllocBytes=5494374, DiskWriteElapsedMicroseconds=34, ZooKeeperTransactions=2, ZooKeeperExists=1, ZooKeeperMulti=1, ZooKeeperMulti|
clickhouse-01|d9608402-f658-4850-8168-01d9e0f10d1a|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:05:14|2025-10-30 19:05:14.495466000|          1|learn_db|mart_student_lesson|f67f3766-b80f-46c3-a5ad-41d4b3d1cf23|all_3_3_0  |all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/f67/f67f3766-b80f-46c3-a5ad-41d4b3d1cf23/tmp_insert_all_3_3_0/|      1|         1869|[]                                                                         |               727|        0|         0|                0|  999|Code: 999. Coordination::Exception: Transaction failed (No node): Op #0, path: /clickhouse/tables/01/mart_student_lesson/replicas/01/host. (KEEPER_EXCEPTION), Stack trace (when copying this message, always include the lines below):¶¶0. DB::Exception::Exce|{FileOpen=10, WriteBufferFromFileDescriptorWrite=10, WriteBufferFromFileDescriptorWriteBytes=3504, IOBufferAllocs=28, IOBufferAllocBytes=6545124, DiskWriteElapsedMicroseconds=40, ZooKeeperTransactions=2, ZooKeeperExists=1, ZooKeeperMulti=1, ZooKeeperMulti|
clickhouse-01|4a47c762-7498-4d33-8ec5-c5d66fcdd389|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:04:41|2025-10-30 19:04:41.575476000|          1|learn_db|mart_student_lesson|f67f3766-b80f-46c3-a5ad-41d4b3d1cf23|all_2_2_0  |all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/f67/f67f3766-b80f-46c3-a5ad-41d4b3d1cf23/tmp_insert_all_2_2_0/|      1|         1880|[]                                                                         |               654|        0|         0|                0|  999|Code: 999. Coordination::Exception: Transaction failed (No node): Op #0, path: /clickhouse/tables/01/mart_student_lesson/replicas/01/host. (KEEPER_EXCEPTION), Stack trace (when copying this message, always include the lines below):¶¶0. DB::Exception::Exce|{FileOpen=10, WriteBufferFromFileDescriptorWrite=10, WriteBufferFromFileDescriptorWriteBytes=3452, IOBufferAllocs=26, IOBufferAllocBytes=5494374, DiskWriteElapsedMicroseconds=41, ZooKeeperTransactions=2, ZooKeeperExists=1, ZooKeeperMulti=1, ZooKeeperMulti|
clickhouse-01|01c56a12-9972-4e7d-9517-ad441809ff1e|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:04:20|2025-10-30 19:04:20.090620000|          1|learn_db|mart_student_lesson|f67f3766-b80f-46c3-a5ad-41d4b3d1cf23|all_1_1_0  |all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/f67/f67f3766-b80f-46c3-a5ad-41d4b3d1cf23/tmp_insert_all_1_1_0/|      1|         1875|[]                                                                         |               649|        0|         0|                0|  999|Code: 999. Coordination::Exception: Transaction failed (No node): Op #0, path: /clickhouse/tables/01/mart_student_lesson/replicas/01/host. (KEEPER_EXCEPTION), Stack trace (when copying this message, always include the lines below):¶¶0. DB::Exception::Exce|{FileOpen=10, WriteBufferFromFileDescriptorWrite=10, WriteBufferFromFileDescriptorWriteBytes=3447, IOBufferAllocs=26, IOBufferAllocBytes=5494374, DiskWriteElapsedMicroseconds=41, ZooKeeperTransactions=2, ZooKeeperExists=1, ZooKeeperMulti=1, ZooKeeperMulti|
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:12:07|2025-10-28 18:12:07.747258000|          0|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_2_2_0  |all         |tuple()  |Wide     |         |                                                                                        | 556183|     16799442|[]                                                                         |          45464486|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:12:07|2025-10-28 18:12:07.747258000|          0|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_4_4_0  |all         |tuple()  |Wide     |         |                                                                                        | 555933|     16791507|[]                                                                         |          45445604|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:12:07|2025-10-28 18:12:07.747258000|          0|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_3_3_0  |all         |tuple()  |Wide     |         |                                                                                        | 556399|     16806921|[]                                                                         |          45492477|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:12:07|2025-10-28 18:12:07.747258000|          0|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_5_5_0  |all         |tuple()  |Wide     |         |                                                                                        | 555751|     16788388|[]                                                                         |          45436158|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:12:07|2025-10-28 18:12:07.747258000|          0|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_1_1_0  |all         |tuple()  |Wide     |         |                                                                                        | 555751|     16788522|[]                                                                         |          45432818|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:12:07|2025-10-28 18:12:07.747258000|          0|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_0_0_0  |all         |tuple()  |Wide     |         |                                                                                        | 555917|     16792764|[]                                                                         |          45449071|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|06b237c0-5dec-469b-881f-e1caf5bab5f1|NewPart        |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:49|2025-10-28 18:01:49.230629000|        266|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_8_8_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_8_8_0/           | 552224|     16679742|[]                                                                         |          45148369|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=51, WriteBufferFromFileDescriptorWriteBytes=16682870, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=11098, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, M|
clickhouse-01|06b237c0-5dec-469b-881f-e1caf5bab5f1|NewPart        |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:48|2025-10-28 18:01:48.115682000|        437|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_7_7_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_7_7_0/           | 556569|     16817377|[]                                                                         |          45499751|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16820506, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=13668, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, M|
clickhouse-01|                                    |MergeParts     |RegularMerge|Vertical       |2025-10-28|2025-10-28 18:01:47|2025-10-28 18:01:47.879187000|       3049|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_0_5_1  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_0_5_1/           |3335934|    100735121|['all_0_0_0','all_1_1_0','all_2_2_0','all_3_3_0','all_4_4_0','all_5_5_0']  |         272718554|  3335934| 365956198|         12572281|    0|                                                                                                                                                                                                                                                               |{FileOpen=146, ReadBufferFromFileDescriptorRead=864, ReadBufferFromFileDescriptorReadBytes=101156533, WriteBufferFromFileDescriptorWrite=132, WriteBufferFromFileDescriptorWriteBytes=100766272, ReadCompressedBytes=101156533, CompressedReadBufferBlocks=3126|
clickhouse-01|06b237c0-5dec-469b-881f-e1caf5bab5f1|NewPart        |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:46|2025-10-28 18:01:46.425763000|        549|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_6_6_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_6_6_0/           | 555978|     16792762|[]                                                                         |          45454754|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16795894, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=16434, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, M|
clickhouse-01|                                    |MergePartsStart|RegularMerge|Undecided      |2025-10-28|2025-10-28 18:01:44|2025-10-28 18:01:44.829695000|          0|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_0_5_1  |all         |tuple()  |Unknown  |         |                                                                                        |      0|            0|['all_0_0_0','all_1_1_0','all_2_2_0','all_3_3_0','all_4_4_0','all_5_5_0']  |                 0|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{}                                                                                                                                                                                                                                                             |
clickhouse-01|06b237c0-5dec-469b-881f-e1caf5bab5f1|NewPart        |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:44|2025-10-28 18:01:44.708644000|        436|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_5_5_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_5_5_0/           | 555751|     16788388|[]                                                                         |          45436158|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16791516, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=13391, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, M|
clickhouse-01|06b237c0-5dec-469b-881f-e1caf5bab5f1|NewPart        |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:43|2025-10-28 18:01:43.361222000|        415|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_4_4_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_4_4_0/           | 555933|     16791507|[]                                                                         |          45445604|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16794636, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=13143, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, M|
clickhouse-01|06b237c0-5dec-469b-881f-e1caf5bab5f1|NewPart        |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:41|2025-10-28 18:01:41.810152000|        448|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_3_3_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_3_3_0/           | 556399|     16806921|[]                                                                         |          45492477|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16810052, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=14460, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, M|
clickhouse-01|06b237c0-5dec-469b-881f-e1caf5bab5f1|NewPart        |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:40|2025-10-28 18:01:40.730746000|        278|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_2_2_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_2_2_0/           | 556183|     16799442|[]                                                                         |          45464486|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16802574, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=9020, ZooKeeperTransactions=2, ZooKeeperMulti=2, ZooKeeperMultiWrite=2, Me|
clickhouse-01|06b237c0-5dec-469b-881f-e1caf5bab5f1|NewPart        |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:39|2025-10-28 18:01:39.840386000|        261|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_1_1_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_1_1_0/           | 555751|     16788522|[]                                                                         |          45432818|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16791653, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=8979, ZooKeeperTransactions=3, ZooKeeperExists=1, ZooKeeperMulti=2, ZooKee|
clickhouse-01|06b237c0-5dec-469b-881f-e1caf5bab5f1|NewPart        |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:38|2025-10-28 18:01:38.884745000|        226|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_0_0_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_0_0_0/           | 555917|     16792764|[]                                                                         |          45449071|        0|         0|                0|    0|                                                                                                                                                                                                                                                               |{FileOpen=42, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16795890, IOBufferAllocs=150, IOBufferAllocBytes=38200554, DiskWriteElapsedMicroseconds=7844, ZooKeeperTransactions=4, ZooKeeperExists=1, ZooKeeperMulti=3, ZooKee|
```

Результат - нода 3
```text
hostname     |query_id                            |event_type     |merge_reason|merge_algorithm|event_date|event_time         |event_time_microseconds      |duration_ms|database|table              |table_uuid                          |part_name  |partition_id|partition|part_type|disk_name|path_on_disk                                                                   |rows   |size_in_bytes|merged_from                                                                |bytes_uncompressed|read_rows|read_bytes|peak_memory_usage|error|exception|ProfileEvents                                                                                                                                                                                                                                                  |
-------------+------------------------------------+---------------+------------+---------------+----------+-------------------+-----------------------------+-----------+--------+-------------------+------------------------------------+-----------+------------+---------+---------+---------+-------------------------------------------------------------------------------+-------+-------------+---------------------------------------------------------------------------+------------------+---------+----------+-----------------+-----+---------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:46:23|2025-10-30 19:46:23.398783000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_9_9_0  |all         |tuple()  |Wide     |         |                                                                               |1111953|     33587653|[]                                                                         |          90905331|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:46:23|2025-10-30 19:46:23.398783000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_8_8_0  |all         |tuple()  |Wide     |         |                                                                               |1111953|     33578699|[]                                                                         |          90913121|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:46:23|2025-10-30 19:46:23.398783000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_7_7_0  |all         |tuple()  |Wide     |         |                                                                               |1111953|     33583561|[]                                                                         |          90902449|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:46:23|2025-10-30 19:46:23.398783000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_6_6_0  |all         |tuple()  |Wide     |         |                                                                               |1111953|     33585231|[]                                                                         |          90914034|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:46:23|2025-10-30 19:46:23.398783000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_5_5_0  |all         |tuple()  |Wide     |         |                                                                               |1111953|     33586844|[]                                                                         |          90905825|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:46:23|2025-10-30 19:46:23.398783000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_1_1_0_4|all         |tuple()  |Compact  |         |                                                                               |      1|         1973|[]                                                                         |               681|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:41:22|2025-10-30 19:41:22.473054000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_1_1_0_3|all         |tuple()  |Compact  |         |                                                                               |      1|         1874|[]                                                                         |               648|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:41:22|2025-10-30 19:41:22.473054000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_1_1_0_2|all         |tuple()  |Compact  |         |                                                                               |      1|         1874|[]                                                                         |               648|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:57|2025-10-30 19:36:57.060870000|        121|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_13_13_0|all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_13_13_0/|1104376|     33348869|[]                                                                         |          90281546|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4817, WriteBufferFromFileDescriptorWrite=66, WriteBufferFromFileDescriptorWriteBytes=33352005, ReadCompressedBytes=2272, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:57|2025-10-30 19:36:57.045207000|        120|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_12_12_0|all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_12_12_0/|1111953|     33586440|[]                                                                         |          90905969|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4798, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33589575, ReadCompressedBytes=2253, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |MergeParts     |RegularMerge|Vertical       |2025-10-30|2025-10-30 19:36:56|2025-10-30 19:36:56.579143000|       3118|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_1_9_1_4|all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_1_9_1_4/|5559765|    167883715|['all_1_1_0_4','all_5_5_0','all_6_6_0','all_7_7_0','all_8_8_0','all_9_9_0']|         454539112|  5559766| 615491834|         12047725|    0|         |{FileOpen=145, Seek=16, ReadBufferFromFileDescriptorRead=1382, ReadBufferFromFileDescriptorReadBytes=168397832, WriteBufferFromFileDescriptorWrite=196, WriteBufferFromFileDescriptorWriteBytes=167921118, ReadCompressedBytes=168397483, CompressedReadBufferB|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:56|2025-10-30 19:36:56.083243000|        226|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_11_11_0|all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_11_11_0/|1111953|     33582733|[]                                                                         |          90900429|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4806, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33585868, ReadCompressedBytes=2261, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:54|2025-10-30 19:36:54.812329000|        117|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_10_10_0|all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_10_10_0/|1111953|     33576228|[]                                                                         |          90904973|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4804, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33579361, ReadCompressedBytes=2259, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |MergePartsStart|RegularMerge|Undecided      |2025-10-30|2025-10-30 19:36:53|2025-10-30 19:36:53.633775000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_1_9_1_4|all         |tuple()  |Unknown  |         |                                                                               |      0|            0|['all_1_1_0_4','all_5_5_0','all_6_6_0','all_7_7_0','all_8_8_0','all_9_9_0']|                 0|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:53|2025-10-30 19:36:53.608050000|         65|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_9_9_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_9_9_0/  |1111953|     33587653|[]                                                                         |          90905331|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4791, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33590788, ReadCompressedBytes=2246, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:52|2025-10-30 19:36:52.771110000|         47|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_8_8_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_8_8_0/  |1111953|     33578699|[]                                                                         |          90913121|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4805, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33581833, ReadCompressedBytes=2260, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:52|2025-10-30 19:36:52.005189000|         68|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_7_7_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_7_7_0/  |1111953|     33583561|[]                                                                         |          90902449|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4798, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33586696, ReadCompressedBytes=2253, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:51|2025-10-30 19:36:51.281652000|         53|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_6_6_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_6_6_0/  |1111953|     33585231|[]                                                                         |          90914034|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4799, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33588366, ReadCompressedBytes=2254, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:50|2025-10-30 19:36:50.564104000|         66|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_5_5_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_5_5_0/  |1111953|     33586844|[]                                                                         |          90905825|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4806, WriteBufferFromFileDescriptorWrite=67, WriteBufferFromFileDescriptorWriteBytes=33589979, ReadCompressedBytes=2261, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:16|2025-10-30 19:36:16.762363000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_1_1_0  |all         |tuple()  |Compact  |         |                                                                               |      1|         1874|[]                                                                         |               648|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:36:16|2025-10-30 19:36:16.762363000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_0_0_1_2|all         |tuple()  |Compact  |         |                                                                               |      0|            1|[]                                                                         |                 0|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |MutatePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:33:15|2025-10-30 19:33:15.620968000|         11|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_1_1_0_4|all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_1_1_0_4/|      1|         1973|['all_1_1_0_3']                                                            |               681|        1|       109|          5861945|    0|         |{QueriesWithSubqueries=1, SelectQueriesWithSubqueries=1, FileOpen=11, ReadBufferFromFileDescriptorRead=11, ReadBufferFromFileDescriptorReadBytes=621, WriteBufferFromFileDescriptorWrite=10, WriteBufferFromFileDescriptorWriteBytes=3617, ReadCompressedBytes=|
clickhouse-03|                                    |MutatePartStart|NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:33:15|2025-10-30 19:33:15.610055000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_1_1_0_4|all         |tuple()  |Unknown  |         |                                                                               |      0|            0|['all_1_1_0_3']                                                            |                 0|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:31:07|2025-10-30 19:31:07.732594000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_0_0_0_2|all         |tuple()  |Compact  |         |                                                                               |      0|            1|[]                                                                         |                 0|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |MutatePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:30:43|2025-10-30 19:30:43.583094000|         22|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_1_1_0_3|all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_1_1_0_3/|      1|         1874|['all_1_1_0_2']                                                            |               648|        1|       108|          5824252|    0|         |{QueriesWithSubqueries=1, SelectQueriesWithSubqueries=1, FileOpen=11, ReadBufferFromFileDescriptorRead=11, ReadBufferFromFileDescriptorReadBytes=621, WriteBufferFromFileDescriptorWrite=10, WriteBufferFromFileDescriptorWriteBytes=3446, ReadCompressedBytes=|
clickhouse-03|                                    |MutatePartStart|NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:30:43|2025-10-30 19:30:43.562123000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_1_1_0_3|all         |tuple()  |Unknown  |         |                                                                               |      0|            0|['all_1_1_0_2']                                                            |                 0|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:26:07|2025-10-30 19:26:07.268082000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_0_0_0  |all         |tuple()  |Compact  |         |                                                                               |      1|         1878|[]                                                                         |               652|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |MutatePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:25:59|2025-10-30 19:25:59.112484000|         43|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_1_1_0_2|all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_1_1_0_2/|      0|         1874|['all_1_1_0']                                                              |               648|        0|         0|           196240|    0|         |{QueriesWithSubqueries=1, SelectQueriesWithSubqueries=1, FileOpen=8, ReadBufferFromFileDescriptorRead=16, ReadBufferFromFileDescriptorReadBytes=2783, ReadCompressedBytes=347, CompressedReadBufferBlocks=2, CompressedReadBufferBytes=767, OpenedFileCacheMiss|
clickhouse-03|                                    |MutatePartStart|NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:25:59|2025-10-30 19:25:59.071218000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_1_1_0_2|all         |tuple()  |Unknown  |         |                                                                               |      0|            0|['all_1_1_0']                                                              |                 0|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |MutatePart     |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:25:59|2025-10-30 19:25:59.055409000|          9|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_0_0_0_2|all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_0_0_0_2/|      0|            1|['all_0_0_0']                                                              |                 0|        0|         0|          4412691|    0|         |{QueriesWithSubqueries=1, SelectQueriesWithSubqueries=1, FileOpen=9, WriteBufferFromFileDescriptorWrite=6, WriteBufferFromFileDescriptorWriteBytes=1469, IOBufferAllocs=24, IOBufferAllocBytes=5488104, FunctionExecute=1, QueryConditionCacheMisses=1, DiskWri|
clickhouse-03|                                    |MutatePartStart|NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:25:59|2025-10-30 19:25:59.046483000|          0|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_0_0_0_2|all         |tuple()  |Unknown  |         |                                                                               |      0|            0|['all_0_0_0']                                                              |                 0|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|2aa63062-250d-4249-9a61-5ec2ff6e629b|NewPart        |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:23:04|2025-10-30 19:23:04.702208000|          1|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_1_1_0  |all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_1_1_0/  |      1|         1874|[]                                                                         |               648|        0|         0|                0|    0|         |{FileOpen=10, WriteBufferFromFileDescriptorWrite=10, WriteBufferFromFileDescriptorWriteBytes=3446, IOBufferAllocs=26, IOBufferAllocBytes=5494374, DiskWriteElapsedMicroseconds=71, ZooKeeperTransactions=3, ZooKeeperExists=1, ZooKeeperMulti=2, ZooKeeperMulti|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-30|2025-10-30 19:20:33|2025-10-30 19:20:33.670985000|         11|learn_db|mart_student_lesson|4bc9722b-1258-4b4f-ab9d-278b8a51aa0c|all_0_0_0  |all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/4bc/4bc9722b-1258-4b4f-ab9d-278b8a51aa0c/all_0_0_0/  |      1|         1878|[]                                                                         |               652|        0|         0|                0|    0|         |{FileOpen=18, ReadBufferFromFileDescriptorRead=16, ReadBufferFromFileDescriptorReadBytes=2783, WriteBufferFromFileDescriptorWrite=10, WriteBufferFromFileDescriptorWriteBytes=3450, ReadCompressedBytes=347, CompressedReadBufferBlocks=2, CompressedReadBuffer|
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:12:22|2025-10-28 18:12:22.954173000|          0|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_1_1_0  |all         |tuple()  |Wide     |         |                                                                               | 555751|     16788522|[]                                                                         |          45432818|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:12:22|2025-10-28 18:12:22.954173000|          0|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_4_4_0  |all         |tuple()  |Wide     |         |                                                                               | 555933|     16791507|[]                                                                         |          45445604|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:12:22|2025-10-28 18:12:22.954173000|          0|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_3_3_0  |all         |tuple()  |Wide     |         |                                                                               | 556399|     16806921|[]                                                                         |          45492477|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:12:22|2025-10-28 18:12:22.954173000|          0|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_2_2_0  |all         |tuple()  |Wide     |         |                                                                               | 556183|     16799442|[]                                                                         |          45464486|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:12:22|2025-10-28 18:12:22.954173000|          0|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_5_5_0  |all         |tuple()  |Wide     |         |                                                                               | 555751|     16788388|[]                                                                         |          45436158|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |RemovePart     |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:12:22|2025-10-28 18:12:22.954173000|          0|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_0_0_0  |all         |tuple()  |Wide     |         |                                                                               | 555917|     16792764|[]                                                                         |          45449071|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:49|2025-10-28 18:01:49.284336000|         38|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_8_8_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_8_8_0/  | 552224|     16679742|[]                                                                         |          45148369|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4580, WriteBufferFromFileDescriptorWrite=51, WriteBufferFromFileDescriptorWriteBytes=16682870, ReadCompressedBytes=2052, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:48|2025-10-28 18:01:48.183642000|         49|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_7_7_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_7_7_0/  | 556569|     16817377|[]                                                                         |          45499751|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4581, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16820506, ReadCompressedBytes=2053, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |MergeParts     |RegularMerge|Vertical       |2025-10-28|2025-10-28 18:01:47|2025-10-28 18:01:47.843229000|       3017|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_0_5_1  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_0_5_1/  |3335934|    100735121|['all_0_0_0','all_1_1_0','all_2_2_0','all_3_3_0','all_4_4_0','all_5_5_0']  |         272718554|  3335934| 365956198|         12439081|    0|         |{FileOpen=146, ReadBufferFromFileDescriptorRead=864, ReadBufferFromFileDescriptorReadBytes=101156533, WriteBufferFromFileDescriptorWrite=132, WriteBufferFromFileDescriptorWriteBytes=100766272, ReadCompressedBytes=101156533, CompressedReadBufferBlocks=3126|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:46|2025-10-28 18:01:46.648158000|        191|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_6_6_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_6_6_0/  | 555978|     16792762|[]                                                                         |          45454754|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4585, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16795894, ReadCompressedBytes=2056, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |MergePartsStart|RegularMerge|Undecided      |2025-10-28|2025-10-28 18:01:44|2025-10-28 18:01:44.825684000|          0|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_0_5_1  |all         |tuple()  |Unknown  |         |                                                                               |      0|            0|['all_0_0_0','all_1_1_0','all_2_2_0','all_3_3_0','all_4_4_0','all_5_5_0']  |                 0|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:44|2025-10-28 18:01:44.793890000|         71|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_5_5_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_5_5_0/  | 555751|     16788388|[]                                                                         |          45436158|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4581, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16791516, ReadCompressedBytes=2052, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:43|2025-10-28 18:01:43.452900000|         67|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_4_4_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_4_4_0/  | 555933|     16791507|[]                                                                         |          45445604|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4581, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16794636, ReadCompressedBytes=2053, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:41|2025-10-28 18:01:41.927026000|         70|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_3_3_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_3_3_0/  | 556399|     16806921|[]                                                                         |          45492477|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4584, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16810052, ReadCompressedBytes=2055, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:40|2025-10-28 18:01:40.789821000|         42|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_2_2_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_2_2_0/  | 556183|     16799442|[]                                                                         |          45464486|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4585, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16802574, ReadCompressedBytes=2056, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:39|2025-10-28 18:01:39.890208000|         37|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_1_1_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_1_1_0/  | 555751|     16788522|[]                                                                         |          45432818|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4583, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16791653, ReadCompressedBytes=2055, CompressedReadBufferBlocks=2, CompressedReadB|
clickhouse-03|                                    |DownloadPart   |NotAMerge   |Undecided      |2025-10-28|2025-10-28 18:01:38|2025-10-28 18:01:38.932367000|         37|learn_db|mart_student_lesson|d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37|all_0_0_0  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/d0b/d0bdbcd8-f60f-4f8a-8e1e-1772a27a1c37/all_0_0_0/  | 555917|     16792764|[]                                                                         |          45449071|        0|         0|                0|    0|         |{FileOpen=50, ReadBufferFromFileDescriptorRead=14, ReadBufferFromFileDescriptorReadBytes=4579, WriteBufferFromFileDescriptorWrite=52, WriteBufferFromFileDescriptorWriteBytes=16795890, ReadCompressedBytes=2050, CompressedReadBufferBlocks=2, CompressedReadB|
```

### 17. Посмотрим размер блока для вставки данных

```
select * from system.settings WHERE name = 'max_insert_block_size';
```

Результат
```text
name                 |value  |changed|description                                                                                                                                                                                                                                                    |min|max|disallowed_values|readonly|type         |default|alias_for|is_obsolete|tier      |
---------------------+-------+-------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---+---+-----------------+--------+-------------+-------+---------+-----------+----------+
max_insert_block_size|1048449|      0|The size of blocks (in a count of rows) to form for insertion into a table.¶This setting only applies in cases when the server forms the blocks.¶For example, for an INSERT via the HTTP interface, the server parses the data format and forms blocks of the s|   |   |[]               |       0|NonZeroUInt64|1048449|         |          0|Production|
```

Это максимальное количество строк, которое вставляется в одну часть за раз. И каждая часть данных затем синхронизировалась с репликой независимо.

### 18. Смотрим записи в ClickHouse Keeper

```sql
SELECT 
	*
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/mart_student_lesson';
```

Результат
```text
name                      |value                                                                                                                                                                                                                                                          |path                                     |
--------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------------------------+
pinned_part_uuids         |{"part_uuids":"[]"}                                                                                                                                                                                                                                            |/clickhouse/tables/01/mart_student_lesson|
block_numbers             |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/mart_student_lesson|
leader_election           |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/mart_student_lesson|
metadata                  |metadata format version: 1¶date column: ¶sampling expression: ¶index granularity: 8192¶mode: 0¶sign column: ¶primary key: lesson_date¶data format version: 1¶partition key: ¶sorting key: lesson_date¶granularity bytes: 10485760¶merge parameters format versi|/clickhouse/tables/01/mart_student_lesson|
columns                   |columns format version: 1¶16 columns:¶`student_profile_id` Int32¶`person_id` String¶`person_id_int` Int32 CODEC(Delta(4), ZSTD(1))¶`educational_organization_id` Int16¶`parallel_id` Int16¶`class_id` Int16¶`lesson_date` Date32¶`lesson_month_digits` String¶`|/clickhouse/tables/01/mart_student_lesson|
alter_partition_version   |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/mart_student_lesson|
nonincrement_block_numbers|                                                                                                                                                                                                                                                               |/clickhouse/tables/01/mart_student_lesson|
replicas                  |last added replica: 02                                                                                                                                                                                                                                         |/clickhouse/tables/01/mart_student_lesson|
part_moves_shard          |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/mart_student_lesson|
lost_part_count           |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/mart_student_lesson|
log                       |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/mart_student_lesson|
async_blocks              |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/mart_student_lesson|
blocks                    |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/mart_student_lesson|
quorum                    |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/mart_student_lesson|
temp                      |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/mart_student_lesson|
table_shared_id           |09c91824-f4d4-451a-a445-eee11fefb702                                                                                                                                                                                                                           |/clickhouse/tables/01/mart_student_lesson|
mutations                 |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/mart_student_lesson|
lightweight_updates       |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/mart_student_lesson|
```

### 19. Смотрим записи в логе

```sql
SELECT 
	*
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/mart_student_lesson/log';
```

Результат
```text
name          |value                                                                                                                                                                                |path                                         |
--------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------+
log-0000000015|format version: 4¶create_time: 2025-10-30 16:36:56¶source replica: 01¶block_id: all_7701835813759986690_8639394927090974159¶get¶all_12_12_0¶                                         |/clickhouse/tables/01/mart_student_lesson/log|
log-0000000016|format version: 4¶create_time: 2025-10-30 16:36:56¶source replica: 01¶block_id: all_17551751901831051858_15434558463619914125¶get¶all_13_13_0¶                                       |/clickhouse/tables/01/mart_student_lesson/log|
log-0000000007|format version: 4¶create_time: 2025-10-30 16:36:50¶source replica: 01¶block_id: all_10158413130226456495_11206346181967369555¶get¶all_5_5_0¶                                         |/clickhouse/tables/01/mart_student_lesson/log|
log-0000000008|format version: 4¶create_time: 2025-10-30 16:36:51¶source replica: 01¶block_id: all_12951208209131918488_10411956241868872681¶get¶all_6_6_0¶                                         |/clickhouse/tables/01/mart_student_lesson/log|
log-0000000009|format version: 4¶create_time: 2025-10-30 16:36:51¶source replica: 01¶block_id: all_15205408943320654978_16638047591482774501¶get¶all_7_7_0¶                                         |/clickhouse/tables/01/mart_student_lesson/log|
log-0000000010|format version: 4¶create_time: 2025-10-30 16:36:52¶source replica: 01¶block_id: all_16920191430922961787_3959712754250291380¶get¶all_8_8_0¶                                          |/clickhouse/tables/01/mart_student_lesson/log|
log-0000000011|format version: 4¶create_time: 2025-10-30 16:36:53¶source replica: 01¶block_id: all_4104553496225167033_935950602872231977¶get¶all_9_9_0¶                                            |/clickhouse/tables/01/mart_student_lesson/log|
log-0000000012|format version: 4¶create_time: 2025-10-30 16:36:53¶source replica: 02¶block_id: ¶merge¶all_1_1_0_4¶all_5_5_0¶all_6_6_0¶all_7_7_0¶all_8_8_0¶all_9_9_0¶into¶all_1_9_1_4¶deduplicate: 0¶|/clickhouse/tables/01/mart_student_lesson/log|
log-0000000013|format version: 4¶create_time: 2025-10-30 16:36:54¶source replica: 01¶block_id: all_12973614887863570047_12337923535607713889¶get¶all_10_10_0¶                                       |/clickhouse/tables/01/mart_student_lesson/log|
log-0000000014|format version: 4¶create_time: 2025-10-30 16:36:55¶source replica: 01¶block_id: all_1188892686880382233_5673743019009136270¶get¶all_11_11_0¶                                         |/clickhouse/tables/01/mart_student_lesson/log|
```

### 20. Выключим первую ноду и проверим количество записей на третьей ноде

```sql
SELECT count(*) FROM learn_db.mart_student_lesson;
```

Результат
```text
10000000
```

### 21. Вставляем строку на третью ноду

```sql
INSERT INTO learn_db.mart_student_lesson 
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
FROM numbers(1);
```
### 22. Включим первую ноту и проверим количество записей на обоих машинах

```sql
SELECT count(*) FROM learn_db.mart_student_lesson;
```

Результат - нода 1
```text
10000001
```

Результат - нода 3
```text
10000001
```

Clickhouse синхронизировал записи после включения первой ноды.

### 23. Создаем таблицу для формирования набора строк, которые затем будем несколько раз вставлять в основную таблицу

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_temp;
CREATE TABLE learn_db.mart_student_lesson_temp 
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
	PRIMARY KEY(lesson_date)
) ENGINE = MergeTree;
```

### 24. Наполняем временную таблицу строками

```sql
INSERT INTO learn_db.mart_student_lesson_temp
SELECT * FROM learn_db.mart_student_lesson where person_id_int < 10 order by person_id_int, lesson_date;
```

### 25. Проверим количество записей в таблице mart_student_lesson

```sql
SELECT count(*) FROM learn_db.mart_student_lesson;
```

Результат
```text
10000001
```

### 25. Несколько раз выполняем вставку строк из временной таблицы в основную и проверяем, меняется ли количество строк в основной таблице

1 итерация 

```sql
INSERT INTO learn_db.mart_student_lesson
SELECT * FROM learn_db.mart_student_lesson_temp;
```

```sql
SELECT count(*) FROM learn_db.mart_student_lesson;
```

Результат
```text
10000005
```

2 итерация 

```sql
INSERT INTO learn_db.mart_student_lesson
SELECT * FROM learn_db.mart_student_lesson_temp;
```

```sql
SELECT count(*) FROM learn_db.mart_student_lesson;
```

Результат
```text
10000005
```

3 итерация 

```sql
INSERT INTO learn_db.mart_student_lesson
SELECT * FROM learn_db.mart_student_lesson_temp;
```

```sql
SELECT count(*) FROM learn_db.mart_student_lesson;
```

Результат
```text
10000005
```

Вставка одних и тех же строк в одном и том же порядке повторно произведена не была. Но важно понимать что этот механизм реализован только для таблиц с движком Replicated*.

### 26. Создаем таблицу с движком MergeTree и несколько раз вставляем в нее один и тот же набор строк. Убеждаемся, что строки вставляются несколько раз

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_temp2;
CREATE TABLE learn_db.mart_student_lesson_temp2 
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
	PRIMARY KEY(lesson_date)
) ENGINE = MergeTree;
```

1 итерация 

```sql
INSERT INTO learn_db.mart_student_lesson_temp2
SELECT * FROM learn_db.mart_student_lesson_temp;
```

```sql
SELECT count(*) FROM learn_db.mart_student_lesson_temp2;
```

Результат
```text
4
```

2 итерация 

```sql
INSERT INTO learn_db.mart_student_lesson_temp2
SELECT * FROM learn_db.mart_student_lesson_temp;
```

```sql
SELECT count(*) FROM learn_db.mart_student_lesson_temp2;
```

Результат
```text
8
```

3 итерация 

```sql
INSERT INTO learn_db.mart_student_lesson_temp2
SELECT * FROM learn_db.mart_student_lesson_temp;
```

```sql
SELECT count(*) FROM learn_db.mart_student_lesson_temp2;
```

Результат
```text
12
```

### 27. Пересоздадим таблицу с универсальной конфигурацией для ZooKeeper

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
	PRIMARY KEY(lesson_date)
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/{database}.{table}', '{replica}');
```

### 28. Посмотрим записи в ClickHouse Keeper

```sql
SELECT 
	*
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/learn_db.mart_student_lesson';
```

Результат
```text
name                      |value                                                                                                                                                                                                                                                          |path                                              |
--------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------------------------------------+
replicas                  |last added replica: 02                                                                                                                                                                                                                                         |/clickhouse/tables/01/learn_db.mart_student_lesson|
table_shared_id           |f291bebf-f184-4084-85f1-ce49bf9587d0                                                                                                                                                                                                                           |/clickhouse/tables/01/learn_db.mart_student_lesson|
metadata                  |metadata format version: 1¶date column: ¶sampling expression: ¶index granularity: 8192¶mode: 0¶sign column: ¶primary key: lesson_date¶data format version: 1¶partition key: ¶sorting key: lesson_date¶granularity bytes: 10485760¶merge parameters format versi|/clickhouse/tables/01/learn_db.mart_student_lesson|
alter_partition_version   |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
block_numbers             |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
leader_election           |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
part_moves_shard          |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
nonincrement_block_numbers|                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
columns                   |columns format version: 1¶16 columns:¶`student_profile_id` Int32¶`person_id` String¶`person_id_int` Int32 CODEC(Delta(4), ZSTD(1))¶`educational_organization_id` Int16¶`parallel_id` Int16¶`class_id` Int16¶`lesson_date` Date32¶`lesson_month_digits` String¶`|/clickhouse/tables/01/learn_db.mart_student_lesson|
log                       |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
async_blocks              |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
blocks                    |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
quorum                    |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
pinned_part_uuids         |{"part_uuids":"[]"}                                                                                                                                                                                                                                            |/clickhouse/tables/01/learn_db.mart_student_lesson|
mutations                 |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
lost_part_count           |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
temp                      |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
lightweight_updates       |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
```

### 29. Изменим конфигурацию, задав значения по умолчанию для движка таблицы Replicated

```xml
<default_replica_path>/clickhouse/tables/{shard}/{database}/{table}</default_replica_path>
<default_replica_name>{replica}</default_replica_name>
```

### 30. Пересоздадим таблицу с универсальной конфигурацией для ZooKeeper

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
	PRIMARY KEY(lesson_date)
) ENGINE = ReplicatedMergeTree;
```

### 31. Перепроверим записи в ClickHouse Keeper

```sql
SELECT 
	*
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/learn_db.mart_student_lesson';
```

Результат
```text
name                      |value                                                                                                                                                                                                                                                          |path                                              |
--------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------------------------------------+
blocks                    |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
part_moves_shard          |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
async_blocks              |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
log                       |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
replicas                  |last added replica: 02                                                                                                                                                                                                                                         |/clickhouse/tables/01/learn_db.mart_student_lesson|
alter_partition_version   |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
leader_election           |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
columns                   |columns format version: 1¶16 columns:¶`student_profile_id` Int32¶`person_id` String¶`person_id_int` Int32 CODEC(Delta(4), ZSTD(1))¶`educational_organization_id` Int16¶`parallel_id` Int16¶`class_id` Int16¶`lesson_date` Date32¶`lesson_month_digits` String¶`|/clickhouse/tables/01/learn_db.mart_student_lesson|
nonincrement_block_numbers|                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
pinned_part_uuids         |{"part_uuids":"[]"}                                                                                                                                                                                                                                            |/clickhouse/tables/01/learn_db.mart_student_lesson|
block_numbers             |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
mutations                 |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
lightweight_updates       |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
temp                      |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
metadata                  |metadata format version: 1¶date column: ¶sampling expression: ¶index granularity: 8192¶mode: 0¶sign column: ¶primary key: lesson_date¶data format version: 1¶partition key: ¶sorting key: lesson_date¶granularity bytes: 10485760¶merge parameters format versi|/clickhouse/tables/01/learn_db.mart_student_lesson|
table_shared_id           |f291bebf-f184-4084-85f1-ce49bf9587d0                                                                                                                                                                                                                           |/clickhouse/tables/01/learn_db.mart_student_lesson|
quorum                    |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
lost_part_count           |                                                                                                                                                                                                                                                               |/clickhouse/tables/01/learn_db.mart_student_lesson|
```
