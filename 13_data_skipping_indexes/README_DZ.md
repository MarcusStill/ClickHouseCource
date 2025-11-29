## Задание 1: Создайте minmax-индекс на колонку lesson_date

Добавьте minmax-индекс на поле lesson_date с granularity = 2. Проверьте, как изменится скорость выполнения запроса с фильтром по дате.

#### 1. Создаем таблицу mart_student_lesson
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
	PRIMARY KEY(class_id)
) ENGINE = MergeTree();
```

#### 2. Вставляем данные в таблицу
```sql
INSERT INTO mart_student_lesson
(
	student_profile_id, 
	person_id, 
	person_id_int, 
	educational_organization_id, 
	parallel_id, 
	class_id, 
	lesson_date, 
	lesson_month_digits, 
	lesson_month_text, 
	lesson_year, 
	load_date, t, 
	teacher_id, 
	subject_id, 
	subject_name, 
	mark
)
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

#### 3. Выполним запрос без индекса

```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	lesson_date = '2025-09-03'::date;
```

Результат
```text
Query id: 7defb81a-7df8-4901-b17b-d71b83162774

       ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬───t─┬─teacher_id─┬─subject_id─┬─subject_name────────┬─mark─┐
    1. │            1548270 │ 1548270   │       1548270 │                           4 │          21 │      774 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-03 │ 103 │        679 │         11 │ Информатика         │ ᴺᵁᴸᴸ │
    2. │            1548269 │ 1548269   │       1548269 │                           4 │          21 │      774 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │ 128 │        704 │         14 │ Информатика         │ ᴺᵁᴸᴸ │
    3. │            1549797 │ 1549797   │       1549797 │                           4 │          21 │      774 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │ 103 │        680 │         11 │ Информатика         │    4 │
    4. │            1550136 │ 1550136   │       1550136 │                           4 │          21 │      775 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │ 124 │        701 │         13 │ Информатика         │    4 │
    5. │            1550565 │ 1550565   │       1550565 │                           4 │          21 │      775 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │ 101 │        678 │         11 │ Информатика         │    5 │
21716. │            6476359 │ 6476359   │       6476359 │                          17 │          88 │     3238 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-04 │  72 │       2485 │          8 │ Физическая культура │    4 │
       └─student_profile_id─┴─person_id─┴─person_id_int─┴─educational_organization_id─┴─parallel_id─┴─class_id─┴─lesson_date─┴─lesson_month_digits─┴─lesson_month_text─┴─lesson_year─┴──load_date─┴───t─┴─teacher_id─┴─subject_id─┴─subject_name────────┴─mark─┘
Showed 1000 out of 27421 rows.

27421 rows in set. Elapsed: 1.256 sec. Processed 10.00 million rows, 1.14 GB (7.96 million rows/s., 905.51 MB/s.)
Peak memory usage: 48.14 MiB.
```

```sql
explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	lesson_date = '2025-09-03'::date;
```

Результат
```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 4/4                                  |
        Granules: 1223/1223                         |
```

#### 4. Добавляем minmax-индекс на поле lesson_date

```sql
ALTER TABLE learn_db.mart_student_lesson ADD INDEX idx_lesson_date lesson_date TYPE minmax GRANULARITY 2;
ALTER TABLE learn_db.mart_student_lesson MATERIALIZE INDEX idx_lesson_date;
```

#### Повторим выполнение запроса

```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	lesson_date = '2025-09-03'::date;
```

