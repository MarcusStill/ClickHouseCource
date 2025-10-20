# Движок таблиц Buffer Table

### 1. Пересоздаем таблицу learn_db.mart_student_lesson

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
) ENGINE = MergeTree();
```

### 2. Создаем таблицу с движком типа Buffer

```sql
CREATE TABLE learn_db.mart_student_lesson_buffer AS learn_db.mart_student_lesson ENGINE = Buffer(learn_db, mart_student_lesson, 1, 10, 100, 10000, 1000000, 10000000, 100000000)
```

### 3. Вставляем данные несколько раз в таблицу с движком Buffer

```
INSERT INTO learn_db.mart_student_lesson_buffer
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
FROM numbers(10);
```
### 4. Проверяем содержимое основной таблицы и буфферной

```sql
SELECT COUNT(*) FROM learn_db.mart_student_lesson_buffer;
```

Результат
```text
COUNT()|
-------+
     10|
```

```sql
SELECT COUNT(*) FROM learn_db.mart_student_lesson;
```

Результат
```text
COUNT()|
-------+
     10|
```

### 5. Смотрим, какие части данных сформировались

```sql
SELECT * FROM system.part_log where table='mart_student_lesson' and event_type = 1 ORDER BY event_time DESC;
```

Результат
```text
student_profile_id|person_id|person_id_int|educational_organization_id|parallel_id|class_id|lesson_date|lesson_month_digits|lesson_month_text|lesson_year|load_date |t  |teacher_id|subject_id|subject_name|mark|_part    |
------------------+---------+-------------+---------------------------+-----------+--------+-----------+-------------------+-----------------+-----------+----------+---+----------+----------+------------+----+---------+
           1017478|1017478  |      1017478|                          2|         13|     508| 2024-12-07|2024-12            |2024 December    |       2024|2024-12-07| 90|       469|        10|Информатика |    |all_1_1_0|
           9796897|9796897  |      9796897|                         26|        134|    4898| 2024-12-22|2024-12            |2024 December    |       2024|2024-12-23|114|      3764|        12|Информатика |2   |all_1_1_0|
           8004474|8004474  |      8004474|                         21|        109|    4002| 2025-02-11|2025-02            |2025 February    |       2025|2025-02-12|  4|      2986|         0|Информатика |    |all_1_1_0|
           3830119|3830119  |      3830119|                         10|         52|    1915| 2025-02-23|2025-02            |2025 February    |       2025|2025-02-25| 83|      1510|         9|Информатика |    |all_1_1_0|
           6923054|6923054  |      6923054|                         18|         94|    3461| 2025-02-26|2025-02            |2025 February    |       2025|2025-02-27| 44|      2623|         4|Физика      |    |all_1_1_0|
           2547073|2547073  |      2547073|                          6|         34|    1273| 2025-04-22|2025-04            |2025 April       |       2025|2025-04-23|  4|       953|         0|Информатика |    |all_1_1_0|
           6819042|6819042  |      6819042|                         18|         93|    3409| 2025-05-17|2025-05            |2025 May         |       2025|2025-05-18|109|      2649|        12|Информатика |    |all_1_1_0|
           4750010|4750010  |      4750010|                         13|         65|    2375| 2025-06-09|2025-06            |2025 June        |       2025|2025-06-11|132|      1901|        14|Информатика |5   |all_1_1_0|
           8586965|8586965  |      8586965|                         23|        117|    4293| 2025-06-19|2025-06            |2025 June        |       2025|2025-06-21|  8|      3207|         0|Информатика |    |all_1_1_0|
           8800373|8800373  |      8800373|                         24|        120|    4400| 2025-07-24|2025-07            |2025 July        |       2025|2025-07-25| 92|      3371|        10|Информатика |    |all_1_1_0|
