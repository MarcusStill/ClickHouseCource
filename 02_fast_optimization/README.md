# Экспресс-оптимизации для ускорения дашборда

## 1. Определяем метод замера скорости

#### 1.1 Открываем Docker Desktop и заходим внутрь контейнера Clickhouse
Открываем вкладку "Exec" выполняем команду "bash". Сохраняем все запросы, выполняемые при открытии дашборда, в файл queries.tsv
```console
clickhouse-client --query="SELECT query FROM system.query_log WHERE NOT query LIKE '%query_log%' AND http_user_agent = 'DataLens' AND 'QueryFinish' = type ORDER BY event_time desc LIMIT 9" > queries.tsv
```

#### 1.2 С помощью утилиты clickhouse-benchmark выполняем каждый запрос 5 раз
```console
clickhouse-benchmark -i 45 -c 1 < queries.tsv
```

#### 1.3 С помощью утилиты clickhouse-benchmark выполняем каждый запрос 5 раз
```sql
with queries as (
	SELECT * 
	FROM system.query_log 
	WHERE NOT query LIKE '%query_log%'
	AND 'QueryFinish' = type
	and client_name = 'ClickHouse benchmark'
	ORDER BY event_time desc
	limit 45
)
select 
	query,
	avg(read_rows) AS read_rows,
	avg(read_bytes) AS read_bytes,
	count(*) as cnt,
	avg(query_duration_ms) as avg,
	min(query_duration_ms) as min,
	max(query_duration_ms) as max,
	any(partitions) as partitions,
	any(projections) as projections
from 
	queries
group by 
	query
order by 
	query;
```

#### 1.4 Сохраняем результат запроса в Excel

#### 2. Редактируем индикатор подсчета количества учеников
с `countd(str([person_id]))` на `countd([person_id_int])`.

#### 2.1 Обновляем дашборд, в докере выполняем комануд получения запросов и запускаем clickhouse-benchmark

#### 3. Добавляем первичный ключ. Удаляем таблицу. И при создании вместо `PRIMARY KEY(tuple()` указываем `PRIMARY KEY(person_id_int)`.
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson;

CREATE TABLE learn_db.mart_student_lesson
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
	PRIMARY KEY(person_id_int)
) ENGINE = MergeTree() 
AS SELECT
	floor(randUniform(2, 1300000)) as student_profile_id,
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
FROM numbers(200000000);
```

#### 3.1 Обновляем дашборд, в докере выполняем команд получения запросов и запускаем clickhouse-benchmark

#### 4. Смотрим на два самых долгих запроса
```sql
SELECT t1.lesson_month_text AS res_0, count(t1.mark) AS res_1, t1.mark AS res_2
FROM learn_db.mart_student_lesson AS t1
GROUP BY res_0, res_2
LIMIT 1000001
FORMAT JSONCompact
```
и
```sql
SELECT t1.subject_name AS res_0, avg(t1.mark) AS res_1
FROM learn_db.mart_student_lesson AS t1
GROUP BY res_0
LIMIT 1000001
FORMAT JSONCompact
```
В ddl таблицы поле `mark` Nullable(UInt8) - может принимать пустые значения. В рекомендациях работы с CH указано что Nullable поля работают медленнее.
#### 4.1 Изменим конструкцию case (значение `null` заменим на -1), изменим тип поля в ddl таблицы на int8, пересоздадим таблицу. А в дашборде откроем датасет и создадим вычисляемое поле mark_real.
```sql
DROP TABLE learn_db.mart_student_lesson;

CREATE TABLE learn_db.mart_student_lesson
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
	`mark` Int8, -- Оценка
	PRIMARY KEY(person_id_int)
) ENGINE = MergeTree() 
AS SELECT
	floor(randUniform(2, 1300000)) as student_profile_id,
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
    		THEN -1
    		ELSE 
    			CASE
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 5 THEN ROUND(randUniform(4, 5))
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 9 THEN ROUND(randUniform(3, 5))
	    			ELSE ROUND(randUniform(2, 5))
    			END				
    END AS mark
FROM numbers(200000000);
```
#### 4.2 Обновляем дашборд, в докере выполняем команд получения запросов и запускаем clickhouse-benchmark

#### 5. Смотрим на два самых долгих запроса
```sql
SELECT t1.lesson_month_text AS res_0, count(t1.mark) AS res_1, if(t1.mark = -1, NULL, t1.mark) AS res_2
FROM learn_db.mart_student_lesson AS t1
GROUP BY res_0, res_2
LIMIT 1000001
FORMAT JSONCompact
```
и
```sql
SELECT t1.subject_name AS res_0, avg(if(t1.mark = -1, NULL, t1.mark)) AS res_1
FROM learn_db.mart_student_lesson AS t1
GROUP BY res_0
LIMIT 1000001
FORMAT JSONCompact
```
#### 5.1. В обоих функциях агрегация по полю mark. Добавим его в первичный индекс.
```sql
DROP TABLE learn_db.mart_student_lesson;

