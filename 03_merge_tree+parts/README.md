# Изучаем части (parts) таблицы с движком MergeTree и их фоновые соединения (merge)

## 1. Создаем таблицу заново.

#### 1.1. Удаляем таблицу learn_db.mart_student_lesson

```sql
DROP TABLE learn_db.mart_student_lesson;
```

#### 1.2. Создаем таблицу learn_db.mart_student_lesson с 10 строками

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
) ENGINE = MergeTree()
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
FROM numbers(10);
```

#### 1.3. Смотрим созданные парты

Для начала можно выполнить запрос без фильтрации. 

```sql
SELECT * FROM system.parts;
```
Если обратить внимание на значения атрибута table, то можно увидеть что системные таблицы так же содержат много партов.

Выполним запрос с фильтрацией по созданной ранее таблице.

```sql
SELECT * FROM system.parts where table = 'mart_student_lesson';
```

Результат
```text
partition|name     |uuid                                |part_type|active|marks|rows|bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table              |engine   |disk_name|path                                                                         |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock|removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+---------+------------------------------------+---------+------+-----+----+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+-------------------+---------+---------+-----------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+----------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
tuple()  |all_1_1_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|  10|         2509|                 1171|                    842|              52|        109|                                 0|                                   0|                            0|2025-11-26 10:40:46|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|               1|    0|           1|                         18|                                  401|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_1_0/|52dcdf8948966dbdf5eeaeef1ae1b3d9|7207c14d8dda0f4cff97ad64ee9e2fea|acf58c0e1ad7c70cd2f973818eba65f4     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