Результат
```text
Query id: b8f7396e-215d-4c21-a08e-0c8d9d93c0fc

       ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬───t─┬─teacher_id─┬─subject_id─┬─subject_name────────┬─mark─┐
    1. │               1365 │ 1365      │          1365 │                           0 │           0 │        0 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │  61 │         61 │          6 │ География           │    4 │
    2. │                456 │ 456       │           456 │                           0 │           0 │        0 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-04 │  33 │         33 │          3 │ Литература          │ ᴺᵁᴸᴸ │
    3. │               3575 │ 3575      │          3575 │                           0 │           0 │        1 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-04 │  90 │         91 │         10 │ Информатика         │ ᴺᵁᴸᴸ │
    4. │               4116 │ 4116      │          4116 │                           0 │           0 │        2 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │ 114 │        115 │         12 │ Информатика         │ ᴺᵁᴸᴸ │
    5. │               4986 │ 4986      │          4986 │                           0 │           0 │        2 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │  63 │         64 │          7 │ Биология            │ ᴺᵁᴸᴸ │
21370. │            7537813 │ 7537813   │       7537813 │                          20 │         103 │     3768 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-03 │  27 │       2835 │          3 │ Литература          │ ᴺᵁᴸᴸ │
       └─student_profile_id─┴─person_id─┴─person_id_int─┴─educational_organization_id─┴─parallel_id─┴─class_id─┴─lesson_date─┴─lesson_month_digits─┴─lesson_month_text─┴─lesson_year─┴──load_date─┴───t─┴─teacher_id─┴─subject_id─┴─subject_name────────┴─mark─┘
Showed 1000 out of 27421 rows.

27421 rows in set. Elapsed: 1.263 sec. Processed 10.00 million rows, 1.14 GB (7.92 million rows/s., 900.43 MB/s.)
Peak memory usage: 47.92 MiB.
```

```sql
explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	lesson_date = '2025-09-03'::date;
```

Результат
```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 4/4                                  |
        Granules: 1223/1223                         |
      Skip                                          |
        Name: idx_lesson_date                       |
        Description: minmax GRANULARITY 2           |
        Parts: 4/4                                  |
        Granules: 1223/1223                         |
```

**Вывод:** Minmax-индекс эффективен когда данные отсортированы по индексируемому полю. В моем случае PRIMARY KEY (class_id), поэтому данные физически отсортированы по class_id, а не по lesson_date. Minmax-индекс не может эффективно отсекать гранулы.

## Задание 2: Создайте set-индекс на колонку subject_id

Задание: Добавьте set-индекс на поле subject_id с размером множества 100. 
Выполните запрос с фильтром по редкому значению subject_id и сравните план выполнения.

###  5. Проверяем выполнение запроса до добавления индекса

```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	subject_id = 5;
```

Результат

```text
Query id: 1fe905c6-357d-4f22-ab2e-3113b183ceeb

        ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬──t─┬─teacher_id─┬─subject_id─┬─subject_name─┬─mark─┐
     1. │                348 │ 348       │           348 │                           0 │           0 │        0 │  2025-03-02 │ 2025-03             │ 2025 March        │        2025 │ 2025-03-04 │ 46 │         46 │          5 │ Химия        │ ᴺᵁᴸᴸ │
     2. │               1295 │ 1295      │          1295 │                           0 │           0 │        0 │  2025-01-08 │ 2025-01             │ 2025 January      │        2025 │ 2025-01-08 │ 53 │         53 │          5 │ Химия        │    3 │
     3. │                587 │ 587       │           587 │                           0 │           0 │        0 │  2024-11-30 │ 2024-11             │ 2024 November     │        2024 │ 2024-12-01 │ 52 │         52 │          5 │ Химия        │ ᴺᵁᴸᴸ │
     4. │                774 │ 774       │           774 │                           0 │           0 │        0 │  2025-08-17 │ 2025-08             │ 2025 August       │        2025 │ 2025-08-17 │ 53 │         53 │          5 │ Химия        │    4 │
     5. │                611 │ 611       │           611 │                           0 │           0 │        0 │  2024-10-22 │ 2024-10             │ 20
214470. │             455652 │ 455652    │        455652 │                           1 │           6 │      227 │  2025-07-20 │ 2025-07             │ 2025 July         │        2025 │ 2025-07-22 │ 47 │        216 │          5 │ Химия        │ ᴺᵁᴸᴸ │
        └─student_profile_id─┴─person_id─┴─person_id_int─┴─educational_organization_id─┴─parallel_id─┴─class_id─┴─lesson_date─┴─lesson_month_digits─┴─lesson_month_text─┴─lesson_year─┴──load_date─┴──t─┴─teacher_id─┴─subject_id─┴─subject_name─┴─mark─┘
Showed 1000 out of 1332803 rows.

1332803 rows in set. Elapsed: 4.811 sec. Processed 20.00 million rows, 2.27 GB (4.16 million rows/s., 472.71 MB/s.)
Peak memory usage: 56.92 MiB.
```

