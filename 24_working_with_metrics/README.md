# Работа с метриками

## Описание 

В данном случае мы симулируем проблемную ситуацию, которая могла случиться на кластере. Сделаем остановку мутаций и выполните запрос, 
который приведёт к ошибке. Затем проведём диагностические запросы, чтобы увидеть, как эта ситуация отобразится в разных метриках.

### 1. Очистим таблицы логов

```sql
TRUNCATE TABLE system.metric_log ON CLUSTER cluster_2S_2R;
TRUNCATE TABLE system.query_log ON CLUSTER cluster_2S_2R;
```

### 2. Подготовим тестовую таблицу на первом хосте

```sql
DROP TABLE IF EXISTS my_tbl ON CLUSTER cluster_2S_2R SYNC;
CREATE TABLE my_tbl ON CLUSTER cluster_2S_2R
(
    EventDate DateTime,
    CounterID UInt32,
    UserID UInt32,
    SomeID UInt32
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/my_tbl', '{replica}')
PARTITION BY toYYYYMM(EventDate)
ORDER BY (CounterID, EventDate, intHash32(UserID));
```

### 3. Остановим слияния для тестовой таблицы

```sql
SYSTEM STOP MERGES my_tbl;
```

### 4. Создадим два куска

```sql
INSERT INTO my_tbl (EventDate, CounterID, UserID, SomeID)
    SELECT
        toDate('2020-02-01') + (number / 10000),
        rand(1) % 10000, 
        number,
        rand(1) % 10000
    FROM numbers(1000000);

INSERT INTO my_tbl (EventDate, CounterID, UserID, SomeID)
    SELECT
        toDate('2020-02-01') + (number / 10000),
        rand(1) % 10000, 
        number,
        rand(1) % 10000
    FROM numbers(1000000);
```
       
### 5. Выполним мутацию

```sql
ALTER TABLE my_tbl UPDATE SomeID = SomeID + 1 WHERE UserID > 1000000/2;
```

Команда ALTER TABLE реплицируется на все реплики шарда, поэтому в данном случае выражение ON CLUSTER не указано.

### 6. Попробуем выполнить ещё одну мутацию. В результате получится ошибка

```sql
ALTER TABLE my_tbl UPDATE CounterID = CounterID + 1 WHERE UserID > 1000000/2;
```

Результат
```text
Cannot UPDATE key column `CounterID` 
```

Теперь посмотрим на различные метрики.

### 7. Проверим все события с точки зрения ключевых метрик

```sql
SELECT event, value
FROM system.events
WHERE event ilike '%query%'
ORDER BY value DESC
```

Результат
```text
event                      |value  |
---------------------------+-------+
QueryTimeMicroseconds      |3113322|
OtherQueryTimeMicroseconds |2172288|
InsertQueryTimeMicroseconds| 551385|
SelectQueryTimeMicroseconds| 389649|
QueryProfilerRuns          |    582|
Query                      |     37|
InitialQuery               |     33|
SelectQuery                |     23|
InsertQuery                |      2|
FailedQuery                |      2|
```

Здесь видим, что ошибка учтена в метрике FailedQuery.

### 8. Теперь посмотрим статистику по конкретным метрикам в разрезе хостов

```sql
SELECT
    _shard_num,
    hostname(),
    sum(ProfileEvent_FailedQuery),
    sum(ProfileEvent_FailedSelectQuery),
    sum(ProfileEvent_FailedInsertQuery),
    sum(ProfileEvent_ReplicatedPartFailedFetches),
    sum(ProfileEvent_ReplicatedPartChecksFailed),
    sum(ProfileEvent_DistributedConnectionFailTry),
    sum(ProfileEvent_ReplicatedDataLoss)
FROM clusterAllReplicas(cluster_2S_2R, system.metric_log)
WHERE event_time > now() - interval 24 hour
GROUP BY _shard_num, hostname();
```

