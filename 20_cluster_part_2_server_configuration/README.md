# Конфигурирование сервера

### 1. Меняем таблицу, в которую будут сохраняться логи по выполненным запросам - /etc/clickhouse-server/config.d/query_log.xml

```bash
cp config.xml config.d/system_query_log.xml
cd config.d

```

Оставим в фале следующее содержимое
```xml
    <query_log>
        <!-- What table to insert data. If table is not exist, it will be created.
             When query log structure is changed after system update,
              then old table will be renamed and new table will be created automatically.
        -->
        <database>system</database>
        <table>query_log_new</table>
        <!--
            PARTITION BY expr: https://clickhouse.com/docs/en/table_engines/mergetree-family/custom_partitioning_key/
            Example:
                event_date
                toMonday(event_date)
                toYYYYMM(event_date)
                toStartOfHour(event_time)
        -->
        <partition_by>toYYYYMM(event_date)</partition_by>
    </query_log>
```

### 2. Проверим чтение конфигурационной таблицы

```sql
SELECT * FROM system.query_log
```

Результат
```text
SQL Error [22000]: Code: 60. DB::Exception: Unknown table expression identifier 'system.query_log_new' in scope SELECT * FROM system.query_log_new. (UNKNOWN_TABLE) (version 25.4.13.22 (official build)) 
```

После перезапуска сервера получаем данные из новой системной таблицы.
```sql
SELECT * FROM system.query_log_new
```

Результат
```text
hostname    |type                |event_date|event_time         |event_time_microseconds      |query_start_time   |query_start_time_microseconds|query_duration_ms|read_rows|read_bytes|written_rows|written_bytes|result_rows|result_bytes|memory_usage|current_database|query                             |formatted_query|normalized_query_hash|query_kind|databases|tables|columns|partitions|projections|views|exception_code|exception                                                                                                                                                                                                                                                      |stack_trace                                                                                                                                                                                                                                                    |is_initial_query|user    |query_id                            |address    |port |initial_user|initial_query_id                    |initial_address|initial_port|initial_query_start_time|initial_query_start_time_microseconds|interface|is_secure|os_user|client_hostname|client_name|client_revision|client_version_major|client_version_minor|client_version_patch|script_query_number|script_line_number|http_method|http_user_agent                                                                                              |http_referer|forwarded_for|quota_key|distributed_depth|revision|log_comment|thread_ids|peak_threads_usage|ProfileEvents|Settings                                                                                   |used_aggregate_functions|used_aggregate_function_combinators|used_database_engines|used_data_type_families|used_dictionaries|used_formats|used_functions|used_storages|used_table_functions|used_row_policies|used_privileges|missing_privileges|transaction_id                                    |query_cache_usage|asynchronous_read_counters|
------------+--------------------+----------+-------------------+-----------------------------+-------------------+-----------------------------+-----------------+---------+----------+------------+-------------+-----------+------------+------------+----------------+----------------------------------+---------------+---------------------+----------+---------+------+-------+----------+-----------+-----+--------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------+--------+------------------------------------+-----------+-----+------------+------------------------------------+---------------+------------+------------------------+-------------------------------------+---------+---------+-------+---------------+-----------+---------------+--------------------+--------------------+--------------------+-------------------+------------------+-----------+-------------------------------------------------------------------------------------------------------------+------------+-------------+---------+-----------------+--------+-----------+----------+------------------+-------------+-------------------------------------------------------------------------------------------+------------------------+-----------------------------------+---------------------+-----------------------+-----------------+------------+--------------+-------------+--------------------+-----------------+---------------+------------------+--------------------------------------------------+-----------------+--------------------------+
19294dc11be6|ExceptionBeforeStart|2025-10-22|2025-10-22 13:32:11|2025-10-22 13:32:11.916920000|2025-10-22 13:32:11|2025-10-22 13:32:11.915102000|                1|        0|         0|           0|            0|          0|           0|           0|learn_db        |SELECT * FROM system.query_log_new|               | 13134533445725451702|Select    |[]       |[]    |[]     |[]        |[]         |[]   |            60|Code: 60. DB::Exception: Unknown table expression identifier 'system.query_log_new' in scope SELECT * FROM system.query_log_new. (UNKNOWN_TABLE) (version 25.4.13.22 (official build))                                                                         |0. DB::Exception::Exception(DB::Exception::MessageMasked&&, int, bool) @ 0x000000000f2f30db¶1. DB::Exception::Exception(PreformattedMessage&&, int) @ 0x0000000009c7de6c¶2. DB::Exception::Exception<String const&, String>(int, FormatStringHelperImpl<std::ty|               1|username|acf79ed0-1ef3-4e9f-b13e-a4a55d7a0fb5|/172.17.0.1|47746|username    |acf79ed0-1ef3-4e9f-b13e-a4a55d7a0fb5|/172.17.0.1    |       47746|     2025-10-22 13:32:11|        2025-10-22 13:32:11.915102000|        2|        0|       |               |           |              0|                   0|                   0|                   0|                  0|                 0|          2|DBeaver 25.2.3 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                0|   54509|           |[]        |                 0|{}           |{max_result_rows=1000, result_overflow_mode=break, parallel_replicas_for_cluster_engines=0}|[]                      |[]                                 |[]                   |[]                     |[]               |[]          |[]            |[]           |[]                  |[]               |[]             |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|Unknown          |{}                        |
19294dc11be6|ExceptionBeforeStart|2025-10-22|2025-10-22 13:32:22|2025-10-22 13:32:22.892237000|2025-10-22 13:32:22|2025-10-22 13:32:22.890755000|                1|        0|         0|           0|            0|          0|           0|           0|learn_db        |ELECT * FROM system.query_log     |               | 11674103626314096738|None      |[]       |[]    |[]     |[]        |[]         |[]   |            62|Code: 62. DB::Exception: Syntax error: failed at position 1 (ELECT): ELECT * FROM system.query_log. Expected one of: Query, Query with output, EXPLAIN, EXPLAIN, SELECT query, possibly with UNION, list of union elements, SELECT query, subquery, possibly wi|0. DB::Exception::Exception(DB::Exception::MessageMasked&&, int, bool) @ 0x000000000f2f30db¶1. DB::Exception::createDeprecated(String const&, int, bool) @ 0x000000000f3b926d¶2. DB::parseQueryAndMovePosition(DB::IParser&, char const*&, char const*, String |               1|username|1f2ee9f4-bb95-48f4-b81c-0bc82af8b0db|/172.17.0.1|54988|username    |1f2ee9f4-bb95-48f4-b81c-0bc82af8b0db|/172.17.0.1    |       54988|     2025-10-22 13:32:22|        2025-10-22 13:32:22.890755000|        2|        0|       |               |           |              0|                   0|                   0|                   0|                  0|                 0|          2|DBeaver 25.2.3 - Main jdbc-v2/0.8.5 clickhouse-java-v2/0.8.5 (Windows 10; jvm:21.0.8) Apache-HttpClient/5.2.1|            |             |         |                0|   54509|           |[]        |                 0|{}           |{}                                                                                         |[]                      |[]                                 |[]                   |[]                     |[]               |[]          |[]            |[]           |[]                  |[]               |[]             |[]                |{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|Unknown          |{}                        |
```

