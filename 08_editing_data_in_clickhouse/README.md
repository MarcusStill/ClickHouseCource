# Удаление данных в Clickhouse

Рассмотрим 4 метода удаления данных.

### 1. Пересоздаем таблицу learn_db.mart_student_lesson и наполняем ее 50 млн. строками

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
PARTITION BY (educational_organization_id)
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
FROM numbers(50000000);
```

Результат
```text

```

### 3. Посмотрим на содержимое поля educational_organization_id

```sql
SELECT DISTINCT educational_organization_id FROM learn_db.mart_student_lesson ORDER BY educational_organization_id;
```

Результат
```text
Query id: 91a97d04-063c-4935-9f36-0ab2bb179353

   ┌─educational_organization_id─┐
1. │                           0 │
2. │                           1 │
3. │                           2 │
4. │                           3 │
   └─────────────────────────────┘

4 rows in set. Elapsed: 0.013 sec. Processed 50.00 million rows, 71.92 MB (4.00 billion rows/s., 5.75 GB/s.)
Peak memory usage: 1.73 MiB.
```

### 2. Выполняем мутацию удаление

В dbeaver подготовим запрос 

```sql
ALTER TABLE learn_db.mart_student_lesson DELETE WHERE educational_organization_id = 2;
```

Зайдем в docker-контейнер и на вкладке Exec выполним команды

```bash
bash
clickhouse-client
```

Запускаем ALTER TABLE dbeaver и сразу же выполним второй запрос в clickhouse-client

```sql
SELECT DISTINCT educational_organization_id FROM learn_db.mart_student_lesson ORDER BY educational_organization_id;
```

Мы получим все тот же результат

```text
Query id: f4bab7d4-5885-4810-9e00-57008a702fad

   ┌─educational_organization_id─┐
1. │                           0 │
2. │                           1 │
3. │                           2 │
4. │                           3 │
   └─────────────────────────────┘

4 rows in set. Elapsed: 0.013 sec. Processed 50.00 million rows, 71.92 MB (3.76 billion rows/s., 5.41 GB/s.)
Peak memory usage: 828.85 KiB.
```

Если через 1-2 минуты мы повторим это тоже самый запрос, то увидим что строки с id=2 уже удалены.

```text
Query id: 62d7b5b9-16ef-4a09-9ae3-f8c3a8e11723

   ┌─educational_organization_id─┐
1. │                           0 │
2. │                           1 │
3. │                           3 │
   └─────────────────────────────┘

3 rows in set. Elapsed: 0.011 sec. Processed 35.96 million rows, 43.85 MB (3.32 billion rows/s., 4.05 GB/s.)
Peak memory usage: 1.83 MiB.
```
Для удаления строк нужно некоторое время.

### 3. Выполняем легковесное удаление

```sql
DELETE FROM learn_db.mart_student_lesson WHERE educational_organization_id = 1;
```

Запрос отрабатывает быстрее. И при выполнении выборки мы уже не получаем промежуточные результат

```sql
SELECT DISTINCT educational_organization_id FROM learn_db.mart_student_lesson ORDER BY educational_organization_id;
```

### 4. Удаляем партицию

```sql
ALTER TABLE learn_db.mart_student_lesson DROP PARTITION 0;
SELECT DISTINCT educational_organization_id FROM learn_db.mart_student_lesson ORDER BY educational_organization_id;
```

Это самый быстрый метод удаления.


### 5. Удаляем все данные из таблицы

```sql
TRUNCATE TABLE learn_db.mart_student_lesson;
SELECT * FROM learn_db.mart_student_lesson;
```

# Редактирование данных в Clickhouse

### 6. Пересоздаем таблицу learn_db.mart_student_lesson и наполняем ее 100 млн. строками

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
FROM numbers(100000000);
```

### 7. Смотрим, в какое количество частей данных распределены данные таблицы learn_db.mart_student_lesson

```sql
SELECT * FROM system.parts WHERE table = 'mart_student_lesson';
```

