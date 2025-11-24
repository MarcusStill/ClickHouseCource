# Применение движка Distributed

### 1. Пересоздаем таблицу learn_db.mart_student_lesson

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson ON CLUSTER cluster_2S_2R; 

SYSTEM DROP REPLICA '01' FROM ZKPATH '/clickhouse/tables/01/learn_db/mart_student_lesson';
SYSTEM DROP REPLICA '02' FROM ZKPATH '/clickhouse/tables/01/learn_db/mart_student_lesson';
SYSTEM DROP REPLICA '01' FROM ZKPATH '/clickhouse/tables/02/learn_db/mart_student_lesson';
SYSTEM DROP REPLICA '02' FROM ZKPATH '/clickhouse/tables/02/learn_db/mart_student_lesson';

CREATE TABLE learn_db.mart_student_lesson ON CLUSTER cluster_2S_2R
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
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/{database}/{table}', '{replica}');
```

### 2. Создаем распределенную таблицу со случайным распределением

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_distributed ON CLUSTER cluster_2S_2R; 

CREATE TABLE IF NOT EXISTS learn_db.mart_student_lesson_distributed
ON CLUSTER cluster_2S_2R
ENGINE = Distributed('cluster_2S_2R', 'learn_db', 'mart_student_lesson', rand());
```

### 3. Вставляем строки в распределенную таблицу

```sql
INSERT INTO learn_db.mart_student_lesson_distributed
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

### 4. Смотрим, сколько строк всего в распределенной таблице и сколько на 1ом шарде

```sql
SELECT COUNT(*) FROM learn_db.mart_student_lesson_distributed;
```

Результат
```text
COUNT() |
--------+
10000000|
```


```sql
SELECT COUNT(*) FROM learn_db.mart_student_lesson;
```

Результат - шард 1 реплика 1
```text
COUNT()|
-------+
5001175|
```

Результат - шард 1 реплика 2
```text
COUNT()|
-------+
4998825|
```

Данные были разделены между двумя шардами.

Теперь создадим таблицу с осознанным распределением строк.

### 5. Удаляем данные из распределенной таблицы

```sql
ALTER TABLE learn_db.mart_student_lesson ON CLUSTER cluster_2S_2R
DELETE WHERE TRUE;

TRUNCATE TABLE learn_db.mart_student_lesson ON CLUSTER cluster_2S_2R SYNC;
```

### 6. Или пересоздаем таблицу learn_db.mart_student_lesson

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson ON CLUSTER cluster_2S_2R; 

SYSTEM DROP REPLICA '01' FROM ZKPATH '/clickhouse/tables/01/learn_db/mart_student_lesson';
SYSTEM DROP REPLICA '02' FROM ZKPATH '/clickhouse/tables/01/learn_db/mart_student_lesson';
SYSTEM DROP REPLICA '01' FROM ZKPATH '/clickhouse/tables/02/learn_db/mart_student_lesson';
SYSTEM DROP REPLICA '02' FROM ZKPATH '/clickhouse/tables/02/learn_db/mart_student_lesson';

CREATE TABLE learn_db.mart_student_lesson ON CLUSTER cluster_2S_2R
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
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/{database}/{table}', '{replica}');
```

### 7. Создаем распределенную таблицу с распределением по полю person_id_int

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_distributed_person_id_int ON CLUSTER cluster_2S_2R;
CREATE TABLE IF NOT EXISTS learn_db.mart_student_lesson_distributed_person_id_int ON CLUSTER cluster_2S_2R
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
	`mark` Nullable(UInt8) -- Оценка
)
ENGINE = Distributed('cluster_2S_2R', 'learn_db', 'mart_student_lesson', person_id_int);
```

### 8. Вставляем данные в новую распределенную таблицу

```sql
INSERT INTO learn_db.mart_student_lesson_distributed_person_id_int
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

### 9. Смотрим, как данные были распределены между шардами

```sql
SELECT COUNT(*) FROM learn_db.mart_student_lesson_distributed_person_id_int;
```

Результат
```text
COUNT() |
--------+
10000000|
```

```sql
SELECT COUNT(*) FROM learn_db.mart_student_lesson;
```

Результат - шард 1 реплика 1
```text
COUNT()|
-------+
5000994|
```

Результат - шард 1 реплика 2
```text
COUNT()|
-------+
4999006|
```

### 10. Посмотрим на распределение данных

```sql
SELECT * FROM learn_db.mart_student_lesson limit 20;
```