Результат
```text
_shard_num|hostname()   |sum(ProfileEvent_FailedQuery)|sum(ProfileEvent_FailedSelectQuery)|sum(ProfileEvent_FailedInsertQuery)|sum(ProfileEvent_ReplicatedPartFailedFetches)|sum(ProfileEvent_ReplicatedPartChecksFailed)|sum(ProfileEvent_DistributedConnectionFailTry)|sum(ProfileEvent_ReplicatedDataLoss)|
----------+-------------+-----------------------------+-----------------------------------+-----------------------------------+---------------------------------------------+--------------------------------------------+----------------------------------------------+------------------------------------+
         3|clickhouse-02|                            0|                                  0|                                  0|                                            0|                                           0|                                             0|                                   0|
         2|clickhouse-03|                            0|                                  0|                                  0|                                            0|                                           0|                                             0|                                   0|
         4|clickhouse-04|                            0|                                  0|                                  0|                                            0|                                           0|                                             0|                                   0|
         1|clickhouse-01|                            2|                                  1|                                  0|                                            0|                                           0|                                             0|                                   0|
```

В результате видно, что проблема есть только на одном шарде.
При этом нет ошибок SELECT и INSERT, значит, всё дело в мутации.  

### 9. Проверим счётчики ошибок, и увидим, что ошибка учтена в CANNOT_UPDATE_COLUMN

```sql
SELECT _shard_num, hostname(), name, sum(value) as v
FROM clusterAllReplicas(cluster_2S_2R, system.errors)
GROUP BY _shard_num, name
ORDER BY v DESC;
```

Результат
```text
_shard_num|hostname()   |name                |v|
----------+-------------+--------------------+-+
         1|clickhouse-01|CLUSTER_DOESNT_EXIST|3|
         1|clickhouse-01|CANNOT_UPDATE_COLUMN|1|
```

### 10. Найдем последние запросы, завершившиеся ошибкой

```sql
SELECT
  _shard_num,
  max(query_start_time) as q_start_time_max,
  max(query_duration_ms),
  query,
  type,
  max(read_rows),
  max(read_bytes),
  max(memory_usage),
  exception,
  exception_code
FROM clusterAllReplicas(cluster_2S_2R, system.query_log)
WHERE
  exception_code <> 0 and
  query_start_time > now() - interval 2 day
GROUP BY _shard_num, query, type, exception, exception_code
ORDER BY q_start_time_max DESC
LIMIT 50
```

Результат
```text
_shard_num|q_start_time_max   |max(query_duration_ms)|query                                                                                                                                                                                                                                                          |type                |max(read_rows)|max(read_bytes)|max(memory_usage)|exception                                                                                                                                                                                                                                                      |exception_code|
----------+-------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+--------------+---------------+-----------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------+
         1|2025-11-13 15:10:28|                     2|SELECT¶  _shard_num,¶  max(query_start_time) as q_start_time_max,¶  max(query_duration_ms),¶  query,¶  type,¶  max(read_rows),¶  max(read_bytes),¶  max(memory_usage),¶  exception,¶  exception_code¶FROM clusterAllReplicas(cluster_2S_2R, system.query_log)¶W|ExceptionBeforeStart|             0|              0|                0|Code: 62. DB::Exception: Syntax error: failed at position 447 (\G) (line 18, col 9): \G. Expected one of: token, DoubleColon, OR, AND, IS NOT DISTINCT FROM, IS NULL, IS NOT NULL, BETWEEN, NOT BETWEEN, LIKE, ILIKE, NOT LIKE, NOT ILIKE, REGEXP, IN, NOT IN, |            62|
         1|2025-11-13 15:05:38|                     1|SELECT _shard_num, hostname(), name, sum(value) as v¶FROM clusterAllReplicas(test_cluster, system.errors)¶GROUP BY _shard_num, name¶ORDER BY v DESC                                                                                                            |ExceptionBeforeStart|             0|              0|          2098174|Code: 701. DB::Exception: Requested cluster 'test_cluster' not found. (CLUSTER_DOESNT_EXIST) (version 25.9.3.48 (official build))                                                                                                                              |           701|
         1|2025-11-13 15:03:17|                     5|SELECT¶    _shard_num,¶    hostname(),¶    sum(ProfileEvent_FailedQuery),¶    sum(ProfileEvent_FailedSelectQuery),¶    sum(ProfileEvent_FailedInsertQuery),¶    sum(ProfileEvent_ReplicatedPartFailedFetches),¶    sum(ProfileEvent_ReplicatedPartChecksFailed)|ExceptionBeforeStart|             0|              0|          2098174|Code: 701. DB::Exception: Requested cluster 'test_cluster' not found. (CLUSTER_DOESNT_EXIST) (version 25.9.3.48 (official build))                                                                                                                              |           701|
         1|2025-11-13 14:56:34|                     2|ALTER TABLE my_tbl UPDATE CounterID = CounterID + 1 WHERE UserID > 1000000/2                                                                                                                                                                                   |ExceptionBeforeStart|             0|              0|          2098174|Code: 420. DB::Exception: Cannot UPDATE key column `CounterID`. (CANNOT_UPDATE_COLUMN) (version 25.9.3.48 (official build))                                                                                                                                    |           420|
```