Результат
```text
Query id: 8fc47a43-ef9f-47b1-9394-f6bae51fb935

     ┌─partition─┬─name────────┬─uuid─────────────────────────────────┬─part_type─┬─active─┬─marks─┬────rows─┬─bytes_on_disk─┬─data_compressed_bytes─┬─data_uncompressed_bytes─┬─primary_key_size─┬─marks_bytes─┬─secondary_indices_compressed_bytes─┬─secondary_indices_uncompressed_bytes─┬─secondary_indices_marks_bytes─┬───modification_time─┬─────────remove_time─┬─refcount─┬───min_date─┬───max_date─┬────────────min_time─┬────────────max_time─┬─partition_id─┬─min_block_number─┬─max_block_number─┬─level─┬─data_version─┬─primary_key_bytes_in_memory─┬─primary_key_bytes_in_memory_allocated─┬─index_granularity_bytes_in_memory─┬─index_granularity_bytes_in_memory_allocated─┬─is_frozen─┬─database─┬─table───────────────┬─engine────┬─disk_name─┬─path────────────────────────────────────────────────────────────────────────────┬─hash_of_all_files────────────────┬─hash_of_uncompressed_files───────┬─uncompressed_hash_of_compressed_files─┬─delete_ttl_info_min─┬─delete_ttl_info_max─┬─move_ttl_info.expression─┬─move_ttl_info.min─┬─move_ttl_info.max─┬─default_compression_codec─┬─recompressio⋯.expression─┬─recompression_ttl_info.min─┬─recompression_ttl_info.max─┬─group_by_ttl_info.expression─┬─group_by_ttl_info.min─┬─group_by_ttl_info.max─┬─rows_where_t⋯.expression─┬─rows_where_ttl_info.min─┬─rows_where_ttl_info.max─┬─projections─┬─visible─┬─creation_tid─────────────────────────────────┬─────removal_tid_lock─┬─removal_tid──────────────────────────────────┬─creation_csn─┬─removal_csn─┬─has_lightweight_delete─┬─last_removal_attempt_time─┬─removal_state────────────────────────────┐
  1. │ 0         │ 0_3_3_0     │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  312024 │       6552243 │               6547703 │                24186559 │              321 │        2953 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:33 │ 2025-11-29 14:37:38 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │                3 │                3 │     0 │            3 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_3_3_0/     │ aff4fd6f70158ca9da3a06971c5c7737 │ 66ed7147613e1e635cbde95061f318c0 │ eeac3463781785f56a79cd6da3492609      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:39:04 │ Part hasn't reached removal time yet     │
  2. │ 0         │ 0_3_23_1    │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │   233 │ 1873871 │      36284488 │              36272168 │               145250576 │             1500 │        9534 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:38 │ 2025-11-29 14:38:01 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │                3 │               23 │     1 │            3 │                           0 │                                     0 │                              1864 │                                        1864 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_3_23_1/    │ 8fb0bb662acc57a03cba1517a7410456 │ 5455fe81f3d0ba89c61516b3faa927a6 │ 1fcd8c6e9baa7524ec65fd5c6cf0ac7e      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:39:04 │ Part hasn't reached removal time yet     │
  3. │ 0         │ 0_3_111_2   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      1 │  1083 │ 8745311 │     159157908 │             159113570 │               677875090 │             5254 │       37796 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:38:01 │ 1970-01-01 00:00:00 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │                3 │              111 │     2 │            3 │                        8664 │                                  8920 │                              8664 │                                        8664 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_3_111_2/   │ e5de3c13a78029d162ba21cd53235e03 │ 851067bc03a06bbf9ccba17ab0783e6c │ 615ed9a62f389403f9ff5ab924517ad1      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       1 │ (1,1,'00000000-0000-0000-0000-000000000000') │                    0 │ (0,0,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       1970-01-01 00:00:00 │ Cleanup thread hasn't seen this part yet │
  4. │ 0         │ 0_8_8_0     │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  312185 │       6552956 │               6548428 │                24202147 │              323 │        2939 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:34 │ 2025-11-29 14:37:38 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │                8 │                8 │     0 │            8 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_8_8_0/     │ c7731b341e4eb494c3082d0de1de3149 │ 723787d5a29d2203360efa9b54fead68 │ 1016a9cc8901a0e116e34f4b153f29a5      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:39:04 │ Part hasn't reached removal time yet     │
  5. │ 0         │ 0_10_10_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  312184 │       6556622 │               6552099 │                24199787 │              321 │        2936 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:34 │ 2025-11-29 14:37:38 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               10 │               10 │     0 │           10 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_10_10_0/   │ 816f473aac5729098d965b3238bfd3bf │ 2f8d3178f1703a981be1d72476a53e92 │ 01f3911fb45586867b6dabe829c4886c      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:39:04 │ Part hasn't reached removal time yet     │
  6. │ 0         │ 0_16_16_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  312487 │       6563731 │               6559185 │                24217583 │              318 │        2962 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:35 │ 2025-11-29 14:37:38 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               16 │               16 │     0 │           16 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_16_16_0/   │ 696a9bbee1f59c8ee05dbce0656b3252 │ 835a785121970739249fd35425466f2e │ b6d14590c0161eb0f891af478518ae85      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:39:04 │ Part hasn't reached removal time yet     │
  7. │ 0         │ 0_18_18_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  313027 │       6569851 │               6565318 │                24261123 │              324 │        2943 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:36 │ 2025-11-29 14:37:38 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               18 │               18 │     0 │           18 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_18_18_0/   │ 8215c8d525f76969f0b86fcbbc3e5b63 │ 27b76278a411568de286317ebdb7483f │ 17f48582adae6544baffd5bb009e806e      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:39:04 │ Part hasn't reached removal time yet     │
  8. │ 0         │ 0_23_23_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  311964 │       6550571 │               6546019 │                24181424 │              323 │        2963 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:37 │ 2025-11-29 14:37:38 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               23 │               23 │     0 │           23 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_23_23_0/   │ f6c583eda01bb76fd89dbeb66d853cb4 │ e68ba2641b5bcda4c3097a0927a6dd3d │ f27020e56462463a53e0c72c5b92b86d      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:39:04 │ Part hasn't reached removal time yet     │
  9. │ 0         │ 0_27_27_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  311546 │       6543121 │               6538614 │                24150998 │              321 │        2920 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:38 │ 2025-11-29 14:37:42 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               27 │               27 │     0 │           27 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_27_27_0/   │ 3354a482e6f76eee7d56d181b67d0b7e │ a14e8573fa2d4f33b61b778b8c928f21 │ 6cf70b77f249a67e0804894ddeadb2e6      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:39:04 │ Part hasn't reached removal time yet     │
 10. │ 0         │ 0_27_45_1   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │   232 │ 1872557 │      36263054 │              36250846 │               145143955 │             1483 │        9439 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:42 │ 2025-11-29 14:38:01 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               27 │               45 │     1 │           27 │                           0 │                                     0 │                              1856 │                                        1856 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_27_45_1/   │ 326aeb988b67c50c119a48771e7b81a2 │ 913cbd4db55fc7ecc3d6bae4472e1b91 │ ca08ae52687c9fe990825266dc40c56e      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:39:04 │ Part hasn't reached removal time yet     │
...
424. │ 3         │ 3_357_357_0 │ 00000000-0000-0000-0000-000000000000 │ Wide      │      1 │    21 │  163806 │       3381486 │               3377965 │                13239739 │              197 │        2070 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:38:46 │ 1970-01-01 00:00:00 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 3            │              357 │              357 │     0 │          357 │                          84 │                                   212 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/3_357_357_0/ │ 1bf0cea708461597a03a6ccf5c83bc0e │ d1cd43bc8cada2efcae767ae0f0d55f5 │ 3b3a1b0b5434f263877f4e223a26e6a5      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       1 │ (1,1,'00000000-0000-0000-0000-000000000000') │                    0 │ (0,0,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       1970-01-01 00:00:00 │ Cleanup thread hasn't seen this part yet │
     └─partition─┴─name────────┴─uuid─────────────────────────────────┴─part_type─┴─active─┴─marks─┴────rows─┴─bytes_on_disk─┴─data_compressed_bytes─┴─data_uncompressed_bytes─┴─primary_key_size─┴─marks_bytes─┴─secondary_indices_compressed_bytes─┴─secondary_indices_uncompressed_bytes─┴─secondary_indices_marks_bytes─┴───modification_time─┴─────────remove_time─┴─refcount─┴───min_date─┴───max_date─┴────────────min_time─┴────────────max_time─┴─partition_id─┴─min_block_number─┴─max_block_number─┴─level─┴─data_version─┴─primary_key_bytes_in_memory─┴─primary_key_bytes_in_memory_allocated─┴─index_granularity_bytes_in_memory─┴─index_granularity_bytes_in_memory_allocated─┴─is_frozen─┴─database─┴─table───────────────┴─engine────┴─disk_name─┴─path────────────────────────────────────────────────────────────────────────────┴─hash_of_all_files────────────────┴─hash_of_uncompressed_files───────┴─uncompressed_hash_of_compressed_files─┴─delete_ttl_info_min─┴─delete_ttl_info_max─┴─move_ttl_info.expression─┴─move_ttl_info.min─┴─move_ttl_info.max─┴─default_compression_codec─┴─recompressio⋯.expression─┴─recompression_ttl_info.min─┴─recompression_ttl_info.max─┴─group_by_ttl_info.expression─┴─group_by_ttl_info.min─┴─group_by_ttl_info.max─┴─rows_where_t⋯.expression─┴─rows_where_ttl_info.min─┴─rows_where_ttl_info.max─┴─projections─┴─visible─┴─creation_tid─────────────────────────────────┴─────removal_tid_lock─┴─removal_tid──────────────────────────────────┴─creation_csn─┴─removal_csn─┴─has_lightweight_delete─┴─last_removal_attempt_time─┴─removal_state────────────────────────────┘

424 rows in set. Elapsed: 0.009 sec.
```

