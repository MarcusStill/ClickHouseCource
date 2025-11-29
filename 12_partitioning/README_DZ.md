## Задание 1: Создание двух версий таблиц продаж.

Создайте две таблицы с одинаковой структурой, но разными схемами партиционирования.

**Таблица sales_manager**:
* date (Date) – дата продажи
* product_id (UInt32) – идентификатор товара
* manager_id (UInt32) – идентификатор продавца
* quantity (UInt32) – количество проданных единиц
* price (Decimal(10, 2)) – цена за единицу
* total_amount (Decimal(10, 2)) – общая сумма продажи (quantity * price)

Движок таблицы: MergeTree. Первичный ключ по полю product_id. Партиционирование по полю manager_id.

**Таблица sales_month:**
Такие же поля как в sales_manager. Движок таблицы: MergeTree. Первичный ключ по полю product_id. Партиционирование по месяцу из даты продажи (toYYYYMM(date)).

#### Создаем таблицу sales_manager
```sql
DROP TABLE IF EXISTS learn_db.sales_manager;
CREATE TABLE learn_db.sales_manager (
	product_id UInt32,
	manager_id UInt32,
	quantity UInt32,
	price Decimal(10, 2),
	total_amount Decimal(10, 2) DEFAULT quantity * price,
	date Date,
	PRIMARY KEY(product_id)
)
ENGINE = MergeTree()
PARTITION BY product_id
ORDER BY (product_id);
```

#### Создаем таблицу sales_month
```sql
DROP TABLE IF EXISTS learn_db.sales_month;
CREATE TABLE learn_db.sales_month (
	product_id UInt32,
	manager_id UInt32,
	quantity UInt32,
	price Decimal(10, 2),
	total_amount Decimal(10, 2) DEFAULT quantity * price,
	date Date,
	PRIMARY KEY(product_id)
)
ENGINE = MergeTree()
PARTITION BY toYear(date)
ORDER BY (product_id);
```

## Задание 2: Генерация и вставка тестовых данных

1. Сгенерируйте и вставьте 100 000 строк тестовых данных в таблицу sales_manager

2. Скопируйте те же данные в таблицу sales_month

#### Генерируем тестовые данные

```sql
INSERT INTO learn_db.sales_manager (product_id, manager_id, quantity, price, date)
SELECT
    number as product_id,    
    rand() % 9 + 1 as manager_id,     
    rand() % 100 + 1 as quantity,
    round(randUniform(10, 1000), 2) as price,    
    today() - (rand() % 365) as date
FROM numbers(100000);
```

```sql
INSERT INTO learn_db.sales_month
SELECT * from learn_db.sales_manager
```

## Задание 3: Анализ структуры партиций

Для обеих таблиц выполните:
1. Определите количество партиций и частей данных
2. Сравните результаты между таблицами

#### Анализируем партиции и парты

```sql
SELECT * FROM system.parts where table = 'sales_manager';
```

Результат

