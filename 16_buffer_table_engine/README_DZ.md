## Задание

Максимальное ускорение загрузки данных в таблицу.
Повторите вставку 1000 раз по 1000 строк в таблицу mart_student_lesson, которая была показана на уроке. Делайте вставки с помощью утилиты clickhouse-benchmark в 5 параллельных потоков.

Поэксперементируйте с подбором параметров движка таблиц Buffer. Попробуйте по крайней мере 3 варианта разных комбинаций параметров. Определите при каких параметрах вы смогли достигнуть максимальной скорости загрузки.
В качестве ответа напишите, какие параметры вы пробовали и какую скорость загрузки в MB/sec смогли достичь.

### 1. Пересоздадим буферную таблицу, указав что нам нужно 5 потоков

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_buffer; 
CREATE TABLE learn_db.mart_student_lesson_buffer AS learn_db.mart_student_lesson ENGINE = Buffer(learn_db, mart_student_lesson, 5, 10, 100, 10000, 1000000, 10000000, 100000000)
```

### 2. Запускаем 1000 вставок по 1000 строк в один поток в буферную таблицу параллельно в 5 потоков
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
" --iterations=1000 --concurrency=5
```

Время вставки последней порции данных
```text
Queries executed: 1000.

localhost:9000, queries: 1000, QPS: 432.181, RPS: 432180.917, MiB/s: 3.297, result RPS: 0.000, result MiB/s: 0.000.

0%              0.006 sec.
10%             0.008 sec.
20%             0.008 sec.
30%             0.009 sec.
40%             0.009 sec.
50%             0.010 sec.
60%             0.010 sec.
70%             0.011 sec.
80%             0.012 sec.
90%             0.013 sec.
95%             0.014 sec.
99%             0.020 sec.
99.9%           0.022 sec.
99.99%          0.022 sec.
```

### 3. Пересоздадим буферную таблицу, указав что нам нужно 16 потоков

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_buffer; 
CREATE TABLE learn_db.mart_student_lesson_buffer AS learn_db.mart_student_lesson ENGINE = BufferCREATE TABLE learn_db.mart_student_lesson_buffer AS learn_db.mart_student_lesson ENGINE = Buffer(learn_db, mart_student_lesson, 16, 60, 300, 100000, 5000000, 50000000, 500000000)
```

### 4. Запускаем 1000 вставок по 1000 строк в один поток в буферную таблицу параллельно в 16 потоков
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
" --iterations=1000 --concurrency=16
```

Время вставки последней порции данных
```text
Queries executed: 1000.

localhost:9000, queries: 1000, QPS: 793.466, RPS: 793466.470, MiB/s: 6.054, result RPS: 0.000, result MiB/s: 0.000.

0%              0.008 sec.
10%             0.010 sec.
20%             0.011 sec.
30%             0.012 sec.
40%             0.014 sec.
50%             0.015 sec.
60%             0.016 sec.
70%             0.018 sec.
80%             0.021 sec.
90%             0.024 sec.
95%             0.028 sec.
99%             0.036 sec.
99.9%           0.045 sec.
99.99%          0.054 sec.
```

### 5. Смотрим, какие части данных сформировались

```sql
SELECT * FROM system.part_log where table='mart_student_lesson' and event_type = 1 ORDER BY event_time DESC;
```