### 3. Генерируем пароль и хэш от него

```bash
PASSWORD=$(base64 < /dev/urandom | head -c8); echo "$PASSWORD"; echo -n "$PASSWORD" | sha256sum | tr -d '-'
```

```text
d1be30da8006017e72f5293d392b9432847652ff86a185f065457edc864b4cd7
```

### 4. Создаем пользователя в /etc/clickhouse-server/users.xml

```xml
    <users>
        <testovtt>
            <password_sha256_hex>d1be30da8006017e72f5293d392b9432847652ff86a185f065457edc864b4cd7</password_sha256_hex>
        </testovtt>
    </users>
```

### 5. Выдадим права для пользователя

```xml
    <users>
        <testovtt>
            <password_sha256_hex>d1be30da8006017e72f5293d392b9432847652ff86a185f065457edc864b4cd7</password_sha256_hex>
            <grants>
                <query>GRANT learn_db.*</query>
            </grants>
        </testovtt>
    </users>
```

### 6. Создаем роль в /etc/clickhouse-server/users.xml

```xml
    <roles>
        <learn_db_full>
            <grants>
                <query>GRANT ALL ON learn_db.*</query>
            </grants>
        </learn_db_full>
    </roles>
```

### 7. Применим роль для пользователя

```xml
    <users>
        <testovtt>
            <password_sha256_hex>d1be30da8006017e72f5293d392b9432847652ff86a185f065457edc864b4cd7</password_sha256_hex>
            <grants>
                <query>GRANT learn_db_full</query>
            </grants>
        </testovtt>
    </users>
```