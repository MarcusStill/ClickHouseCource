# Сжатие и кодирование данных

### 1. Скачиваем набор данных

Источник: https://clickhouse.com/blog/real-world-data-noaa-climate-data

### 2. Создаем папку и переходим в нее

```bash
mkdir weather_data
cd weather_data
```

### 3. Внутри контейнера загружаем файлы

```bash
for i in {1900..1930}; do wget https://noaa-ghcn-pds.s3.amazonaws.com/csv.gz/${i}.csv.gz; done
```

### 4. Объединяем части в один файл

```bash
for i in {1900..1930}
do
clickhouse-local --query "SELECT station_id,
       toDate32(date) as date,
       anyIf(value, measurement = 'TAVG') as tempAvg,
       anyIf(value, measurement = 'TMAX') as tempMax,
       anyIf(value, measurement = 'TMIN') as tempMin,
       anyIf(value, measurement = 'PRCP') as precipitation,
       anyIf(value, measurement = 'SNOW') as snowfall,
       anyIf(value, measurement = 'SNWD') as snowDepth,
       anyIf(value, measurement = 'PSUN') as percentDailySun,
       anyIf(value, measurement = 'AWND') as averageWindSpeed,
       anyIf(value, measurement = 'WSFG') as maxWindSpeed,
       toUInt8OrZero(replaceOne(anyIf(measurement, startsWith(measurement, 'WT') AND value = 1), 'WT', '')) as weatherType
FROM file('$i.csv.gz', CSV, 'station_id String, date String, measurement String, value Int64, mFlag String, qFlag String, sFlag String, obsTime String')
WHERE qFlag = ''
GROUP BY station_id, date
ORDER BY station_id, date FORMAT TSV" >> "noaa.tsv";
done
```

### 5. Скачиваем еще один файл

```bash
wget http://noaa-ghcn-pds.s3.amazonaws.com/ghcnd-stations.txt
```

### 6. Обогащаем данные

```bash
clickhouse-local --query "WITH stations AS (SELECT id, lat, lon, elevation, name FROM file('ghcnd-stations.txt', Regexp, 'id String, lat Float64, lon Float64, elevation Float32, name String'))
SELECT station_id,
       date,
       tempAvg,
       tempMax,
       tempMin,
       precipitation,
       snowfall,
       snowDepth,
       percentDailySun,
       averageWindSpeed,
       maxWindSpeed,
       weatherType,
       tuple(lon, lat) as location,
       elevation,
       name
FROM file('noaa.tsv', TSV,
          'station_id String, date Date32, tempAvg Int32, tempMax Int32, tempMin Int32, precipitation Int32, snowfall Int32, snowDepth Int32, percentDailySun Int8, averageWindSpeed Int32, maxWindSpeed Int32, weatherType UInt8') as noaa LEFT OUTER
         JOIN stations ON noaa.station_id = stations.id FORMAT TSV SETTINGS format_regexp='^(.{11})\s+(\-?\d{1,2}\.\d{4})\s+(\-?\d{1,3}\.\d{1,4})\s+(\-?\d*\.\d*)\s+(.*?)\s+.*$'" > noaa_enriched.tsv
```

### 7. Создаем таблицу
```sql
CREATE TABLE learn_db.noaa
(
   `station_id` LowCardinality(String),
   `date` Date32,
   `tempAvg` Int32 COMMENT 'Average temperature (tenths of a degrees C)',
   `tempMax` Int32 COMMENT 'Maximum temperature (tenths of degrees C)',
   `tempMin` Int32 COMMENT 'Minimum temperature (tenths of degrees C)',
   `precipitation` UInt32 COMMENT 'Precipitation (tenths of mm)',
   `snowfall` UInt32 COMMENT 'Snowfall (mm)',
   `snowDepth` UInt32 COMMENT 'Snow depth (mm)',
   `percentDailySun` UInt8 COMMENT 'Daily percent of possible sunshine (percent)',
   `averageWindSpeed` UInt32 COMMENT 'Average daily wind speed (tenths of meters per second)',
   `maxWindSpeed` UInt32 COMMENT 'Peak gust wind speed (tenths of meters per second)',
   `weatherType` Enum8('Normal' = 0, 'Fog' = 1, 'Heavy Fog' = 2, 'Thunder' = 3, 'Small Hail' = 4, 'Hail' = 5, 'Glaze' = 6, 'Dust/Ash' = 7, 'Smoke/Haze' = 8, 'Blowing/Drifting Snow' = 9, 'Tornado' = 10, 'High Winds' = 11, 'Blowing Spray' = 12, 'Mist' = 13, 'Drizzle' = 14, 'Freezing Drizzle' = 15, 'Rain' = 16, 'Freezing Rain' = 17, 'Snow' = 18, 'Unknown Precipitation' = 19, 'Ground Fog' = 21, 'Freezing Fog' = 22),
   `location` Point,
   `elevation` Float32,
   `name` LowCardinality(String)
) ENGINE = MergeTree() ORDER BY (station_id, date);
```

### 8. Наполняем таблицу данными

```sql
INSERT INTO learn_db.noaa(
    station_id, 
    date, 
    tempAvg, 
    tempMax, 
    tempMin, 
    precipitation, 
    snowfall, 
    snowDepth, 
    percentDailySun, 
    averageWindSpeed, 
    maxWindSpeed, 
    weatherType, 
    location, 
    elevation, 
    name
)
FROM
	INFILE '/weather_data/noaa_enriched.tsv' FORMAT TSV;
```

Результат
```text
Query id: 481cc477-b677-4449-beaf-07072c52c866

Ok.

131881032 rows in set. Elapsed: 50.762 sec. Processed 131.88 million rows, 7.93 GB (2.60 million rows/s., 156.13 MB/s.)
Peak memory usage: 228.29 MiB.
```

### 9. Запрос для проверки объема читаемых данных

```sql
SELECT
    tempMax / 10 AS maxTemp,
    location,
    name,
    date
FROM learn_db.noaa
WHERE tempMax > 500
ORDER BY
    tempMax DESC,
    date ASC
LIMIT 5;
```

Результат
```text
Query id: b255492e-2d9a-4337-b4be-f92d00bb7606

   ┌─maxTemp─┬─location──────────┬─name─┬───────date─┐
1. │    56.7 │ (-116.8667,36.45) │ CA   │ 1913-07-10 │
2. │      55 │ (-116.8667,36.45) │ CA   │ 1913-07-13 │
3. │    54.4 │ (-116.8667,36.45) │ CA   │ 1913-07-12 │
4. │    53.9 │ (-116.8667,36.45) │ CA   │ 1913-07-09 │
5. │    53.9 │ (-116.8667,36.45) │ CA   │ 1913-07-11 │
   └─────────┴───────────────────┴──────┴────────────┘

5 rows in set. Elapsed: 0.142 sec. Processed 124.54 million rows, 1.49 GB (874.58 million rows/s., 10.47 GB/s.)
Peak memory usage: 1.05 MiB.
```