```

```sql
SELECT * FROM system.part_log where table='mart_student_lesson' and event_type = 1 ORDER BY event_time DESC;
```

Результат
```text
Query id: d9f107b9-be9b-4a7f-be35-06126746c853

      ┌─hostname─────┬─query_id─────────────────────────────┬─event_type──────┬─merge_reason─┬─merge_algorithm─┬─event_date─┬──────────event_time─┬────event_time_microseconds─┬─duration_ms─┬─database─┬─table───────────────┬─table_uuid───────────────────────────┬─part_name────┬─partition_id─┬─partition─┬─part_type─┬─disk_name─┬─path_on_disk─────────────────────────────────────────────────────────────────────┬─────rows─┬─size_in_bytes─┬─merged_from────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┬─bytes_uncompressed─┬─read_rows─┬─read_bytes─┬─peak_memory_usage─┬─error─┬─exception─┬─ProfileEvents──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
   1. │ 19294dc11be6 │                                      │ NewPart         │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 07:29:48 │ 2025-10-20 07:29:48.763699 │           1 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_1_1_0    │ all          │ tuple()   │ Compact   │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_1_1_0/    │       10 │          2268 │ []                                                                                                                                         │               1356 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':9,'WriteBufferFromFileDescriptorWrite':9,'WriteBufferFromFileDescriptorWriteBytes':2928,'IOBufferAllocs':23,'IOBufferAllocBytes':4439465,'DiskWriteElapsedMicroseconds':35,'MergeTreeDataWriterRows':10,'MergeTreeDataWriterUncompressedBytes':1140,'MergeTreeDataWriterCompressedBytes':2268,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':3,'MergeTreeDataWriterMergingBlocksMicroseconds':1,'ContextLock':12,'PartsLockHoldMicroseconds':54,'LogTrace':4,'LoggerElapsedNanoseconds':61600} │
   2. │ 19294dc11be6 │                                      │ RemovePart      │ NotAMerge    │ Undecided       │ 2025-10-16 │ 2025-10-16 19:49:30 │ 2025-10-16 19:49:30.018664 │           0 │ learn_db │ mart_student_lesson │ b15612fb-cbf8-4b30-8b57-fd090ecb5870 │ all_1_6_1_29 │ all          │ tuple()   │ Wide      │           │                                                                                  │  6671718 │     212784801 │ []                                                                                                                                         │          545442661 │         0 │          0 │                 0 │     0 │           │ {}                                                                                                                                                                                                                                                         │
   3. │ 19294dc11be6 │                                      │ RemovePart      │ NotAMerge    │ Undecided       │ 2025-10-16 │ 2025-10-16 19:49:26 │ 2025-10-16 19:49:26.438130 │           0 │ learn_db │ mart_student_lesson │ b15612fb-cbf8-4b30-8b57-fd090ecb5870 │ all_9_9_0_29 │ all          │ tuple()   │ Wide      │           │                                                                                  │  1104376 │      35226871 │ []                                                                                                                                         │           90286426 │         0 │          0 │                 0 │     0 │           │ {}                                                                                                                                                                                                                                                         │
   4. │ 19294dc11be6 │                                      │ RemovePart      │ NotAMerge    │ Undecided       │ 2025-10-16 │ 2025-10-16 19:49:26 │ 2025-10-16 19:49:26.438130 │           0 │ learn_db │ mart_student_lesson │ b15612fb-cbf8-4b30-8b57-fd090ecb5870 │ all_8_8_0_29 │ all          │ tuple()   │ Wide      │           │                                                                                  │  1111953 │      35473453 │ []                                                                                                                                         │           90913335 │         0 │          0 │                 0 │     0 │           │ {}                                                                                                                                                                                                                                                         │
   5. │ 19294dc11be6 │                                      │ RemovePart      │ NotAMerge    │ Undecided       │ 2025-10-16 │ 2025-10-16 19:49:26 │ 2025-10-16 19:49:26.438130 │           0 │ learn_db │ mart_student_lesson │ b15612fb-cbf8-4b30-8b57-fd090ecb5870 │ all_7_7_0_29 │ all          │ tuple()   │ Wide      │           │                                                                                  │  1111953 │      35467929 │ []                                                                                                                                         │           90905502 │         0 │          0 │                 0 │     0 │           │ {}                                                                                                                                                                                                                                                         │
3069. │ 19294dc11be6 │ d67284a5-399b-4d10-99c2-018bb4a1b07d │ NewPart         │ NotAMerge    │ Undecided       │ 2025-09-09 │ 2025-09-09 13:15:58 │ 2025-09-09 13:15:58.629685 │         244 │ learn_db │ mart_student_lesson │ fcc5fdee-d5a8-49b9-a1f0-a78a0c27e00e │ all_1_1_0    │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/fcc/fcc5fdee-d5a8-49b9-a1f0-a78a0c27e00e/all_1_1_0/    │  1111953 │      42881158 │ []                                                                                                                                         │           90081915 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':40,'WriteBufferFromFileDescriptorWrite':72,'WriteBufferFromFileDescriptorWriteBytes':42883334,'IOBufferAllocs':145,'IOBufferAllocBytes':37077935,'DiskWriteElapsedMicroseconds':24680,'MergeTreeDataWriterRows':1111953,'MergeTreeDataWriterUncompressedBytes':125608515,'MergeTreeDataWriterCompressedBytes':42881158,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterMergingBlocksMicroseconds':1,'ContextLock':10,'ContextLockWaitMicroseconds':1,'PartsLockHoldMicroseconds':78,'QueryProfilerRuns':1,'LogTrace':4,'LoggerElapsedNanoseconds':123300} │
      └─hostname─────┴─query_id─────────────────────────────┴─event_type──────┴─merge_reason─┴─merge_algorithm─┴─event_date─┴──────────event_time─┴────event_time_microseconds─┴─duration_ms─┴─database─┴─table───────────────┴─table_uuid───────────────────────────┴─part_name────┴─partition_id─┴─partition─┴─part_type─┴─disk_name─┴─path_on_disk─────────────────────────────────────────────────────────────────────┴─────rows─┴─size_in_bytes─┴─merged_from────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┴─bytes_uncompressed─┴─read_rows─┴─read_bytes─┴─peak_memory_usage─┴─error─┴─exception─┴─ProfileEvents──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
Showed 1000 out of 3069 rows.