Так как данные вставлены напрямую из insert`a, то параметр level равен 0. Скопируем путь к контейнеру и посмотрим на его содержимое. Откроем Docker Desktop, перейдем на вкладку Containers, нажмем на название контейнера clickhouse-course и перейдем на вкладку Files и открываем нужный нам каталог. 

```sql
SELECT * FROM system.part_log where table = 'mart_student_lesson' ORDER BY event_time desc;
```

Результат
```text
hostname    |query_id                            |event_type     |merge_reason|merge_algorithm|event_date|event_time         |event_time_microseconds      |duration_ms|database|table              |table_uuid                          |part_name  |partition_id|partition|part_type|disk_name|path_on_disk                                                                   |rows   |size_in_bytes|merged_from                                                                          |bytes_uncompressed|read_rows|read_bytes|peak_memory_usage|error|exception|ProfileEvents                                                                                                                                                                                                                                                  |
------------+------------------------------------+---------------+------------+---------------+----------+-------------------+-----------------------------+-----------+--------+-------------------+------------------------------------+-----------+------------+---------+---------+---------+-------------------------------------------------------------------------------+-------+-------------+-------------------------------------------------------------------------------------+------------------+---------+----------+-----------------+-----+---------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
19294dc11be6|e7d2b9a8-c001-44c7-8ca3-c25b50fed069|NewPart        |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:40:46|2025-11-26 10:40:46.803607000|          1|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_1_1_0  |all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_1_0/  |     10|         2509|[]                                                                                   |              1388|        0|         0|                0|    0|         |{FileOpen=9, WriteBufferFromFileDescriptorWrite=9, WriteBufferFromFileDescriptorWriteBytes=3158, IOBufferAllocs=25, IOBufferAllocBytes=5490215, DiskWriteElapsedMicroseconds=48, MergeTreeDataWriterRows=10, MergeTreeDataWriterUncompressedBytes=1162, MergeTr|
```

#### 1.4. Добавляем еще 10 строк

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
	numbers(10);
```

#### 1.5. Смотрим созданные парты

```sql
SELECT * FROM system.parts where table = 'mart_student_lesson';
```

Результат
```text
partition|name     |uuid                                |part_type|active|marks|rows|bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table              |engine   |disk_name|path                                                                         |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock|removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+---------+------------------------------------+---------+------+-----+----+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+-------------------+---------+---------+-----------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+----------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
tuple()  |all_1_1_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|  10|         2509|                 1171|                    842|              52|        109|                                 0|                                   0|                            0|2025-11-26 10:40:46|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|               1|    0|           1|                         18|                                  401|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_1_0/|52dcdf8948966dbdf5eeaeef1ae1b3d9|7207c14d8dda0f4cff97ad64ee9e2fea|acf58c0e1ad7c70cd2f973818eba65f4     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_2_2_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|  10|         2450|                 1112|                    783|              52|        109|                                 0|                                   0|                            0|2025-11-26 10:43:12|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               2|               2|    0|           2|                         18|                                  401|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_2_2_0/|1846949c83ee7d4977acb4b0aafd64e1|3e5dbdd4268aabe22941e0d3fe428287|1831b0d99c928be8f16715e285b24b47     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

```sql
SELECT * FROM system.part_log ORDER BY event_time desc;
```

Результат
```text
query_id                            |event_type     |merge_reason|merge_algorithm|event_date|event_time         |event_time_microseconds      |duration_ms|database|table              |table_uuid                          |part_name  |partition_id|partition|part_type|disk_name|path_on_disk                                                                   |rows    |size_in_bytes|merged_from                                                                          |bytes_uncompressed|read_rows|read_bytes|peak_memory_usage|error|exception|ProfileEvents                                                                                                                                                                                                                                                  |
------------------------------------+---------------+------------+---------------+----------+-------------------+-----------------------------+-----------+--------+-------------------+------------------------------------+-----------+------------+---------+---------+---------+-------------------------------------------------------------------------------+--------+-------------+-------------------------------------------------------------------------------------+------------------+---------+----------+-----------------+-----+---------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
acffa967-bd33-49bd-95e8-0d9ad205cb08|NewPart        |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:43:12|2025-11-26 10:43:12.061979000|          1|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_2_2_0  |all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_2_2_0/  |      10|         2450|[]                                                                                   |              1329|        0|         0|                0|    0|         |{FileOpen=9, WriteBufferFromFileDescriptorWrite=9, WriteBufferFromFileDescriptorWriteBytes=3099, IOBufferAllocs=25, IOBufferAllocBytes=5490215, DiskWriteElapsedMicroseconds=101, MergeTreeDataWriterRows=10, MergeTreeDataWriterUncompressedBytes=1103, MergeT|
e7d2b9a8-c001-44c7-8ca3-c25b50fed069|NewPart        |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:40:46|2025-11-26 10:40:46.803607000|          1|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_1_1_0  |all         |tuple()  |Compact  |default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_1_0/  |      10|         2509|[]                                                                                   |              1388|        0|         0|                0|    0|         |{FileOpen=9, WriteBufferFromFileDescriptorWrite=9, WriteBufferFromFileDescriptorWriteBytes=3158, IOBufferAllocs=25, IOBufferAllocBytes=5490215, DiskWriteElapsedMicroseconds=48, MergeTreeDataWriterRows=10, MergeTreeDataWriterUncompressedBytes=1162, MergeTr|
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:40:46|2025-11-26 10:40:46.100300000|          0|learn_db|mt_3               |fcf1215d-f31e-45e7-bb65-36f8e74b604a|all_9_9_0  |all         |tuple()  |Compact  |         |                                                                               | 1104376|      8878252|[]                                                                                   |           8840992|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:40:46|2025-11-26 10:40:46.100300000|          0|learn_db|mt_3               |fcf1215d-f31e-45e7-bb65-36f8e74b604a|all_8_8_0  |all         |tuple()  |Compact  |         |                                                                               | 1111953|      8939173|[]                                                                                   |           8901652|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:40:46|2025-11-26 10:40:46.100300000|          0|learn_db|mt_3               |fcf1215d-f31e-45e7-bb65-36f8e74b604a|all_7_7_0  |all         |tuple()  |Compact  |         |                                                                               | 1111953|      8939166|[]                                                                                   |           8901652|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:39:59|2025-11-26 10:39:59.123566000|          0|learn_db|test_tbl           |2fe862e3-08d2-4023-8acc-90af6ae40d30|all_25_30_1|all         |tuple()  |Wide     |         |                                                                               | 5992423|     48078055|[]                                                                                   |          71950124|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:39:59|2025-11-26 10:39:59.123566000|          0|learn_db|test_tbl           |2fe862e3-08d2-4023-8acc-90af6ae40d30|all_19_24_1|all         |tuple()  |Wide     |         |                                                                               | 6671718|     53528219|[]                                                                                   |          80106312|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:39:59|2025-11-26 10:39:59.123566000|          0|learn_db|test_tbl           |2fe862e3-08d2-4023-8acc-90af6ae40d30|all_13_18_1|all         |tuple()  |Wide     |         |                                                                               | 5992423|     48077580|[]                                                                                   |          71950124|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:39:59|2025-11-26 10:39:59.123566000|          0|learn_db|test_tbl           |2fe862e3-08d2-4023-8acc-90af6ae40d30|all_7_12_1 |all         |tuple()  |Wide     |         |                                                                               | 6671718|     53528121|[]                                                                                   |          80106312|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:39:59|2025-11-26 10:39:59.123566000|          0|learn_db|test_tbl           |2fe862e3-08d2-4023-8acc-90af6ae40d30|all_1_6_1  |all         |tuple()  |Wide     |         |                                                                               | 6671718|     53527029|[]                                                                                   |          80106312|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:39:11|2025-11-26 10:39:11.333945000|          0|learn_db|mt_1               |43e52701-1535-44e6-9b04-547ae90f63c8|all_9_9_0  |all         |tuple()  |Compact  |         |                                                                               | 1104376|      8878252|[]                                                                                   |           8840992|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:39:11|2025-11-26 10:39:11.333945000|          0|learn_db|mt_1               |43e52701-1535-44e6-9b04-547ae90f63c8|all_8_8_0  |all         |tuple()  |Compact  |         |                                                                               | 1111953|      8939173|[]                                                                                   |           8901652|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:39:11|2025-11-26 10:39:11.333945000|          0|learn_db|mt_1               |43e52701-1535-44e6-9b04-547ae90f63c8|all_7_7_0  |all         |tuple()  |Compact  |         |                                                                               | 1111953|      8939166|[]                                                                                   |           8901652|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:39:09|2025-11-26 10:39:09.071211000|          0|learn_db|mt_2               |c904da8b-347d-46d4-8d7e-600b31f61760|all_9_9_0  |all         |tuple()  |Compact  |         |                                                                               | 1104376|      8878252|[]                                                                                   |           8840992|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:39:09|2025-11-26 10:39:09.071211000|          0|learn_db|mt_2               |c904da8b-347d-46d4-8d7e-600b31f61760|all_8_8_0  |all         |tuple()  |Compact  |         |                                                                               | 1111953|      8939173|[]                                                                                   |           8901652|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 10:39:09|2025-11-26 10:39:09.071211000|          0|learn_db|mt_2               |c904da8b-347d-46d4-8d7e-600b31f61760|all_7_7_0  |all         |tuple()  |Compact  |         |                                                                               | 1111953|      8939166|[]                                                                                   |           8901652|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |MergeParts     |RegularMerge|Horizontal     |2025-11-26|2025-11-26 10:30:59|2025-11-26 10:30:59.532928000|       3934|learn_db|test_tbl           |2fe862e3-08d2-4023-8acc-90af6ae40d30|all_1_30_2 |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/2fe/2fe862e3-08d2-4023-8acc-90af6ae40d30/all_1_30_2/ |32000000|    176734517|['all_1_6_1','all_7_12_1','all_13_18_1','all_19_24_1','all_25_30_1']                 |         384218848| 32000000| 384000000|         10126344|    0|         |{FileOpen=21, ReadBufferFromFileDescriptorRead=1963, ReadBufferFromFileDescriptorReadBytes=256712594, WriteBufferFromFileDescriptorWrite=182, WriteBufferFromFileDescriptorWriteBytes=176734953, ReadCompressedBytes=256712594, CompressedReadBufferBlocks=5865|
                                    |MergeParts     |RegularMerge|Horizontal     |2025-11-26|2025-11-26 10:30:55|2025-11-26 10:30:55.994635000|        391|learn_db|mt_1               |43e52701-1535-44e6-9b04-547ae90f63c8|all_7_9_1  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/43e/43e52701-1535-44e6-9b04-547ae90f63c8/all_7_9_1/  | 3328282|     26744747|['all_7_7_0','all_8_8_0','all_9_9_0']                                                |          26647472|  3328282|  26626256|          6742663|    0|         |{FileOpen=17, ReadBufferFromFileDescriptorRead=817, ReadBufferFromFileDescriptorReadBytes=26755091, WriteBufferFromFileDescriptorWrite=35, WriteBufferFromFileDescriptorWriteBytes=26745143, ReadCompressedBytes=26755091, CompressedReadBufferBlocks=817, Comp|
                                    |MergeParts     |RegularMerge|Horizontal     |2025-11-26|2025-11-26 10:30:55|2025-11-26 10:30:55.968420000|        372|learn_db|mt_2               |c904da8b-347d-46d4-8d7e-600b31f61760|all_7_9_1  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/c90/c904da8b-347d-46d4-8d7e-600b31f61760/all_7_9_1/  | 3328282|     26744747|['all_7_7_0','all_8_8_0','all_9_9_0']                                                |          26647472|  3328282|  26626256|          6742663|    0|         |{FileOpen=17, ReadBufferFromFileDescriptorRead=817, ReadBufferFromFileDescriptorReadBytes=26755091, WriteBufferFromFileDescriptorWrite=35, WriteBufferFromFileDescriptorWriteBytes=26745143, ReadCompressedBytes=26755091, CompressedReadBufferBlocks=817, Comp|
                                    |MergeParts     |RegularMerge|Horizontal     |2025-11-26|2025-11-26 10:30:55|2025-11-26 10:30:55.955025000|        359|learn_db|mt_3               |fcf1215d-f31e-45e7-bb65-36f8e74b604a|all_7_9_1  |all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/fcf/fcf1215d-f31e-45e7-bb65-36f8e74b604a/all_7_9_1/  | 3328282|     26744747|['all_7_7_0','all_8_8_0','all_9_9_0']                                                |          26647472|  3328282|  26626256|          6742727|    0|         |{FileOpen=17, ReadBufferFromFileDescriptorRead=817, ReadBufferFromFileDescriptorReadBytes=26755091, WriteBufferFromFileDescriptorWrite=35, WriteBufferFromFileDescriptorWriteBytes=26745143, ReadCompressedBytes=26755091, CompressedReadBufferBlocks=817, Comp|
                                    |MergePartsStart|RegularMerge|Undecided      |2025-11-26|2025-11-26 10:30:55|2025-11-26 10:30:55.603162000|          0|learn_db|mt_1               |43e52701-1535-44e6-9b04-547ae90f63c8|all_7_9_1  |all         |tuple()  |Unknown  |         |                                                                               |       0|            0|['all_7_7_0','all_8_8_0','all_9_9_0']                                                |                 0|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |MergePartsStart|RegularMerge|Undecided      |2025-11-26|2025-11-26 10:30:55|2025-11-26 10:30:55.597780000|          0|learn_db|test_tbl           |2fe862e3-08d2-4023-8acc-90af6ae40d30|all_1_30_2 |all         |tuple()  |Unknown  |         |                                                                               |       0|            0|['all_1_6_1','all_7_12_1','all_13_18_1','all_19_24_1','all_25_30_1']                 |                 0|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |MergePartsStart|RegularMerge|Undecided      |2025-11-26|2025-11-26 10:30:55|2025-11-26 10:30:55.595709000|          0|learn_db|mt_2               |c904da8b-347d-46d4-8d7e-600b31f61760|all_7_9_1  |all         |tuple()  |Unknown  |         |                                                                               |       0|            0|['all_7_7_0','all_8_8_0','all_9_9_0']                                                |                 0|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |MergePartsStart|RegularMerge|Undecided      |2025-11-26|2025-11-26 10:30:55|2025-11-26 10:30:55.595674000|          0|learn_db|mt_3               |fcf1215d-f31e-45e7-bb65-36f8e74b604a|all_7_9_1  |all         |tuple()  |Unknown  |         |                                                                               |       0|            0|['all_7_7_0','all_8_8_0','all_9_9_0']                                                |                 0|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-13|2025-11-13 17:43:43|2025-11-13 17:43:43.738947000|          0|learn_db|test_tbl           |2fe862e3-08d2-4023-8acc-90af6ae40d30|all_30_30_0|all         |tuple()  |Compact  |         |                                                                               |  432658|      3472391|[]                                                                                   |           5194488|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
```

```sql
SELECT * FROM system.merges;
```

Результат
```text
database|table|elapsed|progress|num_parts|source_part_names|result_part_name|source_part_paths|result_part_path|partition_id|partition|is_mutation|total_size_bytes_compressed|total_size_bytes_uncompressed|total_size_marks|bytes_read_uncompressed|rows_read|bytes_written_uncompressed|rows_written|columns_written|memory_usage|thread_id|merge_type|merge_algorithm|
--------+-----+-------+--------+---------+-----------------+----------------+-----------------+----------------+------------+---------+-----------+---------------------------+-----------------------------+----------------+-----------------------+---------+--------------------------+------------+---------------+------------+---------+----------+---------------+
```

Видим что процесс объединения партов не начался.

#### 1.6. Сохраняем в файл query.sql запрос вставки 10 строк в таблицу learn_db.mart_student_lesson

Переходим в контейнере на вкладку Exec и вводим команду bash. Далее вставляем следующий запрос.

```sql
echo "INSERT INTO learn_db.mart_student_lesson
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
	numbers(10);" > query.sql
```

В файле query.sql будет сохранен код запроса, который дальше мы будем запускать через команду clickhouse-benchmark.

Выполнив команду ls, увидим результат

```bash
root@19294dc11be6:/# ls
bin   dev                         entrypoint.sh  home  lib32  libx32  mnt  proc       result.tsv  run   srv  tmp  var
boot  docker-entrypoint-initdb.d  etc            lib   lib64  media   opt  query.sql  root        sbin  sys  usr  weather_data
```

#### 1.6. Запускаем 10000 запросов на вставку по 10 строк в 10 параллельных потоков. То есть всего вставляем 10 000 строк.

```bash
clickhouse-benchmark -i 10000 -c 10 --query "`cat query.sql`"
```

Сразу же переключаемся в dbeaver и смотрим на выполняемые слияния.

```sql
SELECT * FROM system.merges;
```

Результат
```text
database|table              |elapsed  |progress|num_parts|source_part_names                                                                                            |result_part_name|source_part_paths                                                                                                                                                                                                                                              |result_part_path                                                                   |partition_id|partition|is_mutation|total_size_bytes_compressed|total_size_bytes_uncompressed|total_size_marks|bytes_read_uncompressed|rows_read|bytes_written_uncompressed|rows_written|columns_written|memory_usage|thread_id|merge_type|merge_algorithm|
--------+-------------------+---------+--------+---------+-------------------------------------------------------------------------------------------------------------+----------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------------------------------------------------------------------+------------+---------+-----------+---------------------------+-----------------------------+----------------+-----------------------+---------+--------------------------+------------+---------------+------------+---------+----------+---------------+
learn_db|mart_student_lesson| 0.055654|     1.0|        2|['all_1_3166_140','all_3167_3182_2']                                                                         |all_1_3182_141  |['/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_3166_140/','/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_3167_3182_2/']                                                                                   |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_3182_141/ |all         |tuple()  |          0|                     756572|                      2545274|               7|                3563514|    31820|                   5906432|       31820|              0|     6359189|      660|Regular   |Horizontal     |
learn_db|mart_student_lesson|0.0129794|     1.0|        6|['all_3233_3233_0','all_3234_3234_0','all_3235_3235_0','all_3236_3236_0','all_3237_3237_0','all_3238_3238_0']|all_3233_3238_1 |['/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_3233_3233_0/','/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_3234_3234_0/','/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_3235_3235_|/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_3233_3238_1/|all         |tuple()  |          0|                      14905|                         4744|              12|                   6664|       60|                     70656|          60|              0|     5388657|      661|Regular   |Horizontal     |
```

В контейнере видим что вставка данныхъ завершилась
```text
Queries executed: 10000.

localhost:9000, queries: 10000, QPS: 520.168, RPS: 5201.677, MiB/s: 0.040, result RPS: 0.000, result MiB/s: 0.000.

0%              0.007 sec.
10%             0.011 sec.
20%             0.012 sec.
30%             0.013 sec.
40%             0.014 sec.
50%             0.016 sec.
60%             0.017 sec.
70%             0.019 sec.
80%             0.021 sec.
90%             0.027 sec.
95%             0.033 sec.
99%             0.051 sec.
99.9%           0.098 sec.
99.99%          0.112 sec.


root@19294dc11be6:/#
```

И в таблице system.merges записей нет.

#### 1.7. Смотрим распределение частей по уровням и соединения частей

```sql
SELECT * FROM system.parts where table = 'mart_student_lesson';
```

Результат
```text
partition|name         |uuid                                |part_type|active|marks|rows |bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table              |engine   |disk_name|path                                                                             |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock    |removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                       |
---------+-------------+------------------------------------+---------+------+-----+-----+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+-------------------+---------+---------+---------------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+--------------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+------------------------------------+
tuple()  |all_1_1_0    |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|   10|         2509|                 1171|                    842|              52|        109|                                 0|                                   0|                            0|2025-11-26 10:40:46|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|               1|    0|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_1_0/    |52dcdf8948966dbdf5eeaeef1ae1b3d9|7207c14d8dda0f4cff97ad64ee9e2fea|acf58c0e1ad7c70cd2f973818eba65f4     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_6_1    |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|   60|         4422|                 3074|                   4819|              52|        118|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|               6|    1|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_6_1/    |04d3baeadb71136b95cde856ea275d51|b48d3f2bcbe0aa8e435010aa5ad21d62|6e09939078c9075511e770df7ab36735     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_16_2   |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  160|         7794|                 6424|                  12817|              52|        123|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|              16|    2|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_16_2/   |4c1878b252f5727ee1fbda38dc95a1e3|15fa22bdbc78dc542e843618d5ed7478|c3802a8fb818167b0b508fb1d765cccf     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_33_3   |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  330|        12918|                11543|                  26415|              52|        126|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|              33|    3|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_33_3/   |cf85ac400bc189e3d2898eea263d308f|5f1f1faa76f2f2ba1129ca86a5debed0|07c19dfa158ddbcd06738dfb6a36ec69     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_51_4   |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  510|        18211|                16835|                  40885|              52|        126|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|              51|    4|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_51_4/   |a766f17b48224cb73ed186de960a600b|1c35ac23658aa6326af438e44f7b0696|11227b90245c44eefe7d8504dda842eb     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_61_5   |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  610|        20974|                19598|                  48940|              52|        126|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|              61|    5|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_61_5/   |e07cc0c080146bee46a32b60f00358cb|68d65872ffc88d1a9c4646353273f001|5ff86aad9f389d22bd75b0af93370d3a     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_71_6   |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  710|        23818|                22445|                  57018|              52|        123|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|              71|    6|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_71_6/   |313ac9d06fb7081fe670725d1b6bfc69|0c7ee62ee80ed2a5d767ef29d8fd2c76|e0e91473b599514c1e0fbe52995e71f0     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_78_7   |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  780|        25775|                24401|                  62667|              52|        124|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|              78|    7|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_78_7/   |3a75f4ee131711069bf39826d9b6afd0|e55c5fbaf2ea4eb6f46a06134828b840|e499b251687bb08e986f7f7adb687388     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_89_8   |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  890|        28795|                27422|                  71551|              52|        123|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|              89|    8|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_89_8/   |82373fac9018d2f59f989bf64d2d8ee8|2f6fe5c88866d777a898d0e55584929a|6891a05a0697f6a698cfaa6b1cc18ffd     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_95_9   |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  950|        30348|                28974|                  76366|              52|        124|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|              95|    9|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_95_9/   |44641e3b525378c5cef206759c9b206a|e32e04de0134428bee114c3ffd902b55|515696c0bdcf262acedf1a92f0830f12     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_106_10 |00000000-0000-0000-0000-000000000000|Compact  |     0|    2| 1060|        33273|                31883|                  85198|              52|        123|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|             106|   10|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_106_10/ |20ed68a0df46d0e8fba003a96e1652f1|103a47738bfecc4362096a80170c9817|7076623ce7d706389199ce4db4377e3f     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_123_11 |00000000-0000-0000-0000-000000000000|Compact  |     0|    2| 1230|        37851|                36461|                  98858|              52|        123|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|             123|   11|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_123_11/ |3752be32fa35288d23073e043125138c|ab4912092059820460656b8e677d4524|cba0e7a133a9611345332df5e602f6fe     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_136_12 |00000000-0000-0000-0000-000000000000|Compact  |     0|    2| 1360|        41175|                39782|                 109195|              52|        126|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|             136|   12|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_136_12/ |184b4db04ce09d4a5800809422f7d848|350dd6901533cc8963f34958f8276ce5|491eb18b4dc59c546f8f087881bef636     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_155_13 |00000000-0000-0000-0000-000000000000|Compact  |     0|    2| 1550|        46133|                44740|                 124518|              52|        126|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|             155|   13|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_155_13/ |8133cc4c5de663b72bef17e90c6cf3bf|db8ea0cac380b36f0d13e36d57f9b4de|eba1c76914e6fd979acdc023a113f00d     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_171_14 |00000000-0000-0000-0000-000000000000|Compact  |     0|    2| 1710|        50327|                48934|                 137351|              52|        126|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|             171|   14|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_171_14/ |eb281343ca94248f32920e29ed91cd96|6470bc3aed52ba7bc781de52748bd904|b4c6f765bbc3783cc94f126e9f34cd8a     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_183_15 |00000000-0000-0000-0000-000000000000|Compact  |     0|    2| 1830|        53330|                51936|                 146879|              52|        126|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|             183|   15|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_183_15/ |7e75cffed5a6692f4c09f810e799a595|349f358e145c1216032b55f9fe6e4fd9|eb485bbd1266d9e0340c8137f046731f     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_197_16 |00000000-0000-0000-0000-000000000000|Compact  |     0|    2| 1970|        56903|                55511|                 158067|              52|        124|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|             197|   16|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_197_16/ |b357faf18f9349c6fbf74da1f0474fb4|326b3e083109cd6bda76e9f65bd58004|9cafbb17380789b84d06f039150044ca     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_204_17 |00000000-0000-0000-0000-000000000000|Compact  |     0|    2| 2040|        58825|                57435|                 163632|              52|        123|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|             204|   17|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_204_17/ |96fa9135a30b12eb156caca7ecbbc7c8|6e8da19ac95e9462456c621691fc30fb|88a57825eac0b8fa3efa620aa87f1c97     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_214_18 |00000000-0000-0000-0000-000000000000|Compact  |     0|    2| 2140|        61279|                59885|                 171683|              52|        126|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|             214|   18|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_214_18/ |c99608f920290fb4399f824442aa746d|1f345683053396286aa204d9fc787915|aa9b3793895bd8f60d67771957439434     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_226_19 |00000000-0000-0000-0000-000000000000|Compact  |     0|    2| 2260|        64412|                63017|                 181206|              52|        126|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|             226|   19|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_226_19/ |2e9a7c96b5704397adcd09fc1fc078f9|342028c5b390acd4ef365009a3a8f230|e40d190d2256f013771a5a3304029b79     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_240_20 |00000000-0000-0000-0000-000000000000|Compact  |     0|    2| 2400|        67858|                66455|                 192384|              52|        135|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|             240|   20|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_240_20/ |f7bd094ecf375675bf0ad7157096115a|f81ecb2181f73af586ac2cf091bf641d|d70100b092a1f3f8dd34a74dca9a3be8     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
tuple()  |all_1_252_21 |00000000-0000-0000-0000-000000000000|Compact  |     0|    2| 2520|        70865|                69467|                 202060|              52|        129|                                 0|                                   0|                            0|2025-11-26 11:20:43|2025-11-26 11:20:43|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|             252|   21|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_252_21/ |9b7ad4554cc6b8e2c72713c803185444|5c0295eddfbf9cc2301994f8f3e29dbf|5a01891d110e176406caae71d2191f80     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 11:28:06|Part hasn't reached removal time yet|
...
```

Видим множество записей с active=0. Это значит что парты неактивны, т.е. они были созданы и попали в более крупные парты. Строк (rows) в каких-то 10, в каких-то 60 и больше. В атрибуте level видим уровень, он идет по увеличению.

Выполним следующий запрос

```sql
SELECT level, active, count(*), sum(rows) FROM system.parts where table = 'mart_student_lesson' GROUP BY level, active ORDER BY level;
```

Результат
```text
level|active|count()|sum(rows)|
-----+------+-------+---------+
    0|     1|      2|       20|
    3|     1|      1|      290|
  223|     1|      1|    99710|
```

Видим 3 активных парта. У одного уровень 223.

```sql
SELECT * FROM system.merges;
```

Результат
```text
database|table|elapsed|progress|num_parts|source_part_names|result_part_name|source_part_paths|result_part_path|partition_id|partition|is_mutation|total_size_bytes_compressed|total_size_bytes_uncompressed|total_size_marks|bytes_read_uncompressed|rows_read|bytes_written_uncompressed|rows_written|columns_written|memory_usage|thread_id|merge_type|merge_algorithm|
--------+-----+-------+--------+---------+-----------------+----------------+-----------------+----------------+------------+---------+-----------+---------------------------+-----------------------------+----------------+-----------------------+---------+--------------------------+------------+---------------+------------+---------+----------+---------------+
```

```sql
SELECT * FROM system.parts where table = 'mart_student_lesson' and active;
```

Результат
```text
partition|name             |uuid                                |part_type|active|marks|rows |bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table              |engine   |disk_name|path                                                                                 |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock|removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+-----------------+------------------------------------+---------+------+-----+-----+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+-------------------+---------+---------+-------------------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+----------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
tuple()  |all_1_9971_223   |00000000-0000-0000-0000-000000000000|Compact  |     1|   14|99710|      2246461|              2244340|                7974330|             148|        733|                                 0|                                   0|                            0|2025-11-26 11:21:02|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|            9971|  223|           1|                         56|                                  184|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_9971_223/   |69c65ef043fd645155f582540d9671a7|09ed6d2615eca9e7ee9ed62f7a0acf85|8ce2fb6a12c9731407d16378697efa06     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_9972_10000_3 |00000000-0000-0000-0000-000000000000|Compact  |     1|    2|  290|        11710|                10335|                  23063|              52|        126|                                 0|                                   0|                            0|2025-11-26 11:21:02|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |            9972|           10000|    3|        9972|                         18|                                  401|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_9972_10000_3/ |7e1dd519454704440899c8ecd0ee49c7|99bf5c9bc5682351c3894341bb85044e|a32c71607cd91cbf7b793d4ffe826e85     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_10001_10001_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|   10|         2467|                 1129|                    778|              52|        109|                                 0|                                   0|                            0|2025-11-26 11:21:02|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |           10001|           10001|    0|       10001|                         18|                                  401|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_10001_10001_0/|4b6348ba28a43ebc9fb928822db6c51d|76c5dc333b4c2d41d8caf0c5081e5d4b|3385d876da7c20465980329d24985a07     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_10002_10002_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|   10|         2503|                 1165|                    825|              52|        109|                                 0|                                   0|                            0|2025-11-26 11:21:02|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |           10002|           10002|    0|       10002|                         18|                                  401|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_10002_10002_0/|463d9bff02d69ab948550329a8c9f670|4d9d6c0f1257822808dbb172cd87c879|8ef14fb7db2d8439e5f69504315ae8dc     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

#### 1.8. Сделаем запрос к таблице к таблице system.part_log

```sql
SELECT * FROM system.part_log ORDER BY event_time DESC;
```

Результат
```text
hostname    |query_id|event_type|merge_reason|merge_algorithm|event_date|event_time         |event_time_microseconds      |duration_ms|database|table              |table_uuid                          |part_name      |partition_id|partition|part_type|disk_name|path_on_disk|rows|size_in_bytes|merged_from|bytes_uncompressed|read_rows|read_bytes|peak_memory_usage|error|exception|ProfileEvents|
------------+--------+----------+------------+---------------+----------+-------------------+-----------------------------+-----------+--------+-------------------+------------------------------------+---------------+------------+---------+---------+---------+------------+----+-------------+-----------+------------------+---------+----------+-----------------+-----+---------+-------------+
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3653_3653_0|all         |tuple()  |Compact  |         |            |  10|         2466|[]         |              1320|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3652_3652_0|all         |tuple()  |Compact  |         |            |  10|         2497|[]         |              1354|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3651_3658_1|all         |tuple()  |Compact  |         |            |  80|         5082|[]         |              6955|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3651_3651_0|all         |tuple()  |Compact  |         |            |  10|         2480|[]         |              1327|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3650_3650_0|all         |tuple()  |Compact  |         |            |  10|         2477|[]         |              1362|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3649_3649_0|all         |tuple()  |Compact  |         |            |  10|         2530|[]         |              1377|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3648_3648_0|all         |tuple()  |Compact  |         |            |  10|         2496|[]         |              1354|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3647_3647_0|all         |tuple()  |Compact  |         |            |  10|         2475|[]         |              1357|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3646_3646_0|all         |tuple()  |Compact  |         |            |  10|         2469|[]         |              1345|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3645_3693_2|all         |tuple()  |Compact  |         |            | 490|        17581|[]         |             39847|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3645_3650_1|all         |tuple()  |Compact  |         |            |  60|         4380|[]         |              5407|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3645_3645_0|all         |tuple()  |Compact  |         |            |  10|         2454|[]         |              1342|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3644_3644_0|all         |tuple()  |Compact  |         |            |  10|         2490|[]         |              1362|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3643_3643_0|all         |tuple()  |Compact  |         |            |  10|         2479|[]         |              1318|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3642_3642_0|all         |tuple()  |Compact  |         |            |  10|         2486|[]         |              1352|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3641_3641_0|all         |tuple()  |Compact  |         |            |  10|         2484|[]         |              1321|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3640_3640_0|all         |tuple()  |Compact  |         |            |  10|         2483|[]         |              1358|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3639_3639_0|all         |tuple()  |Compact  |         |            |  10|         2508|[]         |              1338|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3638_3638_0|all         |tuple()  |Compact  |         |            |  10|         2475|[]         |              1367|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3637_3637_0|all         |tuple()  |Compact  |         |            |  10|         2523|[]         |              1346|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3636_3636_0|all         |tuple()  |Compact  |         |            |  10|         2498|[]         |              1354|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3635_3635_0|all         |tuple()  |Compact  |         |            |  10|         2489|[]         |              1311|        0|         0|                0|    0|         |{}           |
19294dc11be6|        |RemovePart|NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:30:16|2025-11-26 11:30:16.209203000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_3634_3641_1|all         |tuple()  |Compact  |         |            |  80|         5108|[]         |              6924|        0|         0|                0|    0|         |{}           |
...
```

Видим большое количество записей. В этом журнале можно посмотреть информацию о том, какие парты создавались, какие были объединены/удалены. По прошествии 10 минут видим большое количество удаленных партов.

[//]: # (#### 1.9. Смотрим сколько понадобилось соединений частей)

[//]: # ()
[//]: # (```sql)

[//]: # (SELECT * FROM system.part_log WHERE table = 'mart_student_lesson' AND table_uuid = '1f98a761-620a-45ac-8337-46e28f167453' AND event_type = 'MergeParts' ORDER BY event_time DESC;)

[//]: # (```)

#### 1.9. Форсируем объединение частей в одну

```sql
OPTIMIZE TABLE learn_db.mart_student_lesson FINAL;

SELECT * FROM system.parts where table = 'mart_student_lesson';
```

Результат
```text
partition|name             |uuid                                |part_type|active|marks|rows  |bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table              |engine   |disk_name|path                                                                                 |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock    |removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+-----------------+------------------------------------+---------+------+-----+------+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+-------------------+---------+---------+-------------------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+--------------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
tuple()  |all_1_9971_223   |00000000-0000-0000-0000-000000000000|Compact  |     0|   14| 99710|      2246461|              2244340|                7974330|             148|        733|                                 0|                                   0|                            0|2025-11-26 11:21:02|2025-11-26 11:46:17|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|            9971|  223|           1|                         56|                                  184|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_9971_223/   |69c65ef043fd645155f582540d9671a7|09ed6d2615eca9e7ee9ed62f7a0acf85|8ce2fb6a12c9731407d16378697efa06     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_1_10002_224  |00000000-0000-0000-0000-000000000000|Compact  |     1|   14|100020|      2253473|              2251334|                7998996|             146|        736|                                 0|                                   0|                            0|2025-11-26 11:46:17|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|           10002|  224|           1|                         56|                                  184|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_1_10002_224/  |57e6b18a17feb1a374e64c8941adcf7d|33277037c959fac5010f89a7f7527451|2168dca0fa3ab6ae9efea4344af3dc3b     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|                   0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_9972_10000_3 |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|   290|        11710|                10335|                  23063|              52|        126|                                 0|                                   0|                            0|2025-11-26 11:21:02|2025-11-26 11:46:17|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |            9972|           10000|    3|        9972|                         18|                                  401|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_9972_10000_3/ |7e1dd519454704440899c8ecd0ee49c7|99bf5c9bc5682351c3894341bb85044e|a32c71607cd91cbf7b793d4ffe826e85     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_10001_10001_0|00000000-0000-0000-0000-000000000000|Compact  |     0|    2|    10|         2467|                 1129|                    778|              52|        109|                                 0|                                   0|                            0|2025-11-26 11:21:02|2025-11-26 11:46:17|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |           10001|           10001|    0|       10001|                         18|                                  401|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_10001_10001_0/|4b6348ba28a43ebc9fb928822db6c51d|76c5dc333b4c2d41d8caf0c5081e5d4b|3385d876da7c20465980329d24985a07     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_10002_10002_0|00000000-0000-0000-0000-000000000000|Compact  |     0|    2|    10|         2503|                 1165|                    825|              52|        109|                                 0|                                   0|                            0|2025-11-26 11:21:02|2025-11-26 11:46:17|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |           10002|           10002|    0|       10002|                         18|                                  401|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_10002_10002_0/|463d9bff02d69ab948550329a8c9f670|4d9d6c0f1257822808dbb172cd87c879|8ef14fb7db2d8439e5f69504315ae8dc     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

Видим один активный парт со всеми вставленными строками (100020). Операция OPTIMIZE TABLE <name>> FINAL достаточно ресурсоемкая и в документации не рекомендуют ее использовать вручную. Плюс эта операция игнорирует оптимальный размер партов, к которому стремится ClickHouse. И она постарается все объединить все в один парт. Фоновые процесс объединения не создает такую сильную нагрузку и одновременные запросы пользователей могут выполняться вполне комфортно.

#### 1.10. Вставляем разом 1 000 000 строк

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
	numbers(1000000);
```

Execute time = 0.8 s

#### 1.11. Смотрим, что произошло с частями

```sql
SELECT * FROM system.part_log ORDER BY event_time desc;
```

Результат
```text
hostname    |query_id                            |event_type     |merge_reason|merge_algorithm|event_date|event_time         |event_time_microseconds      |duration_ms|database|table              |table_uuid                          |part_name        |partition_id|partition|part_type|disk_name|path_on_disk                                                                         |rows   |size_in_bytes|merged_from                                                                  |bytes_uncompressed|read_rows|read_bytes|peak_memory_usage|error|exception|ProfileEvents                                                                                                                                                                                                                                                  |
------------+------------------------------------+---------------+------------+---------------+----------+-------------------+-----------------------------+-----------+--------+-------------------+------------------------------------+-----------------+------------+---------+---------+---------+-------------------------------------------------------------------------------------+-------+-------------+-----------------------------------------------------------------------------+------------------+---------+----------+-----------------+-----+---------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
19294dc11be6|                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:54:18|2025-11-26 11:54:18.449399000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_1_9971_223   |all         |tuple()  |Compact  |         |                                                                                     |  99710|      2246461|[]                                                                           |           7978152|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
19294dc11be6|                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:54:18|2025-11-26 11:54:18.449399000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_9972_10000_3 |all         |tuple()  |Compact  |         |                                                                                     |    290|        11710|[]                                                                           |             23609|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
19294dc11be6|                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:54:18|2025-11-26 11:54:18.449399000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_10001_10001_0|all         |tuple()  |Compact  |         |                                                                                     |     10|         2467|[]                                                                           |              1324|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
19294dc11be6|                                    |RemovePart     |NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:54:18|2025-11-26 11:54:18.449399000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_10002_10002_0|all         |tuple()  |Compact  |         |                                                                                     |     10|         2503|[]                                                                           |              1371|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
19294dc11be6|abcc1bef-4360-4062-88d8-4fe15bc37c2b|NewPart        |NotAMerge   |Undecided      |2025-11-26|2025-11-26 11:54:14|2025-11-26 11:54:14.525535000|        510|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_10003_10003_0|all         |tuple()  |Wide     |default  |/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_10003_10003_0/|1000000|     21446075|[]                                                                           |          80003733|        0|         0|                0|    0|         |{FileOpen=39, WriteBufferFromFileDescriptorWrite=54, WriteBufferFromFileDescriptorWriteBytes=21448188, IOBufferAllocs=141, IOBufferAllocBytes=36020915, DiskWriteElapsedMicroseconds=15620, MergeTreeDataWriterRows=1000000, MergeTreeDataWriterUncompressedByt|
19294dc11be6|                                    |MergePartsStart|RegularMerge|Undecided      |2025-11-26|2025-11-26 11:46:17|2025-11-26 11:46:17.674111000|          0|learn_db|mart_student_lesson|ac27b509-7d61-40ae-883b-bcc5abd3eeac|all_1_10002_224  |all         |tuple()  |Unknown  |         |                                                                                     |      0|            0|['all_1_9971_223','all_9972_10000_3','all_10001_10001_0','all_10002_10002_0']|                 0|        0|         0|                0|    0|         |{}                                                                                                                                                                                                                                                             |
```

Видим созданный парт с 1 млн. строк. Тип парта (part_type) - Wide. Посмотрим что расположено в указанном каталоге. 

Выполняем на вкладке Exec команды
```bash
cd /var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_10003_10003_0/
ls - lh
```

Результат
```text
root@19294dc11be6:/var/lib/clickhouse/store/ac2/ac27b509-7d61-40ae-883b-bcc5abd3eeac/all_10003_10003_0# ls -lh
total 21M
-rw-r----- 1 clickhouse clickhouse 1.7K Nov 26 08:54 checksums.txt
-rw-r----- 1 clickhouse clickhouse 828K Nov 26 08:54 class_id.bin
-rw-r----- 1 clickhouse clickhouse  301 Nov 26 08:54 class_id.cmrk2
-rw-r----- 1 clickhouse clickhouse  376 Nov 26 08:54 columns.txt
-rw-r----- 1 clickhouse clickhouse    7 Nov 26 08:54 count.txt
-rw-r----- 1 clickhouse clickhouse   10 Nov 26 08:54 default_compression_codec.txt
-rw-r----- 1 clickhouse clickhouse  18K Nov 26 08:54 educational_organization_id.bin
-rw-r----- 1 clickhouse clickhouse  282 Nov 26 08:54 educational_organization_id.cmrk2
-rw-r----- 1 clickhouse clickhouse  20K Nov 26 08:54 lesson_date.bin
-rw-r----- 1 clickhouse clickhouse  375 Nov 26 08:54 lesson_date.cmrk2
-rw-r----- 1 clickhouse clickhouse  36K Nov 26 08:54 lesson_month_digits.bin
-rw-r----- 1 clickhouse clickhouse  323 Nov 26 08:54 lesson_month_digits.cmrk2
-rw-r----- 1 clickhouse clickhouse  53K Nov 26 08:54 lesson_month_text.bin
-rw-r----- 1 clickhouse clickhouse  322 Nov 26 08:54 lesson_month_text.cmrk2
-rw-r----- 1 clickhouse clickhouse 8.8K Nov 26 08:54 lesson_year.bin
-rw-r----- 1 clickhouse clickhouse  284 Nov 26 08:54 lesson_year.cmrk2
-rw-r----- 1 clickhouse clickhouse 1.2M Nov 26 08:54 load_date.bin
-rw-r----- 1 clickhouse clickhouse  308 Nov 26 08:54 load_date.cmrk2
-rw-r----- 1 clickhouse clickhouse 656K Nov 26 08:54 mark.bin
-rw-r----- 1 clickhouse clickhouse  252 Nov 26 08:54 mark.cmrk2
-rw-r----- 1 clickhouse clickhouse    1 Nov 26 08:54 metadata_version.txt
-rw-r----- 1 clickhouse clickhouse  50K Nov 26 08:54 parallel_id.bin
-rw-r----- 1 clickhouse clickhouse  285 Nov 26 08:54 parallel_id.cmrk2
-rw-r----- 1 clickhouse clickhouse 6.1M Nov 26 08:54 person_id.bin
-rw-r----- 1 clickhouse clickhouse  513 Nov 26 08:54 person_id.cmrk2
-rw-r----- 1 clickhouse clickhouse 1.8M Nov 26 08:54 person_id_int.bin
-rw-r----- 1 clickhouse clickhouse  416 Nov 26 08:54 person_id_int.cmrk2
-rw-r----- 1 clickhouse clickhouse  850 Nov 26 08:54 primary.cidx
-rw-r----- 1 clickhouse clickhouse 1.3K Nov 26 08:54 serialization.json
-rw-r----- 1 clickhouse clickhouse 3.9M Nov 26 08:54 student_profile_id.bin
-rw-r----- 1 clickhouse clickhouse  439 Nov 26 08:54 student_profile_id.cmrk2
-rw-r----- 1 clickhouse clickhouse 780K Nov 26 08:54 subject_id.bin
-rw-r----- 1 clickhouse clickhouse  300 Nov 26 08:54 subject_id.cmrk2
-rw-r----- 1 clickhouse clickhouse 2.7M Nov 26 08:54 subject_name.bin
-rw-r----- 1 clickhouse clickhouse  397 Nov 26 08:54 subject_name.cmrk2
-rw-r----- 1 clickhouse clickhouse 1.3M Nov 26 08:54 t.bin
-rw-r----- 1 clickhouse clickhouse  311 Nov 26 08:54 t.cmrk2
-rw-r----- 1 clickhouse clickhouse 1.4M Nov 26 08:54 teacher_id.bin
-rw-r----- 1 clickhouse clickhouse  403 Nov 26 08:54 teacher_id.cmrk2
```

Теперь у нас на каждую колонку 2 файла: с данными (bin) и со смещениями (cmrk2), которые позволяют быстро найти нужную гранулу.

## 2. Асинхронная вставка

Если со стороны клиента нельзя накапливать данные, то можно использовать механизм асинхронной вставки данных.

#### 2.1. Создаем таблицу для тестирования асинхронной вставки

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

#### 2.2. Включаем асинхронную вставку для сессии

```sql
SET async_insert = 1;
```

#### 2.3. Сохраняем в файл query.sql запрос вставки 1 строки в асинхронном режиме в таблицу learn_db.mart_student_lesson

```bash
echo "INSERT INTO learn_db.async_test SETTINGS async_insert=1 VALUES (1, now(), 'async data');" > query.sql
```

И выполним эту команду в контейнере на вкладке Exec

#### 2.4. Запускаем 1000 000 запросов на вставку по 1 строк в 10 параллельных потоков. То есть всего вставляем 1000 000 строк.

```bash
clickhouse-benchmark -i 1000000 -c 10 --query "`cat query.sql`"
```

#### 2.5. Наблюдаем за асинхронной вставкой

```sql
SELECT * FROM system.asynchronous_inserts;
```

Результат
```text
query                                                                                        |database|table     |format|first_update                 |total_bytes|entries.query_id                                                                                                                                                                                                                                               |entries.bytes                  |
---------------------------------------------------------------------------------------------+--------+----------+------+-----------------------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------+
INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|2025-11-26 12:19:31.367558000|        250|['e25f5e96-3c91-4c8b-90ed-746a95e18dd7','3bc90ad0-2965-4e0e-a92e-79e8533b983a','8b3fb1b0-52a4-42f0-9540-b6785d3aa549','cce048ea-c6eb-4473-81f5-92a722347c12','be58fd23-58d2-4c5f-ba8d-da64fa14a4a3','66612eb2-b6c9-4965-89a5-90499a7afa7d','6c58e1bb-224a-41ec-|[25,25,25,25,25,25,25,25,25,25]|
```

В этой таблице отображается текущая асинхронная вставка. Создан некий буфер, он ожидает появления данных и дальше он их объединит.

```sql
SELECT * FROM system.asynchronous_insert_log;
```

Результат
```text
hostname    |event_date|event_time         |event_time_microseconds      |query                                                                                        |database|table     |format|query_id                            |bytes|rows|exception|status|data_kind|flush_time         |flush_time_microseconds      |flush_query_id                      |timeout_milliseconds|
------------+----------+-------------------+-----------------------------+---------------------------------------------------------------------------------------------+--------+----------+------+------------------------------------+-----+----+---------+------+---------+-------------------+-----------------------------+------------------------------------+--------------------+
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.390542000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|2eb1ef15-7642-4b1b-a37f-f67685b5009f|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.562777000|485c638e-c37f-48f4-9f10-d7d4f3e0ba49|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.390678000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|5d6b347a-7996-451e-b41b-4c3222d33074|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.562777000|485c638e-c37f-48f4-9f10-d7d4f3e0ba49|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.390627000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|39007ca0-6ae5-4423-b748-9fd3c7671604|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.562777000|485c638e-c37f-48f4-9f10-d7d4f3e0ba49|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.390803000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|f5b2577d-67de-4f40-a706-7b0177864b91|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.562777000|485c638e-c37f-48f4-9f10-d7d4f3e0ba49|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.390944000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|57aef424-30ba-4926-914a-7ddbf808c5dc|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.562777000|485c638e-c37f-48f4-9f10-d7d4f3e0ba49|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.390990000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|54717cff-9879-452c-a0c7-9741d1879c46|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.562777000|485c638e-c37f-48f4-9f10-d7d4f3e0ba49|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.390627000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|a4e43491-6e75-47b8-a25b-0c7f0a4e6976|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.562777000|485c638e-c37f-48f4-9f10-d7d4f3e0ba49|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.390882000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|6eb9cac6-121e-4abe-a041-2175f26c6d9d|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.562777000|485c638e-c37f-48f4-9f10-d7d4f3e0ba49|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.390652000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|b01c2764-3811-4118-9e2b-e8f307acf44e|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.562777000|485c638e-c37f-48f4-9f10-d7d4f3e0ba49|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.390599000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|02b03324-12f2-474e-85a8-3453c6d7b27a|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.562777000|485c638e-c37f-48f4-9f10-d7d4f3e0ba49|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.563830000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|e4c3c0f2-f763-413a-ae0d-a8e5d559c10b|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.736529000|420dd655-23da-4722-b5f7-ef4dedb4316f|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.563848000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|d6c06525-152f-4e38-94c4-d88eb23d6288|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.736529000|420dd655-23da-4722-b5f7-ef4dedb4316f|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.563971000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|91fbb704-13d4-444b-a83f-2899a6ba210d|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.736529000|420dd655-23da-4722-b5f7-ef4dedb4316f|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.563873000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|1b5c74fd-7206-4110-92dd-35b41d538c4a|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.736529000|420dd655-23da-4722-b5f7-ef4dedb4316f|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.563881000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|2abb300e-0d49-42a9-b560-509ad06a1e3d|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.736529000|420dd655-23da-4722-b5f7-ef4dedb4316f|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.563898000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|1db37a7c-0ced-4a18-b8b6-57b2856194ce|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.736529000|420dd655-23da-4722-b5f7-ef4dedb4316f|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.564216000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|ad3bb7bb-1e28-4ed3-9fca-359009a89513|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.736529000|420dd655-23da-4722-b5f7-ef4dedb4316f|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.563892000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|4121427c-ef35-46cf-92e6-79ccc281d82e|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.736529000|420dd655-23da-4722-b5f7-ef4dedb4316f|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.564363000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|1fdc9ed0-7f44-48ae-b07d-3fd16713779f|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.736529000|420dd655-23da-4722-b5f7-ef4dedb4316f|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.563861000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|60b1381e-dc28-4d41-9275-06b54fa27b44|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.736529000|420dd655-23da-4722-b5f7-ef4dedb4316f|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.737703000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|e697a362-2564-4d7e-94d5-9aad6cf494c5|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.910501000|18939a34-8a98-482b-9d24-b52a29ad2469|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.737703000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|56496764-bd29-48f2-ad92-8effd4432964|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.910501000|18939a34-8a98-482b-9d24-b52a29ad2469|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.737844000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|a2c59d30-ab2e-42bf-b8b0-d23e2341f1b3|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.910501000|18939a34-8a98-482b-9d24-b52a29ad2469|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.737701000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|bf38a456-978b-4065-a4c3-3bfde86f8c44|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.910501000|18939a34-8a98-482b-9d24-b52a29ad2469|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.737716000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|4195936b-d20b-446d-8b67-fd28f8583648|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.910501000|18939a34-8a98-482b-9d24-b52a29ad2469|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.737724000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|318c3155-b4a8-401e-97d1-737937f35cce|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.910501000|18939a34-8a98-482b-9d24-b52a29ad2469|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.737740000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|cc2cf417-5019-4bc7-9265-d9b272c7325d|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.910501000|18939a34-8a98-482b-9d24-b52a29ad2469|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.737771000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|8ab29119-2578-422f-bd4d-48e3c19c062d|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.910501000|18939a34-8a98-482b-9d24-b52a29ad2469|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.737701000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|66806bc1-eaa3-4baa-9384-db537e3d319f|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.910501000|18939a34-8a98-482b-9d24-b52a29ad2469|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.737733000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|df1345fb-78dd-474a-8e99-02dfa41776bb|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:55|2025-11-26 12:21:55.910501000|18939a34-8a98-482b-9d24-b52a29ad2469|                 166|
19294dc11be6|2025-11-26|2025-11-26 12:21:55|2025-11-26 12:21:55.911634000|INSERT INTO learn_db.async_test (id, timestamp, data) SETTINGS async_insert = 1 FORMAT Values|learn_db|async_test|Values|8ddce9e0-73f1-4fb4-b66c-66b6c51b3eb6|   25|   1|         |Ok    |Parsed   |2025-11-26 12:21:56|2025-11-26 12:21:56.086180000|f3ee727a-430e-4b56-ae81-861e60b14fed|                 166|
...
```

Здесь мы видим все факты асинхронной вставки.

```sql
SELECT * FROM system.parts where table = 'async_test' and level = 0;
```

Результат
```text
partition|name         |uuid                                |part_type|active|marks|rows|bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table     |engine   |disk_name|path                                                                             |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock    |removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                       |
---------+-------------+------------------------------------+---------+------+-----+----+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+----------+---------+---------+---------------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+--------------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+------------------------------------+
tuple()  |all_1_1_0    |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:18|2025-11-26 12:19:19|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|               1|    0|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_1_1_0/    |d84cb5c8262ef735da99bdb5d89074a5|1cb28792615afcb67ed67d18f7e7b4ef|ac85f1b846a0117260f8f446dbff6f6c     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_2_2_0    |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:18|2025-11-26 12:19:19|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               2|               2|    0|           2|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_2_2_0/    |d84cb5c8262ef735da99bdb5d89074a5|1cb28792615afcb67ed67d18f7e7b4ef|ac85f1b846a0117260f8f446dbff6f6c     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_3_3_0    |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:18|2025-11-26 12:19:19|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               3|               3|    0|           3|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_3_3_0/    |d84cb5c8262ef735da99bdb5d89074a5|1cb28792615afcb67ed67d18f7e7b4ef|ac85f1b846a0117260f8f446dbff6f6c     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_4_4_0    |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:18|2025-11-26 12:19:19|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               4|               4|    0|           4|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_4_4_0/    |d84cb5c8262ef735da99bdb5d89074a5|1cb28792615afcb67ed67d18f7e7b4ef|ac85f1b846a0117260f8f446dbff6f6c     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_5_5_0    |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|   1|          427|                   97|                     19|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:18|2025-11-26 12:19:19|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               5|               5|    0|           5|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_5_5_0/    |c863fb6cb6857e9b993dd11a6bbf766b|25efaa379742c46456cdff80a645250d|14f50e4f5c9ac918e7be1dc36bcb47cf     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_6_6_0    |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:19|2025-11-26 12:19:19|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               6|               6|    0|           6|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_6_6_0/    |e5c67600777917002538de7c32cfaf21|1cb28792615afcb67ed67d18f7e7b4ef|bbb18edfe9aa3be4667340105ab9259d     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_7_7_0    |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:19|2025-11-26 12:19:19|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               7|               7|    0|           7|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_7_7_0/    |e5c67600777917002538de7c32cfaf21|1cb28792615afcb67ed67d18f7e7b4ef|bbb18edfe9aa3be4667340105ab9259d     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_8_8_0    |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:19|2025-11-26 12:19:19|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               8|               8|    0|           8|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_8_8_0/    |e5c67600777917002538de7c32cfaf21|1cb28792615afcb67ed67d18f7e7b4ef|bbb18edfe9aa3be4667340105ab9259d     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_9_9_0    |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:19|2025-11-26 12:19:19|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               9|               9|    0|           9|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_9_9_0/    |e5c67600777917002538de7c32cfaf21|1cb28792615afcb67ed67d18f7e7b4ef|bbb18edfe9aa3be4667340105ab9259d     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_10_10_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:19|2025-11-26 12:19:19|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              10|              10|    0|          10|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_10_10_0/  |e5c67600777917002538de7c32cfaf21|1cb28792615afcb67ed67d18f7e7b4ef|bbb18edfe9aa3be4667340105ab9259d     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_11_11_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:19|2025-11-26 12:19:19|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              11|              11|    0|          11|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_11_11_0/  |e5c67600777917002538de7c32cfaf21|1cb28792615afcb67ed67d18f7e7b4ef|bbb18edfe9aa3be4667340105ab9259d     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_12_12_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:20|2025-11-26 12:19:20|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              12|              12|    0|          12|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_12_12_0/  |414f399d5eeb82ac155b31c60b03286e|1cb28792615afcb67ed67d18f7e7b4ef|83a953b8d47c3e9461134f9628afb4b3     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_13_13_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:20|2025-11-26 12:19:20|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              13|              13|    0|          13|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_13_13_0/  |414f399d5eeb82ac155b31c60b03286e|1cb28792615afcb67ed67d18f7e7b4ef|83a953b8d47c3e9461134f9628afb4b3     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_14_14_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:20|2025-11-26 12:19:20|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              14|              14|    0|          14|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_14_14_0/  |414f399d5eeb82ac155b31c60b03286e|1cb28792615afcb67ed67d18f7e7b4ef|83a953b8d47c3e9461134f9628afb4b3     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_15_15_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:20|2025-11-26 12:19:20|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              15|              15|    0|          15|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_15_15_0/  |414f399d5eeb82ac155b31c60b03286e|1cb28792615afcb67ed67d18f7e7b4ef|83a953b8d47c3e9461134f9628afb4b3     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_16_16_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:20|2025-11-26 12:19:20|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              16|              16|    0|          16|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_16_16_0/  |414f399d5eeb82ac155b31c60b03286e|1cb28792615afcb67ed67d18f7e7b4ef|83a953b8d47c3e9461134f9628afb4b3     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_17_17_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:20|2025-11-26 12:19:21|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              17|              17|    0|          17|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_17_17_0/  |414f399d5eeb82ac155b31c60b03286e|1cb28792615afcb67ed67d18f7e7b4ef|83a953b8d47c3e9461134f9628afb4b3     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_18_18_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:21|2025-11-26 12:19:21|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              18|              18|    0|          18|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_18_18_0/  |289d52ed963be86d6b31ca2c43548c8e|1cb28792615afcb67ed67d18f7e7b4ef|eca2c3381b55a68e9eb4da12c2a7c555     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_19_19_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:21|2025-11-26 12:19:21|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              19|              19|    0|          19|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_19_19_0/  |289d52ed963be86d6b31ca2c43548c8e|1cb28792615afcb67ed67d18f7e7b4ef|eca2c3381b55a68e9eb4da12c2a7c555     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_20_20_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:21|2025-11-26 12:19:21|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              20|              20|    0|          20|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_20_20_0/  |289d52ed963be86d6b31ca2c43548c8e|1cb28792615afcb67ed67d18f7e7b4ef|eca2c3381b55a68e9eb4da12c2a7c555     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_21_21_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:21|2025-11-26 12:19:21|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              21|              21|    0|          21|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_21_21_0/  |289d52ed963be86d6b31ca2c43548c8e|1cb28792615afcb67ed67d18f7e7b4ef|eca2c3381b55a68e9eb4da12c2a7c555     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_22_22_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:21|2025-11-26 12:19:22|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              22|              22|    0|          22|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_22_22_0/  |289d52ed963be86d6b31ca2c43548c8e|1cb28792615afcb67ed67d18f7e7b4ef|eca2c3381b55a68e9eb4da12c2a7c555     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_23_23_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:22|2025-11-26 12:19:22|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              23|              23|    0|          23|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_23_23_0/  |8fac73555cde0c2d091d50b34796cb2e|1cb28792615afcb67ed67d18f7e7b4ef|29d2bca8c2fea04277b234c7802a13d4     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_24_24_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:22|2025-11-26 12:19:22|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              24|              24|    0|          24|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_24_24_0/  |8fac73555cde0c2d091d50b34796cb2e|1cb28792615afcb67ed67d18f7e7b4ef|29d2bca8c2fea04277b234c7802a13d4     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_25_25_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:22|2025-11-26 12:19:22|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              25|              25|    0|          25|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_25_25_0/  |8fac73555cde0c2d091d50b34796cb2e|1cb28792615afcb67ed67d18f7e7b4ef|29d2bca8c2fea04277b234c7802a13d4     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_26_26_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:22|2025-11-26 12:19:22|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              26|              26|    0|          26|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_26_26_0/  |8fac73555cde0c2d091d50b34796cb2e|1cb28792615afcb67ed67d18f7e7b4ef|29d2bca8c2fea04277b234c7802a13d4     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_27_27_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:22|2025-11-26 12:19:23|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              27|              27|    0|          27|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_27_27_0/  |8fac73555cde0c2d091d50b34796cb2e|1cb28792615afcb67ed67d18f7e7b4ef|29d2bca8c2fea04277b234c7802a13d4     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_28_28_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:22|2025-11-26 12:19:23|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              28|              28|    0|          28|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_28_28_0/  |8fac73555cde0c2d091d50b34796cb2e|1cb28792615afcb67ed67d18f7e7b4ef|29d2bca8c2fea04277b234c7802a13d4     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_29_29_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:23|2025-11-26 12:19:23|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              29|              29|    0|          29|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_29_29_0/  |ef71b191e17a75ca207617d3a1d9a6fc|1cb28792615afcb67ed67d18f7e7b4ef|d1df768cb7fa9da298012cbc01789ed1     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_30_30_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:23|2025-11-26 12:19:23|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              30|              30|    0|          30|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_30_30_0/  |ef71b191e17a75ca207617d3a1d9a6fc|1cb28792615afcb67ed67d18f7e7b4ef|d1df768cb7fa9da298012cbc01789ed1     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
tuple()  |all_31_31_0  |00000000-0000-0000-0000-000000000000|Compact  |     0|    2|  10|          458|                  124|                    190|              50|         62|                                 0|                                   0|                            0|2025-11-26 12:19:23|2025-11-26 12:19:23|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |              31|              31|    0|          31|                          0|                                    0|                               25|                                         25|        0|learn_db|async_test|MergeTree|default  |/var/lib/clickhouse/store/29d/29d18bc2-92b3-4a10-9f91-712a9fb1bae5/all_31_31_0/  |ef71b191e17a75ca207617d3a1d9a6fc|1cb28792615afcb67ed67d18f7e7b4ef|d1df768cb7fa9da298012cbc01789ed1     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      0|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|15317705874040209379|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      2025-11-26 12:23:48|Part hasn't reached removal time yet|
...
```

Видим что во многих партах по 10 строк. Но процесс асинронной вставки идет очень долго.
```text
Queries executed: 23295.

localhost:9000, queries: 61, QPS: 58.931, RPS: 0.000, MiB/s: 0.000, result RPS: 0.000, result MiB/s: 0.000.

0%              0.171 sec.
10%             0.171 sec.
20%             0.172 sec.
30%             0.172 sec.
40%             0.172 sec.
50%             0.172 sec.
60%             0.172 sec.
70%             0.172 sec.
80%             0.172 sec.
90%             0.173 sec.
95%             0.173 sec.
99%             0.173 sec.
99.9%           0.173 sec.
99.99%          0.173 sec.



Queries executed: 23358.

localhost:9000, queries: 64, QPS: 61.762, RPS: 0.000, MiB/s: 0.000, result RPS: 0.000, result MiB/s: 0.000.

0%              0.170 sec.
10%             0.171 sec.
20%             0.171 sec.
30%             0.171 sec.
40%             0.172 sec.
50%             0.172 sec.
60%             0.172 sec.
70%             0.172 sec.
80%             0.173 sec.
90%             0.176 sec.
95%             0.176 sec.
99%             0.176 sec.
99.9%           0.176 sec.
99.99%          0.176 sec.



Queries executed: 23417.

localhost:9000, queries: 59, QPS: 57.092, RPS: 0.000, MiB/s: 0.000, result RPS: 0.000, result MiB/s: 0.000.

0%              0.171 sec.
10%             0.171 sec.
20%             0.171 sec.
30%             0.171 sec.
40%             0.172 sec.
50%             0.172 sec.
60%             0.172 sec.
70%             0.172 sec.
80%             0.172 sec.
90%             0.173 sec.
95%             0.173 sec.
99%             0.173 sec.
99.9%           0.173 sec.
99.99%          0.173 sec.
```

После того как мы выполняем insert Clickhouse ждет нового появления данных, если приходят - он их объединяет. Завершения процесса вставки 1 млн. строк мы вряд ли дождемся. Эту операцию можно отменить. нажимает crtl+c.


#### 2.6. Пересоздадим таблицу learn_db.mart_student_lesson с партиционированием и с индексом пропуска данных

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
	PRIMARY KEY(lesson_date, person_id_int, mark),
	INDEX idx_lesson_month_digits lesson_month_digits TYPE minmax GRANULARITY 4
) ENGINE = MergeTree()
PARTITION BY lesson_year
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
FROM numbers(10);
```

Укажем индекс пропуска данных по полю `lesson_month_digits` (номер месяца в цифрах).

#### 2.7. Посмотрим на количество партов

```sql
SELECT * FROM system.parts where table = 'mart_student_lesson';
```

Результат
```text
partition|name      |uuid                                |part_type|active|marks|rows|bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table              |engine   |disk_name|path                                                                          |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock|removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+----------+------------------------------------+---------+------+-----+----+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+-------------------+---------+---------+------------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+----------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
2024     |2024_2_2_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|   5|         2226|                  805|                    414|              52|        110|                                43|                                  16|                           50|2025-11-26 12:41:29|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|2024        |               2|               2|    0|           2|                         18|                                  401|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/7ec/7ec7bb89-c4eb-46a2-9290-0fbb258e1603/2024_2_2_0/|2ef62f0bde6d6defff847f60f28ef2d8|e86d480b3c18299dfe2a5b501bf23a58|0b0925b905a1c4fd38661ac5bd26f380     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
2025     |2025_1_1_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|   5|         2288|                  867|                    399|              52|        110|                                43|                                  16|                           50|2025-11-26 12:41:29|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|2025        |               1|               1|    0|           1|                         18|                                  401|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/7ec/7ec7bb89-c4eb-46a2-9290-0fbb258e1603/2025_1_1_0/|7b85cb9eee9246246312708616d7c78b|480ed0d02c4c9d92db5ce660102048d7|7a31ab517149dfc2c96b15a1b4b45e1a     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

Видим 2 парта потому, что у нас используется партицирование по году.
Посмотрим внутрь контейнера на созданные файлы. Для этого выполним команды

```bash
cd /var/lib/clickhouse/store/7ec/7ec7bb89-c4eb-46a2-9290-0fbb258e1603/2024_2_2_0/
ls -lh
```

Результат
```text
root@19294dc11be6:/var/lib/clickhouse/store/7ec/7ec7bb89-c4eb-46a2-9290-0fbb258e1603/2024_2_2_0# ls -lh
total 52K
-rw-r----- 1 clickhouse clickhouse  444 Nov 26 09:41 checksums.txt
-rw-r----- 1 clickhouse clickhouse  376 Nov 26 09:41 columns.txt
-rw-r----- 1 clickhouse clickhouse    1 Nov 26 09:41 count.txt
-rw-r----- 1 clickhouse clickhouse  805 Nov 26 09:41 data.bin
-rw-r----- 1 clickhouse clickhouse  110 Nov 26 09:41 data.cmrk3
-rw-r----- 1 clickhouse clickhouse   10 Nov 26 09:41 default_compression_codec.txt
-rw-r----- 1 clickhouse clickhouse    1 Nov 26 09:41 metadata_version.txt
-rw-r----- 1 clickhouse clickhouse    4 Nov 26 09:41 minmax_lesson_year.idx
-rw-r----- 1 clickhouse clickhouse    2 Nov 26 09:41 partition.dat
-rw-r----- 1 clickhouse clickhouse   52 Nov 26 09:41 primary.cidx
-rw-r----- 1 clickhouse clickhouse 1.2K Nov 26 09:41 serialization.json
-rw-r----- 1 clickhouse clickhouse   50 Nov 26 09:41 skp_idx_idx_lesson_month_digits.cmrk3
-rw-r----- 1 clickhouse clickhouse   43 Nov 26 09:41 skp_idx_idx_lesson_month_digits.idx2
```

Появились 2 файла для индекса пропуска данных (skp_idx_idx_lesson_month_digits), добавился файл связанный с партициями (minmax_lesson_year).

## 3. Практика.

#### 3.1. Удалите и создайте заново таблицу learn_db.mart_student_lesson. Вставляя за один раз в нее по 1000 строк, определите, после какой вставки произойдет первое слияние частей данных и после какой вставки произойдет второе слияние.

#### 3.1.1 Удаляем таблицу learn_db.mart_student_lesson

```sql
DROP TABLE learn_db.mart_student_lesson;
```

#### 3.1.2. Создадим пустую таблицу learn_db.mart_student_lesson

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

#### 3.1.3. Вставку данных по 1000 строк при помощи скрипта query.sql

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

#### 3.1.4. Подготавливаем скрипт для отслеживания процесса вставки

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

#### 3.1.5. Делаем скрипт исполняемым `chmod +x monitor_merges.sh` и выполняем его `./monitor_merges.sh`

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

#### 3.1.6. Анализируем работу merge

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

#### 3.1.7. Анализ выполнения merge можно сделать при помощи следующего запроса

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

#### 3.1.8. Визуализация процесса

```text
Вставки 1-6:   [1][2][3][4][5][6] → merge → [1-6]
Вставки 7-11:  [1-6]+[7][8][9][10][11] → merge → [1-11]
Вставки 12-16: [1-11]+[12][13][14][15][16] → merge → [1-16]
Вставки 17-20: [1-16]+[17][18][19][20] (ожидается merge после 21-й)
```

#### 3.2. Повторим асинхронную вставку в таблицу `learn_db.async_test` и посмотрим на результат выполнения запроса

```sql
SELECT *
FROM system.asynchronous_insert_log;
```

#### 3.2.1. Создадим таблицу

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

#### 3.2.2. Отредактируем скрипт для вставки данных

```bash
#!/bin/bash

INSERT INTO learn_db.async_test SETTINGS async_insert=1 VALUES (1, now(), 'async data');
```

#### 3.2.3. Включаем асинхронную вставку для сессии

`SET async_insert = 1;`

#### 3.2.4. Запускаем 1000 000 запросов на вставку по 1 строк в 10 параллельных потоков. То есть всего вставляем 1000 000 строк.

```text
clickhouse-benchmark -i 1000000 -c 10 --query "`cat query.sql`"
```

#### 3.2.5. Форсируем объединение частей в одну

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

#### 3.2.6. Выполним следующий запрос
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

Рехультат:
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

#### 3.2.7. Выполним следующий запрос

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