## Экспериментируем с кодеками и сжатием

### 10. Создаем таблицу с первыми вариантами кодеков и сжатия и наполняем ее данными

```sql
DROP TABLE IF EXISTS learn_db.noaa_codec_v1;
CREATE TABLE learn_db.noaa_codec_v1
(
   `station_id` String COMMENT 'Id of the station at which the measurement as taken',
   `date` Date32,
   `tempAvg` Int64 COMMENT 'Average temperature (tenths of a degrees C)',
   `tempMax` Int64 COMMENT 'Maximum temperature (tenths of degrees C)',
   `tempMin` Int64 COMMENT 'Minimum temperature (tenths of degrees C)',
   `precipitation` Int64 COMMENT 'Precipitation (tenths of mm)',
   `snowfall` Int64 COMMENT 'Snowfall (mm)',
   `snowDepth` Int64 COMMENT 'Snow depth (mm)',
   `percentDailySun` Int64 COMMENT 'Daily percent of possible sunshine (percent)',
   `averageWindSpeed` Int64 COMMENT 'Average daily wind speed (tenths of meters per second)',
   `maxWindSpeed` Int64 COMMENT 'Peak gust wind speed (tenths of meters per second)',
   `weatherType` String,
   `location` Point,
   `elevation` Float64,
   `name` String
) ENGINE = MergeTree() ORDER BY (station_id, date) AS
SELECT * FROM learn_db.noaa;
```

### 11. Смотрим на размер столбцов в 1 версии таблицы

```sql
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v1'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

Результат
```text
name            |compressed_size|uncompressed_size|ratio |compressed|uncompressed|
----------------+---------------+-----------------+------+----------+------------+
date            |164.00 MiB     |475.08 MiB       |   2.9| 171965493|   498154944|
precipitation   |121.16 MiB     |950.16 MiB       |  7.84| 127042190|   996309888|
tempMin         |104.20 MiB     |950.16 MiB       |  9.12| 109265741|   996309888|
tempMax         |102.53 MiB     |950.16 MiB       |  9.27| 107514380|   996309888|
location        |9.51 MiB       |1.86 GiB         |199.72|   9976915|  1992619776|
weatherType     |7.06 MiB       |830.97 MiB       |117.62|   7407792|   871334574|
station_id      |6.63 MiB       |1.39 GiB         |214.89|   6954418|  1494464832|
snowfall        |5.72 MiB       |14.30 MiB        |   2.5|   5995162|    14999356|
snowDepth       |4.90 MiB       |20.08 MiB        |   4.1|   5134577|    21057346|
elevation       |4.71 MiB       |950.16 MiB       |201.86|   4935749|   996309888|
name            |3.83 MiB       |796.14 MiB       |208.13|   4011065|   834813816|
tempAvg         |3.38 MiB       |9.87 MiB         |  2.92|   3548012|    10353424|
averageWindSpeed|59.04 KiB      |211.97 KiB       |  3.59|     60453|      217053|
maxWindSpeed    |59.04 KiB      |211.97 KiB       |  3.59|     60453|      217053|
percentDailySun |59.04 KiB      |211.97 KiB       |  3.59|     60453|      217053|
```

Колонка station_id сжата в 214 раз. В несжатом виде она занимает 1.39 GiB, а в сжатом 6.63 MiB.


### 12. Смотрим общий размер таблицы версии 1

```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v1'
```

Результат
```text
compressed_size|uncompressed_size|ratio|compressed|uncompressed|
---------------+-----------------+-----+----------+------------+
537.81 MiB     |9.06 GiB         |17.24| 563932853|  9723688779|
```

### 13. Выполняем диагностический запрос

```sql
SELECT
    elevation_range,
    uniq(station_id) AS num_stations,
    max(tempMax) / 10 AS max_temp,
    min(tempMin) / 10 AS min_temp,
    sum(precipitation) AS total_precipitation,
    avg(percentDailySun) AS avg_percent_sunshine,
    max(maxWindSpeed) AS max_wind_speed,
    sum(snowfall) AS total_snowfall
FROM learn_db.noaa_codec_v1
WHERE (date > '1907-01-01') AND (station_id IN ('AG000060590', 'ASN00075063', 3))
GROUP BY floor(elevation, -2) AS elevation_range
ORDER BY elevation_range ASC
```

Результат
```text
Query id: e0e3d003-7974-4a17-8231-7d72a787826d

   ┌─elevation_range─┬─num_stations─┬─max_temp─┬─min_temp─┬─total_precipitation─┬─avg_percent_sunshine─┬─max_wind_speed─┬─total_snowfall─┐
1. │               0 │            1 │        0 │        0 │               71234 │                    0 │              0 │              0 │
2. │             300 │            1 │       48 │       -5 │                 681 │                    0 │              0 │              0 │
   └─────────────────┴──────────────┴──────────┴──────────┴─────────────────────┴──────────────────────┴────────────────┴────────────────┘

2 rows in set. Elapsed: 0.029 sec. Processed 64.08 thousand rows, 2.27 MB (2.18 million rows/s., 77.41 MB/s.)
Peak memory usage: 4.10 MiB.
```

## Выбор оптимального типа данных

### 14. Смотрим максимальные и минимальные значений в столбцах. Выполнять в clickhouse-client

```sql
SELECT
    COLUMNS('Wind|temp|snow|pre') APPLY min,
    COLUMNS('Wind|temp|snow|pre') APPLY max
FROM learn_db.noaa
FORMAT Vertical
```

Результат
```text
Query id: bb648560-7cd0-40c8-8ef7-ff2c6e54038a

Row 1:
──────
min(tempAvg):          -644
min(tempMax):          -634
min(tempMin):          -669
min(precipitation):    0
min(snowfall):         0
min(snowDepth):        0
min(averageWindSpeed): 0
min(maxWindSpeed):     0
max(tempAvg):          396
max(tempMax):          567
max(tempMin):          433
max(precipitation):    9977
max(snowfall):         1600
max(snowDepth):        8306
max(averageWindSpeed): 0
max(maxWindSpeed):     0