3069 rows in set. Elapsed: 0.010 sec. Processed 5.20 thousand rows, 2.08 MB (500.86 thousand rows/s., 200.03 MB/s.)
Peak memory usage: 1.58 MiB.
```

### 6. Запускаем 1000 вставок по 1000 строк в один поток в буфферную таблицу

```sql
clickhouse-benchmark --query "
INSERT INTO learn_db.mart_student_lesson_buffer
SELECT
    floor(randUniform(2, 10000000)) as student_profile_id,
    cast(student_profile_id as String) as person_id,
    cast(person_id as Int32) as  person_id_int,
    student_profile_id / 365000 as educational_organization_id,
    student_profile_id / 73000 as parallel_id,
    student_profile_id / 2000 as class_id,
    cast(now() - randUniform(2, 60*60*24*365) as date) as lesson_date,
    formatDateTime(lesson_date, '%Y-%m') as lesson_month_digits,
    formatDateTime(lesson_date, '%Y %M') AS lesson_month_text,
    toYear(lesson_date) as lesson_year, 
    lesson_date + rand() % 3,
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
FROM numbers(1000)
" --iterations=1000 
```

### 7. Проверяем содержимое основной таблицы и буфферной

```sql
SELECT COUNT(*) FROM learn_db.mart_student_lesson_buffer;
```

Результат
```text
COUNT()|
-------+
1000010|
```

```sql
SELECT COUNT(*) FROM learn_db.mart_student_lesson;
```

Результат
```text
COUNT()|
-------+
1000010|
```

### 8. Смотрим, какие части данных сформировались

```sql
SELECT * FROM system.part_log where table='mart_student_lesson' and event_type = 1 ORDER BY event_time DESC;
```

Результат
```text
Query id: f7611cfa-52d3-4a37-801b-cf2b69041fbd

         ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬───t─┬─teacher_id─┬─subject_id─┬─subject_name────────┬─mark─┬─_part─────┐
      1. │            1017478 │ 1017478   │       1017478 │                           2 │          13 │      508 │  2024-12-07 │ 2024-12             │ 2024 December     │        2024 │ 2024-12-07 │  90 │        469 │         10 │ Информатика         │ ᴺᵁᴸᴸ │ all_1_1_0 │
      2. │            9796897 │ 9796897   │       9796897 │                          26 │         134 │     4898 │  2024-12-22 │ 2024-12             │ 2024 December     │        2024 │ 2024-12-23 │ 114 │       3764 │         12 │ Информатика         │    2 │ all_1_1_0 │
      3. │            8004474 │ 8004474   │       8004474 │                          21 │         109 │     4002 │  2025-02-11 │ 2025-02             │ 2025 February     │        2025 │ 2025-02-12 │   4 │       2986 │          0 │ Информатика         │ ᴺᵁᴸᴸ │ all_1_1_0 │
      4. │            3830119 │ 3830119   │       3830119 │                          10 │          52 │     1915 │  2025-02-23 │ 2025-02             │ 2025 February     │        2025 │ 2025-02-25 │  83 │       1510 │          9 │ Информатика         │ ᴺᵁᴸᴸ │ all_1_1_0 │
      5. │            6923054 │ 6923054   │       6923054 │                          18 │          94 │     3461 │  2025-02-26 │ 2025-02             │ 2025 February     │        2025 │ 2025-02-27 │  44 │       2623 │          4 │ Физика              │ ᴺᵁᴸᴸ │ all_1_1_0 │
1000010. │            1195777 │ 1195777   │       1195777 │                           3 │          16 │      597 │  2025-10-20 │ 2025-10             │ 2025 October      │        2025 │ 2025-10-22 │  84 │        529 │          9 │ Информатика         │ ᴺᵁᴸᴸ │ all_3_3_0 │
         └─student_profile_id─┴─person_id─┴─person_id_int─┴─educational_organization_id─┴─parallel_id─┴─class_id─┴─lesson_date─┴─lesson_month_digits─┴─lesson_month_text─┴─lesson_year─┴──load_date─┴───t─┴─teacher_id─┴─subject_id─┴─subject_name────────┴─mark─┴─_part─────┘
