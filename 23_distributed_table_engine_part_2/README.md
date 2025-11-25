# План выполнения запроса к распределенной таблице

### 1. Останавливаем кластер и затем в конфигурационных файлах серверов ClickHouse

```text
fs/volumes/clickhouse-01/etc/clickhouse-server/config.d/config.xml fs/volumes/clickhouse-02/etc/clickhouse-server/config.d/config.xml
fs/volumes/clickhouse-03/etc/clickhouse-server/config.d/config.xml fs/volumes/clickhouse-04/etc/clickhouse-server/config.d/config.xml
```

Вносим правки
```xml
<clickhouse replace="true">
...
    <remote_servers>
        <cluster_2S_2R>
            <shard>
                <internal_replication>true</internal_replication>
				<weight>1</weight>
                <replica>
                    <host>clickhouse-01</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>clickhouse-03</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <internal_replication>true</internal_replication>
				<weight>1</weight>
                <replica>
                    <host>clickhouse-02</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>clickhouse-04</host>
                    <port>9000</port>
                </replica>
            </shard>
        </cluster_2S_2R>
    </remote_servers>
...
</clickhouse>
```

### 2. Пересоздаем таблицу с оценками на кластере

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_mergetree ON CLUSTER cluster_2S_2R; 

CREATE TABLE learn_db.mart_student_lesson_mergetree ON CLUSTER cluster_2S_2R
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
) ENGINE = MergeTree();
```

### 3. Создаем распределенную таблицу с оценками

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_mergetree_distributed ON CLUSTER cluster_2S_2R; 
CREATE TABLE IF NOT EXISTS learn_db.mart_student_lesson_mergetree_distributed ON CLUSTER cluster_2S_2R
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
	`mark` Nullable(UInt8)
)
ENGINE = Distributed('cluster_2S_2R', 'learn_db', 'mart_student_lesson_mergetree', person_id_int);
```

### 4. Вставляем данные в распределенную таблицу с оценками

```sql
INSERT INTO learn_db.mart_student_lesson_mergetree_distributed
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

### 5. Проверяем количество строк в распределенной таблице и в шарде 1

```sql
SELECT COUNT(*) FROM learn_db.mart_student_lesson_mergetree_distributed;
```

Результат
```text
COUNT() |
--------+
10000000|
```

```sql
SELECT COUNT(*) FROM learn_db.mart_student_lesson_mergetree;
```

Результат - шард 1 реплика 1
```text
COUNT()|
-------+
4997698|
```

Результат - шард 1 реплика 2
```text
COUNT()|
-------+
5002302|
```

Результат - шард 2 реплика 1
```text
COUNT()|
-------+
4997698|
```

Результат - шард 2 реплика 2
```text
COUNT()|
-------+
5002302|
```

ClickHouse записал данные напрямую на каждую реплику каждого шарда.

### 6. Считаем количество уникальных учителей в распределенной таблице

```sql
SELECT 
	uniq(teacher_id)
FROM 
	learn_db.mart_student_lesson_mergetree_distributed;
```

Результат
```text
uniq(teacher_id)|
----------------+
            3861|
```

### 7. Смотрим информацию в system.query_log по запросу к распределенной таблице на шарде 1 реплике 1

```sql
SELECT 
	*
FROM 	
	system.query_log
ORDER BY
	event_time DESC;