1 row in set. Elapsed: 0.216 sec. Processed 124.54 million rows, 1.55 GB (576.82 million rows/s., 7.16 GB/s.)
Peak memory usage: 147.04 KiB.
```

Типы данных в таблице не оптимальны. Например, для tempAvg, tempMax, tempMin подойдет Int16. 

### 15. Создаем таблицу 2 версии

```sql
DROP TABLE IF EXISTS learn_db.noaa_codec_v2;
CREATE TABLE learn_db.noaa_codec_v2
(
  `station_id` String COMMENT 'Id of the station at which the measurement as taken',
  `date` Date32,
  `tempAvg` Int16 COMMENT 'Average temperature (tenths of a degrees C)',
  `tempMax` Int16 COMMENT 'Maximum temperature (tenths of degrees C)',
  `tempMin` Int16 COMMENT 'Minimum temperature (tenths of degrees C)',
  `precipitation` UInt16 COMMENT 'Precipitation (tenths of mm)',
  `snowfall` UInt16 COMMENT 'Snowfall (mm)',
  `snowDepth` UInt16 COMMENT 'Snow depth (mm)',
  `percentDailySun` UInt8 COMMENT 'Daily percent of possible sunshine (percent)',
  `averageWindSpeed` UInt16 COMMENT 'Average daily wind speed (tenths of meters per second)',
  `maxWindSpeed` UInt16 COMMENT 'Peak gust wind speed (tenths of meters per second)',
  `weatherType` String,
  `location` Point,
  `elevation` Int16,
  `name` String
) ENGINE = MergeTree() ORDER BY (station_id, date) AS 
SELECT * FROM learn_db.noaa;
```

### 16. Смотрим на размер столбцов во 2 версии таблицы

```sql
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v2'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

Результат
```text
name            |compressed_size|uncompressed_size|ratio |compressed|uncompressed|
----------------+---------------+-----------------+------+----------+------------+
date            |158.58 MiB     |475.08 MiB       |   3.0| 166285640|   498154944|
precipitation   |77.91 MiB      |237.54 MiB       |  3.05|  81690114|   249077472|
tempMin         |53.62 MiB      |237.54 MiB       |  4.43|  56228217|   249077472|
tempMax         |52.69 MiB      |237.54 MiB       |  4.51|  55246352|   249077472|
location        |9.54 MiB       |1.86 GiB         | 199.2|  10003314|  1992619776|
weatherType     |7.07 MiB       |830.97 MiB       |117.59|   7409872|   871334574|
station_id      |6.64 MiB       |1.39 GiB         |214.58|   6964512|  1494464832|
name            |3.83 MiB       |796.14 MiB       |207.89|   4015655|   834813816|
snowfall        |3.43 MiB       |4.96 MiB         |  1.45|   3599606|     5203771|
snowDepth       |2.75 MiB       |6.66 MiB         |  2.42|   2887592|     6984071|
tempAvg         |2.21 MiB       |3.43 MiB         |  1.55|   2317297|     3594358|
elevation       |1.27 MiB       |237.54 MiB       |186.89|   1332739|   249077472|
averageWindSpeed|54.92 KiB      |210.41 KiB       |  3.83|     56238|      215460|
maxWindSpeed    |54.92 KiB      |210.41 KiB       |  3.83|     56238|      215460|
percentDailySun |54.92 KiB      |210.41 KiB       |  3.83|     56238|      215460|
```

### 17. Смотрим общий размер таблицы версии 2

```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v2';
```

Результат
```text
compressed_size|uncompressed_size|ratio|compressed|uncompressed|
---------------+-----------------+-----+----------+------------+
379.71 MiB     |6.24 GiB         |16.84| 398149624|  6704126410|
```

### 18. Выполняем диагностические запросы

```sql
SELECT
    tempMax / 10 AS maxTemp,
    location,
    name,
    date
FROM learn_db.noaa
WHERE tempMax > 500
ORDER BY
    tempMax DESC,
    date ASC
LIMIT 5;
```

Результат
```text
Query id: e1752c58-1a76-45d2-8d3e-3b3b38988ce8

   ┌─maxTemp─┬─location──────────┬─name─┬───────date─┐
1. │    56.7 │ (-116.8667,36.45) │ CA   │ 1913-07-10 │
2. │      55 │ (-116.8667,36.45) │ CA   │ 1913-07-13 │
3. │    54.4 │ (-116.8667,36.45) │ CA   │ 1913-07-12 │
4. │    53.9 │ (-116.8667,36.45) │ CA   │ 1913-07-09 │
5. │    53.9 │ (-116.8667,36.45) │ CA   │ 1913-07-11 │
   └─────────┴───────────────────┴──────┴────────────┘

5 rows in set. Elapsed: 0.111 sec. Processed 124.54 million rows, 1.49 GB (1.12 billion rows/s., 13.42 GB/s.)
Peak memory usage: 109.68 KiB.
```

```sql
SELECT
    elevation_range,
    uniq(station_id) AS num_stations,
    max(tempMax) / 10 AS max_temp,
    min(tempMin) / 10 AS min_temp,
    sum(precipitation) AS total_precipitation,
    avg(percentDailySun) AS avg_percent_sunshine,
    max(maxWindSpeed) AS max_wind_speed,
    sum(snowfall) AS total_snowfall
FROM learn_db.noaa_codec_v2
WHERE (date > '1907-01-01') AND (station_id IN ('AG000060590', 'ASN00075063', 3))
GROUP BY floor(elevation, -2) AS elevation_range
ORDER BY elevation_range ASC
```

Результат
```text
Query id: ed0a1df3-d844-47e6-9d3b-1a0ee73ff5ed

   ┌─elevation_range─┬─num_stations─┬─max_temp─┬─min_temp─┬─total_precipitation─┬─avg_percent_sunshine─┬─max_wind_speed─┬─total_snowfall─┐
1. │               0 │            1 │        0 │        0 │               71234 │                    0 │              0 │              0 │
2. │             300 │            1 │       48 │       -5 │                 681 │                    0 │              0 │              0 │
   └─────────────────┴──────────────┴──────────┴──────────┴─────────────────────┴──────────────────────┴────────────────┴────────────────┘

2 rows in set. Elapsed: 0.015 sec. Processed 65.54 thousand rows, 1.77 MB (4.32 million rows/s., 116.96 MB/s.)
Peak memory usage: 4.69 MiB.
```

## Применяем LowCardinality и Enum

Изменим типы данных у атрибутов station_id и weatherType (в нем фиксированный набор строк). Так же в поле location у нас использовался тип данных Point.
По умолчанию в нем хранятся 2 значения с типом Float64. Диапазон значений у нас небольшой. И мы сделаем 2 поля lat и lon с типом Float32.
### 19. Создаем таблицы 3 версии