Showed 1000 out of 1000010 rows.
```

```sql
SELECT * FROM system.part_log where table='mart_student_lesson' and event_type = 1 ORDER BY event_time DESC;
```

Результат
```text
Query id: 0400742a-7462-4e70-b4d8-12b8b732be7c

      ┌─hostname─────┬─query_id─────────────────────────────┬─event_type─┬─merge_reason─┬─merge_algorithm─┬─event_date─┬──────────event_time─┬────event_time_microseconds─┬─duration_ms─┬─database─┬─table───────────────┬─table_uuid───────────────────────────┬─part_name───┬─partition_id─┬─partition─┬─part_type─┬─disk_name─┬─path_on_disk────────────────────────────────────────────────────────────────────┬────rows─┬─size_in_bytes─┬─merged_from─┬─bytes_uncompressed─┬─read_rows─┬─read_bytes─┬─peak_memory_usage─┬─error─┬─exception─┬─ProfileEvents──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
   1. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:36:09 │ 2025-10-20 11:36:09.277961 │          79 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_3_3_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_3_3_0/   │  121000 │       3858326 │ []          │            9893306 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3860460,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':2628,'MergeTreeDataWriterRows':121000,'MergeTreeDataWriterUncompressedBytes':13758714,'MergeTreeDataWriterCompressedBytes':3858326,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':6748,'MergeTreeDataWriterMergingBlocksMicroseconds':7,'ContextLock':12,'PartsLockHoldMicroseconds':89,'LogTrace':4,'LoggerElapsedNanoseconds':201300} │
   2. │ 19294dc11be6 │ 3b77ba6e-07b3-4656-a20f-0a863052b130 │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:35:59 │ 2025-10-20 11:35:59.069118 │         743 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_2_2_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_2_2_0/   │  879000 │      28041097 │ []          │           71863277 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':63,'WriteBufferFromFileDescriptorWriteBytes':28043260,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':28615,'MergeTreeDataWriterRows':879000,'MergeTreeDataWriterUncompressedBytes':99946369,'MergeTreeDataWriterCompressedBytes':28041097,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':41420,'MergeTreeDataWriterMergingBlocksMicroseconds':5,'ContextLock':12,'PartsLockHoldMicroseconds':358,'PartsLockWaitMicroseconds':2,'QueryProfilerRuns':2,'LogTrace':4,'LoggerElapsedNanoseconds':301800} │
   3. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 07:29:48 │ 2025-10-20 07:29:48.763699 │           1 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_1_1_0   │ all          │ tuple()   │ Compact   │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_1_1_0/   │      10 │          2268 │ []          │               1356 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':9,'WriteBufferFromFileDescriptorWrite':9,'WriteBufferFromFileDescriptorWriteBytes':2928,'IOBufferAllocs':23,'IOBufferAllocBytes':4439465,'DiskWriteElapsedMicroseconds':35,'MergeTreeDataWriterRows':10,'MergeTreeDataWriterUncompressedBytes':1140,'MergeTreeDataWriterCompressedBytes':2268,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':3,'MergeTreeDataWriterMergingBlocksMicroseconds':1,'ContextLock':12,'PartsLockHoldMicroseconds':54,'LogTrace':4,'LoggerElapsedNanoseconds':61600} │
   4. │ 19294dc11be6 │ 38a1f210-e211-4c6c-b365-3224499a164f │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-16 │ 2025-10-16 19:04:32 │ 2025-10-16 19:04:32.653237 │         542 │ learn_db │ mart_student_lesson │ b15612fb-cbf8-4b30-8b57-fd090ecb5870 │ all_9_9_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/b15/b15612fb-cbf8-4b30-8b57-fd090ecb5870/all_9_9_0/   │ 1104376 │      35226871 │ []          │           90286426 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':68,'WriteBufferFromFileDescriptorWriteBytes':35229035,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':22645,'MergeTreeDataWriterRows':1104376,'MergeTreeDataWriterUncompressedBytes':125570426,'MergeTreeDataWriterCompressedBytes':35226871,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':24118,'MergeTreeDataWriterMergingBlocksMicroseconds':2,'ContextLock':12,'PartsLockHoldMicroseconds':51,'QueryProfilerRuns':2,'LogTrace':4,'LoggerElapsedNanoseconds':119400} │
   5. │ 19294dc11be6 │ 38a1f210-e211-4c6c-b365-3224499a164f │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-16 │ 2025-10-16 19:04:32 │ 2025-10-16 19:04:32.652938 │         583 │ learn_db │ mart_student_lesson │ b15612fb-cbf8-4b30-8b57-fd090ecb5870 │ all_8_8_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/b15/b15612fb-cbf8-4b30-8b57-fd090ecb5870/all_8_8_0/   │ 1111953 │      35473453 │ []          │           90913335 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':69,'WriteBufferFromFileDescriptorWriteBytes':35475617,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':23556,'MergeTreeDataWriterRows':1111953,'MergeTreeDataWriterUncompressedBytes':126439387,'MergeTreeDataWriterCompressedBytes':35473453,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':27771,'MergeTreeDataWriterMergingBlocksMicroseconds':2,'ContextLock':12,'PartsLockHoldMicroseconds':142,'QueryProfilerRuns':1,'LogTrace':4,'LoggerElapsedNanoseconds':108100} │