### 8. Смотрим, в какое количество частей данных распределены данные таблицы learn_db.mart_student_lesson

```sql
SELECT * FROM system.parts WHERE table = 'mart_student_lesson';
```

Результат
```text
Query id: 2e4a6342-dbd3-4416-8a70-f073b351e004

     ┌─partition─┬─name────────┬─uuid─────────────────────────────────┬─part_type─┬─active─┬─marks─┬────rows─┬─bytes_on_disk─┬─data_compressed_bytes─┬─data_uncompressed_bytes─┬─primary_key_size─┬─marks_bytes─┬─secondary_indices_compressed_bytes─┬─secondary_indices_uncompressed_bytes─┬─secondary_indices_marks_bytes─┬───modification_time─┬─────────remove_time─┬─refcount─┬───min_date─┬───max_date─┬────────────min_time─┬────────────max_time─┬─partition_id─┬─min_block_number─┬─max_block_number─┬─level─┬─data_version─┬─primary_key_bytes_in_memory─┬─primary_key_bytes_in_memory_allocated─┬─index_granularity_bytes_in_memory─┬─index_granularity_bytes_in_memory_allocated─┬─is_frozen─┬─database─┬─table───────────────┬─engine────┬─disk_name─┬─path────────────────────────────────────────────────────────────────────────────┬─hash_of_all_files────────────────┬─hash_of_uncompressed_files───────┬─uncompressed_hash_of_compressed_files─┬─delete_ttl_info_min─┬─delete_ttl_info_max─┬─move_ttl_info.expression─┬─move_ttl_info.min─┬─move_ttl_info.max─┬─default_compression_codec─┬─recompressio⋯.expression─┬─recompression_ttl_info.min─┬─recompression_ttl_info.max─┬─group_by_ttl_info.expression─┬─group_by_ttl_info.min─┬─group_by_ttl_info.max─┬─rows_where_t⋯.expression─┬─rows_where_ttl_info.min─┬─rows_where_ttl_info.max─┬─projections─┬─visible─┬─creation_tid─────────────────────────────────┬─────removal_tid_lock─┬─removal_tid──────────────────────────────────┬─creation_csn─┬─removal_csn─┬─has_lightweight_delete─┬─last_removal_attempt_time─┬─removal_state────────────────────────────┐
  1. │ 0         │ 0_3_3_0     │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  312024 │       6552243 │               6547703 │                24186559 │              321 │        2953 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:33 │ 2025-11-29 14:37:38 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │                3 │                3 │     0 │            3 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_3_3_0/     │ aff4fd6f70158ca9da3a06971c5c7737 │ 66ed7147613e1e635cbde95061f318c0 │ eeac3463781785f56a79cd6da3492609      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:43:44 │ Part hasn't reached removal time yet     │
  2. │ 0         │ 0_3_23_1    │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │   233 │ 1873871 │      36284488 │              36272168 │               145250576 │             1500 │        9534 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:38 │ 2025-11-29 14:38:01 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │                3 │               23 │     1 │            3 │                           0 │                                     0 │                              1864 │                                        1864 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_3_23_1/    │ 8fb0bb662acc57a03cba1517a7410456 │ 5455fe81f3d0ba89c61516b3faa927a6 │ 1fcd8c6e9baa7524ec65fd5c6cf0ac7e      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:43:44 │ Part hasn't reached removal time yet     │
  3. │ 0         │ 0_3_111_2   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      1 │  1083 │ 8745311 │     159157908 │             159113570 │               677875090 │             5254 │       37796 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:38:01 │ 1970-01-01 00:00:00 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │                3 │              111 │     2 │            3 │                        8664 │                                  8920 │                              8664 │                                        8664 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_3_111_2/   │ e5de3c13a78029d162ba21cd53235e03 │ 851067bc03a06bbf9ccba17ab0783e6c │ 615ed9a62f389403f9ff5ab924517ad1      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       1 │ (1,1,'00000000-0000-0000-0000-000000000000') │                    0 │ (0,0,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       1970-01-01 00:00:00 │ Cleanup thread hasn't seen this part yet │
  4. │ 0         │ 0_8_8_0     │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  312185 │       6552956 │               6548428 │                24202147 │              323 │        2939 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:34 │ 2025-11-29 14:37:38 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │                8 │                8 │     0 │            8 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_8_8_0/     │ c7731b341e4eb494c3082d0de1de3149 │ 723787d5a29d2203360efa9b54fead68 │ 1016a9cc8901a0e116e34f4b153f29a5      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:43:44 │ Part hasn't reached removal time yet     │
  5. │ 0         │ 0_10_10_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  312184 │       6556622 │               6552099 │                24199787 │              321 │        2936 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:34 │ 2025-11-29 14:37:38 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               10 │               10 │     0 │           10 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_10_10_0/   │ 816f473aac5729098d965b3238bfd3bf │ 2f8d3178f1703a981be1d72476a53e92 │ 01f3911fb45586867b6dabe829c4886c      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:43:44 │ Part hasn't reached removal time yet     │
  6. │ 0         │ 0_16_16_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  312487 │       6563731 │               6559185 │                24217583 │              318 │        2962 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:35 │ 2025-11-29 14:37:38 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               16 │               16 │     0 │           16 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_16_16_0/   │ 696a9bbee1f59c8ee05dbce0656b3252 │ 835a785121970739249fd35425466f2e │ b6d14590c0161eb0f891af478518ae85      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:43:44 │ Part hasn't reached removal time yet     │
  7. │ 0         │ 0_18_18_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  313027 │       6569851 │               6565318 │                24261123 │              324 │        2943 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:36 │ 2025-11-29 14:37:38 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               18 │               18 │     0 │           18 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_18_18_0/   │ 8215c8d525f76969f0b86fcbbc3e5b63 │ 27b76278a411568de286317ebdb7483f │ 17f48582adae6544baffd5bb009e806e      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:43:44 │ Part hasn't reached removal time yet     │
  8. │ 0         │ 0_23_23_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  311964 │       6550571 │               6546019 │                24181424 │              323 │        2963 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:37 │ 2025-11-29 14:37:38 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               23 │               23 │     0 │           23 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_23_23_0/   │ f6c583eda01bb76fd89dbeb66d853cb4 │ e68ba2641b5bcda4c3097a0927a6dd3d │ f27020e56462463a53e0c72c5b92b86d      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:43:44 │ Part hasn't reached removal time yet     │
  9. │ 0         │ 0_27_27_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  311546 │       6543121 │               6538614 │                24150998 │              321 │        2920 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:38 │ 2025-11-29 14:37:42 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               27 │               27 │     0 │           27 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_27_27_0/   │ 3354a482e6f76eee7d56d181b67d0b7e │ a14e8573fa2d4f33b61b778b8c928f21 │ 6cf70b77f249a67e0804894ddeadb2e6      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:43:44 │ Part hasn't reached removal time yet     │
 10. │ 0         │ 0_27_45_1   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │   232 │ 1872557 │      36263054 │              36250846 │               145143955 │             1483 │        9439 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:42 │ 2025-11-29 14:38:01 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               27 │               45 │     1 │           27 │                           0 │                                     0 │                              1856 │                                        1856 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_27_45_1/   │ 326aeb988b67c50c119a48771e7b81a2 │ 913cbd4db55fc7ecc3d6bae4472e1b91 │ ca08ae52687c9fe990825266dc40c56e      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:43:44 │ Part hasn't reached removal time yet     │
 11. │ 0         │ 0_29_29_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  311863 │       6549433 │               6544885 │                24168305 │              322 │        2960 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:38 │ 2025-11-29 14:37:42 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               29 │               29 │     0 │           29 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_29_29_0/   │ 7aada52a658e024095ad1d07120d5f77 │ aad55669e49f84becd7c892a52ccdf47 │ abd5e6273e5baa08fe510660283c12df      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:43:44 │ Part hasn't reached removal time yet     │
 12. │ 0         │ 0_33_33_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  312324 │       6553860 │               6549305 │                24209073 │              327 │        2962 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:39 │ 2025-11-29 14:37:42 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               33 │               33 │     0 │           33 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_33_33_0/   │ e3c9a5e0447d81294fa860fa56caf771 │ a57e423f7f9f121b15d66c407417d771 │ 09e10205c628b0dd444a794a506de40a      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:43:44 │ Part hasn't reached removal time yet     │
 13. │ 0         │ 0_39_39_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  311966 │       6549704 │               6545157 │                24180189 │              323 │        2958 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:40 │ 2025-11-29 14:37:42 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               39 │               39 │     0 │           39 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_39_39_0/   │ fa7504dc722a618aa163bb300fc9bf0f │ 8b3b7254679a006abd6aef37738fecab │ 72e437640761c907032d43ce42f8fa39      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:43:44 │ Part hasn't reached removal time yet     │
 14. │ 0         │ 0_43_43_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  312778 │       6567041 │               6562490 │                24240920 │              325 │        2960 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:41 │ 2025-11-29 14:37:42 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               43 │               43 │     0 │           43 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_43_43_0/   │ f51f4afb0a19ae9d00ff745e724478e4 │ 153368a258093854c32492d95bc96100 │ f44e337836bef7c1f8d53e0e7491283c      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:43:44 │ Part hasn't reached removal time yet     │
 15. │ 0         │ 0_45_45_0   │ 00000000-0000-0000-0000-000000000000 │ Wide      │      0 │    40 │  312080 │       6552825 │               6548281 │                24192526 │              322 │        2956 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:37:41 │ 2025-11-29 14:37:42 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 0            │               45 │               45 │     0 │           45 │                           0 │                                     0 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/0_45_45_0/   │ 783c72b31eab2b4d0b22707bda0f0027 │ 7e4998b3242be322c1f4d4637e7a89ec │ 043d5cd3861689ee2ac56b5ba6fb3ad6      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       0 │ (1,1,'00000000-0000-0000-0000-000000000000') │ 15317705874040209379 │ (1,1,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       2025-11-29 14:43:44 │ Part hasn't reached removal time yet     │
424. │ 3         │ 3_357_357_0 │ 00000000-0000-0000-0000-000000000000 │ Wide      │      1 │    21 │  163806 │       3381486 │               3377965 │                13239739 │              197 │        2070 │                                  0 │                                    0 │                             0 │ 2025-11-29 14:38:46 │ 1970-01-01 00:00:00 │        1 │ 1970-01-01 │ 1970-01-01 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ 3            │              357 │              357 │     0 │          357 │                          84 │                                   212 │                                25 │                                          25 │         0 │ learn_db │ mart_student_lesson │ MergeTree │ default   │ /var/lib/clickhouse/store/265/26542376-2395-421a-be57-39d47071eae9/3_357_357_0/ │ 1bf0cea708461597a03a6ccf5c83bc0e │ d1cd43bc8cada2efcae767ae0f0d55f5 │ 3b3a1b0b5434f263877f4e223a26e6a5      │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │ []                       │ []                │ []                │ LZ4                       │ []                       │ []                         │ []                         │ []                           │ []                    │ []                    │ []                       │ []                      │ []                      │ []          │       1 │ (1,1,'00000000-0000-0000-0000-000000000000') │                    0 │ (0,0,'00000000-0000-0000-0000-000000000000') │            0 │           0 │                      0 │       1970-01-01 00:00:00 │ Cleanup thread hasn't seen this part yet │
     └─partition─┴─name────────┴─uuid─────────────────────────────────┴─part_type─┴─active─┴─marks─┴────rows─┴─bytes_on_disk─┴─data_compressed_bytes─┴─data_uncompressed_bytes─┴─primary_key_size─┴─marks_bytes─┴─secondary_indices_compressed_bytes─┴─secondary_indices_uncompressed_bytes─┴─secondary_indices_marks_bytes─┴───modification_time─┴─────────remove_time─┴─refcount─┴───min_date─┴───max_date─┴────────────min_time─┴────────────max_time─┴─partition_id─┴─min_block_number─┴─max_block_number─┴─level─┴─data_version─┴─primary_key_bytes_in_memory─┴─primary_key_bytes_in_memory_allocated─┴─index_granularity_bytes_in_memory─┴─index_granularity_bytes_in_memory_allocated─┴─is_frozen─┴─database─┴─table───────────────┴─engine────┴─disk_name─┴─path────────────────────────────────────────────────────────────────────────────┴─hash_of_all_files────────────────┴─hash_of_uncompressed_files───────┴─uncompressed_hash_of_compressed_files─┴─delete_ttl_info_min─┴─delete_ttl_info_max─┴─move_ttl_info.expression─┴─move_ttl_info.min─┴─move_ttl_info.max─┴─default_compression_codec─┴─recompressio⋯.expression─┴─recompression_ttl_info.min─┴─recompression_ttl_info.max─┴─group_by_ttl_info.expression─┴─group_by_ttl_info.min─┴─group_by_ttl_info.max─┴─rows_where_t⋯.expression─┴─rows_where_ttl_info.min─┴─rows_where_ttl_info.max─┴─projections─┴─visible─┴─creation_tid─────────────────────────────────┴─────removal_tid_lock─┴─removal_tid──────────────────────────────────┴─creation_csn─┴─removal_csn─┴─has_lightweight_delete─┴─last_removal_attempt_time─┴─removal_state────────────────────────────┘

424 rows in set. Elapsed: 0.011 sec.
```