```sql
DROP TABLE IF EXISTS learn_db.noaa_codec_v3;
CREATE TABLE learn_db.noaa_codec_v3
(
 `station_id` LowCardinality(String) COMMENT 'Id of the station at which the measurement as taken',
 `date` Date32,
 `tempAvg` Int16 COMMENT 'Average temperature (tenths of a degrees C)',
 `tempMax` Int16 COMMENT 'Maximum temperature (tenths of degrees C)',
 `tempMin` Int16 COMMENT 'Minimum temperature (tenths of degrees C)',
 `precipitation` UInt16 COMMENT 'Precipitation (tenths of mm)',
 `snowfall` UInt16 COMMENT 'Snowfall (mm)',
 `snowDepth` UInt16 COMMENT 'Snow depth (mm)',
 `percentDailySun` UInt8 COMMENT 'Daily percent of possible sunshine (percent)',
 `averageWindSpeed` UInt16 COMMENT 'Average daily wind speed (tenths of meters per second)',
 `maxWindSpeed` UInt16 COMMENT 'Peak gust wind speed (tenths of meters per second)',
 `weatherType` Enum8('Normal' = 0, 'Fog' = 1, 'Heavy Fog' = 2, 'Thunder' = 3, 'Small Hail' = 4, 'Hail' = 5, 'Glaze' = 6, 'Dust/Ash' = 7, 'Smoke/Haze' = 8, 'Blowing/Drifting Snow' = 9, 'Tornado' = 10, 'High Winds' = 11, 'Blowing Spray' = 12, 'Mist' = 13, 'Drizzle' = 14, 'Freezing Drizzle' = 15, 'Rain' = 16, 'Freezing Rain' = 17, 'Snow' = 18, 'Unknown Precipitation' = 19, 'Ground Fog' = 21, 'Freezing Fog' = 22),
 `lat` Float32,
 `lon` Float32,
 `elevation` Int16,
 `name` LowCardinality(String)
) ENGINE = MergeTree() ORDER BY (station_id, date) AS 
SELECT 
	station_id, 
	date,
	tempAvg,
	tempMax,
	tempMin,
	precipitation,
	snowfall,
	snowDepth,
	percentDailySun,
	averageWindSpeed,
	maxWindSpeed,
	weatherType,
	location.2 as lat,
	location.1 as lon,
	elevation,
	name
FROM
	learn_db.noaa;
```

### 20. Смотрим на размер столбцов в 3 версии таблицы

```sql
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v3'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

### 21. Смотрим общий размер таблицы версии 3

```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v3'
```

### 22. Выполняем диагностические запросы

```sql
SELECT
    tempMax / 10 AS maxTemp,
    lat,
    lon,
    name,
    date
FROM learn_db.noaa_codec_v3
WHERE tempMax > 500
ORDER BY
    tempMax DESC,
    date ASC
LIMIT 5
```

Результат
```text
Query id: f9291b28-c3ca-4449-be01-a3771ffbbd2d

   ┌─maxTemp─┬───lat─┬───────lon─┬─name─┬───────date─┐
1. │    56.7 │ 36.45 │ -116.8667 │ CA   │ 1913-07-10 │
2. │      55 │ 36.45 │ -116.8667 │ CA   │ 1913-07-13 │
3. │    54.4 │ 36.45 │ -116.8667 │ CA   │ 1913-07-12 │
4. │    53.9 │ 36.45 │ -116.8667 │ CA   │ 1913-07-09 │
5. │    53.9 │ 36.45 │ -116.8667 │ CA   │ 1913-07-11 │
   └─────────┴───────┴───────────┴──────┴────────────┘

5 rows in set. Elapsed: 0.081 sec. Processed 124.54 million rows, 1.25 GB (1.53 billion rows/s., 15.31 GB/s.)
Peak memory usage: 735.21 KiB.
```

```sql
SELECT
    elevation_range,
    uniq(station_id) AS num_stations,
    max(tempMax) / 10 AS max_temp,
    min(tempMin) / 10 AS min_temp,
    sum(precipitation) AS total_precipitation,
    avg(percentDailySun) AS avg_percent_sunshine,
    max(maxWindSpeed) AS max_wind_speed,
    sum(snowfall) AS total_snowfall
FROM learn_db.noaa_codec_v3
WHERE (date > '1907-01-01') AND (station_id IN ('AG000060590', 'ASN00075063', 3))
GROUP BY floor(elevation, -2) AS elevation_range
ORDER BY elevation_range ASC
```

Результат
```text
Query id: b1a3dab4-e706-4783-b879-b4b9c45a5ccb

   ┌─elevation_range─┬─num_stations─┬─max_temp─┬─min_temp─┬─total_precipitation─┬─avg_percent_sunshine─┬─max_wind_speed─┬─total_snowfall─┐
1. │               0 │            1 │        0 │        0 │               71234 │                    0 │              0 │              0 │
2. │             300 │            1 │       48 │       -5 │                 681 │                    0 │              0 │              0 │
   └─────────────────┴──────────────┴──────────┴──────────┴─────────────────────┴──────────────────────┴────────────────┴────────────────┘

2 rows in set. Elapsed: 0.016 sec. Processed 57.34 thousand rows, 436.42 KB (3.61 million rows/s., 27.45 MB/s.)
Peak memory usage: 4.88 MiB.
```

## Применяем различные кодеки общего и специального назначения

### 23. Создаем таблицу 4 версии

```sql
DROP TABLE IF EXISTS learn_db.noaa_codec_v4;
CREATE TABLE learn_db.noaa_codec_v4
(
    `station_id` LowCardinality(String),
    `date` Date32 CODEC(Delta, ZSTD),
    `tempAvg` Int16 CODEC(Delta, ZSTD),
    `tempMax` Int16 CODEC(Delta, ZSTD),
    `tempMin` Int16 CODEC(Delta, ZSTD),
    `precipitation` UInt16 CODEC(Delta, ZSTD),
    `snowfall` UInt16 CODEC(Delta, ZSTD),
    `snowDepth` UInt16 CODEC(Delta, ZSTD),
    `percentDailySun` UInt8 CODEC(Delta, ZSTD),
    `averageWindSpeed` UInt16 CODEC(Delta, ZSTD),
    `maxWindSpeed` UInt16 CODEC(Delta, ZSTD),
    `weatherType` Enum8('Normal' = 0, 'Fog' = 1, 'Heavy Fog' = 2, 'Thunder' = 3, 'Small Hail' = 4, 'Hail' = 5, 'Glaze' = 6, 'Dust/Ash' = 7, 'Smoke/Haze' = 8, 'Blowing/Drifting Snow' = 9, 'Tornado' = 10, 'High Winds' = 11, 'Blowing Spray' = 12, 'Mist' = 13, 'Drizzle' = 14, 'Freezing Drizzle' = 15, 'Rain' = 16, 'Freezing Rain' = 17, 'Snow' = 18, 'Unknown Precipitation' = 19, 'Ground Fog' = 21, 'Freezing Fog' = 22),
    `lat` Float32,
    `lon` Float32,
    `elevation` Int16,
    `name` LowCardinality(String)
)
ENGINE = MergeTree
ORDER BY (station_id, date) AS
SELECT
	station_id,
	date,
	tempAvg,
	tempMax,
	tempMin,
	precipitation,
	snowfall,
	snowDepth,
	percentDailySun,
	averageWindSpeed,
	maxWindSpeed,
	weatherType,
	location.2 as lat,
	location.1 as lon,
	elevation,
	name