1011. │ 19294dc11be6 │ d67284a5-399b-4d10-99c2-018bb4a1b07d │ NewPart    │ NotAMerge    │ Undecided       │ 2025-09-09 │ 2025-09-09 13:15:58 │ 2025-09-09 13:15:58.629685 │         244 │ learn_db │ mart_student_lesson │ fcc5fdee-d5a8-49b9-a1f0-a78a0c27e00e │ all_1_1_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/fcc/fcc5fdee-d5a8-49b9-a1f0-a78a0c27e00e/all_1_1_0/   │ 1111953 │      42881158 │ []          │           90081915 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':40,'WriteBufferFromFileDescriptorWrite':72,'WriteBufferFromFileDescriptorWriteBytes':42883334,'IOBufferAllocs':145,'IOBufferAllocBytes':37077935,'DiskWriteElapsedMicroseconds':24680,'MergeTreeDataWriterRows':1111953,'MergeTreeDataWriterUncompressedBytes':125608515,'MergeTreeDataWriterCompressedBytes':42881158,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterMergingBlocksMicroseconds':1,'ContextLock':10,'ContextLockWaitMicroseconds':1,'PartsLockHoldMicroseconds':78,'QueryProfilerRuns':1,'LogTrace':4,'LoggerElapsedNanoseconds':123300} │
      └─hostname─────┴─query_id─────────────────────────────┴─event_type─┴─merge_reason─┴─merge_algorithm─┴─event_date─┴──────────event_time─┴────event_time_microseconds─┴─duration_ms─┴─database─┴─table───────────────┴─table_uuid───────────────────────────┴─part_name───┴─partition_id─┴─partition─┴─part_type─┴─disk_name─┴─path_on_disk────────────────────────────────────────────────────────────────────┴────rows─┴─size_in_bytes─┴─merged_from─┴─bytes_uncompressed─┴─read_rows─┴─read_bytes─┴─peak_memory_usage─┴─error─┴─exception─┴─ProfileEvents──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
Showed 1000 out of 1011 rows.

1011 rows in set. Elapsed: 0.011 sec. Processed 5.20 thousand rows, 2.08 MB (492.13 thousand rows/s., 196.69 MB/s.)
Peak memory usage: 753.80 KiB.
```

В последние 2 парта были вставлены 879000 и 121000 строк. Я сделал 1000 insert, но было создано всего 2 части данных. Но в этом случае clickhouse-benchmark сделал последовательные операции вставки.

### 8. Запускаем 1000 вставок по 1000 строк в один поток в буферную таблицу параллельно в 10 потоков

```sql
clickhouse-benchmark --query "
INSERT INTO learn_db.mart_student_lesson_buffer
SELECT
    floor(randUniform(2, 10000000)) as student_profile_id,
    cast(student_profile_id as String) as person_id,
    cast(person_id as Int32) as  person_id_int,
    student_profile_id / 365000 as educational_organization_id,
    student_profile_id / 73000 as parallel_id,
    student_profile_id / 2000 as class_id,
    cast(now() - randUniform(2, 60*60*24*365) as date) as lesson_date,
    formatDateTime(lesson_date, '%Y-%m') as lesson_month_digits,
    formatDateTime(lesson_date, '%Y %M') AS lesson_month_text,
    toYear(lesson_date) as lesson_year, 
    lesson_date + rand() % 3,
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
FROM numbers(1000)
" --iterations=1000 --concurrency=10
```

Время вставки последней порции данных

```text
Queries executed: 1000.

localhost:9000, queries: 1000, QPS: 340.451, RPS: 340450.693, MiB/s: 2.597, result RPS: 0.000, result MiB/s: 0.000.