```text
partition|name   |uuid                                |part_type|active|marks|rows |bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table        |engine   |disk_name|path                                                                       |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock|removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+-------+------------------------------------+---------+------+-----+-----+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+-------------+---------+---------+---------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+----------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
1        |1_9_9_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|11079|       202268|               201674|                 332370|              42|         83|                                 0|                                   0|                            0|2025-09-29 20:03:16|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|1           |               9|               9|    0|           9|                          8|                                  136|                               25|                                         25|        0|learn_db|sales_manager|MergeTree|default  |/var/lib/clickhouse/store/4a0/4a04fbe5-b4c9-42b2-a3da-5fafe4c8fbf6/1_9_9_0/|565b00ef87dc4b1f955320ca5e003096|9d055a2742a6a0aef731f85e7fde84f2|29350aad10964a3af9bb5ac1ce3be63f     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
2        |2_1_1_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|11003|       200925|               200330|                 330090|              42|         83|                                 0|                                   0|                            0|2025-09-29 20:03:16|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|2           |               1|               1|    0|           1|                          8|                                  136|                               25|                                         25|        0|learn_db|sales_manager|MergeTree|default  |/var/lib/clickhouse/store/4a0/4a04fbe5-b4c9-42b2-a3da-5fafe4c8fbf6/2_1_1_0/|4280f6c5e39f741d9734f1a169ac2b1b|cb984c732230214b03a2173f30b207e8|b7af9d8b221267ea189dc0f1e63f7df2     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
3        |3_4_4_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|11055|       201853|               201259|                 331650|              42|         83|                                 0|                                   0|                            0|2025-09-29 20:03:16|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|3           |               4|               4|    0|           4|                          8|                                  136|                               25|                                         25|        0|learn_db|sales_manager|MergeTree|default  |/var/lib/clickhouse/store/4a0/4a04fbe5-b4c9-42b2-a3da-5fafe4c8fbf6/3_4_4_0/|01e4653bec5de0a44f73bac48b9f1d1d|0865d9228d62f8d4ba2fc8e9aa43bb39|6c41e813d698762654a0930641921acb     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
4        |4_5_5_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|11180|       204005|               203411|                 335400|              42|         83|                                 0|                                   0|                            0|2025-09-29 20:03:16|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|4           |               5|               5|    0|           5|                          8|                                  136|                               25|                                         25|        0|learn_db|sales_manager|MergeTree|default  |/var/lib/clickhouse/store/4a0/4a04fbe5-b4c9-42b2-a3da-5fafe4c8fbf6/4_5_5_0/|965bc39300bb2a78d7b2ca2a1b836698|e9ee50f58c0e11807231dfc4effe6946|7a058c7693a04849ba346ddc81888019     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
5        |5_2_2_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|11143|       203405|               202811|                 334290|              42|         83|                                 0|                                   0|                            0|2025-09-29 20:03:16|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|5           |               2|               2|    0|           2|                          8|                                  136|                               25|                                         25|        0|learn_db|sales_manager|MergeTree|default  |/var/lib/clickhouse/store/4a0/4a04fbe5-b4c9-42b2-a3da-5fafe4c8fbf6/5_2_2_0/|250f4751a4a2e4cc893e06accd614319|61ed2c022a14147972b7e297ba6ad46b|a31e3616f1c552c84ecc8dd6f1638ad0     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
6        |6_3_3_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|11200|       204622|               204028|                 336000|              42|         83|                                 0|                                   0|                            0|2025-09-29 20:03:16|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|6           |               3|               3|    0|           3|                          8|                                  136|                               25|                                         25|        0|learn_db|sales_manager|MergeTree|default  |/var/lib/clickhouse/store/4a0/4a04fbe5-b4c9-42b2-a3da-5fafe4c8fbf6/6_3_3_0/|e98b5159b336cd6c3bddfdf40794f078|5ad0a68baf149b105caf9a4433c354a3|11b168d89deb782c38406a61ed95f35c     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
7        |7_8_8_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|11057|       201884|               201290|                 331710|              42|         83|                                 0|                                   0|                            0|2025-09-29 20:03:16|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|7           |               8|               8|    0|           8|                          8|                                  136|                               25|                                         25|        0|learn_db|sales_manager|MergeTree|default  |/var/lib/clickhouse/store/4a0/4a04fbe5-b4c9-42b2-a3da-5fafe4c8fbf6/7_8_8_0/|d6d9a7f5b1205da37dfb53cd4c89bc13|8bfec372bdd64afe09dda70ecd9e4f9c|42f05d5f3fc9e91493496f08a4320190     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
8        |8_6_6_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|11188|       204126|               203532|                 335640|              42|         83|                                 0|                                   0|                            0|2025-09-29 20:03:16|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|8           |               6|               6|    0|           6|                          8|                                  136|                               25|                                         25|        0|learn_db|sales_manager|MergeTree|default  |/var/lib/clickhouse/store/4a0/4a04fbe5-b4c9-42b2-a3da-5fafe4c8fbf6/8_6_6_0/|5b4c78620ca9b0439af244a1e3d07804|fd43b202483e49fb65f5c2a04daf2a92|802b89278fc766e33ce0cb8940a49db3     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
9        |9_7_7_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    2|11095|       202582|               201991|                 332850|              42|         80|                                 0|                                   0|                            0|2025-09-29 20:03:16|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|9           |               7|               7|    0|           7|                          8|                                  136|                               25|                                         25|        0|learn_db|sales_manager|MergeTree|default  |/var/lib/clickhouse/store/4a0/4a04fbe5-b4c9-42b2-a3da-5fafe4c8fbf6/9_7_7_0/|8de933096d19db160dcf4d6e39ef2ce8|244b46d0c5a2acdb089acc0eebf03e47|ce7b0756f583bef22a7c64e51c6aaec9     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

```sql
SELECT * FROM system.parts where table = 'sales_month';
```

Результат

```text
partition|name      |uuid                                |part_type|active|marks|rows |bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table      |engine   |disk_name|path                                                                          |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock|removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+----------+------------------------------------+---------+------+-----+-----+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+-----------+---------+---------+------------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+----------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
2024     |2024_2_2_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    4|25501|       521256|               520587|                 765030|              50|        156|                                 0|                                   0|                            0|2025-09-29 20:03:42|1970-01-01 03:00:00|       1|2024-09-30|2024-12-31|1970-01-01 03:00:00|1970-01-01 03:00:00|2024        |               2|               2|    0|           2|                         16|                                  144|                               25|                                         25|        0|learn_db|sales_month|MergeTree|default  |/var/lib/clickhouse/store/57d/57d2cd44-81c8-4d10-94d0-1a51c1805b36/2024_2_2_0/|1cbcb2210046cbf752d21455a039af9e|280d05d2707e83eefdc74a27a931d890|60ee885c3451449c13deeaec388118b9     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
2024     |2024_3_3_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    4|25501|       521256|               520587|                 765030|              50|        156|                                 0|                                   0|                            0|2025-09-29 20:58:41|1970-01-01 03:00:00|       1|2024-09-30|2024-12-31|1970-01-01 03:00:00|1970-01-01 03:00:00|2024        |               3|               3|    0|           3|                         16|                                  144|                               25|                                         25|        0|learn_db|sales_month|MergeTree|default  |/var/lib/clickhouse/store/57d/57d2cd44-81c8-4d10-94d0-1a51c1805b36/2024_3_3_0/|1cbcb2210046cbf752d21455a039af9e|280d05d2707e83eefdc74a27a931d890|60ee885c3451449c13deeaec388118b9     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
2025     |2025_1_1_0|00000000-0000-0000-0000-000000000000|Compact  |     1|   10|74499|      1546352|              1545510|                2234970|              74|        305|                                 0|                                   0|                            0|2025-09-29 20:03:42|1970-01-01 03:00:00|       1|2025-01-01|2025-09-29|1970-01-01 03:00:00|1970-01-01 03:00:00|2025        |               1|               1|    0|           1|                         40|                                  168|                               25|                                         25|        0|learn_db|sales_month|MergeTree|default  |/var/lib/clickhouse/store/57d/57d2cd44-81c8-4d10-94d0-1a51c1805b36/2025_1_1_0/|3ac4c01957549d6b1a211c02028b7f08|004d3fd3353ddb1302b4d22eca00016b|150dac48599b6042e8f6e1ad9bb8665d     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
2025     |2025_4_4_0|00000000-0000-0000-0000-000000000000|Compact  |     1|   10|74499|      1546352|              1545510|                2234970|              74|        305|                                 0|                                   0|                            0|2025-09-29 20:58:41|1970-01-01 03:00:00|       1|2025-01-01|2025-09-29|1970-01-01 03:00:00|1970-01-01 03:00:00|2025        |               4|               4|    0|           4|                         40|                                  168|                               25|                                         25|        0|learn_db|sales_month|MergeTree|default  |/var/lib/clickhouse/store/57d/57d2cd44-81c8-4d10-94d0-1a51c1805b36/2025_4_4_0/|3ac4c01957549d6b1a211c02028b7f08|004d3fd3353ddb1302b4d22eca00016b|150dac48599b6042e8f6e1ad9bb8665d     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