FROM
	learn_db.noaa;
```

### 24. Смотрим на размер столбцов в 4 версии таблицы

```sql
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v4'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

Результат
```text
name            |compressed_size|uncompressed_size|ratio |compressed|uncompressed|
----------------+---------------+-----------------+------+----------+------------+
precipitation   |62.63 MiB      |237.54 MiB       |  3.79|  65672485|   249077472|
tempMin         |36.36 MiB      |237.54 MiB       |  6.53|  38127995|   249077472|
tempMax         |35.88 MiB      |237.54 MiB       |  6.62|  37625010|   249077472|
snowfall        |2.62 MiB       |4.97 MiB         |   1.9|   2746891|     5206710|
lat             |2.41 MiB       |475.08 MiB       |196.99|   2528829|   498154944|
lon             |2.40 MiB       |475.08 MiB       |197.58|   2521323|   498154944|
snowDepth       |1.77 MiB       |6.42 MiB         |  3.63|   1856222|     6728984|
date            |1.68 MiB       |475.08 MiB       |283.04|   1760026|   498154944|
name            |1.58 MiB       |233.77 MiB       |147.57|   1661015|   245121866|
station_id      |1.58 MiB       |230.31 MiB       |145.68|   1657682|   241498639|
tempAvg         |1.39 MiB       |3.43 MiB         |  2.48|   1452891|     3597225|
elevation       |1.26 MiB       |237.54 MiB       |188.15|   1323821|   249077472|
weatherType     |1.04 MiB       |1.66 MiB         |   1.6|   1089771|     1745310|
averageWindSpeed|33.30 KiB      |213.19 KiB       |   6.4|     34099|      218304|
maxWindSpeed    |33.30 KiB      |213.19 KiB       |   6.4|     34099|      218304|
percentDailySun |33.30 KiB      |213.19 KiB       |   6.4|     34099|      218304|
```

### 25. Смотрим общий размер таблицы версии 4

```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    sum(data_uncompressed_bytes) / sum(data_compressed_bytes) AS compression_ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v4'
```

Результат
```text
compressed_size|uncompressed_size|compression_ratio |compressed|uncompressed|
---------------+-----------------+------------------+----------+------------+
152.71 MiB     |2.79 GiB         |18.706041116629354| 160126258|  2995328366|
```

### 26. Смотрим информацию обо всех столбцах в 4 версии таблицы

```sql
SELECT
    *
FROM system.columns
WHERE table = 'noaa_codec_v4'
```

Результат
```text
database|table        |name            |type                                                                                                                                                                                                                                                           |position|default_kind|default_expression|data_compressed_bytes|data_uncompressed_bytes|marks_bytes|comment|is_in_partition_key|is_in_sorting_key|is_in_primary_key|is_in_sampling_key|compression_codec       |character_octet_length|numeric_precision|numeric_precision_radix|numeric_scale|datetime_precision|serialization_hint|
--------+-------------+----------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------+------------+------------------+---------------------+-----------------------+-----------+-------+-------------------+-----------------+-----------------+------------------+------------------------+----------------------+-----------------+-----------------------+-------------+------------------+------------------+
learn_db|noaa_codec_v4|station_id      |LowCardinality(String)                                                                                                                                                                                                                                         |       1|            |                  |              1657682|              241498639|      27656|       |                  0|                1|                1|                 0|                        |                      |                 |                       |             |                  |                  |
learn_db|noaa_codec_v4|date            |Date32                                                                                                                                                                                                                                                         |       2|            |                  |              1760026|              498154944|      37082|       |                  0|                1|                1|                 0|CODEC(Delta(4), ZSTD(1))|                      |                 |                       |             |0                 |Default           |
learn_db|noaa_codec_v4|tempAvg         |Int16                                                                                                                                                                                                                                                          |       3|            |                  |              1452891|                3597225|      37510|       |                  0|                0|                0|                 0|CODEC(Delta(2), ZSTD(1))|                      |16               |2                      |0            |                  |Sparse            |
learn_db|noaa_codec_v4|tempMax         |Int16                                                                                                                                                                                                                                                          |       4|            |                  |             37625010|              249077472|      30052|       |                  0|                0|                0|                 0|CODEC(Delta(2), ZSTD(1))|                      |16               |2                      |0            |                  |Default           |
learn_db|noaa_codec_v4|tempMin         |Int16                                                                                                                                                                                                                                                          |       5|            |                  |             38127995|              249077472|      29969|       |                  0|                0|                0|                 0|CODEC(Delta(2), ZSTD(1))|                      |16               |2                      |0            |                  |Default           |
learn_db|noaa_codec_v4|precipitation   |UInt16                                                                                                                                                                                                                                                         |       6|            |                  |             65672485|              249077472|      33138|       |                  0|                0|                0|                 0|CODEC(Delta(2), ZSTD(1))|                      |16               |2                      |0            |                  |Default           |
learn_db|noaa_codec_v4|snowfall        |UInt16                                                                                                                                                                                                                                                         |       7|            |                  |              2746891|                5206710|      50236|       |                  0|                0|                0|                 0|CODEC(Delta(2), ZSTD(1))|                      |16               |2                      |0            |                  |Sparse            |
learn_db|noaa_codec_v4|snowDepth       |UInt16                                                                                                                                                                                                                                                         |       8|            |                  |              1856222|                6728984|      48980|       |                  0|                0|                0|                 0|CODEC(Delta(2), ZSTD(1))|                      |16               |2                      |0            |                  |Sparse            |
learn_db|noaa_codec_v4|percentDailySun |UInt8                                                                                                                                                                                                                                                          |       9|            |                  |                34099|                 218304|      37489|       |                  0|                0|                0|                 0|CODEC(Delta(1), ZSTD(1))|                      |8                |2                      |0            |                  |Sparse            |
learn_db|noaa_codec_v4|averageWindSpeed|UInt16                                                                                                                                                                                                                                                         |      10|            |                  |                34099|                 218304|      37489|       |                  0|                0|                0|                 0|CODEC(Delta(2), ZSTD(1))|                      |16               |2                      |0            |                  |Sparse            |
learn_db|noaa_codec_v4|maxWindSpeed    |UInt16                                                                                                                                                                                                                                                         |      11|            |                  |                34099|                 218304|      37489|       |                  0|                0|                0|                 0|CODEC(Delta(2), ZSTD(1))|                      |16               |2                      |0            |                  |Sparse            |
learn_db|noaa_codec_v4|weatherType     |Enum8('Normal' = 0, 'Fog' = 1, 'Heavy Fog' = 2, 'Thunder' = 3, 'Small Hail' = 4, 'Hail' = 5, 'Glaze' = 6, 'Dust/Ash' = 7, 'Smoke/Haze' = 8, 'Blowing/Drifting Snow' = 9, 'Tornado' = 10, 'High Winds' = 11, 'Blowing Spray' = 12, 'Mist' = 13, 'Drizzle' = 14, |      12|            |                  |              1089771|                1745310|      47262|       |                  0|                0|                0|                 0|                        |                      |                 |                       |             |                  |Sparse            |
learn_db|noaa_codec_v4|lat             |Float32                                                                                                                                                                                                                                                        |      13|            |                  |              2528829|              498154944|      40095|       |                  0|                0|                0|                 0|                        |                      |                 |                       |             |                  |Default           |
learn_db|noaa_codec_v4|lon             |Float32                                                                                                                                                                                                                                                        |      14|            |                  |              2521323|              498154944|      39980|       |                  0|                0|                0|                 0|                        |                      |                 |                       |             |                  |Default           |
learn_db|noaa_codec_v4|elevation       |Int16                                                                                                                                                                                                                                                          |      15|            |                  |              1323821|              249077472|      31177|       |                  0|                0|                0|                 0|                        |                      |16               |2                      |0            |                  |Default           |
learn_db|noaa_codec_v4|name            |LowCardinality(String)                                                                                                                                                                                                                                         |      16|            |                  |              1661015|              245121866|      30501|       |                  0|                0|                0|                 0|                        |                      |                 |                       |             |                  |                  |
```