Создано 434 парта. Количество строк разное: где-то 300 тыс., где-то больше 1 млн. Некоторые парты уже объединены (level <> 0).

### 9. Смотрим, в каком количестве частей данных попадают строки с предметом "Математика"

```sql
SELECT 
	DISTINCT _part
FROM 
	learn_db.mart_student_lesson
WHERE 
	subject_name = 'Математика';
```

Результат
```text
_part      |
-----------+
0_3_110_2  |
2_226_334_2|
1_116_221_2|
0_225_333_2|
3_109_213_2|
1_2_111_2  |
1_227_335_2|
2_338_359_1|
2_115_222_2|
0_113_224_2|
2_1_112_2  |
3_4_106_2  |
0_340_360_1|
3_219_321_2|
1_339_357_1|
3_325_345_1|
3_350_350_0|
3_355_355_0|
3_358_358_0|
```
       
### 10. Запускаем изменение названия предметов уроков с "Математика" на "Математическая наука"

```sql
ALTER TABLE learn_db.mart_student_lesson
UPDATE subject_name = 'Математическая наука' WHERE subject_name = 'Математика';
```
Так как мы выполняем стандартный update, Clickhouse найдет нужные строки, начнет их читать с диска, загружать в RAM, раскодировать/править, опять записывать на жесткий диск и мы будем получать 'Математическая наука' после того, как данные записаны на жд. А до этого при запросе данных получим Математику. И в какой-то момент времени увидим оба названия предмета в таблице.