Проблемный запрос, это ALTER TABLE my_tbl UPDATE CounterID = CounterID + 1 WHERE UserID > 1000000/2.

### 11. Посмотрим на мутации и проверим успешные

```sql
SELECT _shard_num, hostname() database, table, mutation_id, command, create_time
FROM clusterAllReplicas(test_cluster, system.mutations)
WHERE
    is_done = 1 and
    create_time  > now() - interval 24 hour
ORDER BY create_time DESC
LIMIT 100;
```

Результат
```text
_shard_num|database     |table |mutation_id|command                                                  |create_time        |
----------+-------------+------+-----------+---------------------------------------------------------+-------------------+
         2|clickhouse-03|my_tbl|0000000000 |(UPDATE SomeID = SomeID + 1 WHERE UserID > (1000000 / 2))|2025-11-13 14:55:00|
```

Слияния на первом хосте остановлены, поэтому видны операции только со второй реплики. Это логично, ведь мы делали остановку только на одном хосте.

### 12. Проверим не завершившиеся мутации, и увидим, что одна такая есть

```sql
SELECT _shard_num,hostname(), database, table, mutation_id, command, latest_fail_reason, latest_fail_time
FROM clusterAllReplicas(cluster_2S_2R, system.mutations)
WHERE
    is_done = 0 
LIMIT 100;
```

Результат
```text
_shard_num|hostname()   |database|table |mutation_id|command                                                  |latest_fail_reason|latest_fail_time   |
----------+-------------+--------+------+-----------+---------------------------------------------------------+------------------+-------------------+
         1|clickhouse-01|default |my_tbl|0000000000 |(UPDATE SomeID = SomeID + 1 WHERE UserID > (1000000 / 2))|                  |1970-01-01 03:00:00|
```

### 13. Проверим мутации, завершившиеся ошибкой. Увидим, что их нет: 

```sql
SELECT _shard_num, database, table, mutation_id, command, latest_fail_reason, latest_fail_time
FROM clusterAllReplicas(cluster_2S_2R, system.mutations)
WHERE
    is_done = 0 and
    latest_fail_time > now() - interval 24 hour
ORDER BY latest_fail_reason DESC
LIMIT 100;
```

Результат
```text
_shard_num|database|table|mutation_id|command|latest_fail_reason|latest_fail_time|
----------+--------+-----+-----------+-------+------------------+----------------+
```

Как видели на предыдущем шаге, есть одна зависшая мутация. Но сейчас нет ошибок, а значит, нет и глобальных проблем с кластером.

### 14. Теперь поработаем с метриками асинхронных процессов. Выполним запрос и посмотрим метрики из system.asynchronous_metrics по репликации

```
SELECT
    metric,
    value
FROM system.asynchronous_metrics
WHERE metric ILIKE '%repl%'
ORDER BY value DESC;
```

Результат
```text
metric                   |value|
-------------------------+-----+
ReplicasSumQueueSize     |  8.0|
ReplicasMaxQueueSize     |  8.0|
ReplicasMaxAbsoluteDelay |  0.0|
ReplicasSumInsertsInQueue|  0.0|
ReplicasMaxRelativeDelay |  0.0|
ReplicasMaxMergesInQueue |  0.0|
ReplicasSumMergesInQueue |  0.0|
ReplicasMaxInsertsInQueue|  0.0|
```

Здесь видно, что в очереди находятся 8 элементов.

### 15. Проверим очередь репликации по таблице с группировкой по полям table, type, is_currently_executing

```sql
SELECT
    table, type, is_currently_executing, count() AS cnt
FROM system.replication_queue
GROUP BY table, type, is_currently_executing
ORDER BY is_currently_executing ASC, cnt DESC;
```

Результат
```text
table |type       |is_currently_executing|cnt|
------+-----------+----------------------+---+
my_tbl|MUTATE_PART|                     0|  8|
```

Из вывода видно, что есть зависшая мутация и 8 операций в очереди.