### 27. Выполняем диагностические запросы

```sql
SELECT
    tempMax / 10 AS maxTemp,
    lat,
    lon,
    name,
    date
FROM learn_db.noaa_codec_v4
WHERE tempMax > 500
ORDER BY
    tempMax DESC,
    date ASC
LIMIT 5
```

Результат
```text
Query id: 3204c5cd-23ba-4d88-829a-af860220c1de

   ┌─maxTemp─┬───lat─┬───────lon─┬─name─┬───────date─┐
1. │    56.7 │ 36.45 │ -116.8667 │ CA   │ 1913-07-10 │
2. │      55 │ 36.45 │ -116.8667 │ CA   │ 1913-07-13 │
3. │    54.4 │ 36.45 │ -116.8667 │ CA   │ 1913-07-12 │
4. │    53.9 │ 36.45 │ -116.8667 │ CA   │ 1913-07-09 │
5. │    53.9 │ 36.45 │ -116.8667 │ CA   │ 1913-07-11 │
   └─────────┴───────┴───────────┴──────┴────────────┘

5 rows in set. Elapsed: 0.015 sec. Processed 929.38 thousand rows, 9.62 MB (62.81 million rows/s., 649.93 MB/s.)
Peak memory usage: 40.52 KiB.
```

```sql
SELECT
    elevation_range,
    uniq(station_id) AS num_stations,
    max(tempMax) / 10 AS max_temp,
    min(tempMin) / 10 AS min_temp,
    sum(precipitation) AS total_precipitation,
    avg(percentDailySun) AS avg_percent_sunshine,
    max(maxWindSpeed) AS max_wind_speed,
    sum(snowfall) AS total_snowfall
FROM learn_db.noaa_codec_v3
WHERE (date > '1907-01-01') AND (station_id IN ('AG000060590', 'ASN00075063', 3))
GROUP BY floor(elevation, -2) AS elevation_range
ORDER BY elevation_range ASC
```

Результат
```text
Query id: 3631f6ae-2b2f-4c91-953d-e5e523e2abcd

   ┌─elevation_range─┬─num_stations─┬─max_temp─┬─min_temp─┬─total_precipitation─┬─avg_percent_sunshine─┬─max_wind_speed─┬─total_snowfall─┐
1. │               0 │            1 │        0 │        0 │               71234 │                    0 │              0 │              0 │
2. │             300 │            1 │       48 │       -5 │                 681 │                    0 │              0 │              0 │
   └─────────────────┴──────────────┴──────────┴──────────┴─────────────────────┴──────────────────────┴────────────────┴────────────────┘

2 rows in set. Elapsed: 0.011 sec. Processed 57.34 thousand rows, 436.42 KB (5.08 million rows/s., 38.62 MB/s.)
Peak memory usage: 554.31 KiB.
```

### 28. Смотрим, сколько нулевых значений в столбце precipitation

```sql
SELECT
    countIf(precipitation = 0) AS num_empty,
    countIf(precipitation > 0) AS num_non_zero,
    num_empty / (num_empty + num_non_zero) AS ratio
FROM learn_db.noaa
```

```text
Query id: f0c7cfb5-b444-4fb5-92b3-de844bab934d

   ┌─num_empty─┬─num_non_zero─┬──────────────ratio─┐
1. │  96470114 │     28068622 │ 0.7746193441372329 │
   └───────────┴──────────────┴────────────────────┘

1 row in set. Elapsed: 0.102 sec. Processed 124.54 million rows, 498.15 MB (1.22 billion rows/s., 4.89 GB/s.)
Peak memory usage: 133.74 KiB.
```

### 29. Смотрим, сколько нулевых значений в столбце snowDepth

```sql
SELECT
    countIf(snowDepth = 0) AS num_empty,
    countIf(snowDepth > 0) AS num_non_zero,
    num_empty / (num_empty + num_non_zero) AS ratio
FROM learn_db.noaa
```

Результат
```text
Query id: 1a5e6874-6c63-406f-b735-23b64d9c3323

   ┌─num_empty─┬─num_non_zero─┬──────────────ratio─┐
1. │ 122388191 │      2150545 │ 0.9827319188465186 │
   └───────────┴──────────────┴────────────────────┘

1 row in set. Elapsed: 0.035 sec. Processed 124.54 million rows, 25.84 MB (3.57 billion rows/s., 741.47 MB/s.)
Peak memory usage: 387.79 KiB.
```