```
explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	subject_id = 5;
```

Результат

```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
```

###  6. Добавим индекс

```sql
ALTER TABLE learn_db.mart_student_lesson
ADD INDEX subject_id_set_index (subject_id) TYPE set(100) GRANULARITY 1;


ALTER TABLE learn_db.mart_student_lesson
MATERIALIZE INDEX subject_id_set_index;
```

###  7. Проверим скорость выполнения запроса

```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	subject_id = 5;
```

Результат

```text
Query id: e761227d-5fde-4445-a54c-4b371326afc0

        ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬──t─┬─teacher_id─┬─subject_id─┬─subject_name─┬─mark─┐
     1. │            2801607 │ 2801607   │       2801607 │                           7 │          38 │     1400 │  2025-04-08 │ 2025-04             │ 2025 April        │        2025 │ 2025-04-10 │ 46 │       1089 │          5 │ Химия        │ ᴺᵁᴸᴸ │
432737. │            1045101 │ 1045101   │       1045101 │                           2 │          14 │      522 │  2024-12-22 │ 2024-12             │ 2024 December     │        2024 │ 2024-12-24 │ 46 │        435 │          5 │ Химия        │    3 │
        └─student_profile_id─┴─person_id─┴─person_id_int─┴─educational_organization_id─┴─parallel_id─┴─class_id─┴─lesson_date─┴─lesson_month_digits─┴─lesson_month_text─┴─lesson_year─┴──load_date─┴──t─┴─teacher_id─┴─subject_id─┴─subject_name─┴─mark─┘
Showed 1000 out of 1332803 rows.

1332803 rows in set. Elapsed: 2.949 sec. Processed 20.00 million rows, 2.27 GB (6.78 million rows/s., 771.07 MB/s.)
Peak memory usage: 57.88 MiB.
```

```
explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	subject_id = 5;
```

Результат

```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
      Skip                                          |
        Name: subject_id_set_index                  |
        Description: set GRANULARITY 1              |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
```

**Вывод:** Set-индекс ускорил запрос на 63%. Он хорошо работает для колонок с низкой кардинальностью (малым количеством уникальных значений). 

## Задание 3: Используйте bloom_filter для поиска по person_id

Задание: Добавьте bloom_filter-индекс на поле person_id.
Выполните поиск по конкретному person_id и оцените, сколько блоков было пропущено.


###  8. Проверяем выполнение запроса до добавления индекса

```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	person_id = '183';
```

Результат

```text
Query id: 6b780fea-df5b-4129-ad25-d1dcc05851d6

   ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬──t─┬─teacher_id─┬─subject_id─┬─subject_name─┬─mark─┐
1. │                183 │ 183       │           183 │                           0 │           0 │        0 │  2025-10-02 │ 2025-10             │ 2025 October      │        2025 │ 2025-10-03 │  4 │          4 │          0 │ Информатика  │ ᴺᵁᴸᴸ │
2. │                183 │ 183       │           183 │                           0 │           0 │        0 │  2024-12-09 │ 2024-12             │ 2024 December     │        2024 │ 2024-12-10 │ 19 │         19 │          2 │ Русский язык │ ᴺᵁᴸᴸ │
   └────────────────────┴───────────┴───────────────┴─────────────────────────────┴─────────────┴──────────┴─────────────┴─────────────────────┴───────────────────┴─────────────┴────────────┴────┴────────────┴────────────┴──────────────┴──────┘

2 rows in set. Elapsed: 0.109 sec. Processed 20.00 million rows, 317.85 MB (183.57 million rows/s., 2.92 GB/s.)
Peak memory usage: 3.07 MiB.
```

