# ClickHouse: Performance Optimization & Real-Time Analytics

**Практическое погружение в ClickHouse.**  
Этот репозиторий — пример моих навыков в оптимизации запросов, настройке MergeTree-движков, построении real-time витрин и работе с распределёнными кластерами.

## 📁 Структура репозитория

| Раздел | Темы |
|--------|------|
| **Основы** | [`01_install`](01_install) — установка |
| **Оптимизация** | [`02_fast_optimization`](02_fast_optimization), [`03_merge_tree+parts`](03_merge_tree+parts), [`09_primary_index`](09_primary_index), [`12_partitioning`](12_partitioning), [`13_data_skipping_indexes`](13_data_skipping_indexes), [`14_explain`](14_explain), [`15_projections`](15_projections), [`18_compression`](18_compression) |
| **Движки MergeTree** | [`04_collapsing`](04_collapsing_merge_tree), [`05_versioned_collapsing`](05_versioned_collapsing_merge_tree), [`06_replacing`](06_replacing_merge_tree), [`07_summing`](07_summing_merge_tree), [`10_aggregating`](10_aggregating_merge_tree) |
| **Real-time & витрины** | [`11_incremental_materialized_view`](11_incremental_materialized_view) |
| **Мутации** | [`08_editing_data_in_clickhouse`](08_editing_data_in_clickhouse) |
| **Кластер** | [`19_cluster_part_1`](19_cluster_part_1), [`20_cluster_part_2_server_config`](20_cluster_part_2_server_config), [`21_replicated_merge_tree`](21_replicated_merge_tree), [`22_distributed_table_engine`](22_distributed_table_engine), [`23_distributed_table_engine_part_2`](23_distributed_table_engine_part_2) |
| **Доп. движки** | [`16_buffer_table_engine`](16_buffer_table_engine), [`17_join_table_engine`](17_join_table_engine) |
| **Мониторинг** | [`24_working_with_metrics`](24_working_with_metrics) |


## 🎯 Что я умею (доказано кодом)

| Навык | Что реализовано | Папка |
|-------|----------------|-------|
| **EXPLAIN и профилирование** | Анализ планов запросов, поиск узких мест | [`14_explain`](14_explain) |
| **Первичный индекс** | Настройка индексов для ускорения фильтрации | [`09_primary_index`](09_primary_index) |
| **Партиционирование** | Дата-партиции, управление частями данных | [`12_partitioning`](12_partitioning) |
| **Data Skipping Indexes** | Граннулярные и блум-фильтр индексы | [`13_data_skipping_indexes`](13_data_skipping_indexes) |
| **Проекции** | Автоматическая агрегация при вставке | [`15_projections`](15_projections) |
| **Сжатие** | Выбор алгоритмов сжатия (LZ4, ZSTD) | [`18_compression`](18_compression) |
| **CollapsingMergeTree** | Работа с часто меняющимися строками | [`04_collapsing_merge_tree`](04_collapsing_merge_tree) |
| **VersionedCollapsingMergeTree** | То же, но с версионированием | [`05_versioned_collapsing_merge_tree`](05_versioned_collapsing_merge_tree) |
| **ReplacingMergeTree** | Дедупликация по ключу | [`06_replacing_merge_tree`](06_replacing_merge_tree) |
| **SummingMergeTree** | Предрасчёт сумм для агрегаций | [`07_summing_merge_tree`](07_summing_merge_tree) |
| **AggregatingMergeTree** | Хранение промежуточных состояний агрегатов | [`10_aggregating_merge_tree`](10_aggregating_merge_tree) |
| **Инкрементальные материализованные представления** | Real-time обновление витрин для дашбордов | [`11_incremental_materialized_view`](11_incremental_materialized_view) |
| **Мутации (UPDATE/DELETE)** | Эффективное изменение больших данных | [`08_editing_data_in_clickhouse`](08_editing_data_in_clickhouse) |
| **Распределённые таблицы** | Шардирование данных по кластеру | [`22_distributed_table_engine`](22_distributed_table_engine) |
| **Реплицируемые таблицы** | Отказоустойчивость (ReplicatedMergeTree) | [`21_replicated_merge_tree`](21_replicated_merge_tree) |
| **Настройка кластера** | Конфигурация серверов, ZooKeeper | [`19_cluster_part_1`](19_cluster_part_1) |
| **Buffer/Join движки** | Оптимизация для частых вставок/джойнов | [`16_buffer_table_engine`](16_buffer_table_engine) |


## 🚀 Пример: реальная оптимизация

Один из кейсов — ускорение дашборда с 8 секунд до 0.3 секунды:

```sql
-- До: полное сканирование
SELECT sum(amount) FROM sales WHERE date = '2025-03-15'  -- 8.4s

-- После: партиции + проекция
CREATE PROJECTION sales_projection AS 
SELECT date, sum(amount) GROUP BY date;
-- Запрос идёт в проекцию: 0.3s
