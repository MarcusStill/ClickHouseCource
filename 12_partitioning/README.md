# Партиционирование таблиц

### 1. Создаем не партиционированную таблицу с заказами orders

```sql
DROP TABLE IF EXISTS learn_db.orders;
CREATE TABLE learn_db.orders (
	order_id UInt32,
	user_id UInt32,
	product_id UInt32,
	amount Decimal(18, 2),
	order_date Date
)
ENGINE = MergeTree()
ORDER BY (product_id);
```

### 2. Вставляем в таблицу с заказами 10 000 000 строк

```sql
INSERT INTO learn_db.orders
SELECT
	number as order_id,
	round(randUniform(1, 100000)) as user_id,
	round(randUniform(1, 10000)) as product_id,
	randUniform(1, 5000) as amount,
	date_add(day, rand() % (365 * 10 + 2), today() - INTERVAL 10 YEAR),
FROM 
	numbers(10000000);
```

### 3. Создаем таблицу с заказами, партиционированную по годам

```sql
DROP TABLE IF EXISTS learn_db.orders_partition_by_year;
CREATE TABLE learn_db.orders_partition_by_year (
	order_id UInt32,
	user_id UInt32,
	product_id UInt32,
	amount Decimal(18, 2),
	order_date Date
)
ENGINE = MergeTree()
PARTITION BY toYear(order_date)
ORDER BY (product_id);
```

### 4. Вставляем данные в партиционированную таблицу

```sql
INSERT INTO learn_db.orders_partition_by_year
SELECT * FROM learn_db.orders;
```

### 5. Получаем список частей данных и партиций по двум таблицам с заказами

```sql
SELECT DISTINCT _partition_id, _part FROM learn_db.orders ORDER BY _partition_id, _part;
```

Результат
```text
_partition_id|_part    |
-------------+---------+
all          |all_1_6_1|
all          |all_7_7_0|
all          |all_8_8_0|
all          |all_9_9_0|
```

```sql
SELECT DISTINCT _partition_id, _part FROM learn_db.orders_partition_by_year ORDER BY _partition_id, _part;
```

Результат
```text
_partition_id|_part         |
-------------+--------------+
2015         |2015_109_109_0|
2015         |2015_70_70_0  |
2015         |2015_7_66_1   |
2015         |2015_83_83_0  |
2015         |2015_95_95_0  |
2016         |2016_100_100_0|
2016         |2016_10_64_1  |
2016         |2016_77_77_0  |
2016         |2016_81_81_0  |
2016         |2016_92_92_0  |
2017         |2017_108_108_0|
2017         |2017_3_65_1   |
2017         |2017_68_68_0  |
2017         |2017_85_85_0  |
2017         |2017_93_93_0  |
2018         |2018_106_106_0|
2018         |2018_76_76_0  |
2018         |2018_82_82_0  |
2018         |2018_8_56_1   |
2018         |2018_98_98_0  |
2019         |2019_102_102_0|
2019         |2019_11_63_1  |
2019         |2019_74_74_0  |
2019         |2019_79_79_0  |
2019         |2019_97_97_0  |
2020         |2020_104_104_0|
2020         |2020_6_60_1   |
2020         |2020_75_75_0  |
2020         |2020_87_87_0  |
2020         |2020_96_96_0  |
2021         |2021_105_105_0|
2021         |2021_1_61_1   |
2021         |2021_69_69_0  |
2021         |2021_78_78_0  |
2021         |2021_94_94_0  |
2022         |2022_103_103_0|
2022         |2022_67_67_0  |
2022         |2022_80_80_0  |
2022         |2022_91_91_0  |
2022         |2022_9_59_1   |
2023         |2023_107_107_0|
2023         |2023_2_62_1   |
2023         |2023_73_73_0  |
2023         |2023_84_84_0  |
2023         |2023_90_90_0  |
2024         |2024_101_101_0|
2024         |2024_4_57_1   |
2024         |2024_72_72_0  |
2024         |2024_88_88_0  |
2024         |2024_99_99_0  |
2025         |2025_110_110_0|
2025         |2025_5_58_1   |
2025         |2025_71_71_0  |
2025         |2025_86_86_0  |
2025         |2025_89_89_0  |
```