```
explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	person_id = '183';
```

Результат

```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
```

###  9. Добавим индекс

```sql
ALTER TABLE learn_db.mart_student_lesson
ADD INDEX person_id_bloom_filter_idx person_id TYPE bloom_filter(0.001) GRANULARITY 1

ALTER TABLE learn_db.mart_student_lesson
MATERIALIZE INDEX person_id_bloom_filter_idx;
```

###  10. Проверим скорость выполнения запроса

```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	person_id = '183';
```

Результат

```text
Query id: 21e353ed-a856-4350-b31a-0f36111428c7

   ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬──t─┬─teacher_id─┬─subject_id─┬─subject_name─┬─mark─┐
1. │                183 │ 183       │           183 │                           0 │           0 │        0 │  2025-10-02 │ 2025-10             │ 2025 October      │        2025 │ 2025-10-03 │  4 │          4 │          0 │ Информатика  │ ᴺᵁᴸᴸ │
2. │                183 │ 183       │           183 │                           0 │           0 │        0 │  2024-12-09 │ 2024-12             │ 2024 December     │        2024 │ 2024-12-10 │ 19 │         19 │          2 │ Русский язык │ ᴺᵁᴸᴸ │
   └────────────────────┴───────────┴───────────────┴─────────────────────────────┴─────────────┴──────────┴─────────────┴─────────────────────┴───────────────────┴─────────────┴────────────┴────┴────────────┴────────────┴──────────────┴──────┘

2 rows in set. Elapsed: 0.019 sec. Processed 40.96 thousand rows, 703.81 KB (2.21 million rows/s., 38.01 MB/s.)
Peak memory usage: 2.43 MiB.
```

```
explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	person_id = '183';
```

Результат

```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
      Skip                                          |
        Name: person_id_bloom_filter_idx            |
        Description: bloom_filter GRANULARITY 1     |
        Parts: 3/3                                  |
        Granules: 5/2446                            |
```

**Вывод:** Bloom-filter индекс очень эффективен для строковых полей с высокой кардинальностью. Количество обработанных гранул уменьшилось с 2446 до 5!

## Задание 4: Проверьте влияние индекса на колонку с высокой кардинальностью

Добавьте set-индекс на поле student_profile_id и выполните поиск по конкретному значению.
Оцените, насколько индекс помогает (или нет).


###  11. Проверяем выполнение запроса до добавления индекса

```sql
SELECT 
	student_profile_id
FROM
	learn_db.mart_student_lesson
WHERE 
	person_id = '5017140';
```

Результат

```text
Query id: 8d928a9c-0bf5-489b-a91b-ce9f0781524a

   ┌─student_profile_id─┐
1. │            5017140 │
2. │            5017140 │
   └────────────────────┘

2 rows in set. Elapsed: 0.018 sec. Processed 40.96 thousand rows, 686.50 KB (2.26 million rows/s., 37.88 MB/s.)
Peak memory usage: 27.95 KiB.
```

```
explain indexes = 1
SELECT 
	student_profile_id
FROM
	learn_db.mart_student_lesson
WHERE 
	person_id = '5017140';
```

Результат

```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
      Skip                                          |
        Name: person_id_bloom_filter_idx            |
        Description: bloom_filter GRANULARITY 1     |
        Parts: 2/3                                  |
        Granules: 5/2446                            |
```

###  12. Добавим индекс

```sql
ALTER TABLE learn_db.mart_student_lesson
ADD INDEX student_profile_id_set_index (student_profile_id) TYPE set(100) GRANULARITY 1;


ALTER TABLE learn_db.mart_student_lesson
MATERIALIZE INDEX student_profile_id_set_index;
```