Зайдем в docker-контейнер и на вкладке Exec выполним команды

```bash
bash
clickhouse-client
```

### 11. Посмотрим за процессом редактирования данных

Будем в clickhouse-client выполнять запрос на получение названий предметов уроков

```sql
SELECT subject_name, count(*) as cnt FROM learn_db.mart_student_lesson GROUP BY subject_name;
```

Но перед этим запустим в dbeaver запрос на изменение названия предметов уроков с "Математика" на "Математическая наука"

```sql
ALTER TABLE learn_db.mart_student_lesson
UPDATE subject_name = 'Математическая наука' WHERE subject_name = 'Математика';
```

Запускаем в dbeaver update и сразу же переключаемся в clickhouse-client и выполняем select. В результате увидим, что у нас есть и "Математическая наука" и "Математика".

Т.е. пока парты полностью не перезапишутся мы можем получать промежуточные данные в процессе изменения.

После завершения обновления у нас будет следующая информация

Результат
```text
subject_name        |cnt     |
--------------------+--------+
Физика              | 6666872|
Биология            | 6672889|
Химия               | 6667011|
Русский язык        | 6662697|
География           | 6668558|
Литература          | 6664351|
Информатика         |46661462|
Математическая наука| 6667516|
Физическая культура | 6668644|
```