### 6. Смотрим статистику по партициям в двух таблицах с заказами

```sql
SELECT
    partition,
    count() AS parts,
    sum(rows) AS rows
FROM system.parts
WHERE (database = 'learn_db') AND (`table` = 'orders') AND active
GROUP BY partition
ORDER BY partition ASC;
```

Результат
```text
partition|parts|rows    |
---------+-----+--------+
tuple()  |    4|10000000|
```

```sql
SELECT
    partition,
    count() AS parts,
    sum(rows) AS rows
FROM system.parts
WHERE (database = 'learn_db') AND (`table` = 'orders_partition_by_year') AND active
GROUP BY partition
ORDER BY partition ASC;
```

Результат
```text
partition|parts|rows   |
---------+-----+-------+
2015     |    5| 266179|
2016     |    5|1003162|
2017     |    5| 999904|
2018     |    5| 999471|
2019     |    5| 999824|
2020     |    5|1003033|
2021     |    5| 999162|
2022     |    5| 998693|
2023     |    5| 999337|
2024     |    5| 999945|
2025     |    5| 731290|
```

### 7. Сравниваем скорость выполнения запроса из таблицы без партиционирования с запросом из таблицы с партиционированием с фильтрацией по полю, по которому таблица партиционирована

```sql
SELECT *, _part FROM learn_db.orders_agg;
```

Результат
```text
Query id: bfeb8cd7-e863-42d9-bf4f-106f9a39ae44

        ┌─order_id─┬─user_id─┬─product_id─┬──amount─┬─order_date─┐
     1. │  5810019 │   52513 │       1585 │  381.57 │ 2024-06-19 │
     2. │  5847301 │   24735 │       1585 │ 1704.39 │ 2024-07-18 │
     3. │  5868838 │    9136 │       1585 │ 2108.82 │ 2024-07-08 │
     4. │  5956566 │    6613 │       1585 │ 2261.88 │ 2024-11-08 │
     5. │  6033165 │   74278 │       1585 │ 4014.82 │ 2024-04-27 │
200090. │  9986540 │   10175 │       3000 │  175.62 │ 2024-04-11 │
        └─order_id─┴─user_id─┴─product_id─┴──amount─┴─order_date─┘
Showed 1000 out of 200090 rows.

200090 rows in set. Elapsed: 0.072 sec. Processed 2.03 million rows, 44.70 MB (28.13 million rows/s., 618.90 MB/s.)
Peak memory usage: 7.13 MiB.
```

```sql
SELECT 
	*
FROM
	learn_db.orders_partition_by_year
WHERE
	`product_id` between 1000 and 3000
	AND `order_date` BETWEEN '2024-01-01' AND '2024-12-31';
```

Результат
```text
Query id: 13861490-3ed0-4029-aa05-6bbdb4f3615d

        ┌─order_id─┬─user_id─┬─product_id─┬──amount─┬─order_date─┐
     1. │  6097095 │   43128 │       1487 │ 4070.73 │ 2024-12-30 │
     2. │  6263307 │   49293 │       1487 │ 4042.27 │ 2024-09-26 │
     3. │  6335882 │   41610 │       1487 │ 2021.68 │ 2024-11-15 │
     4. │  6364188 │   27577 │       1487 │ 1723.43 │ 2024-01-21 │
     5. │  6428629 │   52942 │       1487 │  472.62 │ 2024-03-01 │
200090. │  9986540 │   10175 │       3000 │  175.62 │ 2024-04-11 │
        └─order_id─┴─user_id─┴─product_id─┴──amount─┴─order_date─┘
Showed 1000 out of 200090 rows.

200090 rows in set. Elapsed: 0.016 sec. Processed 212.99 thousand rows, 4.69 MB (13.09 million rows/s., 288.06 MB/s.)
Peak memory usage: 4.28 MiB.
```