## Задание 4: Сравнение производительности

Выполните и сравните:
1. Запрос на выборку всех строк с product_id от 1 до 10 из sales_manager
2. Такой же запрос к таблице sales_month
Зафиксируйте время выполнения каждого запроса

#### Делаем запрос к таблице sales_manager

```sql
SELECT * FROM learn_db.sales_manager WHERE product_id BETWEEN 1 AND 10;
```

Результат

```text
Query id: 4c7f11ec-4dab-4630-8007-2a7e1e648f06

    ┌─product_id─┬─manager_id─┬─quantity─┬──price─┬─total_amount─┬───────date─┐
 1. │          9 │          2 │       34 │ 376.61 │     12804.74 │ 2025-03-10 │
 2. │          5 │          6 │       80 │ 307.93 │      24634.4 │ 2025-02-07 │
 3. │          1 │          5 │       23 │ 849.89 │     19547.47 │ 2025-03-06 │
 4. │          7 │          5 │       62 │ 232.81 │     14434.22 │ 2025-04-21 │
 5. │          3 │          3 │       21 │ 365.73 │      7680.33 │ 2024-10-14 │
 6. │          4 │          3 │       31 │ 403.27 │     12501.37 │ 2025-01-27 │
 7. │         10 │          7 │       33 │ 158.54 │      5231.82 │ 2025-03-26 │
 8. │          2 │          4 │       42 │ 441.99 │     18563.58 │ 2024-10-28 │
 9. │          8 │          4 │       25 │ 492.94 │      12323.5 │ 2025-08-01 │
10. │          6 │          9 │        7 │ 132.08 │       924.56 │ 2025-02-05 │
    └────────────┴────────────┴──────────┴────────┴──────────────┴────────────┘

10 rows in set. Elapsed: 0.008 sec. Processed 88.88 thousand rows, 2.38 MB (10.97 million rows/s., 294.04 MB/s.)
Peak memory usage: 381.57 KiB.
```