###  13. Проверим скорость выполнения запроса

```sql
SELECT 
	student_profile_id
FROM
	learn_db.mart_student_lesson
WHERE 
	person_id = '5017140';
```

Результат
```text
Query id: 5b7f8dfd-182f-46c6-a4f7-20073b7928c9

   ┌─student_profile_id─┐
1. │            5017140 │
2. │            5017140 │
   └────────────────────┘

2 rows in set. Elapsed: 0.014 sec. Processed 40.96 thousand rows, 686.50 KB (3.01 million rows/s., 50.45 MB/s.)
Peak memory usage: 235.79 KiB.
```

```
explain indexes = 1
SELECT 
	student_profile_id
FROM
	learn_db.mart_student_lesson
WHERE 
	person_id = '5017140';
```

Результат
```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
      Skip                                          |
        Name: person_id_bloom_filter_idx            |
        Description: bloom_filter GRANULARITY 1     |
        Parts: 3/3                                  |
        Granules: 5/2446                            |
```

**Вывод**: Set-индексы не эффективны для полей с высокой кардинальностью (почти 10 млн уникальных значений). ClickHouse тратит больше времени на проверку индекса чем на прямой поиск.

## Задание 5: Сравните эффективность minmax-индекса на lesson_year и lesson_date
Задание: Добавьте minmax-индексы на оба поля и выполните запросы с фильтрацией по году и по дате. Сравните, какой индекс эффективнее.


###  14. Проверяем выполнение запроса до добавления индексов

```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	lesson_year = '2025' -- 15468042 количество строк за год
	and 
	lesson_date = '2025-09-03'; -- 54953 количество строк за дату
```

Результат
```text
Query id: c8b1167b-3ea4-4ee6-9f81-1f9a6b9d2dcc

       ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬───t─┬─teacher_id─┬─subject_id─┬─subject_name────────┬─mark─┐
    1. │               1163 │ 1163      │          1163 │                           0 │           0 │        0 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-04 │  70 │         70 │          7 │ Биология            │    2 │
    2. │               1469 │ 1469      │          1469 │                           0 │           0 │        0 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-03 │   9 │          9 │          1 │ Математика          │    4 │
    3. │                 17 │ 17        │            17 │                           0 │           0 │        0 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-03 │  78 │         78 │          8 │ Физическая культура │ ᴺᵁᴸᴸ │
    4. │               1205 │ 1205      │          1205 │                           0 │           0 │        0 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-04 │  81 │         81 │          9 │ Информатика         │ ᴺᵁᴸᴸ │
    5. │               1906 │ 1906      │          1906 │                           0 │           0 │        0 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-03 │  27 │         27 │          3 │ Литература          │    4 │
17018. │            3370174 │ 3370174   │       3370174 │                           9 │          46 │     1685 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-04 │  91 │       1346 │         10 │ Информатика         │ ᴺᵁᴸᴸ │
       └─student_profile_id─┴─person_id─┴─person_id_int─┴─educational_organization_id─┴─parallel_id─┴─class_id─┴─lesson_date─┴─lesson_month_digits─┴─lesson_month_text─┴─lesson_year─┴──load_date─┴───t─┴─teacher_id─┴─subject_id─┴─subject_name────────┴─mark─┘
Showed 1000 out of 54953 rows.

54953 rows in set. Elapsed: 2.981 sec. Processed 20.00 million rows, 2.27 GB (6.71 million rows/s., 762.88 MB/s.)
Peak memory usage: 51.32 MiB.
```

```
explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	lesson_year = '2025'
	and 
	lesson_date = '2025-09-03'; 
```

Результат
```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
      Skip                                          |
        Name: idx_lesson_date                       |
        Description: minmax GRANULARITY 2           |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
```

###  15. Добавим индексы