### 8. Исследуем планы выполнения запросов

```sql
EXPLAIN indexes = 1
SELECT 
	*
FROM
	learn_db.orders
WHERE
	`product_id` between 1000 and 3000
	AND `order_date` BETWEEN '2024-01-01' AND '2024-12-31';
```

Результат
```text
explain                                                                           |
----------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                         |
  Expression                                                                      |
    ReadFromMergeTree (learn_db.orders)                                           |
    Indexes:                                                                      |
      PrimaryKey                                                                  |
        Keys:                                                                     |
          product_id                                                              |
        Condition: and((product_id in (-Inf, 3000]), (product_id in [1000, +Inf)))|
        Parts: 4/4                                                                |
        Granules: 248/1222                                                        |
```

```sql
EXPLAIN indexes = 1
SELECT 
	*
FROM
	learn_db.orders_partition_by_year
WHERE
	`product_id` between 1000 and 3000
	AND `order_date` BETWEEN '2024-01-01' AND '2024-12-31';
```

Результат
```text
explain                                                                                           |
--------------------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                                         |
  Expression                                                                                      |
    ReadFromMergeTree (learn_db.orders_partition_by_year)                                         |
    Indexes:                                                                                      |
      MinMax                                                                                      |
        Keys:                                                                                     |
          order_date                                                                              |
        Condition: and((order_date in (-Inf, 20088]), (order_date in [19723, +Inf)))              |
        Parts: 5/55                                                                               |
        Granules: 123/1228                                                                        |
      Partition                                                                                   |
        Keys:                                                                                     |
          toYear(order_date)                                                                      |
        Condition: and((toYear(order_date) in (-Inf, 2024]), (toYear(order_date) in [2024, +Inf)))|
        Parts: 5/5                                                                                |
        Granules: 123/123                                                                         |
      PrimaryKey                                                                                  |
        Keys:                                                                                     |
          product_id                                                                              |
        Condition: and((product_id in (-Inf, 3000]), (product_id in [1000, +Inf)))                |
        Parts: 4/5                                                                                |
        Granules: 26/123                                                                          |
```

### 9. Исследуем планы выполнения запросов без фильтрации по полю, по которому партиционирована таблица

```sql
EXPLAIN indexes = 1
SELECT 
	*
FROM
	learn_db.orders
WHERE
	`product_id` between 1000 and 1000;
```

Результат
```text
explain                                                                           |
----------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                         |
  Expression                                                                      |
    ReadFromMergeTree (learn_db.orders)                                           |
    Indexes:                                                                      |
      PrimaryKey                                                                  |
        Keys:                                                                     |
          product_id                                                              |
        Condition: and((product_id in (-Inf, 1000]), (product_id in [1000, +Inf)))|
        Parts: 4/4                                                                |
        Granules: 4/1222                                                          |
```

```sql
EXPLAIN indexes = 1
SELECT 
	*
FROM
	learn_db.orders_partition_by_year
WHERE
	`product_id` between 1000 and 1000;
```

Результат
```text
explain                                                                           |
----------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                         |
  Expression                                                                      |
    ReadFromMergeTree (learn_db.orders_partition_by_year)                         |
    Indexes:                                                                      |
      MinMax                                                                      |
        Condition: true                                                           |
        Parts: 55/55                                                              |
        Granules: 1228/1228                                                       |
      Partition                                                                   |
        Condition: true                                                           |
        Parts: 55/55                                                              |
        Granules: 1228/1228                                                       |
      PrimaryKey                                                                  |
        Keys:                                                                     |
          product_id                                                              |
        Condition: and((product_id in (-Inf, 1000]), (product_id in [1000, +Inf)))|
        Parts: 11/55                                                              |
        Granules: 11/1228                                                         |
```