#### Делаем запрос к таблице sales_month

```sql
SELECT * FROM learn_db.sales_month WHERE product_id BETWEEN 1 AND 10;
```

Результат
```text
Query id: bbf5d360-b8e3-4d8b-9874-35f3961eccb4

    ┌─product_id─┬─manager_id─┬─quantity─┬──price─┬─total_amount─┬───────date─┐
 1. │          1 │          5 │       23 │ 849.89 │     19547.47 │ 2025-03-06 │
 2. │          4 │          3 │       31 │ 403.27 │     12501.37 │ 2025-01-27 │
 3. │          5 │          6 │       80 │ 307.93 │      24634.4 │ 2025-02-07 │
 4. │          6 │          9 │        7 │ 132.08 │       924.56 │ 2025-02-05 │
 5. │          7 │          5 │       62 │ 232.81 │     14434.22 │ 2025-04-21 │
 6. │          8 │          4 │       25 │ 492.94 │      12323.5 │ 2025-08-01 │
 7. │          9 │          2 │       34 │ 376.61 │     12804.74 │ 2025-03-10 │
 8. │         10 │          7 │       33 │ 158.54 │      5231.82 │ 2025-03-26 │
 9. │          2 │          4 │       42 │ 441.99 │     18563.58 │ 2024-10-28 │
10. │          3 │          3 │       21 │ 365.73 │      7680.33 │ 2024-10-14 │
    └────────────┴────────────┴──────────┴────────┴──────────────┴────────────┘

10 rows in set. Elapsed: 0.005 sec. Processed 16.38 thousand rows, 491.52 KB (3.50 million rows/s., 104.94 MB/s.)
Peak memory usage: 62.05 KiB.
```