```sql
ALTER TABLE learn_db.mart_student_lesson ADD INDEX idx_lesson_year lesson_year TYPE minmax GRANULARITY 2;
ALTER TABLE learn_db.mart_student_lesson MATERIALIZE INDEX idx_lesson_year;

ALTER TABLE learn_db.mart_student_lesson ADD INDEX idx_lesson_date lesson_date TYPE minmax GRANULARITY 2;
ALTER TABLE learn_db.mart_student_lesson MATERIALIZE INDEX idx_lesson_date;
```

###  16. Проверим скорость выполнения запроса

```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	lesson_year = '2025'
	and 
	lesson_date = '2025-09-03';
```

Результат
```text
Query id: 032da213-65b3-46ec-b17d-e0ae0fccad90

       ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬───t─┬─teacher_id─┬─subject_id─┬─subject_name────────┬─mark─┐
    1. │            5018705 │ 5018705   │       5018705 │                          13 │          68 │     2509 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-03 │  25 │       1894 │          2 │ Русский язык        │ ᴺᵁᴸᴸ │
    2. │            5018784 │ 5018784   │       5018784 │                          13 │          68 │     2509 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-04 │  38 │       1908 │          4 │ Физика              │ ᴺᵁᴸᴸ │
    3. │            5019909 │ 5019909   │       5019909 │                          13 │          68 │     2509 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │ 125 │       1995 │         13 │ Информатика         │    3 │
    4. │            5019245 │ 5019245   │       5019245 │                          13 │          68 │     2509 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-03 │  64 │       1934 │          7 │ Биология            │ ᴺᵁᴸᴸ │
    5. │            5021043 │ 5021043   │       5021043 │                          13 │          68 │     2510 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-04 │ 126 │       1996 │         14 │ Информатика         │ ᴺᵁᴸᴸ │
15240. │            6030002 │ 6030002   │       6030002 │                          16 │          82 │     3015 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-03 │  25 │       2271 │          2 │ Русский язык        │    4 │
       └─student_profile_id─┴─person_id─┴─person_id_int─┴─educational_organization_id─┴─parallel_id─┴─class_id─┴─lesson_date─┴─lesson_month_digits─┴─lesson_month_text─┴─lesson_year─┴──load_date─┴───t─┴─teacher_id─┴─subject_id─┴─subject_name────────┴─mark─┘
Showed 1000 out of 54953 rows.

54953 rows in set. Elapsed: 4.945 sec. Processed 20.00 million rows, 2.27 GB (4.04 million rows/s., 459.91 MB/s.)
Peak memory usage: 51.30 MiB.
```

```
explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	lesson_year = '2025'
	and 
	lesson_date = '2025-09-03'; 
```

Результат
```text
  explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
      Skip                                          |
        Name: idx_lesson_year                       |
        Description: minmax GRANULARITY 2           |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
      Skip                                          |
        Name: idx_lesson_date                       |
        Description: minmax GRANULARITY 2           |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
```

**Вывод:** Незначительное ухудшение. Тот же эффект что и в задании 1 - данные не отсортированы по этим полям.

## Задание 6: Оцените влияние granularity на эффективность индекса

Задание: Создайте два minmax-индекса на lesson_date с granularity 1 и 10. Сравните размер индекса и количество пропущенных блоков при одинаковом запросе.

### 17. Удалим индекс

```sql
ALTER TABLE learn_db.mart_student_lesson DROP INDEX idx_lesson_date;
```


###  18. Проверяем выполнение запроса до добавления индекса


```sql
SELECT 	*
FROM
	learn_db.mart_student_lesson
WHERE 
	lesson_date = '2025-09-03'; 
```