0%              0.009 sec.
10%             0.013 sec.
20%             0.015 sec.
30%             0.016 sec.
40%             0.017 sec.
50%             0.018 sec.
60%             0.019 sec.
70%             0.021 sec.
80%             0.023 sec.
90%             0.028 sec.
95%             0.031 sec.
99%             0.043 sec.
99.9%           0.749 sec.
99.99%          0.753 sec.
```

### 9. Смотрим, какие части данных сформировались

```sql
SELECT * FROM system.part_log where table='mart_student_lesson' and event_type = 1 ORDER BY event_time DESC;
```

Результат
```text
Query id: f087b149-1ae2-4eba-9218-59351306ea77

      ┌─hostname─────┬─query_id─────────────────────────────┬─event_type─┬─merge_reason─┬─merge_algorithm─┬─event_date─┬──────────event_time─┬────event_time_microseconds─┬─duration_ms─┬─database─┬─table───────────────┬─table_uuid───────────────────────────┬─part_name───┬─partition_id─┬─partition─┬─part_type─┬─disk_name─┬─path_on_disk────────────────────────────────────────────────────────────────────┬────rows─┬─size_in_bytes─┬─merged_from─┬─bytes_uncompressed─┬─read_rows─┬─read_bytes─┬─peak_memory_usage─┬─error─┬─exception─┬─ProfileEvents──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
   1. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:48:44 │ 2025-10-20 11:48:44.329264 │          61 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_5_5_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_5_5_0/   │  121000 │       3858451 │ []          │            9893361 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3860585,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':2090,'MergeTreeDataWriterRows':121000,'MergeTreeDataWriterUncompressedBytes':13758769,'MergeTreeDataWriterCompressedBytes':3858451,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':4572,'MergeTreeDataWriterMergingBlocksMicroseconds':22,'ContextLock':12,'PartsLockHoldMicroseconds':115,'LogTrace':4,'LoggerElapsedNanoseconds':517700} │
   2. │ 19294dc11be6 │ 113cd6d5-93c5-41ad-a33a-24948aa7d08e │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:48:34 │ 2025-10-20 11:48:34.067887 │         728 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_4_4_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_4_4_0/   │  879000 │      28042616 │ []          │           71862972 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':63,'WriteBufferFromFileDescriptorWriteBytes':28044779,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':22905,'MergeTreeDataWriterRows':879000,'MergeTreeDataWriterUncompressedBytes':99946064,'MergeTreeDataWriterCompressedBytes':28042616,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':39390,'MergeTreeDataWriterMergingBlocksMicroseconds':5,'ContextLock':12,'ContextLockWaitMicroseconds':2,'PartsLockHoldMicroseconds':156,'QueryProfilerRuns':2,'LogTrace':4,'LoggerElapsedNanoseconds':171100} │
   3. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:36:09 │ 2025-10-20 11:36:09.277961 │          79 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_3_3_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_3_3_0/   │  121000 │       3858326 │ []          │            9893306 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3860460,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':2628,'MergeTreeDataWriterRows':121000,'MergeTreeDataWriterUncompressedBytes':13758714,'MergeTreeDataWriterCompressedBytes':3858326,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':6748,'MergeTreeDataWriterMergingBlocksMicroseconds':7,'ContextLock':12,'PartsLockHoldMicroseconds':89,'LogTrace':4,'LoggerElapsedNanoseconds':201300} │
   4. │ 19294dc11be6 │ 3b77ba6e-07b3-4656-a20f-0a863052b130 │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:35:59 │ 2025-10-20 11:35:59.069118 │         743 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_2_2_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_2_2_0/   │  879000 │      28041097 │ []          │           71863277 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':63,'WriteBufferFromFileDescriptorWriteBytes':28043260,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':28615,'MergeTreeDataWriterRows':879000,'MergeTreeDataWriterUncompressedBytes':99946369,'MergeTreeDataWriterCompressedBytes':28041097,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':41420,'MergeTreeDataWriterMergingBlocksMicroseconds':5,'ContextLock':12,'PartsLockHoldMicroseconds':358,'PartsLockWaitMicroseconds':2,'QueryProfilerRuns':2,'LogTrace':4,'LoggerElapsedNanoseconds':301800} │
   5. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 07:29:48 │ 2025-10-20 07:29:48.763699 │           1 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_1_1_0   │ all          │ tuple()   │ Compact   │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_1_1_0/   │      10 │          2268 │ []          │               1356 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':9,'WriteBufferFromFileDescriptorWrite':9,'WriteBufferFromFileDescriptorWriteBytes':2928,'IOBufferAllocs':23,'IOBufferAllocBytes':4439465,'DiskWriteElapsedMicroseconds':35,'MergeTreeDataWriterRows':10,'MergeTreeDataWriterUncompressedBytes':1140,'MergeTreeDataWriterCompressedBytes':2268,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':3,'MergeTreeDataWriterMergingBlocksMicroseconds':1,'ContextLock':12,'PartsLockHoldMicroseconds':54,'LogTrace':4,'LoggerElapsedNanoseconds':61600} │
1013. │ 19294dc11be6 │ d67284a5-399b-4d10-99c2-018bb4a1b07d │ NewPart    │ NotAMerge    │ Undecided       │ 2025-09-09 │ 2025-09-09 13:15:58 │ 2025-09-09 13:15:58.629685 │         244 │ learn_db │ mart_student_lesson │ fcc5fdee-d5a8-49b9-a1f0-a78a0c27e00e │ all_1_1_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/fcc/fcc5fdee-d5a8-49b9-a1f0-a78a0c27e00e/all_1_1_0/   │ 1111953 │      42881158 │ []          │           90081915 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':40,'WriteBufferFromFileDescriptorWrite':72,'WriteBufferFromFileDescriptorWriteBytes':42883334,'IOBufferAllocs':145,'IOBufferAllocBytes':37077935,'DiskWriteElapsedMicroseconds':24680,'MergeTreeDataWriterRows':1111953,'MergeTreeDataWriterUncompressedBytes':125608515,'MergeTreeDataWriterCompressedBytes':42881158,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterMergingBlocksMicroseconds':1,'ContextLock':10,'ContextLockWaitMicroseconds':1,'PartsLockHoldMicroseconds':78,'QueryProfilerRuns':1,'LogTrace':4,'LoggerElapsedNanoseconds':123300} │
      └─hostname─────┴─query_id─────────────────────────────┴─event_type─┴─merge_reason─┴─merge_algorithm─┴─event_date─┴──────────event_time─┴────event_time_microseconds─┴─duration_ms─┴─database─┴─table───────────────┴─table_uuid───────────────────────────┴─part_name───┴─partition_id─┴─partition─┴─part_type─┴─disk_name─┴─path_on_disk────────────────────────────────────────────────────────────────────┴────rows─┴─size_in_bytes─┴─merged_from─┴─bytes_uncompressed─┴─read_rows─┴─read_bytes─┴─peak_memory_usage─┴─error─┴─exception─┴─ProfileEvents──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
Showed 1000 out of 1013 rows.

1013 rows in set. Elapsed: 0.009 sec. Processed 5.20 thousand rows, 2.08 MB (610.73 thousand rows/s., 243.87 MB/s.)
Peak memory usage: 690.42 KiB.
```

Видим что в 2 парта была произведена вставка все того же количества строк: 879000 и 121000. 

### 10. Пересоздадим буферную таблицу, указав что нам нужно 10 слоев

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_buffer; 
CREATE TABLE learn_db.mart_student_lesson_buffer AS learn_db.mart_student_lesson ENGINE = Buffer(learn_db, mart_student_lesson, 10, 10, 100, 10000, 1000000, 10000000, 100000000)
```