```

Результат
```text
hostname     |type                    |event_date|event_time         |event_time_microseconds      |query_start_time   |query_start_time_microseconds|query_duration_ms|read_rows|read_bytes|written_rows|written_bytes|result_rows|result_bytes|memory_usage|current_database|query                                                                                                                                                                                                                                                          |formatted_query|normalized_query_hash|query_kind|databases                    |tables                                                                                                                 |columns                                                                                                                                                                                                                                                        |partitions                                   |projections|views|exception_code|exception                                                                                                                                                                                                                                                      |stack_trace                                                                                                                                                                                                                                                    |is_initial_query|user   |query_id                            |address    |port |initial_user|initial_query_id                    |initial_address|initial_port|initial_query_start_time|initial_query_start_time_microseconds|interface|is_secure|os_user|client_hostname|client_name      |client_revision|client_version_major|client_version_minor|client_version_patch|script_query_number|script_line_number|http_method|http_user_agent                                                                                              |http_referer|forwarded_for|quota_key|distributed_depth|revision|log_comment|thread_ids                                                                                                                  |peak_threads_usage|ProfileEvents                                                                                                                                                                                                                                                  |Settings                                                                                                                                                                                  |used_aggregate_functions    |used_aggregate_function_combinators|used_database_engines|used_data_type_families                                                                                                 |used_dictionaries|used_formats                  |used_functions                                                                                                                                                                                                                       |used_storages          |used_table_functions|used_executable_user_defined_functions|used_sql_user_defined_functions|used_row_policies|used_privileges                                                                                                                                                                                                                                                |missing_privileges|transaction_id                                    |query_cache_usage|asynchronous_read_counters|
-------------+------------------------+----------+-------------------+-----------------------------+-------------------+-----------------------------+-----------------+---------+----------+------------+-------------+-----------+------------+------------+----------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+---------------------+----------+-----------------------------+-----------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------+-----------+-----+--------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------+-------+------------------------------------+-----------+-----+------------+------------------------------------+---------------+------------+------------------------+-------------------------------------+---------+---------+-------+---------------+-----------------+---------------+--------------------+--------------------+--------------------+-------------------+------------------+-----------+-------------------------------------------------------------------------------------------------------------+------------+-------------+---------+-----------------+--------+-----------+----------------------------------------------------------------------------------------------------------------------------+------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------------+-----------------------------------+---------------------+------------------------------------------------------------------------------------------------------------------------+-----------------+------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------+--------------------+--------------------------------------+-------------------------------+-----------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------------+--------------------------------------------------+-----------------+--------------------------+
clickhouse-01|QueryStart              |2025-11-25|2025-11-25 16:41:27|2025-11-25 16:41:27.875156000|2025-11-25 16:41:27|2025-11-25 16:41:27.875156000|                0|        0|         0|           0|            0|          0|           0|           0|default         |SELECT ¶ uniq(teacher_id)¶FROM ¶ mart_student_lesson_mergetree_distributed                                                                                                                                                                                     |               | 17279150729616781600|Select    |['default']                  |['default.mart_student_lesson_mergetree','default.mart_student_lesson_mergetree_distributed']                          |['default.mart_student_lesson_mergetree.teacher_id','default.mart_student_lesson_mergetree_distributed.teacher_id']                                                                                                                                            |['default.mart_student_lesson_mergetree.all']|[]         |[]   |             0|                                                                                                                                                                                                                                                               |                                                                                                                                                                                                                                                               |               1|default|92ca4662-47ef-4cda-9b92-7c61b675fc33|/172.23.0.1|46952|default     |92ca4662-47ef-4cda-9b92-7c61b675fc33|/172.23.0.1    |       46952|     2025-11-25 16:41:27|        2025-11-25 16:41:27.875156000|        2|        0|       |               |                 |              0|                   0|                   0|                   0|                  0|                 0|          2|DBeaver 25.2.5 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                0|   54504|           |[]                                                                                                                          |                 0|{}                                                                                                                                                                                                                                                             |{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_result_rows=200, result_overflow_mode=break, max_memory_usage=10000000000, parallel_replicas_for_cluster_engines=0}|[]                          |[]                                 |[]                   |[]                                                                                                                      |[]               |[]                            |[]                                                                                                                                                                                                                                   |[]                     |[]                  |[]                                    |[]                             |[]               |[]                                                                                                                                                                                                                                                             |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|Unknown          |{}                        |
clickhouse-01|QueryFinish             |2025-11-25|2025-11-25 16:41:27|2025-11-25 16:41:27.910840000|2025-11-25 16:41:27|2025-11-25 16:41:27.875156000|               36| 10000000|  40000000|           0|            0|          1|         136|     6008008|default         |SELECT ¶ uniq(teacher_id)¶FROM ¶ mart_student_lesson_mergetree_distributed                                                                                                                                                                                     |               | 17279150729616781600|Select    |['default']                  |['default.mart_student_lesson_mergetree','default.mart_student_lesson_mergetree_distributed']                          |['default.mart_student_lesson_mergetree.teacher_id','default.mart_student_lesson_mergetree_distributed.teacher_id']                                                                                                                                            |['default.mart_student_lesson_mergetree.all']|[]         |[]   |             0|                                                                                                                                                                                                                                                               |                                                                                                                                                                                                                                                               |               1|default|92ca4662-47ef-4cda-9b92-7c61b675fc33|/172.23.0.1|46952|default     |92ca4662-47ef-4cda-9b92-7c61b675fc33|/172.23.0.1    |       46952|     2025-11-25 16:41:27|        2025-11-25 16:41:27.875156000|        2|        0|       |               |                 |              0|                   0|                   0|                   0|                  0|                 0|          2|DBeaver 25.2.5 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                0|   54504|           |[741,754,726,771,757,774,728,722,746,733,779,773,720,775,85,778,761,783,782,759,781,770]                                    |                18|{Query=1, SelectQuery=1, InitialQuery=1, QueriesWithSubqueries=6, SelectQueriesWithSubqueries=6, FileOpen=8, ReadBufferFromFileDescriptorReadBytes=12619071, ReadCompressedBytes=11990353, CompressedReadBufferBlocks=316, CompressedReadBufferBytes=20283315, |{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_result_rows=200, result_overflow_mode=break, max_memory_usage=10000000000, parallel_replicas_for_cluster_engines=0}|['min','uniq','max','count']|[]                                 |[]                   |['AggregateFunction(uniq, Int32)','Int32','String','DateTime','UInt64','Enum8('increment' = 1, 'gauge' = 2)','Int64']   |[]               |['RowBinaryWithNamesAndTypes']|[]                                                                                                                                                                                                                                   |[]                     |[]                  |[]                                    |[]                             |[]               |['SELECT(teacher_id) ON default.mart_student_lesson_mergetree_distributed','SELECT(teacher_id) ON default.mart_student_lesson_mergetree']                                                                                                                      |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|None             |{}                        |
clickhouse-01|ExceptionBeforeStart    |2025-11-25|2025-11-25 16:41:20|2025-11-25 16:41:20.168605000|2025-11-25 16:41:20|2025-11-25 16:41:20.168138000|                0|        0|         0|           0|            0|          0|           0|     2098174|default         |SELECT ¶ uniq(teacher_id)¶FROM ¶ learn_db.mart_student_lesson_mergetree_distributed                                                                                                                                                                            |               | 12552188881140405286|Select    |[]                           |[]                                                                                                                     |[]                                                                                                                                                                                                                                                             |[]                                           |[]         |[]   |            81|Code: 81. DB::Exception: Database learn_db does not exist. (UNKNOWN_DATABASE) (version 25.9.3.48 (official build))                                                                                                                                             |0. DB::Exception::Exception(DB::Exception::MessageMasked&&, int, bool) @ 0x00000000137a855f¶1. DB::Exception::Exception(String&&, int, String, bool) @ 0x000000000cae7e8e¶2. DB::Exception::Exception(PreformattedMessage&&, int) @ 0x000000000cae7940¶3. DB::E|               1|default|f118962c-7703-43d3-8f08-e10d14846228|/172.23.0.1|46952|default     |f118962c-7703-43d3-8f08-e10d14846228|/172.23.0.1    |       46952|     2025-11-25 16:41:20|        2025-11-25 16:41:20.168138000|        2|        0|       |               |                 |              0|                   0|                   0|                   0|                  0|                 0|          2|DBeaver 25.2.5 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                0|   54504|           |[85]                                                                                                                        |                 1|{Query=1, SelectQuery=1, InitialQuery=1, FailedQuery=1, FailedSelectQuery=1, IOBufferAllocs=2, IOBufferAllocBytes=2097278, ContextLock=10, RealTimeMicroseconds=643, UserTimeMicroseconds=165, SystemTimeMicroseconds=41, OSCPUWaitMicroseconds=8, OSCPUVirtual|{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_result_rows=200, result_overflow_mode=break, max_memory_usage=10000000000, parallel_replicas_for_cluster_engines=0}|[]                          |[]                                 |[]                   |[]                                                                                                                      |[]               |[]                            |[]                                                                                                                                                                                                                                   |[]                     |[]                  |[]                                    |[]                             |[]               |[]                                                                                                                                                                                                                                                             |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|Unknown          |{}                        |
```

Видим что запрос обработал 10 млн строк. Хотя это суммарное количество строк на всем шарде. Но он сначала сделал на реплике 1 и получил предагрегированный результат со второй реплики.

Запомним показатели:
read_rows = 10000000
query_id = 92ca4662-47ef-4cda-9b92-7c61b675fc33
event_time = 2025-11-25 16:41:27

Выполним тот же самый запрос, но на реплике 2.

Результат
```text
hostname     |type                |event_date|event_time         |event_time_microseconds      |query_start_time   |query_start_time_microseconds|query_duration_ms|read_rows|read_bytes|written_rows|written_bytes|result_rows|result_bytes|memory_usage|current_database|query                                                                                                                                                                                                                                                          |formatted_query|normalized_query_hash|query_kind|databases          |tables                                               |columns                                                                                                                                                                                                                                                        |partitions                                   |projections|views|exception_code|exception                                                                                                                                                                                                                                                      |stack_trace                                                                                                                                                                                                                                                    |is_initial_query|user   |query_id                            |address    |port |initial_user|initial_query_id                    |initial_address|initial_port|initial_query_start_time|initial_query_start_time_microseconds|interface|is_secure|os_user|client_hostname|client_name      |client_revision|client_version_major|client_version_minor|client_version_patch|script_query_number|script_line_number|http_method|http_user_agent                                                                                              |http_referer|forwarded_for|quota_key|distributed_depth|revision|log_comment|thread_ids                                                              |peak_threads_usage|ProfileEvents                                                                                                                                                                                                                                                  |Settings                                                                                                                                                                                  |used_aggregate_functions    |used_aggregate_function_combinators|used_database_engines|used_data_type_families                                                      |used_dictionaries|used_formats                  |used_functions                                                                                           |used_storages          |used_table_functions|used_executable_user_defined_functions|used_sql_user_defined_functions|used_row_policies|used_privileges                                                                                                                                                                                                                                                |missing_privileges|transaction_id                                    |query_cache_usage|asynchronous_read_counters|
-------------+--------------------+----------+-------------------+-----------------------------+-------------------+-----------------------------+-----------------+---------+----------+------------+-------------+-----------+------------+------------+----------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+---------------------+----------+-------------------+-----------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------+-----------+-----+--------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------+-------+------------------------------------+-----------+-----+------------+------------------------------------+---------------+------------+------------------------+-------------------------------------+---------+---------+-------+---------------+-----------------+---------------+--------------------+--------------------+--------------------+-------------------+------------------+-----------+-------------------------------------------------------------------------------------------------------------+------------+-------------+---------+-----------------+--------+-----------+------------------------------------------------------------------------+------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------------+-----------------------------------+---------------------+-----------------------------------------------------------------------------+-----------------+------------------------------+---------------------------------------------------------------------------------------------------------+-----------------------+--------------------+--------------------------------------+-------------------------------+-----------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------------+--------------------------------------------------+-----------------+--------------------------+
clickhouse-02|QueryStart          |2025-11-25|2025-11-25 16:41:27|2025-11-25 16:41:27.885950000|2025-11-25 16:41:27|2025-11-25 16:41:27.885950000|                0|        0|         0|           0|            0|          0|           0|           0|default         |SELECT uniq(`__table1`.`teacher_id`) AS `uniq(teacher_id)` FROM `default`.`mart_student_lesson_mergetree` AS `__table1`                                                                                                                                        |               |  2629711370036620604|Select    |['default']        |['default.mart_student_lesson_mergetree']            |['default.mart_student_lesson_mergetree.teacher_id']                                                                                                                                                                                                           |['default.mart_student_lesson_mergetree.all']|[]         |[]   |             0|                                                                                                                                                                                                                                                               |                                                                                                                                                                                                                                                               |               0|default|1f43ada3-dd6b-4021-984c-22dd3c6b2387|/172.23.0.6|51232|default     |92ca4662-47ef-4cda-9b92-7c61b675fc33|/172.23.0.1    |       46952|     2025-11-25 16:41:27|        2025-11-25 16:41:27.875156000|        2|        0|       |               |ClickHouse server|          54480|                  25|                   9|                   0|                  0|                 0|          2|DBeaver 25.2.5 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                1|   54504|           |[]                                                                      |                 0|{}                                                                                                                                                                                                                                                             |{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_result_rows=200, result_overflow_mode=break, max_memory_usage=10000000000, parallel_replicas_for_cluster_engines=0}|[]                          |[]                                 |[]                   |[]                                                                           |[]               |[]                            |[]                                                                                                       |[]                     |[]                  |[]                                    |[]                             |[]               |[]                                                                                                                                                                                                                                                             |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|Unknown          |{}                        |
clickhouse-02|QueryFinish         |2025-11-25|2025-11-25 16:41:27|2025-11-25 16:41:27.908319000|2025-11-25 16:41:27|2025-11-25 16:41:27.885950000|               23|  5002302|  20009208|           0|            0|          1|         256|      431816|default         |SELECT uniq(`__table1`.`teacher_id`) AS `uniq(teacher_id)` FROM `default`.`mart_student_lesson_mergetree` AS `__table1`                                                                                                                                        |               |  2629711370036620604|Select    |['default']        |['default.mart_student_lesson_mergetree']            |['default.mart_student_lesson_mergetree.teacher_id']                                                                                                                                                                                                           |['default.mart_student_lesson_mergetree.all']|[]         |[]   |             0|                                                                                                                                                                                                                                                               |                                                                                                                                                                                                                                                               |               0|default|1f43ada3-dd6b-4021-984c-22dd3c6b2387|/172.23.0.6|51232|default     |92ca4662-47ef-4cda-9b92-7c61b675fc33|/172.23.0.1    |       46952|     2025-11-25 16:41:27|        2025-11-25 16:41:27.875156000|        2|        0|       |               |ClickHouse server|          54480|                  25|                   9|                   0|                  0|                 0|          2|DBeaver 25.2.5 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                1|   54504|           |[724,759,742,781,794,771,786,789,778,787,761,739,795,790,86]            |                14|{Query=1, SelectQuery=1, QueriesWithSubqueries=1, SelectQueriesWithSubqueries=1, FileOpen=8, ReadBufferFromFileDescriptorReadBytes=12702763, ReadCompressedBytes=12026808, CompressedReadBufferBlocks=317, CompressedReadBufferBytes=20351756, OpenedFileCacheH|{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_result_rows=200, result_overflow_mode=break, max_memory_usage=10000000000, parallel_replicas_for_cluster_engines=0}|['min','uniq','max','count']|[]                                 |[]                   |['UInt32']                                                                   |[]               |[]                            |[]                                                                                                       |[]                     |[]                  |[]                                    |[]                             |[]               |['SELECT(teacher_id) ON default.mart_student_lesson_mergetree']                                                                                                                                                                                                |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|None             |{}                        |
clickhouse-02|QueryStart          |2025-11-25|2025-11-25 16:31:47|2025-11-25 16:31:47.073719000|2025-11-25 16:31:47|2025-11-25 16:31:47.073719000|                0|        0|         0|           0|            0|          0|           0|           0|default         |¶SELECT COUNT(*) FROM mart_student_lesson_mergetree                                                                                                                                                                                                            |               | 12248379267681461444|Select    |[]                 |[]                                                   |[]                                                                                                                                                                                                                                                             |[]                                           |[]         |[]   |             0|                                                                                                                                                                                                                                                               |                                                                                                                                                                                                                                                               |               1|default|c1930b8f-c43b-4382-adce-a93419e38f9b|/172.23.0.1|51710|default     |c1930b8f-c43b-4382-adce-a93419e38f9b|/172.23.0.1    |       51710|     2025-11-25 16:31:47|        2025-11-25 16:31:47.073719000|        2|        0|       |               |                 |              0|                   0|                   0|                   0|                  0|                 0|          2|DBeaver 25.2.5 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                0|   54504|           |[]                                                                      |                 0|{}                                                                                                                                                                                                                                                             |{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_result_rows=200, result_overflow_mode=break, max_memory_usage=10000000000, parallel_replicas_for_cluster_engines=0}|[]                          |[]                                 |[]                   |[]                                                                           |[]               |[]                            |[]                                                                                                       |[]                     |[]                  |[]                                    |[]                             |[]               |[]                                                                                                                                                                                                                                                             |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|Unknown          |{}                        |
clickhouse-02|QueryFinish         |2025-11-25|2025-11-25 16:31:47|2025-11-25 16:31:47.075440000|2025-11-25 16:31:47|2025-11-25 16:31:47.073719000|                2|        1|        16|           0|            0|          1|         136|     5344227|default         |¶SELECT COUNT(*) FROM mart_student_lesson_mergetree                                                                                                                                                                                                            |               | 12248379267681461444|Select    |[]                 |[]                                                   |[]                                                                                                                                                                                                                                                             |[]                                           |[]         |[]   |             0|                                                                                                                                                                                                                                                               |                                                                                                                                                                                                                                                               |               1|default|c1930b8f-c43b-4382-adce-a93419e38f9b|/172.23.0.1|51710|default     |c1930b8f-c43b-4382-adce-a93419e38f9b|/172.23.0.1    |       51710|     2025-11-25 16:31:47|        2025-11-25 16:31:47.073719000|        2|        0|       |               |                 |              0|                   0|                   0|                   0|                  0|                 0|          2|DBeaver 25.2.5 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                0|   54504|           |[753,741,727,793,788,87]                                                |                 6|{Query=1, SelectQuery=1, InitialQuery=1, QueriesWithSubqueries=2, SelectQueriesWithSubqueries=2, IOBufferAllocs=6, IOBufferAllocBytes=6291834, ArenaAllocChunks=2, ArenaAllocBytes=8192, NetworkSendElapsedMicroseconds=207, NetworkSendBytes=827, GlobalThread|{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_result_rows=200, result_overflow_mode=break, max_memory_usage=10000000000, parallel_replicas_for_cluster_engines=0}|['count']                   |[]                                 |[]                   |[]                                                                           |[]               |['RowBinaryWithNamesAndTypes']|[]                                                                                                       |[]                     |[]                  |[]                                    |[]                             |[]               |['SELECT(lesson_year) ON default.mart_student_lesson_mergetree']                                                                                                                                                                                               |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|None             |{}                        |
```

Показатели запроса:
read_rows = 5002302
query = SELECT uniq(`__table1`.`teacher_id`) AS `uniq(teacher_id)` FROM `default`.`mart_student_lesson_mergetree` AS `__table1`
query_id = 1f43ada3-dd6b-4021-984c-22dd3c6b2387
event_time = 2025-11-25 16:41:27

Идентификаторы запросов на разных нодах отличаются. И были обработаны только те строки, которые лежат на этой реплике.

Повторим запрос на шарде 2 реплике 1.

Результат
```text
-------------+--------------------+----------+-------------------+-----------------------------+-------------------+-----------------------------+-----------------+---------+----------+------------+-------------+-----------+------------+------------+----------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+---------------------+----------+-------------------+---------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------+-----------+-----+--------------+------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------+-------+------------------------------------+-----------+-----+------------+------------------------------------+---------------+------------+------------------------+-------------------------------------+---------+---------+-------+---------------+-----------------+---------------+--------------------+--------------------+--------------------+-------------------+------------------+-----------+-------------------------------------------------------------------------------------------------------------+------------+-------------+---------+-----------------+--------+-----------+--------------------------------------------------------------------+------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------------------+-----------------------------------+---------------------+-----------------------------------------------------------------------------+-----------------+------------------------------+---------------------------------------------------------------------------------------------------------+-----------------------+--------------------+--------------------------------------+-------------------------------+-----------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------------+--------------------------------------------------+-----------------+--------------------------+
clickhouse-03|QueryStart          |2025-11-25|2025-11-25 16:32:26|2025-11-25 16:32:26.165812000|2025-11-25 16:32:26|2025-11-25 16:32:26.165812000|                0|        0|         0|           0|            0|          0|           0|           0|default         |SELECT COUNT(*) FROM mart_student_lesson_mergetree                                                                                                                                                                                                             |               | 12248379267681461444|Select    |[]                 |[]                                                       |[]                                                                                                                                                                                                                                                             |[]        |[]         |[]   |             0|                                                                                                                  |                                                                                                                                                                                                                                                               |               1|default|69ae7877-696c-47f0-b488-20081b4def7f|/172.23.0.1|60460|default     |69ae7877-696c-47f0-b488-20081b4def7f|/172.23.0.1    |       60460|     2025-11-25 16:32:26|        2025-11-25 16:32:26.165812000|        2|        0|       |               |                 |              0|                   0|                   0|                   0|                  0|                 0|          2|DBeaver 25.2.5 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                0|   54504|           |[]                                                                  |                 0|{}                                                                                                                                                                                                                                                             |{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_result_rows=200, result_overflow_mode=break, max_memory_usage=10000000000, parallel_replicas_for_cluster_engines=0}|[]                      |[]                                 |[]                   |[]                                                                           |[]               |[]                            |[]                                                                                                       |[]                     |[]                  |[]                                    |[]                             |[]               |[]                                                                                                                                                                                                                                                             |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|Unknown          |{}                        |
clickhouse-03|QueryFinish         |2025-11-25|2025-11-25 16:32:26|2025-11-25 16:32:26.168093000|2025-11-25 16:32:26|2025-11-25 16:32:26.165812000|                2|        1|        16|           0|            0|          1|         136|     5345795|default         |SELECT COUNT(*) FROM mart_student_lesson_mergetree                                                                                                                                                                                                             |               | 12248379267681461444|Select    |[]                 |[]                                                       |[]                                                                                                                                                                                                                                                             |[]        |[]         |[]   |             0|                                                                                                                  |                                                                                                                                                                                                                                                               |               1|default|69ae7877-696c-47f0-b488-20081b4def7f|/172.23.0.1|60460|default     |69ae7877-696c-47f0-b488-20081b4def7f|/172.23.0.1    |       60460|     2025-11-25 16:32:26|        2025-11-25 16:32:26.165812000|        2|        0|       |               |                 |              0|                   0|                   0|                   0|                  0|                 0|          2|DBeaver 25.2.5 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                0|   54504|           |[781,780,747,723,720,81]                                            |                 6|{Query=1, SelectQuery=1, InitialQuery=1, QueriesWithSubqueries=2, SelectQueriesWithSubqueries=2, IOBufferAllocs=6, IOBufferAllocBytes=6291834, ArenaAllocChunks=2, ArenaAllocBytes=8192, NetworkSendElapsedMicroseconds=168, NetworkSendBytes=827, GlobalThread|{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_result_rows=200, result_overflow_mode=break, max_memory_usage=10000000000, parallel_replicas_for_cluster_engines=0}|['count']               |[]                                 |[]                   |[]                                                                           |[]               |['RowBinaryWithNamesAndTypes']|[]                                                                                                       |[]                     |[]                  |[]                                    |[]                             |[]               |['SELECT(lesson_year) ON default.mart_student_lesson_mergetree']                                                                                                                                                                                               |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|None             |{}                        |
clickhouse-03|QueryStart          |2025-11-25|2025-11-25 16:32:25|2025-11-25 16:32:25.073821000|2025-11-25 16:32:25|2025-11-25 16:32:25.073821000|                0|        0|         0|           0|            0|          0|           0|           0|default         |SELECT c1 as NAME, c2 as MAX_LEN, c3 as DEFAULT_VALUE, c4 as DESCRIPTION FROM VALUES (('ApplicationName', 255, '', 'Client application name.'))                                                                                                                |               | 15413437441412256643|Select    |['_table_function']|['_table_function.values']                               |['_table_function.values.c1','_table_function.values.c2','_table_function.values.c3','_table_function.values.c4']                                                                                                                                              |[]        |[]         |[]   |             0|                                                                                                                  |                                                                                                                                                                                                                                                               |               1|default|4a5f7b3d-4eb9-48c2-a30b-d1132f03cdd9|/172.23.0.1|60460|default     |4a5f7b3d-4eb9-48c2-a30b-d1132f03cdd9|/172.23.0.1    |       60460|     2025-11-25 16:32:25|        2025-11-25 16:32:25.073821000|        2|        0|       |               |                 |              0|                   0|                   0|                   0|                  0|                 0|          2|ClickHouse JDBC Driver V2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1    |            |             |         |                0|   54504|           |[]                                                                  |                 0|{}                                                                                                                                                                                                                                                             |{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_memory_usage=10000000000}                                                                                          |[]                      |[]                                 |[]                   |[]                                                                           |[]               |[]                            |[]                                                                                                       |[]                     |[]                  |[]                                    |[]                             |[]               |[]                                                                                                                                                                                                                                                             |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|Unknown          |{}                        |
clickhouse-03|QueryFinish         |2025-11-25|2025-11-25 16:32:25|2025-11-25 16:32:25.075182000|2025-11-25 16:32:25|2025-11-25 16:32:25.073821000|                1|        1|        64|           0|            0|          1|       17023|     5254395|default         |SELECT c1 as NAME, c2 as MAX_LEN, c3 as DEFAULT_VALUE, c4 as DESCRIPTION FROM VALUES (('ApplicationName', 255, '', 'Client application name.'))                                                                                                                |               | 15413437441412256643|Select    |['_table_function']|['_table_function.values']                               |['_table_function.values.c1','_table_function.values.c2','_table_function.values.c3','_table_function.values.c4']                                                                                                                                              |[]        |[]         |[]   |             0|                                                                                                                  |                                                                                                                                                                                                                                                               |               1|default|4a5f7b3d-4eb9-48c2-a30b-d1132f03cdd9|/172.23.0.1|60460|default     |4a5f7b3d-4eb9-48c2-a30b-d1132f03cdd9|/172.23.0.1    |       60460|     2025-11-25 16:32:25|        2025-11-25 16:32:25.073821000|        2|        0|       |               |                 |              0|                   0|                   0|                   0|                  0|                 0|          2|ClickHouse JDBC Driver V2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1    |            |             |         |                0|   54504|           |[717,718,784,759,748,81]                                            |                 6|{Query=1, SelectQuery=1, InitialQuery=1, QueriesWithSubqueries=1, SelectQueriesWithSubqueries=1, IOBufferAllocs=6, IOBufferAllocBytes=6291834, TableFunctionExecute=1, NetworkSendElapsedMicroseconds=92, NetworkSendBytes=905, GlobalThreadPoolJobs=5, LocalTh|{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_memory_usage=10000000000}                                                                                          |[]                      |[]                                 |[]                   |['String','UInt8','Tuple(String, UInt8, String, String)']                    |[]               |['RowBinaryWithNamesAndTypes']|['_CAST']                                                                                                |[]                     |['VALUES']          |[]                                    |[]                             |[]               |[]                                                                                                                                                                                                                                                             |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|None             |{}                        |
```

Повторим запрос на шарде 2 реплике 2.

Результат
```text
hostname     |type                    |event_date|event_time         |event_time_microseconds      |query_start_time   |query_start_time_microseconds|query_duration_ms|read_rows|read_bytes|written_rows|written_bytes|result_rows|result_bytes|memory_usage|current_database|query                                                                                                                                                                                                                                                          |formatted_query|normalized_query_hash|query_kind|databases          |tables                                                   |columns                                                                                                                                                                                                                                                        |partitions|projections|views|exception_code|exception                                                                                                                                                                                                                                       |stack_trace                                                                                                                                                                                                                                                    |is_initial_query|user   |query_id                            |address    |port |initial_user|initial_query_id                    |initial_address|initial_port|initial_query_start_time|initial_query_start_time_microseconds|interface|is_secure|os_user|client_hostname|client_name      |client_revision|client_version_major|client_version_minor|client_version_patch|script_query_number|script_line_number|http_method|http_user_agent                                                                                              |http_referer|forwarded_for|quota_key|distributed_depth|revision|log_comment|thread_ids                                                              |peak_threads_usage|ProfileEvents                                                                                                                                                                                                                                                  |Settings                                                                                                                                                                                  |used_aggregate_functions|used_aggregate_function_combinators|used_database_engines|used_data_type_families                                                                                                 |used_dictionaries|used_formats                  |used_functions                                                                                           |used_storages          |used_table_functions|used_executable_user_defined_functions|used_sql_user_defined_functions|used_row_policies|used_privileges                                                                                                                                                                                                                                                |missing_privileges|transaction_id                                    |query_cache_usage|asynchronous_read_counters|
-------------+------------------------+----------+-------------------+-----------------------------+-------------------+-----------------------------+-----------------+---------+----------+------------+-------------+-----------+------------+------------+----------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+---------------------+----------+-------------------+---------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------+-----------+-----+--------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------+-------+------------------------------------+-----------+-----+------------+------------------------------------+---------------+------------+------------------------+-------------------------------------+---------+---------+-------+---------------+-----------------+---------------+--------------------+--------------------+--------------------+-------------------+------------------+-----------+-------------------------------------------------------------------------------------------------------------+------------+-------------+---------+-----------------+--------+-----------+------------------------------------------------------------------------+------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------------------+-----------------------------------+---------------------+------------------------------------------------------------------------------------------------------------------------+-----------------+------------------------------+---------------------------------------------------------------------------------------------------------+-----------------------+--------------------+--------------------------------------+-------------------------------+-----------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------------+--------------------------------------------------+-----------------+--------------------------+
clickhouse-04|QueryStart              |2025-11-25|2025-11-25 16:41:14|2025-11-25 16:41:14.709311000|2025-11-25 16:41:14|2025-11-25 16:41:14.709311000|                0|        0|         0|           0|            0|          0|           0|           0|default         |SELECT name as TABLE_NAME, engine as TABLE_TYPE, database as TABLE_SCHEM,comment as REMARKS, * FROM system.tables¶WHERE database = 'default' and name='learn_db'                                                                                               |               | 14786696403148937755|Select    |['system']         |['system.tables']                                        |['system.tables.active_on_fly_alter_mutations','system.tables.active_on_fly_data_mutations','system.tables.active_on_fly_metadata_mutations','system.tables.active_parts','system.tables.as_select','system.tables.comment','system.tables.create_table_query',|[]        |[]         |[]   |             0|                                                                                                                                                                                                                                                |                                                                                                                                                                                                                                                               |               1|default|f08cf8e7-de02-44e9-aeaf-a2d47f050745|/172.23.0.1|35008|default     |f08cf8e7-de02-44e9-aeaf-a2d47f050745|/172.23.0.1    |       35008|     2025-11-25 16:41:14|        2025-11-25 16:41:14.709311000|        2|        0|       |               |                 |              0|                   0|                   0|                   0|                  0|                 0|          2|DBeaver 25.2.5 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                0|   54504|           |[]                                                                      |                 0|{}                                                                                                                                                                                                                                                             |{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_memory_usage=10000000000, parallel_replicas_for_cluster_engines=0}                                                 |[]                      |[]                                 |[]                   |[]                                                                                                                      |[]               |[]                            |[]                                                                                                       |[]                     |[]                  |[]                                    |[]                             |[]               |[]                                                                                                                                                                                                                                                             |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|Unknown          |{}                        |
clickhouse-04|QueryFinish             |2025-11-25|2025-11-25 16:41:14|2025-11-25 16:41:14.712413000|2025-11-25 16:41:14|2025-11-25 16:41:14.709311000|                3|        0|         0|           0|            0|          0|           0|     4207036|default         |SELECT name as TABLE_NAME, engine as TABLE_TYPE, database as TABLE_SCHEM,comment as REMARKS, * FROM system.tables¶WHERE database = 'default' and name='learn_db'                                                                                               |               | 14786696403148937755|Select    |['system']         |['system.tables']                                        |['system.tables.active_on_fly_alter_mutations','system.tables.active_on_fly_data_mutations','system.tables.active_on_fly_metadata_mutations','system.tables.active_parts','system.tables.as_select','system.tables.comment','system.tables.create_table_query',|[]        |[]         |[]   |             0|                                                                                                                                                                                                                                                |                                                                                                                                                                                                                                                               |               1|default|f08cf8e7-de02-44e9-aeaf-a2d47f050745|/172.23.0.1|35008|default     |f08cf8e7-de02-44e9-aeaf-a2d47f050745|/172.23.0.1    |       35008|     2025-11-25 16:41:14|        2025-11-25 16:41:14.709311000|        2|        0|       |               |                 |              0|                   0|                   0|                   0|                  0|                 0|          2|DBeaver 25.2.5 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                0|   54504|           |[726,746,749,734,79]                                                    |                 5|{Query=1, SelectQuery=1, InitialQuery=1, QueriesWithSubqueries=1, SelectQueriesWithSubqueries=1, IOBufferAllocs=5, IOBufferAllocBytes=5243195, FunctionExecute=6, NetworkSendElapsedMicroseconds=97, NetworkSendBytes=1394, GlobalThreadPoolJobs=4, LocalThread|{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_memory_usage=10000000000, parallel_replicas_for_cluster_engines=0}                                                 |[]                      |[]                                 |[]                   |[]                                                                                                                      |[]               |['RowBinaryWithNamesAndTypes']|['equals','and','notIn']                                                                                 |[]                     |[]                  |[]                                    |[]                             |[]               |['SELECT(name, engine, database, comment, uuid, is_temporary, data_paths, metadata_path, metadata_modification_time, metadata_version, dependencies_database, dependencies_table, create_table_query, engine_full, as_select, parameterized_view_parameters, pa|[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|None             |{}                        |
clickhouse-04|QueryStart              |2025-11-25|2025-11-25 16:41:14|2025-11-25 16:41:14.715552000|2025-11-25 16:41:14|2025-11-25 16:41:14.715552000|                0|        0|         0|           0|            0|          0|           0|           0|default         |SELECT name as TABLE_NAME, engine as TABLE_TYPE, database as TABLE_SCHEM,comment as REMARKS, * FROM system.tables¶WHERE database = 'default' and name='teacher_id'                                                                                             |               | 14786696403148937755|Select    |['system']         |['system.tables']                                        |['system.tables.active_on_fly_alter_mutations','system.tables.active_on_fly_data_mutations','system.tables.active_on_fly_metadata_mutations','system.tables.active_parts','system.tables.as_select','system.tables.comment','system.tables.create_table_query',|[]        |[]         |[]   |             0|                                                                                                                                                                                                                                                |                                                                                                                                                                                                                                                               |               1|default|62f2ef88-0ed8-41c8-80ed-6ca0267962ea|/172.23.0.1|35008|default     |62f2ef88-0ed8-41c8-80ed-6ca0267962ea|/172.23.0.1    |       35008|     2025-11-25 16:41:14|        2025-11-25 16:41:14.715552000|        2|        0|       |               |                 |              0|                   0|                   0|                   0|                  0|                 0|          2|DBeaver 25.2.5 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                0|   54504|           |[]                                                                      |                 0|{}                                                                                                                                                                                                                                                             |{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_memory_usage=10000000000, parallel_replicas_for_cluster_engines=0}                                                 |[]                      |[]                                 |[]                   |[]                                                                                                                      |[]               |[]                            |[]                                                                                                       |[]                     |[]                  |[]                                    |[]                             |[]               |[]                                                                                                                                                                                                                                                             |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|Unknown          |{}                        |
clickhouse-04|QueryFinish             |2025-11-25|2025-11-25 16:41:14|2025-11-25 16:41:14.718782000|2025-11-25 16:41:14|2025-11-25 16:41:14.715552000|                3|        0|         0|           0|            0|          0|           0|     4206604|default         |SELECT name as TABLE_NAME, engine as TABLE_TYPE, database as TABLE_SCHEM,comment as REMARKS, * FROM system.tables¶WHERE database = 'default' and name='teacher_id'                                                                                             |               | 14786696403148937755|Select    |['system']         |['system.tables']                                        |['system.tables.active_on_fly_alter_mutations','system.tables.active_on_fly_data_mutations','system.tables.active_on_fly_metadata_mutations','system.tables.active_parts','system.tables.as_select','system.tables.comment','system.tables.create_table_query',|[]        |[]         |[]   |             0|                                                                                                                                                                                                                                                |                                                                                                                                                                                                                                                               |               1|default|62f2ef88-0ed8-41c8-80ed-6ca0267962ea|/172.23.0.1|35008|default     |62f2ef88-0ed8-41c8-80ed-6ca0267962ea|/172.23.0.1    |       35008|     2025-11-25 16:41:14|        2025-11-25 16:41:14.715552000|        2|        0|       |               |                 |              0|                   0|                   0|                   0|                  0|                 0|          2|DBeaver 25.2.5 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                0|   54504|           |[741,765,760,79]                                                        |                 4|{Query=1, SelectQuery=1, InitialQuery=1, QueriesWithSubqueries=1, SelectQueriesWithSubqueries=1, IOBufferAllocs=5, IOBufferAllocBytes=5243195, FunctionExecute=6, NetworkSendElapsedMicroseconds=133, NetworkSendBytes=1394, GlobalThreadPoolJobs=4, LocalThrea|{use_uncompressed_cache=0, load_balancing=in_order, log_queries=1, max_memory_usage=10000000000, parallel_replicas_for_cluster_engines=0}                                                 |[]                      |[]                                 |[]                   |[]                                                                                                                      |[]               |['RowBinaryWithNamesAndTypes']|['equals','and','notIn']                                                                                 |[]                     |[]                  |[]                                    |[]                             |[]               |['SELECT(name, engine, database, comment, uuid, is_temporary, data_paths, metadata_path, metadata_modification_time, metadata_version, dependencies_database, dependencies_table, create_table_query, engine_full, as_select, parameterized_view_parameters, pa|[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|None             |{}                        |
```

Видим что на этих машинах таких запросов не было. В данном случае запрос к распределенной таблице выполнили реплики 1 и 2 шарда 1.

### 8. Смотрим план запроса к распределенной таблице

```sql
EXPLAIN distributed=1
SELECT 
	uniq(teacher_id)
FROM 
	learn_db.mart_student_lesson_mergetree_distributed;
```

Результат
```text
explain                                                                               |
--------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                             |
  MergingAggregated                                                                   |
    Union                                                                             |
      Aggregating                                                                     |
        Expression ((Before GROUP BY + Change column names to column identifiers))    |
          ReadFromMergeTree (learn_db.mart_student_lesson_mergetree)                   |
      ReadFromRemote (Read from remote replica)                                       |
        BlocksMarshalling                                                             |
          Aggregating                                                                 |
            Expression ((Before GROUP BY + Change column names to column identifiers))|
              ReadFromMergeTree (learn_db.mart_student_lesson_mergetree)               |
```

Чтение из таблицы mart_student_lesson_mergetree из первой реплики первого шарда. С помощью union данные объединяются с выборкой со второго шарда.
Команда ReadFromRemote показывает что с другого шарда к нам были переданы данные. Блок BlocksMarshalling означает подготовку и отправку данных между машинами.

# Соединение распределенной и не распределенной таблиц

### 9. Создаем на всех нодах кластера таблицу с учителями

```sql
DROP TABLE IF EXISTS learn_db.teachers ON CLUSTER cluster_2S_2R; 

CREATE TABLE learn_db.teachers ON CLUSTER cluster_2S_2R
(
	`teacher_id` Int32 CODEC(Delta, ZSTD), -- Идентификатор учителя
	`teacher_name` String,
	PRIMARY KEY(teacher_id)
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/{database}/{table}', '{replica}');
```

ReplicatedMergeTree будет синхронизировать данные между всеми репликами. А чтобы на каждом шарде были одинаковые данные, мы вручную вставим эти данные. 

### 10. Создаем распределенную таблицу с учителями

```sql
DROP TABLE IF EXISTS learn_db.teachers_distributed ON CLUSTER cluster_2S_2R; 
CREATE TABLE IF NOT EXISTS learn_db.teachers_distributed ON CLUSTER cluster_2S_2R
(
	`teacher_id` Int32 CODEC(Delta, ZSTD), -- Идентификатор учителя
	`teacher_name` String
)
ENGINE = Distributed('cluster_2S_2R', 'learn_db', 'teachers', teacher_id);
```

Распределенная таблица потребуется для оценки общего объема данных во всех таблицах.

### 11. Вставляем данные в таблицу с учителями (выполняем на каждой ноде кластера)

```sql
INSERT INTO learn_db.teachers 
SELECT DISTINCT
	teacher_id,
	CONCAT('Учитель № ', teacher_id)
FROM 
	learn_db.mart_student_lesson_mergetree_distributed;
```

### 12. Проверяем количество строк в распределенной таблице и в локальной таблице

```sql
SELECT COUNT(*) FROM learn_db.teachers_distributed;
```

Результат
```text
COUNT()|
-------+
   3861|
```

Видим что данные пока что только на шарде 1.

```sql
SELECT COUNT(*) FROM learn_db.teachers;
```

Результат
```text
COUNT()|
-------+
   3861|
```

```sql
SELECT distinct _shard_num FROM teachers_distributed
```

Результат
```text
_shard_num|
----------+
         1|
```

Видим что данные расположены на шарде 1.

Переключаемся на шард 2 реплика 1. Повторяем запрос вставки.

Проверяем количество строк в локальной таблице.

```sql
SELECT COUNT(*) FROM learn_db.teachers;
```

Результат
```text
COUNT()|
-------+
   3861|
```

Проверяем количество строк в распределенной таблице.

```sql
SELECT COUNT(*) FROM learn_db.teachers;
```

Результат
```text
COUNT()|
-------+
   7722|
```

На вторых репликах шардов 1 и 2 ничего вставлять не нужно, т.к. данные уже реплицировались. Проверим.

```sql
SELECT COUNT(*) FROM learn_db.teachers;
```

Результат - шард 1 реплика 2
```text
COUNT()|
-------+
   3861|
```

Результат - шард 2 реплика 2
```text
COUNT()|
-------+
   3861|
```

### 13. Выполняем запрос с соединением распределенной и не распределенной таблиц

```sql
SELECT 
	t.teacher_name,
	AVG(sl.mark) as avg_mark,
	COUNT(sl.mark) as cnt_mark
FROM
	learn_db.mart_student_lesson_mergetree_distributed sl
	INNER JOIN learn_db.teachers t
		on sl.teacher_id = t.teacher_id
GROUP BY
	t.teacher_name
limit 50;
```

Результат
```text
teacher_name  |avg_mark          |cnt_mark|
--------------+------------------+--------+
Учитель № 1071| 3.772623574144487|    1315|
Учитель № 439 | 3.757023538344723|    1317|
Учитель № 1813| 3.817344589409056|    1303|
Учитель № 2614|3.7867263236390754|    1341|
Учитель № 3694|3.8001469507714916|    1361|
Учитель № 2237|3.7732919254658386|    1288|
Учитель № 1452|3.7790432801822322|    1317|
Учитель № 186 |3.7873303167420813|    1326|
Учитель № 531 |3.7544510385756675|    1348|
Учитель № 1529|3.7778643803585346|    1283|
Учитель № 465 |3.7753067484662575|    1304|
Учитель № 3777|3.5591787439613527|     828|
Учитель № 3354|3.7852941176470587|    1360|
Учитель № 1968| 3.776255707762557|    1314|
Учитель № 1192|3.7952270977675133|    1299|
Учитель № 372 | 3.769794721407625|    1364|
Учитель № 3370|           3.78125|    1312|
Учитель № 1595|3.7688292319164804|    1341|
Учитель № 3753|3.6821256038647343|    1035|
Учитель № 2368|3.8145941921072226|    1343|
Учитель № 3628| 3.765739385065886|    1366|
Учитель № 2213|3.7888970051132214|    1369|
Учитель № 799 | 3.824456114028507|    1333|
Учитель № 1476|3.8431818181818183|    1320|
Учитель № 1837| 3.788123167155425|    1364|
Учитель № 1055| 3.787833827893175|    1348|
Учитель № 3293|3.7899860917941584|    1438|
Учитель № 2630|3.7376928728875827|    1361|
Учитель № 226 |3.7507716049382718|    1296|
Учитель № 691 | 3.787157287157287|    1386|
Учитель № 3660|3.7720644666155025|    1303|
Учитель № 36  |4.2888283378746594|     367|
Учитель № 323 | 3.789156626506024|    1328|
Учитель № 794 | 3.787878787878788|    1320|
Учитель № 3243| 3.783744557329463|    1378|
Учитель № 2678|3.7611607142857144|    1344|
Учитель № 1085|3.8006230529595015|    1284|
Учитель № 1166| 3.777186311787072|    1315|
Учитель № 1904|3.7581254724111868|    1323|
Учитель № 3338|3.7878086419753085|    1296|
Учитель № 2703|3.7379718726868987|    1351|
Учитель № 277 |3.8202494497432133|    1363|
Учитель № 3783|3.5318066157760812|     786|
Учитель № 2320| 3.744945567651633|    1286|
Учитель № 1545|3.7413155949741315|    1353|
Учитель № 2304|3.8133535660091047|    1318|
Учитель № 1561|3.8087679516250943|    1323|
Учитель № 468 |3.8692476260043827|    1369|
Учитель № 1920| 3.774669774669775|    1287|
Учитель № 1142|3.8250377073906487|    1326|
```

Здесь мы получили объединение данных по обоим шардам.

### 14. Смотрим план выполнения запроса с соединением распределенной и не распределенной таблиц

```sql
EXPLAIN
SELECT 
	t.teacher_name,
	AVG(sl.mark) as avg_mark,
	COUNT(sl.mark) as cnt_mark
FROM
	learn_db.mart_student_lesson_mergetree_distributed sl
	INNER JOIN learn_db.teachers t
		on sl.teacher_id = t.teacher_id
GROUP BY
	t.teacher_name;
```

Результат
```text
explain                                                                    |
---------------------------------------------------------------------------+
Expression ((Project names + Projection))                                  |
  MergingAggregated                                                        |
    Union                                                                  |
      Aggregating                                                          |
        Expression (Before GROUP BY)                                       |
          Expression (Post Join Actions)                                   |
            Join                                                           |
              Expression (Left Pre Join Actions)                           |
                Expression (Change column names to column identifiers)     |
                  ReadFromMergeTree (learn_db.mart_student_lesson_mergetree)|
              Expression (Right Pre Join Actions)                          |
                Expression (Change column names to column identifiers)     |
                  ReadFromMergeTree (learn_db.teachers)                     |
      ReadFromRemote (Read from remote replica)                            |
```

### 15. Смотрим план выполнения запроса с соединением распределенной и не распределенной таблиц с добавлением флага distributed = 1

```sql
EXPLAIN distributed=1
SELECT 
	t.teacher_name,
	AVG(sl.mark) as avg_mark,
	COUNT(sl.mark) as cnt_mark
FROM
	learn_db.mart_student_lesson_mergetree_distributed sl
	INNER JOIN learn_db.teachers t
		on sl.teacher_id = t.teacher_id
GROUP BY
	t.teacher_name;
```

Результат
```text
explain                                                                        |
-------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                      |
  MergingAggregated                                                            |
    Union                                                                      |
      Aggregating                                                              |
        Expression (Before GROUP BY)                                           |
          Expression (Post Join Actions)                                       |
            Join                                                               |
              Expression (Left Pre Join Actions)                               |
                Expression (Change column names to column identifiers)         |
                  ReadFromMergeTree (default.mart_student_lesson_mergetree)    |
              Expression (Right Pre Join Actions)                              |
                Expression (Change column names to column identifiers)         |
                  ReadFromMergeTree (default.teachers)                         |
      ReadFromRemote (Read from remote replica)                                |
        BlocksMarshalling                                                      |
          Aggregating                                                          |
            Expression (Before GROUP BY)                                       |
              Expression (Post Join Actions)                                   |
                Join                                                           |
                  Expression (Left Pre Join Actions)                           |
                    Expression (Change column names to column identifiers)     |
                      ReadFromMergeTree (default.mart_student_lesson_mergetree)|
                  Expression (Right Pre Join Actions)                          |
                    Expression (Change column names to column identifiers)     |
                      ReadFromMergeTree (default.teachers)                     |
```

Видим что вторая половинка данных была получена с шарда 2.

# Соединяем 2 распределенные таблицы

### 16. Пересоздаем таблицу с учителями, сделав ее распределенной

```sql
DROP TABLE IF EXISTS learn_db.teachers ON CLUSTER cluster_2S_2R; 

CREATE TABLE learn_db.teachers ON CLUSTER cluster_2S_2R
(
	`teacher_id` Int32 CODEC(Delta, ZSTD), -- Идентификатор учителя
	`teacher_name` String,
	PRIMARY KEY(teacher_id)
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/{database}/{table}', '{replica}');
```

### 17. Пересоздаем распределенную таблицу с учителями

```sql
DROP TABLE IF EXISTS learn_db.teachers_distributed ON CLUSTER cluster_2S_2R; 
CREATE TABLE IF NOT EXISTS learn_db.teachers_distributed ON CLUSTER cluster_2S_2R
(
	`teacher_id` Int32 CODEC(Delta, ZSTD), -- Идентификатор учителя
	`teacher_name` String
)
ENGINE = Distributed('cluster_2S_2R', 'learn_db', 'teachers', teacher_id);
```


### 18. Создаем локальную (не на всем кластере) временную таблицу с учителями

```sql
DROP TABLE IF EXISTS learn_db.teachers_tmp;
CREATE TABLE learn_db.teachers_tmp AS learn_db.teachers
ENGINE = MergeTree();
```

Мы не будем сразу сохранять в распределенную таблицу teachers_distributed всех наших учителей. 

### 19. Наполняем данными временную таблицу с учителями

```sql
INSERT INTO learn_db.teachers_tmp
SELECT DISTINCT
	teacher_id,
	CONCAT('Учитель № ', teacher_id)
FROM 
	learn_db.mart_student_lesson_mergetree_distributed;
```

### 20. Наполняем распределенную таблицу с учителями данными

```sql
INSERT INTO learn_db.teachers_distributed
SELECT 
	teacher_id,
	teacher_name
FROM 
	learn_db.teachers_tmp;
```

### 21. Смотрим, сколько данных всего в распределенной таблице и в локальной

```sql
SELECT COUNT(*) FROM learn_db.teachers_distributed;
```
Результат
```text
COUNT()|
-------+
   3861|
```

```sql
SELECT COUNT(*) FROM learn_db.teachers;
```

Результат - шард 1
```text
COUNT()|
-------+
   1931|
```

```sql
SELECT distinct _shard_num FROM learn_db.teachers_distributed;
```

Результат
```text
_shard_num|
----------+
         1|
         2|
```

Видим что на данные распределились по двум шардам.

```sql
SELECT *, _shard_num FROM learn_db.teachers_distributed limit 40;
```

Результат
```text
...
3844|Учитель № 3844|         1|
3846|Учитель № 3846|         1|
3848|Учитель № 3848|         1|
3850|Учитель № 3850|         1|
3852|Учитель № 3852|         1|
3854|Учитель № 3854|         1|
3856|Учитель № 3856|         1|
3858|Учитель № 3858|         1|
3860|Учитель № 3860|         1|
3862|Учитель № 3862|         1|
   3|Учитель № 3   |         2|
   5|Учитель № 5   |         2|
   7|Учитель № 7   |         2|
   9|Учитель № 9   |         2|
  11|Учитель № 11  |         2|
  13|Учитель № 13  |         2|
  15|Учитель № 15  |         2|
  17|Учитель № 17  |         2|
  19|Учитель № 19  |         2|
...
```

Видим что четные учителя на первом шарде, а на втором нечетные.

### 22. Пробуем выполнить соединение двух распределенных таблиц с помощью INNER JOIN

```sql
SELECT 
	t.teacher_name,
	AVG(sl.mark) as avg_mark,
	COUNT(sl.mark) as cnt_mark
FROM
	learn_db.mart_student_lesson_mergetree_distributed sl
	INNER JOIN learn_db.teachers_distributed t
		on sl.teacher_id = t.teacher_id
GROUP BY
	t.teacher_name;
```

Получаем ошибку:
```text
SQL Error [22000]: Code: 288. DB::Exception: Double-distributed IN/JOIN subqueries is denied (distributed_product_mode = 'deny'). 
You may rewrite query to use local tables in subqueries, or use GLOBAL keyword, or set distributed_product_mode to suitable value. (DISTRIBUTED_IN_JOIN_SUBQUERY_DENIED) (version 25.9.3.48 (official build)) 
```

Мы не можем с помощью обычных JOIN объединять две таблицы. Перепишем запрос.

### 23. Выполняем соединение двух распределенных таблиц с помощью GLOBAL JOIN

```sql
SELECT 
	t.teacher_name,
	AVG(sl.mark) as avg_mark,
	COUNT(sl.mark) as cnt_mark
FROM
	learn_db.mart_student_lesson_mergetree_distributed sl
	GLOBAL JOIN learn_db.teachers_distributed t
		on sl.teacher_id = t.teacher_id
GROUP BY
	t.teacher_name;
```

Результат
```text
teacher_name  |avg_mark          |cnt_mark|
--------------+------------------+--------+
Учитель № 1071| 3.737381126554499|    1367|
Учитель № 439 | 3.750181554103123|    1377|
Учитель № 1813|3.7890800299177263|    1337|
Учитель № 2614|3.7961240310077518|    1290|
Учитель № 3694|3.8117998506348023|    1339|
Учитель № 2237| 3.757847533632287|    1338|
Учитель № 1452| 3.832456799398948|    1331|
Учитель № 186 |3.7717309145880575|    1323|
Учитель № 531 |3.8062360801781736|    1347|
Учитель № 1529|3.7521676300578033|    1384|
Учитель № 465 |3.7979197622585437|    1346|
Учитель № 3777|3.5508571428571427|     875|
Учитель № 3354|3.8240181268882174|    1324|
Учитель № 1968|3.7887640449438202|    1335|
Учитель № 1192| 3.773211567732116|    1314|
Учитель № 3660|3.8105181747873162|    1293|
Учитель № 36  | 4.294117647058823|     323|
Учитель № 323 |3.8225454545454545|    1375|
Учитель № 794 |3.7779433681073025|    1342|
Учитель № 3243|3.7943107221006565|    1371|
Учитель № 2678|  3.73109243697479|    1309|
Учитель № 1085|  3.77190332326284|    1324|
Учитель № 1166|3.8068350668647843|    1346|
Учитель № 1904|3.7711480362537766|    1324|
Учитель № 3338|3.7847809377401997|    1301|
Учитель № 2703|3.7934700075930143|    1317|
Учитель № 277 |3.8028919330289193|    1314|
Учитель № 3783|3.4898236092265944|     737|
Учитель № 2320| 3.783682634730539|    1336|
Учитель № 1545| 3.784685367702805|    1319|
Учитель № 372 |  3.79037037037037|    1350|
Учитель № 3370|3.8092744951383697|    1337|
Учитель № 1595|3.8068181818181817|    1320|
Учитель № 3753| 3.634237605238541|    1069|
Учитель № 2368| 3.792998477929985|    1314|
Учитель № 3628| 3.832122093023256|    1376|
Учитель № 2213|3.7573696145124718|    1323|
Учитель № 799 | 3.739819004524887|    1326|
Учитель № 1476| 3.786626596543952|    1331|
Учитель № 1837|3.7762863534675617|    1341|
Учитель № 1055|3.7626182965299684|    1268|
Учитель № 3293|3.7992537313432835|    1340|
Учитель № 2630| 3.765671641791045|    1340|
Учитель № 226 |              3.75|    1384|
Учитель № 691 | 3.801059001512859|    1322|
Учитель № 2304|3.8200155159038016|    1289|
Учитель № 1561| 3.725710014947683|    1338|
Учитель № 468 |3.7817764165390506|    1306|
Учитель № 1920|3.7450404114621603|    1361|
Учитель № 1142|3.7931558935361216|    1315|
Учитель № 560 | 3.781710914454277|    1356|
Учитель № 3384| 3.821455938697318|    1305|
Учитель № 2727| 3.759026687598116|    1274|
Учитель № 3267| 3.800155520995334|    1286|
Учитель № 1039|3.8225108225108224|    1386|
Учитель № 434 |3.7684981684981684|    1365|
Учитель № 1482|3.8148710166919577|    1318|
Учитель № 3644| 3.765121759622938|    1273|
Учитель № 2565|3.7740986019131713|    1359|
Учитель № 1398|3.7691154422788604|    1334|
Учитель № 1300| 3.770057306590258|    1396|
Учитель № 175 |3.8302325581395347|    1290|
Учитель № 1723| 3.851408450704225|    1420|
Учитель № 2924| 3.773409578270193|    1399|
Учитель № 2146| 3.782710280373832|    1284|
Учитель № 3406| 3.808770668583753|    1391|
Учитель № 129 | 3.841930116472546|    1202|
Учитель № 1658|3.7291822955738936|    1333|
Учитель № 496 |3.7413925019127774|    1307|
Учитель № 3025| 3.759063444108761|    1324|
Учитель № 2486|3.7591463414634148|    1312|
Учитель № 3847| 3.459119496855346|     159|
Учитель № 841 |3.7983257229832574|    1314|
Учитель № 289 |3.7810871183916603|    1343|
Учитель № 3001| 3.802097902097902|    1430|
...
```

### 24. Смотрим план запроса с соединением двух распределенных таблиц с помощью GLOBAL JOIN

```sql
EXPLAIN distributed=1
SELECT 
	t.teacher_name,
	AVG(sl.mark) as avg_mark,
	COUNT(sl.mark) as cnt_mark
FROM
	learn_db.mart_student_lesson_mergetree_distributed sl
	GLOBAL JOIN learn_db.teachers_distributed t
		on sl.teacher_id = t.teacher_id
GROUP BY
	t.teacher_name;
```

Результат
```text
explain                                                                        |
-------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                      |
  MergingAggregated                                                            |
    Union                                                                      |
      Aggregating                                                              |
        Expression (Before GROUP BY)                                           |
          Expression (Post Join Actions)                                       |
            Join                                                               |
              Expression (Left Pre Join Actions)                               |
                Expression (Change column names to column identifiers)         |
                  ReadFromMergeTree (learn_db.mart_student_lesson_mergetree)    |
              Expression (Right Pre Join Actions)                              |
                Expression (Change column names to column identifiers)         |
                  ReadFromMemoryStorage                                        |
      ReadFromRemote (Read from remote replica)                                |
        BlocksMarshalling                                                      |
          Aggregating                                                          |
            Expression (Before GROUP BY)                                       |
              Expression (Post Join Actions)                                   |
                Join                                                           |
                  Expression (Left Pre Join Actions)                           |
                    Expression (Change column names to column identifiers)     |
                      ReadFromMergeTree (learn_db.mart_student_lesson_mergetree)|
                  Expression (Right Pre Join Actions)                          |
                    Expression (Change column names to column identifiers)     |
                      ReadFromMemoryStorage                                    |
```

Мы видим что чтение таблицы learn_db.mart_student_lesson_mergetree происходит с диска. А чтение правой таблицы с учителями происходит из таблицы с движком  ReadFromMemoryStorage.
Так происходит потому, что перед тем, как объединять данные, наша правая таблица собирается целиком со всех шардов в одном месте на той машине, куда мы отправили запрос.
И со всех шардов в нее собираются данные. После этого полученные данные правой таблицы были скопированы на все шарды. И только после этого началось соединение.