Результат
```text
Query id: 160f46b1-3d07-4d0c-9f43-cb6ee4b26112

       ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬───t─┬─teacher_id─┬─subject_id─┬─subject_name────────┬─mark─┐
    1. │             295460 │ 295460    │        295460 │                           0 │           4 │      147 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │  34 │        144 │          3 │ Литература          │ ᴺᵁᴸᴸ │
    2. │             296149 │ 296149    │        296149 │                           0 │           4 │      148 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-03 │  62 │        172 │          6 │ География           │ ᴺᵁᴸᴸ │
    3. │             296341 │ 296341    │        296341 │                           0 │           4 │      148 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │  71 │        181 │          7 │ Биология            │    3 │
    4. │             296757 │ 296757    │        296757 │                           0 │           4 │      148 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-04 │  35 │        145 │          3 │ Литература          │    4 │
    5. │             299723 │ 299723    │        299723 │                           0 │           4 │      149 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-04 │  64 │        175 │          7 │ Биология            │ ᴺᵁᴸᴸ │
18462. │            1228219 │ 1228219   │       1228219 │                           3 │          16 │      614 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │  52 │        509 │          5 │ Химия               │ ᴺᵁᴸᴸ │
       └─student_profile_id─┴─person_id─┴─person_id_int─┴─educational_organization_id─┴─parallel_id─┴─class_id─┴─lesson_date─┴─lesson_month_digits─┴─lesson_month_text─┴─lesson_year─┴──load_date─┴───t─┴─teacher_id─┴─subject_id─┴─subject_name────────┴─mark─┘
Showed 1000 out of 54953 rows.

54953 rows in set. Elapsed: 3.581 sec. Processed 20.00 million rows, 2.27 GB (5.59 million rows/s., 635.04 MB/s.)
Peak memory usage: 48.21 MiB.
```

```sql
explain indexes = 1
SELECT 	*
FROM
	learn_db.mart_student_lesson
WHERE 
	lesson_date = '2025-09-03'; 
```

Результат
```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
```

###  19. Добавим индекс с гранулярностью 1

```sql
ALTER TABLE learn_db.mart_student_lesson ADD INDEX idx_lesson_date lesson_date TYPE minmax GRANULARITY 1;
ALTER TABLE learn_db.mart_student_lesson MATERIALIZE INDEX idx_lesson_date;
```

###  20. Проверим скорость выполнения запроса

```sql
SELECT 	*
FROM
	learn_db.mart_student_lesson
WHERE 
	lesson_date = '2025-09-03'; 
```

Результат
```text
Query id: e94a1bfa-36ac-42b0-afbe-f65611f900b8

       ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬───t─┬─teacher_id─┬─subject_id─┬─subject_name────────┬─mark─┐
    1. │            2511202 │ 2511202   │       2511202 │                           6 │          34 │     1255 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-03 │  80 │       1015 │          8 │ Физическая культура │    2 │
    2. │            2511606 │ 2511606   │       2511606 │                           6 │          34 │     1255 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │  32 │        967 │          3 │ Литература          │    5 │
    3. │            2513973 │ 2513973   │       2513973 │                           6 │          34 │     1256 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │  67 │       1003 │          7 │ Биология            │ ᴺᵁᴸᴸ │
    4. │            2515243 │ 2515243   │       2515243 │                           6 │          34 │     1257 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │ 103 │       1040 │         11 │ Информатика         │ ᴺᵁᴸᴸ │
    5. │            2515908 │ 2515908   │       2515908 │                           6 │          34 │     1257 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │  33 │        970 │          3 │ Литература          │ ᴺᵁᴸᴸ │
15745. │            8151674 │ 8151674   │       8151674 │                          22 │         111 │     4075 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-03 │ 111 │       3148 │         12 │ Информатика         │    4 │
       └─student_profile_id─┴─person_id─┴─person_id_int─┴─educational_organization_id─┴─parallel_id─┴─class_id─┴─lesson_date─┴─lesson_month_digits─┴─lesson_month_text─┴─lesson_year─┴──load_date─┴───t─┴─teacher_id─┴─subject_id─┴─subject_name────────┴─mark─┘
Showed 1000 out of 54953 rows.

54953 rows in set. Elapsed: 3.477 sec. Processed 20.00 million rows, 2.27 GB (5.75 million rows/s., 654.08 MB/s.)
Peak memory usage: 50.85 MiB.
```