### 30. Смотрим, сколько нулевых значений в столбце tempMax

```sql
SELECT
    countIf(tempMax = 0) AS num_empty,
    countIf(tempMax > 0) AS num_non_zero,
    num_empty / (num_empty + num_non_zero) AS ratio
FROM learn_db.noaa
```

Результат
```text
Query id: df748375-a323-4ba3-ae81-61d16e09013d

   ┌─num_empty─┬─num_non_zero─┬─────────────ratio─┐
1. │  88934997 │     32497596 │ 0.732381602030025 │
   └───────────┴──────────────┴───────────────────┘

1 row in set. Elapsed: 0.067 sec. Processed 124.54 million rows, 493.95 MB (1.85 billion rows/s., 7.35 GB/s.)
Peak memory usage: 330.06 KiB.
```

### 31. Находим лучший вариант кодека для каждого столбца

```sql
SELECT
    name,
    if(argMin(compression_codec, data_compressed_bytes) != '', argMin(compression_codec, data_compressed_bytes), 'DEFAULT') AS best_codec,
    formatReadableSize(min(data_compressed_bytes)) AS compressed_size
FROM system.columns
WHERE table LIKE 'noaa%'
GROUP BY name
```

Результат
```text
Query id: 04ed6f2c-f28b-4617-8ff1-d819d225d984

    ┌─name─────────────┬─best_codec───────────────┬─compressed_size─┐
 1. │ snowfall         │ CODEC(Delta(2), ZSTD(1)) │ 2.62 MiB        │
 2. │ tempMax          │ CODEC(Delta(2), ZSTD(1)) │ 35.88 MiB       │
 3. │ lat              │ DEFAULT                  │ 2.41 MiB        │
 4. │ tempMin          │ CODEC(Delta(2), ZSTD(1)) │ 36.36 MiB       │
 5. │ date             │ CODEC(Delta(4), ZSTD(1)) │ 1.68 MiB        │
 6. │ tempAvg          │ CODEC(Delta(2), ZSTD(1)) │ 1.39 MiB        │
 7. │ lon              │ DEFAULT                  │ 2.40 MiB        │
 8. │ name             │ DEFAULT                  │ 1.58 MiB        │
 9. │ location         │ DEFAULT                  │ 9.51 MiB        │
10. │ weatherType      │ DEFAULT                  │ 1.04 MiB        │
11. │ elevation        │ DEFAULT                  │ 1.26 MiB        │
12. │ station_id       │ DEFAULT                  │ 1.58 MiB        │
13. │ snowDepth        │ CODEC(Delta(2), ZSTD(1)) │ 1.77 MiB        │
14. │ precipitation    │ CODEC(Delta(2), ZSTD(1)) │ 62.63 MiB       │
15. │ averageWindSpeed │ CODEC(Delta(2), ZSTD(1)) │ 33.30 KiB       │
16. │ maxWindSpeed     │ CODEC(Delta(2), ZSTD(1)) │ 33.30 KiB       │
17. │ percentDailySun  │ CODEC(Delta(1), ZSTD(1)) │ 33.30 KiB       │
    └──────────────────┴──────────────────────────┴─────────────────┘

17 rows in set. Elapsed: 0.007 sec.
```

## Финальный вариант

### 32. Создаем таблицы финальной версии

```sql
DROP TABLE IF EXISTS learn_db.noaa_codec_optimal;
CREATE TABLE learn_db.noaa_codec_optimal
(
   `station_id` LowCardinality(String),
   `date` Date32 CODEC(DoubleDelta, ZSTD(1)),
   `tempAvg` Int16 CODEC(T64, ZSTD(1)),
   `tempMax` Int16 CODEC(T64, ZSTD(1)),
   `tempMin` Int16 CODEC(T64, ZSTD(1)) ,
   `precipitation` UInt16 CODEC(T64, ZSTD(1)) ,
   `snowfall` UInt16 CODEC(T64, ZSTD(1)) ,
   `snowDepth` UInt16 CODEC(ZSTD(1)),
   `percentDailySun` UInt8,
   `averageWindSpeed` UInt16 CODEC(T64, ZSTD(1)),
   `maxWindSpeed` UInt16 CODEC(T64, ZSTD(1)),
   `weatherType` Enum8('Normal' = 0, 'Fog' = 1, 'Heavy Fog' = 2, 'Thunder' = 3, 'Small Hail' = 4, 'Hail' = 5, 'Glaze' = 6, 'Dust/Ash' = 7, 'Smoke/Haze' = 8, 'Blowing/Drifting Snow' = 9, 'Tornado' = 10, 'High Winds' = 11, 'Blowing Spray' = 12, 'Mist' = 13, 'Drizzle' = 14, 'Freezing Drizzle' = 15, 'Rain' = 16, 'Freezing Rain' = 17, 'Snow' = 18, 'Unknown Precipitation' = 19, 'Ground Fog' = 21, 'Freezing Fog' = 22),
   `lat` Float32,
   `lon` Float32,
   `elevation` Int16,
   `name` LowCardinality(String)
)
ENGINE = MergeTree
ORDER BY (station_id, date) AS
SELECT
	station_id,
	date,
	tempAvg,
	tempMax,
	tempMin,
	precipitation,
	snowfall,
	snowDepth,
	percentDailySun,
	averageWindSpeed,
	maxWindSpeed,
	weatherType,
	location.2 as lat,
	location.1 as lon,
	elevation,
	name
FROM
	learn_db.noaa;
```

### 33. Смотрим на размер столбцов в финальной версии таблицы

```sql
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_optimal'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

Результат
```text
Query id: 8860b143-b30d-487c-ae0d-e1e2a500be6a

    ┌─name─────────────┬─compressed_size─┬─uncompressed_size─┬──ratio─┬─compressed─┬─uncompressed─┐
 1. │ precipitation    │ 43.35 MiB       │ 237.16 MiB        │   5.47 │   45458654 │    248683496 │
 2. │ tempMax          │ 31.37 MiB       │ 237.16 MiB        │   7.56 │   32894757 │    248683496 │
 3. │ tempMin          │ 31.24 MiB       │ 237.16 MiB        │   7.59 │   32760711 │    248683496 │
 4. │ lat              │ 2.41 MiB        │ 474.33 MiB        │ 197.12 │    2523170 │    497366992 │
 5. │ lon              │ 2.40 MiB        │ 474.33 MiB        │ 197.72 │    2515521 │    497366992 │
 6. │ snowfall         │ 1.98 MiB        │ 4.95 MiB          │    2.5 │    2078368 │      5188587 │
 7. │ date             │ 1.85 MiB        │ 474.33 MiB        │ 256.09 │    1942142 │    497366992 │
 8. │ snowDepth        │ 1.83 MiB        │ 8.17 MiB          │   4.46 │    1921941 │      8570121 │
 9. │ name             │ 1.57 MiB        │ 230.56 MiB        │ 146.48 │    1650394 │    241754833 │