### 11. Запускаем 1000 вставок по 1000 строк в один поток в буфферную таблицу параллельно в 10 потоков
```sql
clickhouse-benchmark --query "
INSERT INTO learn_db.mart_student_lesson_buffer
SELECT
    floor(randUniform(2, 10000000)) as student_profile_id,
    cast(student_profile_id as String) as person_id,
    cast(person_id as Int32) as  person_id_int,
    student_profile_id / 365000 as educational_organization_id,
    student_profile_id / 73000 as parallel_id,
    student_profile_id / 2000 as class_id,
    cast(now() - randUniform(2, 60*60*24*365) as date) as lesson_date,
    formatDateTime(lesson_date, '%Y-%m') as lesson_month_digits,
    formatDateTime(lesson_date, '%Y %M') AS lesson_month_text,
    toYear(lesson_date) as lesson_year, 
    lesson_date + rand() % 3,
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
FROM numbers(1000)
" --iterations=1000 --concurrency=10
```

Время вставки последней порции данных
```text
Queries executed: 1000.

localhost:9000, queries: 1000, QPS: 466.484, RPS: 466483.793, MiB/s: 3.559, result RPS: 0.000, result MiB/s: 0.000.

0%              0.009 sec.
10%             0.014 sec.
20%             0.015 sec.
30%             0.016 sec.
40%             0.017 sec.
50%             0.018 sec.
60%             0.018 sec.
70%             0.020 sec.
80%             0.021 sec.
90%             0.024 sec.
95%             0.026 sec.
99%             0.035 sec.
99.9%           0.038 sec.
99.99%          0.040 sec.
```

### 12. Смотрим, какие части данных сформировались

```sql
SELECT * FROM system.part_log where table='mart_student_lesson' and event_type = 1 ORDER BY event_time DESC;
```