### 10. Реализована автоматическая миграция данных: записи заказов старше 12 месяцев перемещаются на более экономичные, но медленные диски

```sql
DROP TABLE IF EXISTS learn_db.orders_partition_by_year;
CREATE TABLE learn_db.orders_partition_by_year (
	order_id UInt32,
	user_id UInt32,
	product_id UInt32,
	amount Decimal(18, 2),
	order_date Date
)
ENGINE = MergeTree()
PARTITION BY toStartOfMonth(order_date)
ORDER BY (product_id)
TTL order_date + INTERVAL 12 MONTH TO VOLUME 'slow_but_cheap';
```

### 11. Реализовано автоматическое удаление данных: записи заказов старше 12 месяцев удаляются

```sql
DROP TABLE IF EXISTS learn_db.orders_partition_by_year;
CREATE TABLE learn_db.orders_partition_by_year (
	order_id UInt32,
	user_id UInt32,
	product_id UInt32,
	amount Decimal(18, 2),
	order_date Date
)
ENGINE = MergeTree()
PARTITION BY toStartOfMonth(order_date)
ORDER BY (product_id)
TTL order_date + INTERVAL 12 MONTH DELETE;
```

## Управление партициями и частями данных

### 12. Получаем список всех партиций
```
SELECT
    partition,
    count() AS parts,
    sum(rows) AS rows
FROM system.parts
WHERE (database = 'learn_db') AND (`table` = 'orders_partition_by_year') AND active
GROUP BY partition
ORDER BY partition ASC;
```

Результат
```text
partition|parts|rows   |
---------+-----+-------+
2015     |    5| 266179|
2016     |    5|1003162|
2017     |    5| 999904|
2018     |    5| 999471|
2019     |    5| 999824|
2020     |    5|1003033|
2021     |    5| 999162|
2022     |    5| 998693|
2023     |    5| 999337|
2024     |    5| 999945|
2025     |    5| 731290|
```

### 13. Получаем список всех активных частей данных из партиций за 2015 год

```sql
SELECT
    *
FROM system.parts
WHERE (database = 'learn_db') AND (`table` = 'orders_partition_by_year')
	AND `partition` = '2015'
	AND active = true;
```