10. │ station_id       │ 1.57 MiB        │ 228.55 MiB        │ 145.53 │    1646706 │    239648734 │
11. │ elevation        │ 1.26 MiB        │ 237.16 MiB        │ 188.32 │    1320502 │    248683496 │
12. │ tempAvg          │ 1.26 MiB        │ 3.41 MiB          │   2.71 │    1316535 │      3574268 │
13. │ weatherType      │ 1.04 MiB        │ 1.66 MiB          │   1.59 │    1091643 │      1737790 │
14. │ percentDailySun  │ 63.08 KiB       │ 215.27 KiB        │   3.41 │      64591 │       220437 │
15. │ averageWindSpeed │ 34.21 KiB       │ 215.27 KiB        │   6.29 │      35031 │       220437 │
16. │ maxWindSpeed     │ 34.21 KiB       │ 215.27 KiB        │   6.29 │      35031 │       220437 │
    └──────────────────┴─────────────────┴───────────────────┴────────┴────────────┴──────────────┘

16 rows in set. Elapsed: 0.004 sec.
```

### 34. Смотрим общий размер таблицы финальной версии

```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_optimal'
```

Результат
```text
Query id: 9a9dc359-0129-4b92-ac63-5996febad9e5

   ┌─compressed_size─┬─uncompressed_size─┬─ratio─┬─compressed─┬─uncompressed─┐
1. │ 123.27 MiB      │ 2.78 GiB          │ 23.12 │  129255697 │   2987970604 │
   └─────────────────┴───────────────────┴───────┴────────────┴──────────────┘

1 row in set. Elapsed: 0.004 sec.
```

### 35. Выполняем диагностический запрос

```sql
SELECT
    elevation_range,
    uniq(station_id) AS num_stations,
    max(tempMax) / 10 AS max_temp,
    min(tempMin) / 10 AS min_temp,
    sum(precipitation) AS total_precipitation,
    avg(percentDailySun) AS avg_percent_sunshine,
    max(maxWindSpeed) AS max_wind_speed,
    sum(snowfall) AS total_snowfall
FROM learn_db.noaa_codec_optimal
WHERE (date > '1907-01-01') AND (station_id IN ('AG000060590', 'ASN00075063', 3))
GROUP BY floor(elevation, -2) AS elevation_range
ORDER BY elevation_range ASC
```

Результат
```text
Query id: c9a57ad9-bf23-4a3c-a460-40901f2c9906

   ┌─elevation_range─┬─num_stations─┬─max_temp─┬─min_temp─┬─total_precipitation─┬─avg_percent_sunshine─┬─max_wind_speed─┬─total_snowfall─┐
1. │               0 │            1 │        0 │        0 │               71234 │                    0 │              0 │              0 │
2. │             300 │            1 │       48 │       -5 │                 681 │                    0 │              0 │              0 │
   └─────────────────┴──────────────┴──────────┴──────────┴─────────────────────┴──────────────────────┴────────────────┴────────────────┘

2 rows in set. Elapsed: 0.011 sec. Processed 57.34 thousand rows, 540.80 KB (5.04 million rows/s., 47.55 MB/s.)
Peak memory usage: 553.87 KiB.
```

Сравнение версий таблиц
```text
Версия	                    Сжатый размер	Несжатый размер	Коэффициент сжатия	Экономия против v1
v1 (базовая)	            537.81 MiB	    9.06 GiB	    17.24	            -
v2 (оптимальные типы)	    379.71 MiB	    6.24 GiB	    16.84	            29.4%
v3 (LowCardinality + Enum)	-	            -	            -	                -
v4 (кодеки Delta+ZSTD)	    152.71 MiB	    2.79 GiB	    18.71	            71.6%
v5 (финальная)	            123.27 MiB	    2.78 GiB	    23.12	            77.1%
```
## Основные выводы из анализа

### Анализ по версиям таблиц

```text
v1 → v2 (Оптимальные типы данных):
Сжатый размер: 537.81 → 379.71 MiB (▼29.4%)
Наибольший выигрыш: температурные данные (tempMin/Max: 104→54 MiB)

v2 → v3 (LowCardinality + Enum):
Сжатый размер: 379.71 → 372.39 MiB (▼1.9%)
Хороший результат для station_id (6.64 → 1.59 MiB) и name (3.83 → 1.60 MiB)

v3 → v4 (Delta + ZSTD кодеки):
Сжатый размер: 372.39 → 152.71 MiB (▼59.0%)
Значительное улучшение! Особенно для date (169→1.68 MiB) и температур

v4 → v5 (Финальная оптимизация):
Сжатый размер: 152.71 → 123.27 MiB (▼19.3%)
Дополнительная оптимизация кодеков
```

### Топ самых эффективных оптимизаций по столбцам:

```text
* date: 169.20 MiB → 1.85 MiB (▼98.9%) - Delta+ZSTD
* station_id: 6.63 MiB → 1.57 MiB (▼76.3%) - LowCardinality
* tempMax: 104.20 MiB → 31.37 MiB (▼69.9%) - Int16 + T64+ZSTD
* precipitation: 121.16 MiB → 43.35 MiB (▼64.2%) - UInt16 + T64+ZSTD
* name: 3.83 MiB → 1.57 MiB (▼59.0%) - LowCardinality
```

### Эффективность кодеков:

```text
1. Delta + ZSTD показал потрясающие результаты для:
- временных рядов (date)
- последовательно изменяющихся данных (температуры)
2. T64 + ZSTD в финальной версии дал дополнительный выигрыш для:
- числовых данных с ограниченным диапазоном
```

### Выводы

*Общий успех оптимизации:* Удалось уменьшить размер данных в 4.4 раза (с 537.81 MiB до 123.27 MiB) при сохранении производительности запросов.

*Ключевой фактор:* Комбинация правильных типов данных + специализированные кодеки для временных рядов + LowCardinality для строк с повторениями.

*Производительность:* Все диагностические запросы выполняются за доли секунды, что подтверждает эффективность выбранных оптимизаций.