# Легковесное редактирование данных в Clickhouse

На старте курса мы использовали Clickhouse версии 25.4. В нем легковесного редактирования не было. Для теста его запустим новую версию СН.

### 12. Запускаем контейнер с Clickhouse версии 25.5.3.75

```bash
docker run --name clickhouse-course2 -e CLICKHOUSE_DB=learn_db -e CLICKHOUSE_USER=username -e CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT=1 -e CLICKHOUSE_PASSWORD=password -p 8124:8123 -p 9001:9000/tcp -d clickhouse:25.5.3.75
```

### 13. Создаем таблицу learn_db.mart_student_lesson

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
FROM numbers(100000000);
```

### 14. Включение легковесного редактирования на уровне сессии

```sql
SET allow_experimental_lightweight_update = 1;
```

### 15. Вставляем данные

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

### 16. Запускаем редактирование в dbeaver 

```sql
ALTER TABLE learn_db.mart_student_lesson
UPDATE subject_name = 'Математическая наука' WHERE subject_name = 'Математика'
SETTINGS allow_experimental_lightweight_update = 1;
```

### 17. И сразу же в clickhouse-client дважды выполняем запрос на получение названий предметов уроков

```sql
SELECT subject_name, count(*) as cnt FROM learn_db.mart_student_lesson GROUP BY subject_name;
```

Результат 1
```text 
Query id: 044a0e8c-d726-478d-822c-582dae34e66b

   ┌─subject_name────────┬──────cnt─┐