## Задание 4: Сравнение производительности
1. Создайте таблицу sales_month_backup с такой же структурой и схемой партиционирования как sales_month.
2. Определите 3 последние партиции в sales_month (по дате)
3. Скопируйте эти партиции в sales_month_backup
4. Убедитесь, что исходные данные в sales_month остались без изменений
5. Проверьте содержимое таблицы sales_month_backup

#### Создаем таблицу sales_month_backup

```sql
DROP TABLE IF EXISTS learn_db.sales_month_backup;
CREATE TABLE learn_db.sales_month_backup (
	product_id UInt32,
	manager_id UInt32,
	quantity UInt32,
	price Decimal(10, 2),
	total_amount Decimal(10, 2) DEFAULT quantity * price,
	date Date,
	PRIMARY KEY(product_id)
)
ENGINE = MergeTree()
PARTITION BY toYear(date)
ORDER BY (product_id);
```

#### Определяем 3 последние партиции

```sql
SELECT * FROM system.parts where table = 'sales_month';
```

Результат

```text
partition|name      |uuid                                |part_type|active|marks|rows |bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table      |engine   |disk_name|path                                                                          |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock|removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+----------+------------------------------------+---------+------+-----+-----+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+-----------+---------+---------+------------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+----------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
2024     |2024_2_2_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    4|25584|       523549|               522880|                 767520|              50|        156|                                 0|                                   0|                            0|2025-09-29 21:13:04|1970-01-01 03:00:00|       1|2024-09-30|2024-12-31|1970-01-01 03:00:00|1970-01-01 03:00:00|2024        |               2|               2|    0|           2|                         16|                                  144|                               25|                                         25|        0|learn_db|sales_month|MergeTree|default  |/var/lib/clickhouse/store/cb3/cb3308d3-4d1a-438c-8fa9-a330a2930fda/2024_2_2_0/|f42a24f08ad33a1f8b03a4767a513b46|0ca110103291c84d236d6955a0614194|97f9a57b5ba5f6ae6397fa03e7374ac2     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
2025     |2025_1_1_0|00000000-0000-0000-0000-000000000000|Compact  |     1|   10|74416|      1544378|              1543535|                2232480|              74|        305|                                 0|                                   0|                            0|2025-09-29 21:13:04|1970-01-01 03:00:00|       1|2025-01-01|2025-09-29|1970-01-01 03:00:00|1970-01-01 03:00:00|2025        |               1|               1|    0|           1|                         40|                                  168|                               25|                                         25|        0|learn_db|sales_month|MergeTree|default  |/var/lib/clickhouse/store/cb3/cb3308d3-4d1a-438c-8fa9-a330a2930fda/2025_1_1_0/|9f20572c24b5c76e493cff46ca5beea0|fd69b651cb013687744ce1a43388284a|2ad1920cf2d6f70bd90bc9aa5451ef77     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

#### Копируем эти партиции в sales_month_backup

```sql
ALTER TABLE learn_db.sales_month_backup ATTACH PARTITION 2024 FROM learn_db.sales_month;
ALTER TABLE learn_db.sales_month_backup ATTACH PARTITION 2025 FROM learn_db.sales_month;
```

#### Проверяем исходные данные в sales_month

```sql
SELECT * FROM system.parts where table = 'sales_month';
```

Результат

```text
partition|name      |uuid                                |part_type|active|marks|rows |bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table      |engine   |disk_name|path                                                                          |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock|removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+----------+------------------------------------+---------+------+-----+-----+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+-----------+---------+---------+------------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+----------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
2024     |2024_2_2_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    4|25584|       523549|               522880|                 767520|              50|        156|                                 0|                                   0|                            0|2025-09-29 21:13:04|1970-01-01 03:00:00|       1|2024-09-30|2024-12-31|1970-01-01 03:00:00|1970-01-01 03:00:00|2024        |               2|               2|    0|           2|                         16|                                  144|                               25|                                         25|        0|learn_db|sales_month|MergeTree|default  |/var/lib/clickhouse/store/cb3/cb3308d3-4d1a-438c-8fa9-a330a2930fda/2024_2_2_0/|f42a24f08ad33a1f8b03a4767a513b46|0ca110103291c84d236d6955a0614194|97f9a57b5ba5f6ae6397fa03e7374ac2     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
2025     |2025_1_1_0|00000000-0000-0000-0000-000000000000|Compact  |     1|   10|74416|      1544378|              1543535|                2232480|              74|        305|                                 0|                                   0|                            0|2025-09-29 21:13:04|1970-01-01 03:00:00|       1|2025-01-01|2025-09-29|1970-01-01 03:00:00|1970-01-01 03:00:00|2025        |               1|               1|    0|           1|                         40|                                  168|                               25|                                         25|        0|learn_db|sales_month|MergeTree|default  |/var/lib/clickhouse/store/cb3/cb3308d3-4d1a-438c-8fa9-a330a2930fda/2025_1_1_0/|9f20572c24b5c76e493cff46ca5beea0|fd69b651cb013687744ce1a43388284a|2ad1920cf2d6f70bd90bc9aa5451ef77     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