```text
Query id: fec76c1c-134f-4143-ba4b-33af12ee2a4c

      ┌─hostname─────┬─query_id─────────────────────────────┬─event_type─┬─merge_reason─┬─merge_algorithm─┬─event_date─┬──────────event_time─┬────event_time_microseconds─┬─duration_ms─┬─database─┬─table───────────────┬─table_uuid───────────────────────────┬─part_name───┬─partition_id─┬─partition─┬─part_type─┬─disk_name─┬─path_on_disk────────────────────────────────────────────────────────────────────┬────rows─┬─size_in_bytes─┬─merged_from─┬─bytes_uncompressed─┬─read_rows─┬─read_bytes─┬─peak_memory_usage─┬─error─┬─exception─┬─ProfileEvents──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
   1. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.228517 │         114 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_21_21_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_21_21_0/ │  100000 │       3195069 │ []          │            8174053 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197208,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':10159,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11368285,'MergeTreeDataWriterCompressedBytes':3195069,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7482,'MergeTreeDataWriterMergingBlocksMicroseconds':4,'ContextLock':12,'PartsLockHoldMicroseconds':212,'LogTrace':4,'LoggerElapsedNanoseconds':215700} │
   2. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.232240 │         118 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_22_22_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_22_22_0/ │  100000 │       3196931 │ []          │            8173916 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3199070,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':13077,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11368148,'MergeTreeDataWriterCompressedBytes':3196931,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7053,'MergeTreeDataWriterMergingBlocksMicroseconds':5,'ContextLock':12,'PartsLockHoldMicroseconds':120,'LogTrace':4,'LoggerElapsedNanoseconds':160300} │
   3. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.234616 │         121 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_23_23_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_23_23_0/ │  100000 │       3193896 │ []          │            8172228 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196036,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':11688,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11366460,'MergeTreeDataWriterCompressedBytes':3193896,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':6972,'MergeTreeDataWriterMergingBlocksMicroseconds':6,'ContextLock':12,'PartsLockHoldMicroseconds':109,'LogTrace':4,'LoggerElapsedNanoseconds':121700} │
   4. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.235888 │         122 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_24_24_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_24_24_0/ │  100000 │       3194687 │ []          │            8177659 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196826,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':11425,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11371891,'MergeTreeDataWriterCompressedBytes':3194687,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':5686,'MergeTreeDataWriterMergingBlocksMicroseconds':27,'ContextLock':12,'PartsLockHoldMicroseconds':93,'LogTrace':4,'LoggerElapsedNanoseconds':349300} │
   5. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.236364 │         122 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_25_25_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_25_25_0/ │  100000 │       3196960 │ []          │            8175417 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3199097,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':12495,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11369649,'MergeTreeDataWriterCompressedBytes':3196960,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':6948,'MergeTreeDataWriterMergingBlocksMicroseconds':5,'ContextLock':12,'PartsLockHoldMicroseconds':149,'LogTrace':4,'LoggerElapsedNanoseconds':124200} │
   6. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.237169 │         123 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_26_26_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_26_26_0/ │  100000 │       3193785 │ []          │            8176642 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3195924,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':11079,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11370874,'MergeTreeDataWriterCompressedBytes':3193785,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':6851,'MergeTreeDataWriterMergingBlocksMicroseconds':16,'ContextLock':12,'PartsLockHoldMicroseconds':76,'LogTrace':4,'LoggerElapsedNanoseconds':163800} │
   7. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.238095 │         124 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_27_27_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_27_27_0/ │  100000 │       3193979 │ []          │            8174987 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196119,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':11451,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11369219,'MergeTreeDataWriterCompressedBytes':3193979,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7245,'MergeTreeDataWriterMergingBlocksMicroseconds':3,'ContextLock':12,'PartsLockHoldMicroseconds':97,'LogTrace':4,'LoggerElapsedNanoseconds':93900} │
   8. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.239236 │         125 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_28_28_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_28_28_0/ │  100000 │       3196029 │ []          │            8176941 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3198168,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':10489,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11371173,'MergeTreeDataWriterCompressedBytes':3196029,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7090,'MergeTreeDataWriterMergingBlocksMicroseconds':5,'ContextLock':12,'PartsLockHoldMicroseconds':697,'LogTrace':4,'LoggerElapsedNanoseconds':164300} │
   9. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.239608 │         126 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_29_29_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_29_29_0/ │  100000 │       3194428 │ []          │            8177196 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196568,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':12009,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11371428,'MergeTreeDataWriterCompressedBytes':3194428,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7134,'MergeTreeDataWriterMergingBlocksMicroseconds':4,'ContextLock':12,'PartsLockHoldMicroseconds':90,'LogTrace':4,'LoggerElapsedNanoseconds':95700} │
  10. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.241296 │         127 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_30_30_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_30_30_0/ │  100000 │       3195435 │ []          │            8172512 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197574,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':9136,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11366744,'MergeTreeDataWriterCompressedBytes':3195435,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7473,'MergeTreeDataWriterMergingBlocksMicroseconds':6,'ContextLock':12,'PartsLockHoldMicroseconds':103,'LogTrace':4,'LoggerElapsedNanoseconds':140800} │
  11. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:18:38 │ 2025-10-20 12:18:38.787934 │         143 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_20_20_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_20_20_0/ │  200000 │       6383558 │ []          │           16351066 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':42,'WriteBufferFromFileDescriptorWriteBytes':6385708,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':5724,'MergeTreeDataWriterRows':200000,'MergeTreeDataWriterUncompressedBytes':22740354,'MergeTreeDataWriterCompressedBytes':6383558,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7327,'MergeTreeDataWriterMergingBlocksMicroseconds':4,'ContextLock':12,'PartsLockHoldMicroseconds':73,'LogTrace':4,'LoggerElapsedNanoseconds':97200} │
  12. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:18:38 │ 2025-10-20 12:18:38.787121 │         143 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_19_19_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_19_19_0/ │  200000 │       6383556 │ []          │           16350154 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':42,'WriteBufferFromFileDescriptorWriteBytes':6385706,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':4684,'MergeTreeDataWriterRows':200000,'MergeTreeDataWriterUncompressedBytes':22739442,'MergeTreeDataWriterCompressedBytes':6383556,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9282,'MergeTreeDataWriterMergingBlocksMicroseconds':3,'ContextLock':12,'PartsLockHoldMicroseconds':67,'LogTrace':4,'LoggerElapsedNanoseconds':74800} │
  13. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:18:38 │ 2025-10-20 12:18:38.786507 │         142 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_18_18_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_18_18_0/ │  200000 │       6383060 │ []          │           16355063 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':42,'WriteBufferFromFileDescriptorWriteBytes':6385211,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':4592,'MergeTreeDataWriterRows':200000,'MergeTreeDataWriterUncompressedBytes':22744351,'MergeTreeDataWriterCompressedBytes':6383060,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':8979,'MergeTreeDataWriterMergingBlocksMicroseconds':2,'ContextLock':12,'PartsLockHoldMicroseconds':99,'LogTrace':4,'LoggerElapsedNanoseconds':78200} │
  14. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:18:38 │ 2025-10-20 12:18:38.786247 │         142 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_17_17_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_17_17_0/ │  200000 │       6382904 │ []          │           16351166 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':42,'WriteBufferFromFileDescriptorWriteBytes':6385052,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':5650,'MergeTreeDataWriterRows':200000,'MergeTreeDataWriterUncompressedBytes':22740454,'MergeTreeDataWriterCompressedBytes':6382904,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7232,'MergeTreeDataWriterMergingBlocksMicroseconds':8,'ContextLock':12,'PartsLockHoldMicroseconds':116,'LogTrace':4,'LoggerElapsedNanoseconds':188600} │
  15. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:18:38 │ 2025-10-20 12:18:38.783848 │         139 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_16_16_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_16_16_0/ │  200000 │       6381901 │ []          │           16351636 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':42,'WriteBufferFromFileDescriptorWriteBytes':6384053,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':5439,'MergeTreeDataWriterRows':200000,'MergeTreeDataWriterUncompressedBytes':22740924,'MergeTreeDataWriterCompressedBytes':6381901,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7385,'MergeTreeDataWriterMergingBlocksMicroseconds':3,'ContextLock':12,'PartsLockHoldMicroseconds':134,'LogTrace':4,'LoggerElapsedNanoseconds':177600} │
  16. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.975364 │         240 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_15_15_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_15_15_0/ │  100000 │       3194837 │ []          │            8176116 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196979,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':10917,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11370348,'MergeTreeDataWriterCompressedBytes':3194837,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9912,'MergeTreeDataWriterMergingBlocksMicroseconds':2,'ContextLock':12,'PartsLockHoldMicroseconds':156,'LogTrace':4,'LoggerElapsedNanoseconds':314800} │
  17. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.973292 │         238 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_14_14_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_14_14_0/ │  100000 │       3195693 │ []          │            8175689 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197832,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':20567,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11369921,'MergeTreeDataWriterCompressedBytes':3195693,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9562,'MergeTreeDataWriterMergingBlocksMicroseconds':5,'ContextLock':12,'PartsLockHoldMicroseconds':169,'LogTrace':4,'LoggerElapsedNanoseconds':465200} │
  18. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.971251 │         235 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_13_13_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_13_13_0/ │  100000 │       3195268 │ []          │            8171520 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197404,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':17372,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11365752,'MergeTreeDataWriterCompressedBytes':3195268,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9863,'MergeTreeDataWriterMergingBlocksMicroseconds':4,'ContextLock':12,'PartsLockHoldMicroseconds':641,'PartsLockWaitMicroseconds':1,'LogTrace':4,'LoggerElapsedNanoseconds':220400} │
  19. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.967187 │         232 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_12_12_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_12_12_0/ │  100000 │       3195431 │ []          │            8174509 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197572,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':18550,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11368741,'MergeTreeDataWriterCompressedBytes':3195431,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':10112,'MergeTreeDataWriterMergingBlocksMicroseconds':2,'ContextLock':12,'PartsLockHoldMicroseconds':172,'LogTrace':4,'LoggerElapsedNanoseconds':158200} │
  20. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.966149 │         231 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_11_11_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_11_11_0/ │  100000 │       3194702 │ []          │            8180089 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196841,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':17496,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11374321,'MergeTreeDataWriterCompressedBytes':3194702,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9538,'MergeTreeDataWriterMergingBlocksMicroseconds':6,'ContextLock':12,'PartsLockHoldMicroseconds':114,'LogTrace':4,'LoggerElapsedNanoseconds':507500} │
1038. │ 19294dc11be6 │ d67284a5-399b-4d10-99c2-018bb4a1b07d │ NewPart    │ NotAMerge    │ Undecided       │ 2025-09-09 │ 2025-09-09 13:15:58 │ 2025-09-09 13:15:58.629685 │         244 │ learn_db │ mart_student_lesson │ fcc5fdee-d5a8-49b9-a1f0-a78a0c27e00e │ all_1_1_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/fcc/fcc5fdee-d5a8-49b9-a1f0-a78a0c27e00e/all_1_1_0/   │ 1111953 │      42881158 │ []          │           90081915 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':40,'WriteBufferFromFileDescriptorWrite':72,'WriteBufferFromFileDescriptorWriteBytes':42883334,'IOBufferAllocs':145,'IOBufferAllocBytes':37077935,'DiskWriteElapsedMicroseconds':24680,'MergeTreeDataWriterRows':1111953,'MergeTreeDataWriterUncompressedBytes':125608515,'MergeTreeDataWriterCompressedBytes':42881158,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterMergingBlocksMicroseconds':1,'ContextLock':10,'ContextLockWaitMicroseconds':1,'PartsLockHoldMicroseconds':78,'QueryProfilerRuns':1,'LogTrace':4,'LoggerElapsedNanoseconds':123300} │
      └─hostname─────┴─query_id─────────────────────────────┴─event_type─┴─merge_reason─┴─merge_algorithm─┴─event_date─┴──────────event_time─┴────event_time_microseconds─┴─duration_ms─┴─database─┴─table───────────────┴─table_uuid───────────────────────────┴─part_name───┴─partition_id─┴─partition─┴─part_type─┴─disk_name─┴─path_on_disk────────────────────────────────────────────────────────────────────┴────rows─┴─size_in_bytes─┴─merged_from─┴─bytes_uncompressed─┴─read_rows─┴─read_bytes─┴─peak_memory_usage─┴─error─┴─exception─┴─ProfileEvents──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
Showed 1000 out of 1038 rows.

1038 rows in set. Elapsed: 0.008 sec. Processed 5.25 thousand rows, 2.10 MB (626.96 thousand rows/s., 250.74 MB/s.)
Peak memory usage: 726.97 KiB.
```