```sql
explain indexes = 1
SELECT 	*
FROM
	learn_db.mart_student_lesson
WHERE 
	lesson_date = '2025-09-03'; 
```

Результат
```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
      Skip                                          |
        Name: idx_lesson_date                       |
        Description: minmax GRANULARITY 1           |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
```

### 21. Удалим индекс

```sql
ALTER TABLE learn_db.mart_student_lesson DROP INDEX idx_lesson_date;
```

###  22. Добавим индекс с гранулярностью 10

```sql
ALTER TABLE learn_db.mart_student_lesson ADD INDEX idx_lesson_date lesson_date TYPE minmax GRANULARITY 10;
ALTER TABLE learn_db.mart_student_lesson MATERIALIZE INDEX idx_lesson_date;
```

###  23. Проверим скорость выполнения запроса

```sql
SELECT 	*
FROM
	learn_db.mart_student_lesson
WHERE 
	lesson_date = '2025-09-03'; 
```

Результат
```text
Query id: 35bdb66b-1d86-43b4-ada3-8983814e43ae

       ┌─student_profile_id─┬─person_id─┬─person_id_int─┬─educational_organization_id─┬─parallel_id─┬─class_id─┬─lesson_date─┬─lesson_month_digits─┬─lesson_month_text─┬─lesson_year─┬──load_date─┬───t─┬─teacher_id─┬─subject_id─┬─subject_name────────┬─mark─┐
    1. │            2511202 │ 2511202   │       2511202 │                           6 │          34 │     1255 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-03 │  80 │       1015 │          8 │ Физическая культура │    2 │
    2. │            2511606 │ 2511606   │       2511606 │                           6 │          34 │     1255 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │  32 │        967 │          3 │ Литература          │    5 │
    3. │            2513973 │ 2513973   │       2513973 │                           6 │          34 │     1256 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │  67 │       1003 │          7 │ Биология            │ ᴺᵁᴸᴸ │
    4. │            2515243 │ 2515243   │       2515243 │                           6 │          34 │     1257 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │ 103 │       1040 │         11 │ Информатика         │ ᴺᵁᴸᴸ │
    5. │            2515908 │ 2515908   │       2515908 │                           6 │          34 │     1257 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │  33 │        970 │          3 │ Литература          │ ᴺᵁᴸᴸ │
14758. │             652027 │ 652027    │        652027 │                           1 │           8 │      326 │  2025-09-03 │ 2025-09             │ 2025 September    │        2025 │ 2025-09-05 │  11 │        253 │          1 │ Математика          │    4 │
       └─student_profile_id─┴─person_id─┴─person_id_int─┴─educational_organization_id─┴─parallel_id─┴─class_id─┴─lesson_date─┴─lesson_month_digits─┴─lesson_month_text─┴─lesson_year─┴──load_date─┴───t─┴─teacher_id─┴─subject_id─┴─subject_name────────┴─mark─┘
Showed 1000 out of 54953 rows.

54953 rows in set. Elapsed: 4.731 sec. Processed 20.00 million rows, 2.27 GB (4.23 million rows/s., 480.68 MB/s.)
Peak memory usage: 48.96 MiB.
```

```sql
explain indexes = 1
SELECT 	*
FROM
	learn_db.mart_student_lesson
WHERE 
	lesson_date = '2025-09-03'; 
```

Результат
```text
explain                                             |
----------------------------------------------------+
Expression ((Project names + Projection))           |
  Expression                                        |
    ReadFromMergeTree (learn_db.mart_student_lesson)|
    Indexes:                                        |
      PrimaryKey                                    |
        Condition: true                             |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
      Skip                                          |
        Name: idx_lesson_date                       |
        Description: minmax GRANULARITY 10          |
        Parts: 3/3                                  |
        Granules: 2446/2446                         |
```

**Вывод:** Меньшая granularity (1) немного эффективнее чем большая (10), но разница незначительная поскольку основной проблемой остается отсутствие сортировки по индексируемому полю.