Результат
```text
partition|name          |uuid                                |part_type|active|marks|rows  |bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table                   |engine   |disk_name|path                                                                              |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock|removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+--------------+------------------------------------+---------+------+-----+------+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+------------------------+---------+---------+----------------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+----------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
2015     |2015_7_66_1   |00000000-0000-0000-0000-000000000000|Compact  |     1|   22|169600|      2440874|              2439943|                3731200|             111|        422|                                 0|                                   0|                            0|2025-09-26 22:04:06|1970-01-01 03:00:00|       1|2015-09-26|2015-12-31|1970-01-01 03:00:00|1970-01-01 03:00:00|2015        |               7|              66|    1|           7|                         88|                                  216|                               25|                                         25|        0|learn_db|orders_partition_by_year|MergeTree|default  |/var/lib/clickhouse/store/bb3/bb3356d9-c090-4621-859f-a468ec177666/2015_7_66_1/   |c75fae5e2f4fcb3ec7c8cfa9ca9977bf|68b125b938bcb527c565b06f8ec7f8f0|da16b7dcd149dc1f4bca3f380fec45d5     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
2015     |2015_70_70_0  |00000000-0000-0000-0000-000000000000|Compact  |     1|    4| 27897|       408288|               407705|                 613734|              50|        141|                                 0|                                   0|                            0|2025-09-26 22:04:06|1970-01-01 03:00:00|       1|2015-09-26|2015-12-31|1970-01-01 03:00:00|1970-01-01 03:00:00|2015        |              70|              70|    0|          70|                         16|                                  144|                               25|                                         25|        0|learn_db|orders_partition_by_year|MergeTree|default  |/var/lib/clickhouse/store/bb3/bb3356d9-c090-4621-859f-a468ec177666/2015_70_70_0/  |90997b31dbbdae4f83d86478f89fa73d|fe668ff1d922a4ef21ed9b9dfa80bf96|e160de95f6b75169f6b6fe4cebe50c5b     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
2015     |2015_83_83_0  |00000000-0000-0000-0000-000000000000|Compact  |     1|    4| 28221|       415106|               414523|                 620862|              50|        141|                                 0|                                   0|                            0|2025-09-26 22:04:06|1970-01-01 03:00:00|       1|2015-09-26|2015-12-31|1970-01-01 03:00:00|1970-01-01 03:00:00|2015        |              83|              83|    0|          83|                         16|                                  144|                               25|                                         25|        0|learn_db|orders_partition_by_year|MergeTree|default  |/var/lib/clickhouse/store/bb3/bb3356d9-c090-4621-859f-a468ec177666/2015_83_83_0/  |165b9db0226d522ca687c6e79f3a8bcf|01c2057e893285ecc613810a46d483a3|1c481c357f10d23b76b6fed7e7ff818a     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
2015     |2015_95_95_0  |00000000-0000-0000-0000-000000000000|Compact  |     1|    4| 28177|       415835|               415252|                 619894|              50|        141|                                 0|                                   0|                            0|2025-09-26 22:04:06|1970-01-01 03:00:00|       1|2015-09-26|2015-12-31|1970-01-01 03:00:00|1970-01-01 03:00:00|2015        |              95|              95|    0|          95|                         16|                                  144|                               25|                                         25|        0|learn_db|orders_partition_by_year|MergeTree|default  |/var/lib/clickhouse/store/bb3/bb3356d9-c090-4621-859f-a468ec177666/2015_95_95_0/  |e728ce5b05d7a878d9161fdc3fe40093|e11f2ad5140cde8b2600cd9f7487bd2a|ca049d1a262e275ea314f3247fccc7f3     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
2015     |2015_109_109_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2| 12284|       183278|               182766|                 270248|              42|         78|                                 0|                                   0|                            0|2025-09-26 22:04:06|1970-01-01 03:00:00|       1|2015-09-26|2015-12-31|1970-01-01 03:00:00|1970-01-01 03:00:00|2015        |             109|             109|    0|         109|                          8|                                  136|                               25|                                         25|        0|learn_db|orders_partition_by_year|MergeTree|default  |/var/lib/clickhouse/store/bb3/bb3356d9-c090-4621-859f-a468ec177666/2015_109_109_0/|f55f030cff1ec2c7a863a9ee6b3c2f2c|dd7d488fe0d366ee7564151e617cd48c|14a46f1a912302e9709ea7533ac3e18c     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

### 14. Открепляем партицию за 2015 год

```sql
ALTER TABLE learn_db.orders_partition_by_year DETACH PARTITION '2015';
```

### 16. Прикрепляем открепленную партицию за 2015 год

```sql
ALTER TABLE learn_db.orders_partition_by_year ATTACH PARTITION 2015;
ALTER TABLE learn_db.orders_partition_by_year ATTACH PARTITION ALL;
```

### 16. Открепляем и прикрепляем обратно часть данных
```
ALTER TABLE learn_db.orders_partition_by_year DETACH PART '2015_112_112_0';
ALTER TABLE learn_db.orders_partition_by_year ATTACH PART '2015_112_112_0';
```

### 17. Открепляем партицию за 2015 год и затем удаляем ее

```sql
ALTER TABLE learn_db.orders_partition_by_year DETACH PARTITION '2015';
ALTER TABLE learn_db.orders_partition_by_year DROP DETACHED PARTITION '2015'
SETTINGS allow_drop_detached = 1;
```