### 6. Пересоздадим буферную таблицу, указав что нам нужно 8 потоков

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_buffer; 
CREATE TABLE learn_db.mart_student_lesson_buffer AS learn_db.mart_student_lesson ENGINE = Buffer(learn_db, mart_student_lesson, 8, 1, 10, 50000, 1000000, 10000000, 100000000)
```

### 6. Запускаем 1000 вставок по 1000 строк в один поток в буферную таблицу параллельно в 8 потоков

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
" --iterations=1000 --concurrency=8
```

Время вставки последней порции данных
```text
Queries executed: 1000.

localhost:9000, queries: 1000, QPS: 577.740, RPS: 577740.202, MiB/s: 4.408, result RPS: 0.000, result MiB/s: 0.000.

0%              0.006 sec.
10%             0.009 sec.
20%             0.009 sec.
30%             0.010 sec.
40%             0.010 sec.
50%             0.011 sec.
60%             0.011 sec.
70%             0.012 sec.
80%             0.013 sec.
90%             0.015 sec.
95%             0.016 sec.
99%             0.029 sec.
99.9%           0.105 sec.
99.99%          0.105 sec.
```

### 7. Смотрим, какие части данных сформировались

```sql
SELECT * FROM system.part_log where table='mart_student_lesson' and event_type = 1 ORDER BY event_time DESC;
```