```sql
SELECT count(*) FROM learn_db.sales_month;
```

Результат

```text
count()|
-------+
 100000|
```

#### Проверьте содержимое таблицы sales_month_backup

```sql
SELECT * FROM system.parts where table = 'sales_month_backup';
```

Результат
```text
partition|name      |uuid                                |part_type|active|marks|rows |bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table             |engine   |disk_name|path                                                                          |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections|visible|creation_tid                                      |removal_tid_lock|removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+----------+------------------------------------+---------+------+-----+-----+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+------------------+---------+---------+------------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+-----------+-------+--------------------------------------------------+----------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
2024     |2024_1_1_0|00000000-0000-0000-0000-000000000000|Compact  |     1|    4|25584|       523549|               522880|                 767520|              50|        156|                                 0|                                   0|                            0|2025-09-29 21:36:46|1970-01-01 03:00:00|       1|2024-09-30|2024-12-31|1970-01-01 03:00:00|1970-01-01 03:00:00|2024        |               1|               1|    0|           1|                          0|                                    0|                               25|                                         25|        0|learn_db|sales_month_backup|MergeTree|default  |/var/lib/clickhouse/store/e87/e8754759-f922-4651-85a5-7592a0003f63/2024_1_1_0/|f42a24f08ad33a1f8b03a4767a513b46|0ca110103291c84d236d6955a0614194|97f9a57b5ba5f6ae6397fa03e7374ac2     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
2025     |2025_2_2_0|00000000-0000-0000-0000-000000000000|Compact  |     1|   10|74416|      1544378|              1543535|                2232480|              74|        305|                                 0|                                   0|                            0|2025-09-29 21:36:49|1970-01-01 03:00:00|       1|2025-01-01|2025-09-29|1970-01-01 03:00:00|1970-01-01 03:00:00|2025        |               2|               2|    0|           2|                          0|                                    0|                               25|                                         25|        0|learn_db|sales_month_backup|MergeTree|default  |/var/lib/clickhouse/store/e87/e8754759-f922-4651-85a5-7592a0003f63/2025_2_2_0/|9f20572c24b5c76e493cff46ca5beea0|fd69b651cb013687744ce1a43388284a|2ad1920cf2d6f70bd90bc9aa5451ef77     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |[]         |      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

```sql
SELECT count(*) FROM learn_db.sales_month_backup;
```

Результат
```text
count()|
-------+
 100000|
```