## Задание

1. Пересоздаем таблицу learn_db.mart_student_lesson с движком ReplicatedMergeTree на кластере cluster_2S_2R.
2. Пересоздаем распределенную таблицу learn_db.mart_student_lesson_distributed_person_id_int с распределением по person_id_int.
3. Вставляем 1 000 000 строк в распределенную таблицу learn_db.mart_student_lesson_distributed_person_id_int.
4. Напишем любой запрос с оконной функцией к распределенной таблице learn_db.mart_student_lesson_distributed_person_id_int. Получим план выполнения запроса.
5. Напишем запрос к распределенной таблице learn_db.mart_student_lesson_distributed_person_id_int, который выведет следующие столбцы:
   - id ученика (person_id_int);
   - последняя оценка, полученная учеником (среди всех оценок одного ученика выбираем оценку, у которой макcимальная дата lesson_date. Если за максимальную дату несколько оценок, то возьмите максимальную среди них, то есть с самым большим значением mark)
   Получим план выполнения написанного вами запроса.

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

### 2. Пересоздаем распределенную таблицу learn_db.mart_student_lesson_distributed_person_id_int

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

### 3. Наполним таблицу данными

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

### 4. Напишем произвольный запрос с оконной функцией

```sql
select student_profile_id, avg(mark) over (partition by student_profile_id order by lesson_date desc)
from learn_db.mart_student_lesson_distributed_person_id_int
where mark is not null;
```

Результат
```text
student_profile_id|avg_mark          |
------------------+------------------+
                 5|               4.0|
                40|               4.0|
                48|               5.0|
                55|               5.0|
                69|               3.0|
                69|               3.0|
                86|               4.0|
               130|               4.0|
               188|               4.0|
               194|               5.0|
               206|               4.0|
               206|               4.5|
               221|               4.0|
               247|               4.0|
               252|               5.0|
               344|               4.0|
               344|               4.0|
               426|               3.0|
               426|               3.0|
               434|               5.0|
               434|               4.0|
               446|               4.0|
               481|               4.0|
               501|               4.0|
               516|               4.0|
               528|               4.0|
               540|               2.0|
               553|               5.0|
               579|               3.0|
               604|               4.0|
               617|               4.0|
               625|               5.0|
               655|               3.0|
               667|               4.0|
               689|               4.0|
               731|               3.0|
               812|               2.0|
               812|               3.5|
               875|               3.0|
               884|               4.0|
               888|               4.0|
               888|               4.0|
...
```

Данный запрос выведет для каждой строки с ненулевой оценкой:
- идентификатор студента
- скользящее среднее оценок студента, рассчитанное от самой последней даты урока к более ранним

### 5. Посмотрим план выполнения запроса

```sql
EXPLAIN distributed=1
select student_profile_id, avg(mark) over (partition by student_profile_id order by lesson_date desc) as avg_mark
from learn_db.mart_student_lesson_distributed_person_id_int
where mark is not null
```

Результат
```text
explain                                                                                                        |
---------------------------------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                                                      |
  Window (Window step for window 'PARTITION BY __table1.student_profile_id ORDER BY __table1.lesson_date DESC')|
    Sorting (Sorting for window 'PARTITION BY __table1.student_profile_id ORDER BY __table1.lesson_date DESC') |
      Union                                                                                                    |
        Expression (Before WINDOW)                                                                             |
          Expression ((WHERE + Change column names to column identifiers))                                     |
            ReadFromMergeTree (learn_db.mart_student_lesson)                                                    |
        ReadFromRemote (Read from remote replica)                                                              |
          BlocksMarshalling                                                                                    |
            Expression (Before WINDOW)                                                                         |
              Expression ((WHERE + Change column names to column identifiers))                                 |
                ReadFromMergeTree (learn_db.mart_student_lesson)                                                |
```

### 6. Напишем запрос с оконной функцией для вывода id ученика и последняя оценки, полученной учеником (среди всех оценок одного ученика выбираем оценку, у которой макcимальная дата lesson_date

```sql
SELECT 
    person_id_int,
    argMax(mark, (lesson_date, mark)) AS last_mark
FROM learn_db.mart_student_lesson_distributed_person_id_int
WHERE mark IS NOT NULL
GROUP BY person_id_int
```

Результат
```text
person_id_int|last_mark|
-------------+---------+
      5487891|4        |
      3997043|2        |
      9256403|5        |
      3465756|3        |
      7643519|2        |
      1415844|4        |
      7204291|3        |
      9132295|4        |
      3840423|5        |
      6681260|3        |
      9728367|2        |
      7521707|3        |
      2470351|3        |
      3311304|3        |
      4911535|4        |
      7299024|2        |
      3681204|2        |
      5729236|4        |
      3087539|5        |
      8516731|4        |
      2556892|5        |
      4735932|2        |
      6896388|3        |
      9408448|4        |
       424804|3        |
      5121896|4        |
      9298495|4        |
      8151548|4        |
       267931|3        |
      4478456|5        |
       788980|4        |
      7634579|4        |
      9620055|3        |
      5995920|3        |
      2978124|3        |
      1393992|3        |
      6101316|3        |
      6630720|4        |
      8624516|4        |
      9744003|3        |
      7477831|3        |
...
```

### 7. Посмотрим план выполнения запроса

```sql
EXPLAIN distributed=1
SELECT 
    person_id_int,
    argMax(mark, (lesson_date, mark)) AS last_mark
FROM learn_db.mart_student_lesson_distributed_person_id_int
WHERE mark IS NOT NULL
GROUP BY person_id_int
```

Результат
```text
explain                                                                       |
------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                     |
  MergingAggregated                                                           |
    Union                                                                     |
      Aggregating                                                             |
        Expression (Before GROUP BY)                                          |
          Expression ((WHERE + Change column names to column identifiers))    |
            ReadFromMergeTree (default.mart_student_lesson)                   |
      ReadFromRemote (Read from remote replica)                               |
        BlocksMarshalling                                                     |
          Aggregating                                                         |
            Expression (Before GROUP BY)                                      |
              Expression ((WHERE + Change column names to column identifiers))|
                ReadFromMergeTree (default.mart_student_lesson)               |                                           |
```