Результат
```text
Query id: fec76c1c-134f-4143-ba4b-33af12ee2a4c

      ┌─hostname─────┬─query_id─────────────────────────────┬─event_type─┬─merge_reason─┬─merge_algorithm─┬─event_date─┬──────────event_time─┬────event_time_microseconds─┬─duration_ms─┬─database─┬─table───────────────┬─table_uuid───────────────────────────┬─part_name───┬─partition_id─┬─partition─┬─part_type─┬─disk_name─┬─path_on_disk────────────────────────────────────────────────────────────────────┬────rows─┬─size_in_bytes─┬─merged_from─┬─bytes_uncompressed─┬─read_rows─┬─read_bytes─┬─peak_memory_usage─┬─error─┬─exception─┬─ProfileEvents──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
   1. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.228517 │         114 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_21_21_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_21_21_0/ │  100000 │       3195069 │ []          │            8174053 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197208,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':10159,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11368285,'MergeTreeDataWriterCompressedBytes':3195069,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7482,'MergeTreeDataWriterMergingBlocksMicroseconds':4,'ContextLock':12,'PartsLockHoldMicroseconds':212,'LogTrace':4,'LoggerElapsedNanoseconds':215700} │
   2. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.232240 │         118 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_22_22_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_22_22_0/ │  100000 │       3196931 │ []          │            8173916 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3199070,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':13077,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11368148,'MergeTreeDataWriterCompressedBytes':3196931,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7053,'MergeTreeDataWriterMergingBlocksMicroseconds':5,'ContextLock':12,'PartsLockHoldMicroseconds':120,'LogTrace':4,'LoggerElapsedNanoseconds':160300} │
   3. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.234616 │         121 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_23_23_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_23_23_0/ │  100000 │       3193896 │ []          │            8172228 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196036,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':11688,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11366460,'MergeTreeDataWriterCompressedBytes':3193896,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':6972,'MergeTreeDataWriterMergingBlocksMicroseconds':6,'ContextLock':12,'PartsLockHoldMicroseconds':109,'LogTrace':4,'LoggerElapsedNanoseconds':121700} │
   4. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.235888 │         122 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_24_24_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_24_24_0/ │  100000 │       3194687 │ []          │            8177659 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196826,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':11425,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11371891,'MergeTreeDataWriterCompressedBytes':3194687,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':5686,'MergeTreeDataWriterMergingBlocksMicroseconds':27,'ContextLock':12,'PartsLockHoldMicroseconds':93,'LogTrace':4,'LoggerElapsedNanoseconds':349300} │
   5. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.236364 │         122 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_25_25_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_25_25_0/ │  100000 │       3196960 │ []          │            8175417 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3199097,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':12495,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11369649,'MergeTreeDataWriterCompressedBytes':3196960,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':6948,'MergeTreeDataWriterMergingBlocksMicroseconds':5,'ContextLock':12,'PartsLockHoldMicroseconds':149,'LogTrace':4,'LoggerElapsedNanoseconds':124200} │
   6. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.237169 │         123 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_26_26_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_26_26_0/ │  100000 │       3193785 │ []          │            8176642 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3195924,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':11079,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11370874,'MergeTreeDataWriterCompressedBytes':3193785,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':6851,'MergeTreeDataWriterMergingBlocksMicroseconds':16,'ContextLock':12,'PartsLockHoldMicroseconds':76,'LogTrace':4,'LoggerElapsedNanoseconds':163800} │
   7. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.238095 │         124 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_27_27_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_27_27_0/ │  100000 │       3193979 │ []          │            8174987 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196119,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':11451,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11369219,'MergeTreeDataWriterCompressedBytes':3193979,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7245,'MergeTreeDataWriterMergingBlocksMicroseconds':3,'ContextLock':12,'PartsLockHoldMicroseconds':97,'LogTrace':4,'LoggerElapsedNanoseconds':93900} │
   8. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.239236 │         125 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_28_28_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_28_28_0/ │  100000 │       3196029 │ []          │            8176941 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3198168,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':10489,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11371173,'MergeTreeDataWriterCompressedBytes':3196029,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7090,'MergeTreeDataWriterMergingBlocksMicroseconds':5,'ContextLock':12,'PartsLockHoldMicroseconds':697,'LogTrace':4,'LoggerElapsedNanoseconds':164300} │
   9. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.239608 │         126 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_29_29_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_29_29_0/ │  100000 │       3194428 │ []          │            8177196 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196568,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':12009,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11371428,'MergeTreeDataWriterCompressedBytes':3194428,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7134,'MergeTreeDataWriterMergingBlocksMicroseconds':4,'ContextLock':12,'PartsLockHoldMicroseconds':90,'LogTrace':4,'LoggerElapsedNanoseconds':95700} │
  10. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:26:02 │ 2025-10-20 12:26:02.241296 │         127 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_30_30_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_30_30_0/ │  100000 │       3195435 │ []          │            8172512 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197574,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':9136,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11366744,'MergeTreeDataWriterCompressedBytes':3195435,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7473,'MergeTreeDataWriterMergingBlocksMicroseconds':6,'ContextLock':12,'PartsLockHoldMicroseconds':103,'LogTrace':4,'LoggerElapsedNanoseconds':140800} │
  11. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:18:38 │ 2025-10-20 12:18:38.787934 │         143 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_20_20_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_20_20_0/ │  200000 │       6383558 │ []          │           16351066 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':42,'WriteBufferFromFileDescriptorWriteBytes':6385708,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':5724,'MergeTreeDataWriterRows':200000,'MergeTreeDataWriterUncompressedBytes':22740354,'MergeTreeDataWriterCompressedBytes':6383558,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7327,'MergeTreeDataWriterMergingBlocksMicroseconds':4,'ContextLock':12,'PartsLockHoldMicroseconds':73,'LogTrace':4,'LoggerElapsedNanoseconds':97200} │
  12. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:18:38 │ 2025-10-20 12:18:38.787121 │         143 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_19_19_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_19_19_0/ │  200000 │       6383556 │ []          │           16350154 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':42,'WriteBufferFromFileDescriptorWriteBytes':6385706,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':4684,'MergeTreeDataWriterRows':200000,'MergeTreeDataWriterUncompressedBytes':22739442,'MergeTreeDataWriterCompressedBytes':6383556,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9282,'MergeTreeDataWriterMergingBlocksMicroseconds':3,'ContextLock':12,'PartsLockHoldMicroseconds':67,'LogTrace':4,'LoggerElapsedNanoseconds':74800} │
  13. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:18:38 │ 2025-10-20 12:18:38.786507 │         142 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_18_18_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_18_18_0/ │  200000 │       6383060 │ []          │           16355063 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':42,'WriteBufferFromFileDescriptorWriteBytes':6385211,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':4592,'MergeTreeDataWriterRows':200000,'MergeTreeDataWriterUncompressedBytes':22744351,'MergeTreeDataWriterCompressedBytes':6383060,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':8979,'MergeTreeDataWriterMergingBlocksMicroseconds':2,'ContextLock':12,'PartsLockHoldMicroseconds':99,'LogTrace':4,'LoggerElapsedNanoseconds':78200} │
  14. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:18:38 │ 2025-10-20 12:18:38.786247 │         142 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_17_17_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_17_17_0/ │  200000 │       6382904 │ []          │           16351166 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':42,'WriteBufferFromFileDescriptorWriteBytes':6385052,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':5650,'MergeTreeDataWriterRows':200000,'MergeTreeDataWriterUncompressedBytes':22740454,'MergeTreeDataWriterCompressedBytes':6382904,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7232,'MergeTreeDataWriterMergingBlocksMicroseconds':8,'ContextLock':12,'PartsLockHoldMicroseconds':116,'LogTrace':4,'LoggerElapsedNanoseconds':188600} │
  15. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 12:18:38 │ 2025-10-20 12:18:38.783848 │         139 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_16_16_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_16_16_0/ │  200000 │       6381901 │ []          │           16351636 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':42,'WriteBufferFromFileDescriptorWriteBytes':6384053,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':5439,'MergeTreeDataWriterRows':200000,'MergeTreeDataWriterUncompressedBytes':22740924,'MergeTreeDataWriterCompressedBytes':6381901,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':7385,'MergeTreeDataWriterMergingBlocksMicroseconds':3,'ContextLock':12,'PartsLockHoldMicroseconds':134,'LogTrace':4,'LoggerElapsedNanoseconds':177600} │
  16. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.975364 │         240 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_15_15_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_15_15_0/ │  100000 │       3194837 │ []          │            8176116 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196979,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':10917,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11370348,'MergeTreeDataWriterCompressedBytes':3194837,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9912,'MergeTreeDataWriterMergingBlocksMicroseconds':2,'ContextLock':12,'PartsLockHoldMicroseconds':156,'LogTrace':4,'LoggerElapsedNanoseconds':314800} │
  17. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.973292 │         238 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_14_14_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_14_14_0/ │  100000 │       3195693 │ []          │            8175689 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197832,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':20567,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11369921,'MergeTreeDataWriterCompressedBytes':3195693,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9562,'MergeTreeDataWriterMergingBlocksMicroseconds':5,'ContextLock':12,'PartsLockHoldMicroseconds':169,'LogTrace':4,'LoggerElapsedNanoseconds':465200} │
  18. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.971251 │         235 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_13_13_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_13_13_0/ │  100000 │       3195268 │ []          │            8171520 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197404,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':17372,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11365752,'MergeTreeDataWriterCompressedBytes':3195268,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9863,'MergeTreeDataWriterMergingBlocksMicroseconds':4,'ContextLock':12,'PartsLockHoldMicroseconds':641,'PartsLockWaitMicroseconds':1,'LogTrace':4,'LoggerElapsedNanoseconds':220400} │
  19. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.967187 │         232 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_12_12_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_12_12_0/ │  100000 │       3195431 │ []          │            8174509 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3197572,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':18550,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11368741,'MergeTreeDataWriterCompressedBytes':3195431,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':10112,'MergeTreeDataWriterMergingBlocksMicroseconds':2,'ContextLock':12,'PartsLockHoldMicroseconds':172,'LogTrace':4,'LoggerElapsedNanoseconds':158200} │
  20. │ 19294dc11be6 │                                      │ NewPart    │ NotAMerge    │ Undecided       │ 2025-10-20 │ 2025-10-20 11:57:17 │ 2025-10-20 11:57:17.966149 │         231 │ learn_db │ mart_student_lesson │ 41d4607f-3ab3-4431-bd45-47b4c02d09d9 │ all_11_11_0 │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/41d/41d4607f-3ab3-4431-bd45-47b4c02d09d9/all_11_11_0/ │  100000 │       3194702 │ []          │            8180089 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':41,'WriteBufferFromFileDescriptorWrite':41,'WriteBufferFromFileDescriptorWriteBytes':3196841,'IOBufferAllocs':149,'IOBufferAllocBytes':38196395,'DiskWriteElapsedMicroseconds':17496,'MergeTreeDataWriterRows':100000,'MergeTreeDataWriterUncompressedBytes':11374321,'MergeTreeDataWriterCompressedBytes':3194702,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterSortingBlocksMicroseconds':9538,'MergeTreeDataWriterMergingBlocksMicroseconds':6,'ContextLock':12,'PartsLockHoldMicroseconds':114,'LogTrace':4,'LoggerElapsedNanoseconds':507500} │
1038. │ 19294dc11be6 │ d67284a5-399b-4d10-99c2-018bb4a1b07d │ NewPart    │ NotAMerge    │ Undecided       │ 2025-09-09 │ 2025-09-09 13:15:58 │ 2025-09-09 13:15:58.629685 │         244 │ learn_db │ mart_student_lesson │ fcc5fdee-d5a8-49b9-a1f0-a78a0c27e00e │ all_1_1_0   │ all          │ tuple()   │ Wide      │ default   │ /var/lib/clickhouse/store/fcc/fcc5fdee-d5a8-49b9-a1f0-a78a0c27e00e/all_1_1_0/   │ 1111953 │      42881158 │ []          │           90081915 │         0 │          0 │                 0 │     0 │           │ {'FileOpen':40,'WriteBufferFromFileDescriptorWrite':72,'WriteBufferFromFileDescriptorWriteBytes':42883334,'IOBufferAllocs':145,'IOBufferAllocBytes':37077935,'DiskWriteElapsedMicroseconds':24680,'MergeTreeDataWriterRows':1111953,'MergeTreeDataWriterUncompressedBytes':125608515,'MergeTreeDataWriterCompressedBytes':42881158,'MergeTreeDataWriterBlocks':1,'MergeTreeDataWriterMergingBlocksMicroseconds':1,'ContextLock':10,'ContextLockWaitMicroseconds':1,'PartsLockHoldMicroseconds':78,'QueryProfilerRuns':1,'LogTrace':4,'LoggerElapsedNanoseconds':123300} │
      └─hostname─────┴─query_id─────────────────────────────┴─event_type─┴─merge_reason─┴─merge_algorithm─┴─event_date─┴──────────event_time─┴────event_time_microseconds─┴─duration_ms─┴─database─┴─table───────────────┴─table_uuid───────────────────────────┴─part_name───┴─partition_id─┴─partition─┴─part_type─┴─disk_name─┴─path_on_disk────────────────────────────────────────────────────────────────────┴────rows─┴─size_in_bytes─┴─merged_from─┴─bytes_uncompressed─┴─read_rows─┴─read_bytes─┴─peak_memory_usage─┴─error─┴─exception─┴─ProfileEvents──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
Showed 1000 out of 1038 rows.

1038 rows in set. Elapsed: 0.008 sec. Processed 5.25 thousand rows, 2.10 MB (626.96 thousand rows/s., 250.74 MB/s.)
Peak memory usage: 726.97 KiB.
```

**Вывод:** вариант вставки в 16 потоков дал лучший результат. Скорость: 6.054 MiB/s - почти 2x ускорение. В part_log видно создание как партов по 100K, так и по 200K строк.