CREATE TABLE learn_db.mart_student_lesson
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
	`mark` Int8, -- Оценка
	PRIMARY KEY(person_id_int, mark)
) ENGINE = MergeTree() 
AS SELECT
	floor(randUniform(2, 1300000)) as student_profile_id,
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
    		THEN -1
    		ELSE 
    			CASE
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 5 THEN ROUND(randUniform(4, 5))
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 9 THEN ROUND(randUniform(3, 5))
	    			ELSE ROUND(randUniform(2, 5))
    			END				
    END AS mark
FROM numbers(200000000);
```
#### 5.2 Обновляем дашборд, в докере выполняем команд получения запросов и запускаем clickhouse-benchmark

#### 6. Изменим фильтрацию.
Оценки за весь период пользователям вряд ли будут нужны. Т.е. "дата урока" будет самым часто применяемым фильтром.
Взяли данные за месяц, но запросы не ускорились. Поместим поле lession_date в первичный индекс и поместить его на первое место в первичном индексе.
```sql
DROP TABLE if exists learn_db.mart_student_lesson;

CREATE TABLE learn_db.mart_student_lesson
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
	`mark` Int8, -- Оценка
	PRIMARY KEY(lesson_date, person_id_int, mark)
) ENGINE = MergeTree() 
AS SELECT
	floor(randUniform(2, 1300000)) as student_profile_id,
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
    		THEN -1
    		ELSE 
    			CASE
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 5 THEN ROUND(randUniform(4, 5))
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 9 THEN ROUND(randUniform(3, 5))
	    			ELSE ROUND(randUniform(2, 5))
    			END				
    END AS mark
FROM numbers(200000000);
```
В этот раз ускорение выполнения запросов получилось существенным.

#### 7. Добавим партиционирование.
Проверим скорость открытия дашборда по одной школе.

Добавим партицирование по `educational_organization_id`.
```sql
DROP TABLE if exists learn_db.mart_student_lesson;

CREATE TABLE learn_db.mart_student_lesson
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
	`mark` Int8, -- Оценка
	PRIMARY KEY(lesson_date, person_id_int, mark)
) ENGINE = MergeTree() 
PARTITION BY educational_organization_id
AS SELECT
	floor(randUniform(2, 1300000)) as student_profile_id,
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
    		THEN -1
    		ELSE 
    			CASE
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 5 THEN ROUND(randUniform(4, 5))
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 9 THEN ROUND(randUniform(3, 5))
	    			ELSE ROUND(randUniform(2, 5))
    			END				
    END AS mark
FROM numbers(200000000);
```
#### 7.1. Проверим скорость выполнения запросов. Время выполнения запросов изменилось.

#### 8. Добавим проекцию.

#### 8.1. Добавляем проекцию по оценке для случая, если дашборд строится без применения фильтров по дате и школе
```sql
ALTER TABLE learn_db.mart_student_lesson
ADD PROJECTION prj_mark
(
    SELECT *
    ORDER BY mark, person_id_int
);

ALTER TABLE learn_db.mart_student_lesson MATERIALIZE PROJECTION prj_mark;
```
#### 8.2. Проверим скорость выполнения запросов. Она упала.

#### 9. Добавим индексы пропуска данных.
Он хорошо применяется если у нас есть какое-то поле (в нашем случае `lesson_date`) и  есть другие поля, которые весьма близки и их порядок сортировки похож на то поле, по которому у нас происходит сортировка. Если мы создаем индекс по полю `lesson_mpnth_digits`, то на каждые 2 гранулы (или на то количество, которое мы укажем) будут создаваться две доп.засечки: в одной будет указано минимальное значение `lesson_month_digits` (которое попадает в эти гранулы), а в другой будет указано максимальное значение. Поскольку наши строки отсортированы по `lesson_date`, то все значения двух гранул могут попасть в 1 месяц или 2 месяца и с помощью этих засечек мы сможем отфильтровать по месяцу.

#### 9.1. Перед созданием фильтра проверим скорость отрисовки дашборда с применением фильтра по месяцу.
Выбираем в фильтре по месяцу апрель 2025, сохраняем запросы и выполняем без пропуска тесты.

#### 9.2. Добавляем индекс пропуска данных по всем полям с датами
```sql
ALTER TABLE learn_db.mart_student_lesson ADD INDEX lesson_month_text_skipping_index lesson_month_text TYPE minmax GRANULARITY 4;
ALTER TABLE learn_db.mart_student_lesson ADD INDEX lesson_month_digits_skipping_index lesson_month_digits TYPE minmax GRANULARITY 4;
ALTER TABLE learn_db.mart_student_lesson ADD INDEX lesson_year_skipping_index lesson_year TYPE minmax GRANULARITY 4;
ALTER TABLE learn_db.mart_student_lesson ADD INDEX load_date_skipping_index load_date TYPE minmax GRANULARITY 4;

ALTER TABLE learn_db.mart_student_lesson MATERIALIZE INDEX lesson_month_text_skipping_index;
ALTER TABLE learn_db.mart_student_lesson MATERIALIZE INDEX lesson_month_digits_skipping_index;
ALTER TABLE learn_db.mart_student_lesson MATERIALIZE INDEX lesson_year_skipping_index;
ALTER TABLE learn_db.mart_student_lesson MATERIALIZE INDEX load_date_skipping_index;
```
#### 9.3. Повторяем замеры. Видим прирост в скорости выполнения запросов.

## P.S. Результаты тестов в файле ch_benchmark_results.xlsx