Результат
```text
Query id: a25da82e-5390-4be8-908e-9cdc1d38aa21

      ┌─hostname─────┬─query_id─────────────────────────────┬─event_type─┬─merge_reason─┬─merge_algorithm─┬─event_date─┬──────────event_time─┬────event_time_microseconds─┬─duration_ms─┬─database─┬─table───────────────┬─table_uuid───────────────────────────┬─part_name───┬─partition_id─┬─partition─┬─part_type─┬─disk_name─┬─path_on_disk────────────────────────────────────────────────────────────────────┬────rows─┬─size_in_bytes─┬─merged_from─┬─bytes_uncompressed─┬─read_rows─┬─read_bytes─┬─peak_memory_usage─┬─error─┬─exception─┬─ProfileEvents──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
   1. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.944997 │         210 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_6_6_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_6_6_0/   │  100000 │       3195002 │ []          │            8175586 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197141,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':11857,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11369818,'MergeTreeDataWriterCompressedBytes':3195002,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9525,'MergeTreeDataWriterMergingBlocksMicroseconds':20,'ContextLock':12,'PartsLockHoldMicroseconds':207,'PartsLockWaitMicroseconds':1,'LogTrace':4,'LoggerElapsedNanoseconds':492500} │
   2. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.961247 │         226 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_8_8_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_8_8_0/   │  100000 │       3195420 │ []          │            8176492 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197560,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':18050,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11370724,'MergeTreeDataWriterCompressedBytes':3195420,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9547,'MergeTreeDataWriterMergingBlocksMicroseconds':6,'ContextLock':12,'PartsLockHoldMicroseconds':200,'PartsLockWaitMicroseconds':112,'LogTrace':4,'LoggerElapsedNanoseconds':484100} │
   3. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.961067 │         226 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_7_7_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_7_7_0/   │  100000 │       3193881 │ []          │            8175178 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196020,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':16948,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11369410,'MergeTreeDataWriterCompressedBytes':3193881,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9825,'MergeTreeDataWriterMergingBlocksMicroseconds':5,'ContextLock':12,'PartsLockHoldMicroseconds':140,'LogTrace':4,'LoggerElapsedNanoseconds':736600} │
   4. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.963664 │         228 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_9_9_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_9_9_0/   │  100000 │       3194982 │ []          │            8177625 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197124,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':18172,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11371857,'MergeTreeDataWriterCompressedBytes':3194982,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9494,'MergeTreeDataWriterMergingBlocksMicroseconds':8,'ContextLock':12,'PartsLockHoldMicroseconds':244,'LogTrace':4,'LoggerElapsedNanoseconds':569100} │
   5. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.963847 │         229 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_10_10_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_10_10_0/ │  100000 │       3193992 │ []          │            8178628 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196132,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':20056,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11372860,'MergeTreeDataWriterCompressedBytes':3193992,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9844,'MergeTreeDataWriterMergingBlocksMicroseconds':8,'ContextLock':12,'PartsLockHoldMicroseconds':157,'PartsLockWaitMicroseconds':267,'LogTrace':4,'LoggerElapsedNanoseconds':117300} │
   6. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.966149 │         231 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_11_11_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_11_11_0/ │  100000 │       3194702 │ []          │            8180089 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196841,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':17496,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11374321,'MergeTreeDataWriterCompressedBytes':3194702,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9538,'MergeTreeDataWriterMergingBlocksMicroseconds':6,'ContextLock':12,'PartsLockHoldMicroseconds':114,'LogTrace':4,'LoggerElapsedNanoseconds':507500} │
   7. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.967187 │         232 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_12_12_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_12_12_0/ │  100000 │       3195431 │ []          │            8174509 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197572,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':18550,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11368741,'MergeTreeDataWriterCompressedBytes':3195431,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':10112,'MergeTreeDataWriterMergingBlocksMicroseconds':2,'ContextLock':12,'PartsLockHoldMicroseconds':172,'LogTrace':4,'LoggerElapsedNanoseconds':158200} │
   8. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.971251 │         235 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_13_13_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_13_13_0/ │  100000 │       3195268 │ []          │            8171520 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197404,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':17372,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11365752,'MergeTreeDataWriterCompressedBytes':3195268,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9863,'MergeTreeDataWriterMergingBlocksMicroseconds':4,'ContextLock':12,'PartsLockHoldMicroseconds':641,'PartsLockWaitMicroseconds':1,'LogTrace':4,'LoggerElapsedNanoseconds':220400} │
   9. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.973292 │         238 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_14_14_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_14_14_0/ │  100000 │       3195693 │ []          │            8175689 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197832,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':20567,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11369921,'MergeTreeDataWriterCompressedBytes':3195693,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9562,'MergeTreeDataWriterMergingBlocksMicroseconds':5,'ContextLock':12,'PartsLockHoldMicroseconds':169,'LogTrace':4,'LoggerElapsedNanoseconds':465200} │
  10. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.975364 │         240 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_15_15_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_15_15_0/ │  100000 │       3194837 │ []          │            8176116 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196979,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':10917,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11370348,'MergeTreeDataWriterCompressedBytes':3194837,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9912,'MergeTreeDataWriterMergingBlocksMicroseconds':2,'ContextLock':12,'PartsLockHoldMicroseconds':156,'LogTrace':4,'LoggerElapsedNanoseconds':314800} │
  11. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:48:44 │ 2025-10-20 11:48:44.329264 │          61 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_5_5_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_5_5_0/   │  121000 │       3858451 │ []          │            9893361 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3860585,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':2090,'MergeTreeDataWriterRows':121000,'MergeTreeDataWriterUncompressedBytes':13758769,'MergeTreeDataWriterCompressedBytes':3858451,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':4572,'MergeTreeDataWriterMergingBlocksMicroseconds':22,'ContextLock':12,'PartsLockHoldMicroseconds':115,'LogTrace':4,'LoggerElapsedNanoseconds':517700} │
1023. │ 19294dc11be6 │ d67284a5-399b-4d10-99c2-018bb4a1b07d │ NewPart    │ NotAMerge    │ Undecided       │ 2025-09-09 │ 2025-09-09 13:15:58 │ 2025-09-09 13:15:58.629685 │         244 │ learn_db │ mart_student_lesson │ fcc5fdee-d5a8-49b9-a1f0-a78a0c27e00e │ all_1_1_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/fcc/fcc5fdee-d5a8-49b9-a1f0-a78a0c27e00e/all_1_1_0/   │ 1111953 │      42881158 │ []          │           90081915 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':40,'WriteBufferFromFileDescriptorWrite':72,'WriteBufferFromFileDescriptorWriteBytes':42883334,'IOBufferAllocs':145,'IOBufferAllocBytes':37077935,'DiskWriteElapsedMicroseconds':24680,'MergeTreeDataWriterRows':1111953,'MergeTreeDataWriterUncompressedBytes':125608515,'MergeTreeDataWriterCompressedBytes':42881158,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterMergingBlocksMicroseconds':1,'ContextLock':10,'ContextLockWaitMicroseconds':1,'PartsLockHoldMicroseconds':78,'QueryProfilerRuns':1,'LogTrace':4,'LoggerElapsedNanoseconds':123300} │
      └─hostname─────┴─query_id─────────────────────────────┴─event_type─┴─merge_reason─┴─merge_algorithm─┴─event_date─┴──────────event_time─┴────event_time_microseconds─┴─duration_ms─┴─database─┴─table───────────────┴─table_uuid───────────────────────────┴─part_name───┴─partition_id─┴─partition─┴─part_type─┴─disk_name─┴─path_on_disk────────────────────────────────────────────────────────────────────┴────rows─┴─size_in_bytes─┴─merged_from─┴─bytes_uncompressed─┴─read_rows─┴─read_bytes─┴─peak_memory_usage─┴─error─┴─exception─┴─ProfileEvents──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
Showed 1000 out of 1023 rows.

1023 rows in set. Elapsed: 0.015 sec. Processed 5.23 thousand rows, 2.09 MB (360.00 thousand rows/s., 143.66 MB/s.)
Peak memory usage: 727.12 KiB.
```

Видим что было создано 10 партов по 100000 строк.