Результат - шард 1 реплика 1
```text
student_profile_id|person_id|person_id_int|educational_organization_id|parallel_id|class_id|lesson_date|lesson_month_digits|lesson_month_text|lesson_year|load_date |t  |teacher_id|subject_id|subject_name       |mark|
------------------+---------+-------------+---------------------------+-----------+--------+-----------+-------------------+-----------------+-----------+----------+---+----------+----------+-------------------+----+
           4130118|4130118  |      4130118|                         11|         56|    2065| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-24|113|      1651|        12|Информатика        |    |
           6921044|6921044  |      6921044|                         18|         94|    3460| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25| 48|      2626|         5|Химия              |3   |
           6574836|6574836  |      6574836|                         18|         90|    3287| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-24| 38|      2487|         4|Физика             |    |
           2301972|2301972  |      2301972|                          6|         31|    1150| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25|119|       976|        13|Информатика        |    |
             17440|17440    |        17440|                          0|          0|       8| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-24| 91|        97|        10|Информатика        |5   |
           9529156|9529156  |      9529156|                         26|        130|    4764| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-24|110|      3660|        12|Информатика        |5   |
           1713656|1713656  |      1713656|                          4|         23|     856| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25|130|       768|        14|Информатика        |    |
           8000150|8000150  |      8000150|                         21|        109|    4000| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25|103|      3083|        11|Информатика        |    |
           8938412|8938412  |      8938412|                         24|        122|    4469| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25|103|      3433|        11|Информатика        |2   |
           6139118|6139118  |      6139118|                         16|         84|    3069| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25| 52|      2339|         5|Химия              |4   |
           8544486|8544486  |      8544486|                         23|        117|    4272| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-24|105|      3288|        11|Информатика        |    |
           7489540|7489540  |      7489540|                         20|        102|    3744| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25| 18|      2808|         2|Русский язык       |    |
           4733144|4733144  |      4733144|                         12|         64|    2366| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-24| 59|      1822|         6|География          |    |
            517996|517996   |       517996|                          1|          7|     258| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-26| 56|       249|         6|География          |    |
           1963128|1963128  |      1963128|                          5|         26|     981| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-26| 41|       772|         4|Физика             |    |
           3197574|3197574  |      3197574|                          8|         43|    1598| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-26| 84|      1275|         9|Информатика        |4   |
           6882000|6882000  |      6882000|                         18|         94|    3441| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25| 73|      2637|         8|Физическая культура|    |
           6798108|6798108  |      6798108|                         18|         93|    3399| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-26| 72|      2604|         8|Физическая культура|    |
           3776708|3776708  |      3776708|                         10|         51|    1888| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25| 46|      1453|         5|Химия              |4   |
           2604218|2604218  |      2604218|                          7|         35|    1302| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25|113|      1083|        12|Информатика        |    |
```

Видим что значения person_id_int только четные.

Результат - шард 1 реплика 2
```text
student_profile_id|person_id|person_id_int|educational_organization_id|parallel_id|class_id|lesson_date|lesson_month_digits|lesson_month_text|lesson_year|load_date |t  |teacher_id|subject_id|subject_name       |mark|
------------------+---------+-------------+---------------------------+-----------+--------+-----------+-------------------+-----------------+-----------+----------+---+----------+----------+-------------------+----+
           7956671|7956671  |      7956671|                         21|        108|    3978| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-26| 16|      2980|         1|Математика         |4   |
           5083599|5083599  |      5083599|                         13|         69|    2541| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-26| 11|      1905|         1|Математика         |5   |
           5244043|5244043  |      5244043|                         14|         71|    2622| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25| 11|      1964|         1|Математика         |4   |
           6554253|6554253  |      6554253|                         17|         89|    3277| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-26|  6|      2448|         0|Информатика        |5   |
           9900637|9900637  |      9900637|                         27|        135|    4950| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-26| 99|      3788|        11|Информатика        |3   |
           4934695|4934695  |      4934695|                         13|         67|    2467| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-24| 43|      1881|         4|Физика             |5   |
           1326073|1326073  |      1326073|                          3|         18|     663| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-26| 79|       573|         8|Физическая культура|    |
           5683155|5683155  |      5683155|                         15|         77|    2841| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-24| 64|      2181|         7|Биология           |3   |
           2373335|2373335  |      2373335|                          6|         32|    1186| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-26| 41|       925|         4|Физика             |    |
           5032015|5032015  |      5032015|                         13|         68|    2516| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-24|128|      2002|        14|Информатика        |3   |
           8591307|8591307  |      8591307|                         23|        117|    4295| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-26|104|      3305|        11|Информатика        |3   |
           9134157|9134157  |      9134157|                         25|        125|    4567| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-26|114|      3517|        12|Информатика        |    |
           3676667|3676667  |      3676667|                         10|         50|    1838| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-24| 29|      1398|         3|Литература         |3   |
           4465569|4465569  |      4465569|                         12|         61|    2232| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-26| 67|      1730|         7|Биология           |4   |
           4837905|4837905  |      4837905|                         13|         66|    2418| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-26|105|      1907|        11|Информатика        |    |
           1156537|1156537  |      1156537|                          3|         15|     578| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25| 77|       507|         8|Физическая культура|    |
           2166677|2166677  |      2166677|                          5|         29|    1083| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-24| 71|       878|         7|Биология           |4   |
           9351953|9351953  |      9351953|                         25|        128|    4675| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25|104|      3588|        11|Информатика        |    |
           1761787|1761787  |      1761787|                          4|         24|     880| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25|109|       765|        12|Информатика        |    |
           1439861|1439861  |      1439861|                          3|         19|     719| 2024-11-24|2024-11            |2024 November    |       2024|2024-11-25| 52|       588|         5|Химия              |    |
```

### 11. Останавливаем кластер, меняем веса шардов в config.xml, добавляем тег weight и вновь запускаем кластер

```xml
        <cluster_2S_2R>
            <shard>
                <internal_replication>true</internal_replication>
				<с>2</с>
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
```


### 12. Вставляем строки в распределенную таблицу

```sql
INSERT INTO learn_db.mart_student_lesson_distributed_person_id_int
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

### 13. Смотрим, как были распределены строки между шардами

```sql
SELECT COUNT(*) FROM learn_db.mart_student_lesson_distributed_person_id_int;
```

Результат
```text
COUNT() |
--------+
20000000|
```

```sql
SELECT COUNT(*) FROM learn_db.mart_student_lesson;
```

Результат - шард 1 реплика 1
```text
COUNT() |
--------+
11667643|
```

Результат - шард 1 реплика 2
```text
COUNT()|
-------+
8332357|
```

Таким образом можно неравномерно делить данные между шардами.