1. │ Физика              │  6666472 │
2. │ Биология            │  6661183 │
3. │ Химия               │  6665472 │
4. │ Русский язык        │  6662921 │
5. │ География           │  6667066 │
6. │ Литература          │  6669659 │
7. │ Математика          │  6669108 │
8. │ Информатика         │ 46670743 │
9. │ Физическая культура │  6667376 │
   └─────────────────────┴──────────┘

9 rows in set. Elapsed: 0.755 sec. Processed 100.00 million rows, 2.97 GB (132.46 million rows/s., 3.93 GB/s.)
Peak memory usage: 447.69 KiB.
```

Результат 2
```text 
Query id: 044a0e8c-d726-478d-822c-582dae34e66b

   ┌─subject_name─────────┬──────cnt─┐
1. │ Физика               │  6666472 │
2. │ Биология             │  6661183 │
3. │ Химия                │  6665472 │
4. │ Русский язык         │  6662921 │
5. │ География            │  6667066 │
6. │ Литература           │  6669659 │
7. │ Математическая наука │  6669108 │
8. │ Информатика          │ 46670743 │
9. │ Физическая культура  │  6667376 │
   └──────────────────────┴──────────┘

9 rows in set. Elapsed: 0.755 sec. Processed 100.00 million rows, 2.97 GB (132.46 million rows/s., 3.93 GB/s.)
Peak memory usage: 447.69 KiB.
```

Вначале мы увидели только строки "Математика", а затем мы увидели только строки "Математическая наука". У нас не было промежуточного этапа когда были и те, и другие.
С параметром allow_experimental_lightweight_update = 1 запрос отработал гораздо быстрее. 