### 16. Выполним запрос без группировки, и вы увидим, какие именно датапарты ожидают мутации

```
SELECT
    node_name, type, new_part_name, parts_to_merge, num_postponed 
FROM system.replication_queue
ORDER BY is_currently_executing ASC;
```

Результат
```text
node_name       |type       |new_part_name |parts_to_merge  |num_postponed|
----------------+-----------+--------------+----------------+-------------+
queue-0000000008|MUTATE_PART|202002_0_0_0_2|['202002_0_0_0']|           93|
queue-0000000009|MUTATE_PART|202002_1_1_0_2|['202002_1_1_0']|           92|
queue-0000000010|MUTATE_PART|202003_0_0_0_2|['202003_0_0_0']|           91|
queue-0000000011|MUTATE_PART|202003_1_1_0_2|['202003_1_1_0']|           90|
queue-0000000012|MUTATE_PART|202004_0_0_0_2|['202004_0_0_0']|           89|
queue-0000000013|MUTATE_PART|202004_1_1_0_2|['202004_1_1_0']|           88|
queue-0000000014|MUTATE_PART|202005_0_0_0_2|['202005_0_0_0']|           87|
queue-0000000015|MUTATE_PART|202005_1_1_0_2|['202005_1_1_0']|           86|
```

Здесь возникает вопрос, всё ли нормально с репликой.

### 17. Вернемся к репликации и выполним запрос к system.replicas. Так мы проверим состояние реплики

```
SELECT * FROM system.replicas WHERE table = 'my_tbl';
```

Результат
```text
database|table |engine             |is_leader|can_become_leader|is_readonly|readonly_start_time|is_session_expired|future_parts|parts_to_check|zookeeper_name|zookeeper_path              |replica_name|replica_path                            |columns_version|queue_size|inserts_in_queue|merges_in_queue|part_mutations_in_queue|queue_oldest_time  |inserts_oldest_time|merges_oldest_time |part_mutations_oldest_time|oldest_part_to_get|oldest_part_to_merge_to|oldest_part_to_mutate_to|log_max_index|log_pointer|last_queue_update  |absolute_delay|total_replicas|active_replicas|lost_part_count|last_queue_update_exception|zookeeper_exception|replica_is_active|
--------+------+-------------------+---------+-----------------+-----------+-------------------+------------------+------------+--------------+--------------+----------------------------+------------+----------------------------------------+---------------+----------+----------------+---------------+-----------------------+-------------------+-------------------+-------------------+--------------------------+------------------+-----------------------+------------------------+-------------+-----------+-------------------+--------------+--------------+---------------+---------------+---------------------------+-------------------+-----------------+
default |my_tbl|ReplicatedMergeTree|        1|                1|          0|                   |                 0|           0|             0|default       |/clickhouse/tables/01/my_tbl|01          |/clickhouse/tables/01/my_tbl/replicas/01|             -1|         8|               0|              0|                      8|2025-11-13 14:55:00|1970-01-01 03:00:00|1970-01-01 03:00:00|       2025-11-13 14:55:00|                  |                       |202002_0_0_0_2          |           15|         16|2025-11-13 14:55:01|             0|             2|              2|              0|                           |                   |{02=1, 01=1}     |
```

Из вывода видно, что обе реплики активны и с ними всё хорошо. Но также в этом случае видно, что part_mutations_in_queue имеет значение 8 и queue_size равен 8. Это и есть датапарты.

Мы выполнили диагностику и обнаружили на одном шарде зависшие мутации и запрос, выполненный с ошибкой из-за них. Других критичных проблем не обнаружилось.

### 18. Теперь включим мутации снова, немного подождем и проверим, что ситуация исправилась. Проследим динамику у метрик

```sql
SYSTEM START merges my_tbl;
```

### 19. Подождем 30 секунд и проверим, остались ли зависшие мутации

```sql
SELECT _shard_num,hostname(), database, table, mutation_id, command, latest_fail_reason, latest_fail_time
FROM clusterAllReplicas(cluster_2S_2R, system.mutations)
WHERE
    is_done = 0 
limit 100;
```

Результат
```text
_shard_num|hostname()|database|table|mutation_id|command|latest_fail_reason|latest_fail_time|
----------+----------+--------+-----+-----------+-------+------------------+----------------+
```

Отлично. Зависшие мутации ушли, и всё работает штатно.