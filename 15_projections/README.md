# Проекции

### 1. Пересоздаем таблицу и наполняем ее данными
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson; 
CREATE TABLE learn_db.mart_student_lesson
(
	`student_profile_id` Int32 , -- Идентификатор профиля обучающегося
	`person_id` String , -- GUID обучающегося
	`person_id_int` Int32 ,
	`educational_organization_id` Int16 , -- Идентификатор образовательной организации
	`parallel_id` Int16 ,
	`class_id` Int16 , -- Идентификатор класса
	`lesson_date` Date32 , -- Дата урока
	`lesson_month_digits` String ,
	`lesson_month_text` String ,
	`lesson_year` UInt16 ,
	`load_date` Date , -- Дата загрузки данных
	`t` Int16 CODEC(Delta, ZSTD),
	`teacher_id` Int32 , -- Идентификатор учителя
	`subject_id` Int16 , -- Идентификатор предмета
	`subject_name` String ,
	`mark` Nullable(UInt8) , -- Оценка
	PRIMARY KEY(lesson_date)
) ENGINE = MergeTree()
AS SELECT
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

### 2. Находим первый запрос, который приходит из DataLens

```sql
SELECT * FROM system.query_log WHERE http_user_agent = 'DataLens' ORDER BY event_time DESC;
```

### 3. Исследуем эффективность выполнения первого запроса

```sql
EXPLAIN indexes = 1
SELECT t1.subject_name AS res_0, avg(t1.mark) AS res_1
FROM learn_db.mart_student_lesson AS t1
WHERE t1.lesson_date BETWEEN toDate32('2025-10-09') AND toDate32('2025-10-15')
GROUP BY res_0
LIMIT 1000001
```

Результат
```text
explain                                                                                     |
--------------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                                   |
  Limit (preliminary LIMIT (without OFFSET))                                                |
    Aggregating                                                                             |
      Expression (Before GROUP BY)                                                          |
        Expression                                                                          |
          ReadFromMergeTree (learn_db.mart_student_lesson)                                  |
          Indexes:                                                                          |
            PrimaryKey                                                                      |
              Keys:                                                                         |
                lesson_date                                                                 |
              Condition: and((lesson_date in (-Inf, 20376]), (lesson_date in [20370, +Inf)))|
              Parts: 4/4                                                                    |
              Granules: 26/1223                                                             |
```

### 4. Исследуем эффективность выполнения второго запроса

```sql
EXPLAIN indexes = 1
SELECT t1.subject_name AS res_0, avg(t1.mark) AS res_1
FROM learn_db.mart_student_lesson AS t1
WHERE t1.educational_organization_id IN (1) AND t1.lesson_date BETWEEN toDate32('2024-10-16') AND toDate32('2025-10-15')
GROUP BY res_0
LIMIT 1000001
```

Результат
```text
explain                                                                                     |
--------------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                                   |
  Limit (preliminary LIMIT (without OFFSET))                                                |
    Aggregating                                                                             |
      Expression (Before GROUP BY)                                                          |
        Expression                                                                          |
          ReadFromMergeTree (learn_db.mart_student_lesson)                                  |
          Indexes:                                                                          |
            PrimaryKey                                                                      |
              Keys:                                                                         |
                lesson_date                                                                 |
              Condition: and((lesson_date in (-Inf, 20376]), (lesson_date in [20012, +Inf)))|
              Parts: 4/4                                                                    |
              Granules: 1223/1223                                                           |
```

### 5. Пробуем оптимизировать второй запрос, добавив в первичный индекс поле educational_organization_id

```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson; 
CREATE TABLE learn_db.mart_student_lesson
(
	`student_profile_id` Int32 , -- Идентификатор профиля обучающегося
	`person_id` String , -- GUID обучающегося
	`person_id_int` Int32 ,
	`educational_organization_id` Int16 , -- Идентификатор образовательной организации
	`parallel_id` Int16 ,
	`class_id` Int16 , -- Идентификатор класса
	`lesson_date` Date32 , -- Дата урока
	`lesson_month_digits` String ,
	`lesson_month_text` String ,
	`lesson_year` UInt16 ,
	`load_date` Date , -- Дата загрузки данных
	`t` Int16 CODEC(Delta, ZSTD),
	`teacher_id` Int32 , -- Идентификатор учителя
	`subject_id` Int16 , -- Идентификатор предмета
	`subject_name` String ,
	`mark` Nullable(UInt8) , -- Оценка
	PRIMARY KEY(lesson_date, educational_organization_id)
) ENGINE = MergeTree()
AS SELECT
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

### 6. Проверяем эффективность второго запроса

```sql
EXPLAIN indexes = 1
SELECT t1.subject_name AS res_0, avg(t1.mark) AS res_1
FROM learn_db.mart_student_lesson AS t1
WHERE t1.educational_organization_id IN (1) AND t1.lesson_date BETWEEN toDate32('2024-10-16') AND toDate32('2025-10-15')
GROUP BY res_0
LIMIT 1000001
```

Результат
```text
explain                                                                                                                                          |
-------------------------------------------------------------------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                                                                                        |
  Limit (preliminary LIMIT (without OFFSET))                                                                                                     |
    Aggregating                                                                                                                                  |
      Expression (Before GROUP BY)                                                                                                               |
        Expression                                                                                                                               |
          ReadFromMergeTree (learn_db.mart_student_lesson)                                                                                       |
          Indexes:                                                                                                                               |
            PrimaryKey                                                                                                                           |
              Keys:                                                                                                                              |
                lesson_date                                                                                                                      |
                educational_organization_id                                                                                                      |
              Condition: and(and((lesson_date in (-Inf, 20376]), (lesson_date in [20012, +Inf))), (educational_organization_id in 1-element set))|
              Parts: 4/4                                                                                                                         |
              Granules: 829/1223                                                                                                                 |
```
Индекс сработал, но эффективность фильтрации низкая. 

### 7. Смотрим, сколько времени выполняются шаги запроса

```sql
SELECT * FROM system.query_log ORDER BY event_time DESC;
```

Результат
```text
query_id  = 173dfb41-d66e-44f0-8ade-d09653bcda12
```

Запросим информацию о времени выполнения каждого шага запроса.

```sql
SELECT * FROM system.processors_profile_log WHERE query_id = '173dfb41-d66e-44f0-8ade-d09653bcda12' order by processor_uniq_id;
```

Результат
```text
hostname    |event_date|event_time         |event_time_microseconds      |id             |parent_ids                                                                                                                                                                                       |plan_step      |plan_step_name   |plan_step_description             |plan_group|initial_query_id                    |query_id                            |name                                              |elapsed_us|input_wait_elapsed_us|output_wait_elapsed_us|input_rows|input_bytes|output_rows|output_bytes|processor_uniq_id                                    |step_uniq_id       |
------------+----------+-------------------+-----------------------------+---------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+-----------------+----------------------------------+----------+------------------------------------+------------------------------------+--------------------------------------------------+----------+---------------------+----------------------+----------+-----------+-----------+------------+-----------------------------------------------------+-------------------+
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324754840|[140285324478488]                                                                                                                                                                                |140285632238336|Aggregating      |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|AggregatingTransform                              |      1640|                59739|                     0|     19198|     607100|          0|           0|AggregatingTransform_37                              |Aggregating_4      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324757400|[140285324478488]                                                                                                                                                                                |140285632238336|Aggregating      |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|AggregatingTransform                              |      1021|                64612|                     0|     28035|     888180|          0|           0|AggregatingTransform_38                              |Aggregating_4      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324758680|[140285324478488]                                                                                                                                                                                |140285632238336|Aggregating      |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|AggregatingTransform                              |      1234|                62664|                     0|     27940|     884861|          0|           0|AggregatingTransform_39                              |Aggregating_4      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324760600|[140285324478488]                                                                                                                                                                                |140285632238336|Aggregating      |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|AggregatingTransform                              |       927|                58098|                     0|     20905|     661627|          0|           0|AggregatingTransform_40                              |Aggregating_4      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324761240|[140285324478488]                                                                                                                                                                                |140285632238336|Aggregating      |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|AggregatingTransform                              |       803|                59702|                     0|     18291|     580413|          0|           0|AggregatingTransform_41                              |Aggregating_4      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324761880|[140285324478488]                                                                                                                                                                                |140285632238336|Aggregating      |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|AggregatingTransform                              |       920|                58539|                     0|     17336|     548237|          0|           0|AggregatingTransform_42                              |Aggregating_4      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324762520|[140285324478488]                                                                                                                                                                                |140285632238336|Aggregating      |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|AggregatingTransform                              |      2083|                57953|                     0|     41116|    1301534|          0|           0|AggregatingTransform_43                              |Aggregating_4      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285325971480|[140285324478488]                                                                                                                                                                                |140285632238336|Aggregating      |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|AggregatingTransform                              |      1409|                59346|                     0|     40990|    1297729|          0|           0|AggregatingTransform_44                              |Aggregating_4      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285325972120|[140285324478488]                                                                                                                                                                                |140285632238336|Aggregating      |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|AggregatingTransform                              |      3056|                64945|                    19|     52742|    1668541|          9|         340|AggregatingTransform_45                              |Aggregating_4      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285325972760|[140285324478488]                                                                                                                                                                                |140285632238336|Aggregating      |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|AggregatingTransform                              |      2205|                61930|                     0|     40655|    1288233|          0|           0|AggregatingTransform_46                              |Aggregating_4      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285325973400|[140285324478488]                                                                                                                                                                                |140285632238336|Aggregating      |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|AggregatingTransform                              |       513|                60856|                     0|     13339|     423766|          0|           0|AggregatingTransform_47                              |Aggregating_4      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285325974040|[140285324478488]                                                                                                                                                                                |140285632238336|Aggregating      |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|AggregatingTransform                              |      1703|                64079|                     0|     44023|    1391746|          0|           0|AggregatingTransform_48                              |Aggregating_4      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285570695064|[140285325972120]                                                                                                                                                                                |              0|                 |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ConvertingAggregatedToChunksTransform             |       119|                    0|                     0|         0|          0|          9|         340|ConvertingAggregatedToChunksTransform_71             |                   |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285327596056|[140285324456472]                                                                                                                                                                                |140285634447872|Expression       |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |       164|                59251|                  1858|     19198|     722288|      19198|      607100|ExpressionTransform_13                               |Expression_10      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140284836677656|[140285324456984]                                                                                                                                                                                |140285634447872|Expression       |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |       164|                64062|                  1359|     28035|    1056390|      28035|      888180|ExpressionTransform_14                               |Expression_10      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140284838764568|[140285324457496]                                                                                                                                                                                |140285634447872|Expression       |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |       206|                60906|                  2680|     27940|    1052501|      27940|      884861|ExpressionTransform_15                               |Expression_10      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140284838765080|[140285324458008]                                                                                                                                                                                |140285634447872|Expression       |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |       124|                57668|                  1140|     20905|     787057|      20905|      661627|ExpressionTransform_16                               |Expression_10      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140284838765592|[140285324458520]                                                                                                                                                                                |140285634447872|Expression       |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |       125|                59283|                  1025|     18291|     690159|      18291|      580413|ExpressionTransform_17                               |Expression_10      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140284838767640|[140285324459032]                                                                                                                                                                                |140285634447872|Expression       |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |        67|                58160|                  1185|     17336|     652253|      17336|      548237|ExpressionTransform_18                               |Expression_10      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140284838768152|[140285324459544]                                                                                                                                                                                |140285634447872|Expression       |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |       151|                57407|                  2353|     41116|    1548230|      41116|     1301534|ExpressionTransform_19                               |Expression_10      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140284839527448|[140285324476440]                                                                                                                                                                                |140285634447872|Expression       |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |       149|                58860|                  1664|     40990|    1543669|      40990|     1297729|ExpressionTransform_20                               |Expression_10      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140284839528472|[140285324476952]                                                                                                                                                                                |140285634447872|Expression       |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |       185|                64235|                  3354|     52733|    1984599|      52733|     1668201|ExpressionTransform_21                               |Expression_10      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140284839528984|[140284838767128]                                                                                                                                                                                |140285634447872|Expression       |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |       139|                61416|                  2466|     40655|    1532163|      40655|     1288233|ExpressionTransform_22                               |Expression_10      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140284839530008|[140285324477464]                                                                                                                                                                                |140285634447872|Expression       |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |        48|                60670|                   585|     13339|     503800|      13339|      423766|ExpressionTransform_23                               |Expression_10      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324455960|[140285324477976]                                                                                                                                                                                |140285634447872|Expression       |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |       173|                63329|                  2230|     44023|    1655884|      44023|     1391746|ExpressionTransform_24                               |Expression_10      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324456472|[140285324754840]                                                                                                                                                                                |140288467254528|Expression       |Before GROUP BY                   |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |        48|                59590|                  1720|     19198|     607100|      19198|      607100|ExpressionTransform_25                               |Expression_3       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324456984|[140285324757400]                                                                                                                                                                                |140288467254528|Expression       |Before GROUP BY                   |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |        61|                64413|                  1168|     28035|     888180|      28035|      888180|ExpressionTransform_26                               |Expression_3       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324457496|[140285324758680]                                                                                                                                                                                |140288467254528|Expression       |Before GROUP BY                   |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |        67|                62454|                  1339|     27940|     884861|      27940|      884861|ExpressionTransform_27                               |Expression_3       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324458008|[140285324760600]                                                                                                                                                                                |140288467254528|Expression       |Before GROUP BY                   |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |        46|                57949|                   995|     20905|     661627|      20905|      661627|ExpressionTransform_28                               |Expression_3       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324458520|[140285324761240]                                                                                                                                                                                |140288467254528|Expression       |Before GROUP BY                   |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |        44|                59550|                   883|     18291|     580413|      18291|      580413|ExpressionTransform_29                               |Expression_3       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324459032|[140285324761880]                                                                                                                                                                                |140288467254528|Expression       |Before GROUP BY                   |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |        28|                58453|                   963|     17336|     548237|      17336|      548237|ExpressionTransform_30                               |Expression_3       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324459544|[140285324762520]                                                                                                                                                                                |140288467254528|Expression       |Before GROUP BY                   |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |        52|                57765|                  2192|     41116|    1301534|      41116|     1301534|ExpressionTransform_31                               |Expression_3       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324476440|[140285325971480]                                                                                                                                                                                |140288467254528|Expression       |Before GROUP BY                   |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |        54|                59176|                  1488|     40990|    1297729|      40990|     1297729|ExpressionTransform_32                               |Expression_3       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324476952|[140285325972120]                                                                                                                                                                                |140288467254528|Expression       |Before GROUP BY                   |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |        65|                64627|                  3150|     52733|    1668201|      52733|     1668201|ExpressionTransform_33                               |Expression_3       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140284838767128|[140285325972760]                                                                                                                                                                                |140288467254528|Expression       |Before GROUP BY                   |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |        51|                61760|                  2310|     40655|    1288233|      40655|     1288233|ExpressionTransform_34                               |Expression_3       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324477464|[140285325973400]                                                                                                                                                                                |140288467254528|Expression       |Before GROUP BY                   |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |        16|                60786|                   530|     13339|     423766|      13339|      423766|ExpressionTransform_35                               |Expression_3       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324477976|[140285325974040]                                                                                                                                                                                |140288467254528|Expression       |Before GROUP BY                   |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |       100|                63685|                  1838|     44023|    1391746|      44023|     1391746|ExpressionTransform_36                               |Expression_3       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324479512|[140285324735512]                                                                                                                                                                                |140285634446336|Expression       |(Project names + Projection)      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |         4|                68116|                     2|         9|        340|          9|         340|ExpressionTransform_51                               |Expression_9       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324480024|[140285324735512]                                                                                                                                                                                |140285634446336|Expression       |(Project names + Projection)      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |         0|                68085|                     0|         0|          0|          0|           0|ExpressionTransform_52                               |Expression_9       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324480536|[140285324735512]                                                                                                                                                                                |140285634446336|Expression       |(Project names + Projection)      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |         0|                68086|                     0|         0|          0|          0|           0|ExpressionTransform_53                               |Expression_9       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324481048|[140285324735512]                                                                                                                                                                                |140285634446336|Expression       |(Project names + Projection)      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |         0|                68087|                     0|         0|          0|          0|           0|ExpressionTransform_54                               |Expression_9       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324481560|[140285324735512]                                                                                                                                                                                |140285634446336|Expression       |(Project names + Projection)      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |         0|                68087|                     0|         0|          0|          0|           0|ExpressionTransform_55                               |Expression_9       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324482072|[140285324735512]                                                                                                                                                                                |140285634446336|Expression       |(Project names + Projection)      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |         0|                68087|                     0|         0|          0|          0|           0|ExpressionTransform_56                               |Expression_9       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324482584|[140285324735512]                                                                                                                                                                                |140285634446336|Expression       |(Project names + Projection)      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |         0|                68087|                     0|         0|          0|          0|           0|ExpressionTransform_57                               |Expression_9       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324483096|[140285324735512]                                                                                                                                                                                |140285634446336|Expression       |(Project names + Projection)      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |         0|                68087|                     0|         0|          0|          0|           0|ExpressionTransform_58                               |Expression_9       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324483608|[140285324735512]                                                                                                                                                                                |140285634446336|Expression       |(Project names + Projection)      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |         0|                68087|                     0|         0|          0|          0|           0|ExpressionTransform_59                               |Expression_9       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324484120|[140285324735512]                                                                                                                                                                                |140285634446336|Expression       |(Project names + Projection)      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |         0|                68087|                     0|         0|          0|          0|           0|ExpressionTransform_60                               |Expression_9       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324734488|[140285324735512]                                                                                                                                                                                |140285634446336|Expression       |(Project names + Projection)      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |         0|                68088|                     0|         0|          0|          0|           0|ExpressionTransform_61                               |Expression_9       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324735000|[140285324735512]                                                                                                                                                                                |140285634446336|Expression       |(Project names + Projection)      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ExpressionTransform                               |         0|                68088|                     0|         0|          0|          0|           0|ExpressionTransform_62                               |Expression_9       |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324479000|[140285324479512,140285324480024,140285324480536,140285324481048,140285324481560,140285324482072,140285324482584,140285324483096,140285324483608,140285324484120,140285324734488,140285324735000]|140288439440064|Limit            |preliminary LIMIT (without OFFSET)|         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|Limit                                             |         0|                68123|                     8|         9|        340|          9|         340|Limit_50                                             |Limit_6            |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285325974680|[140283050532504]                                                                                                                                                                                |              0|                 |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|LimitsCheckingTransform                           |         1|                68137|                   133|         9|        340|          9|         340|LimitsCheckingTransform_64                           |                   |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140283050532504|[140283050469400]                                                                                                                                                                                |              0|                 |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|MaterializingTransform                            |         0|                68144|                   128|         9|        340|          9|         340|MaterializingTransform_68                            |                   |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285449749016|[140285327596056]                                                                                                                                                                                |140285630729728|ReadFromMergeTree|learn_db.mart_student_lesson      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|     58900|                    0|                  2228|         0|          0|      19198|      722288|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_1 |ReadFromMergeTree_0|
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324067352|[140284839528984]                                                                                                                                                                                |140285630729728|ReadFromMergeTree|learn_db.mart_student_lesson      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|     60863|                    0|                  2838|         0|          0|      40655|     1532163|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_10|ReadFromMergeTree_0|
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285327593496|[140284839530008]                                                                                                                                                                                |140285630729728|ReadFromMergeTree|learn_db.mart_student_lesson      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|     60137|                    0|                   708|         0|          0|      13339|      503800|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_11|ReadFromMergeTree_0|
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285327595544|[140285324455960]                                                                                                                                                                                |140285630729728|ReadFromMergeTree|learn_db.mart_student_lesson      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|     62641|                    0|                  2593|         0|          0|      44023|     1655884|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_12|ReadFromMergeTree_0|
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285451190296|[140284836677656]                                                                                                                                                                                |140285630729728|ReadFromMergeTree|learn_db.mart_student_lesson      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|     63696|                    0|                  1744|         0|          0|      28035|     1056390|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_2 |ReadFromMergeTree_0|
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285451190808|[140284838764568]                                                                                                                                                                                |140285630729728|ReadFromMergeTree|learn_db.mart_student_lesson      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|     60508|                    0|                  3118|         0|          0|      27940|     1052501|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_3 |ReadFromMergeTree_0|
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140288439170072|[140284838765080]                                                                                                                                                                                |140285630729728|ReadFromMergeTree|learn_db.mart_student_lesson      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|     57339|                    0|                  1419|         0|          0|      20905|      787057|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_4 |ReadFromMergeTree_0|
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140288439171608|[140284838765592]                                                                                                                                                                                |140285630729728|ReadFromMergeTree|learn_db.mart_student_lesson      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|     58947|                    0|                  1301|         0|          0|      18291|      690159|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_5 |ReadFromMergeTree_0|
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285322709016|[140284838767640]                                                                                                                                                                                |140285630729728|ReadFromMergeTree|learn_db.mart_student_lesson      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|     57844|                    0|                  1349|         0|          0|      17336|      652253|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_6 |ReadFromMergeTree_0|
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285323801112|[140284838768152]                                                                                                                                                                                |140285630729728|ReadFromMergeTree|learn_db.mart_student_lesson      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|     56982|                    0|                  2736|         0|          0|      41116|     1548230|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_7 |ReadFromMergeTree_0|
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285323801624|[140284839527448]                                                                                                                                                                                |140285630729728|ReadFromMergeTree|learn_db.mart_student_lesson      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|     58464|                    0|                  1988|         0|          0|      40990|     1543669|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_8 |ReadFromMergeTree_0|
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285323804184|[140284839528472]                                                                                                                                                                                |140285630729728|ReadFromMergeTree|learn_db.mart_student_lesson      |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|     63740|                    0|                  3758|         0|          0|      52733|     1984599|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_9 |ReadFromMergeTree_0|
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285447120408|[140283050469400]                                                                                                                                                                                |              0|                 |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|NullSource                                        |         2|                    0|                     0|         0|          0|          0|           0|NullSource_69                                        |                   |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324736024|[140283050469400]                                                                                                                                                                                |              0|                 |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|NullSource                                        |         0|                    0|                     0|         0|          0|          0|           0|NullSource_70                                        |                   |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140283050469400|[]                                                                                                                                                                                               |              0|                 |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|ParallelFormattingOutputFormat                    |       878|                68200|                     0|         9|        340|          0|           0|ParallelFormattingOutputFormat_65                    |                   |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324478488|[140285324479000,140285324479000,140285324479000,140285324479000,140285324479000,140285324479000,140285324479000,140285324479000,140285324479000,140285324479000,140285324479000,140285324479000]|140285632238336|Aggregating      |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|Resize                                            |         0|                68109|                     0|         9|        340|          9|         340|Resize_49                                            |Aggregating_4      |
19294dc11be6|2025-10-15|2025-10-15 22:35:32|2025-10-15 22:35:32.174518000|140285324735512|[140285325974680]                                                                                                                                                                                |              0|                 |                                  |         0|173dfb41-d66e-44f0-8ade-d09653bcda12|173dfb41-d66e-44f0-8ade-d09653bcda12|Resize                                            |         0|                68132|                     0|         9|        340|          9|         340|Resize_63                                            |                   |
```

```sql
SELECT SUM(elapsed_us) FROM system.processors_profile_log WHERE query_id = '173dfb41-d66e-44f0-8ade-d09653bcda12';
```

Результат
```text
SUM(elapsed_us)|
---------------+
         740906|
```

### 8. Добавляем и материализуем проекцию с другим первичным индексом
```sql
ALTER TABLE learn_db.mart_student_lesson
ADD PROJECTION educational_organization_id_pk_projection
(
    SELECT 
    	subject_name,
    	mark,
    	educational_organization_id,
    	lesson_date
    ORDER BY educational_organization_id, lesson_date
);

-- Materialize it so existing data is built in the new order
ALTER TABLE learn_db.mart_student_lesson
MATERIALIZE PROJECTION educational_organization_id_pk_projection;
```

### 9. Проверяем как эффективность второго запроса и время выполнения его шагов

```sql
EXPLAIN indexes = 1
SELECT t1.subject_name AS res_0, avg(t1.mark) AS res_1
FROM learn_db.mart_student_lesson AS t1
WHERE t1.lesson_date BETWEEN toDate32('2024-10-16') AND toDate32('2025-10-15')
GROUP BY res_0
LIMIT 1000001
```

Результат
```text
explain                                                                                                                                        |
-----------------------------------------------------------------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                                                                                      |
  Limit (preliminary LIMIT (without OFFSET))                                                                                                   |
    Aggregating                                                                                                                                |
      Filter                                                                                                                                   |
        ReadFromMergeTree (educational_organization_id_pk_projection)                                                                          |
        Indexes:                                                                                                                               |
          PrimaryKey                                                                                                                           |
            Keys:                                                                                                                              |
              educational_organization_id                                                                                                      |
              lesson_date                                                                                                                      |
            Condition: and((lesson_date in (-Inf, 20376]), and((lesson_date in [20012, +Inf)), (educational_organization_id in 1-element set)))|
            Parts: 4/4                                                                                                                         |
            Granules: 49/1223                                                                                                                  |
```

Количество прочитанных гранул сократилось почти в 17 раз. И чтение происходит не из оригинальной таблицы, а с проекции - ReadFromMergeTree (educational_organization_id_pk_projection).

```sql
SELECT * FROM system.query_log ORDER BY event_time DESC;
```

Результат
```text
query_id = 'edfdc4be-8d38-4686-9267-461fd141c0c2'
```


```sql
SELECT * FROM system.processors_profile_log WHERE query_id = 'edfdc4be-8d38-4686-9267-461fd141c0c2' order by processor_uniq_id;
```

Результат
```text
hostname    |event_date|event_time         |event_time_microseconds      |id             |parent_ids                                                                                                                                                                                       |plan_step      |plan_step_name   |plan_step_description                    |plan_group|initial_query_id                    |query_id                            |name                                              |elapsed_us|input_wait_elapsed_us|output_wait_elapsed_us|input_rows|input_bytes|output_rows|output_bytes|processor_uniq_id                                   |step_uniq_id        |
------------+----------+-------------------+-----------------------------+---------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+-----------------+-----------------------------------------+----------+------------------------------------+------------------------------------+--------------------------------------------------+----------+---------------------+----------------------+----------+-----------+-----------+------------+----------------------------------------------------+--------------------+
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285635224728|[140285634867224]                                                                                                                                                                                |140285636246016|Aggregating      |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|AggregatingTransform                              |      1054|                 7191|                     0|     40362|    1280002|          0|           0|AggregatingTransform_17                             |Aggregating_4       |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285635225368|[140285634867224]                                                                                                                                                                                |140285636246016|Aggregating      |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|AggregatingTransform                              |      4436|                13371|                     9|    243521|    7708942|          9|         340|AggregatingTransform_18                             |Aggregating_4       |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285635226008|[140285634867224]                                                                                                                                                                                |140285636246016|Aggregating      |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|AggregatingTransform                              |       883|                 6212|                     0|     40440|    1279811|          0|           0|AggregatingTransform_19                             |Aggregating_4       |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285635226648|[140285634867224]                                                                                                                                                                                |140285636246016|Aggregating      |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|AggregatingTransform                              |      1029|                 6685|                     0|     40247|    1273212|          0|           0|AggregatingTransform_20                             |Aggregating_4       |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285515397336|[140285635225368]                                                                                                                                                                                |              0|                 |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|ConvertingAggregatedToChunksTransform             |        39|                    0|                     0|         0|          0|          9|         340|ConvertingAggregatedToChunksTransform_43            |                    |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285636287000|[140285551531544]                                                                                                                                                                                |140286249829120|Expression       |(Project names + Projection)             |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|ExpressionTransform                               |         3|                17855|                     1|         9|        340|          9|         340|ExpressionTransform_23                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285636287512|[140285551531544]                                                                                                                                                                                |140286249829120|Expression       |(Project names + Projection)             |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|ExpressionTransform                               |         0|                17845|                     0|         0|          0|          0|           0|ExpressionTransform_24                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285636288024|[140285551531544]                                                                                                                                                                                |140286249829120|Expression       |(Project names + Projection)             |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|ExpressionTransform                               |         0|                17845|                     0|         0|          0|          0|           0|ExpressionTransform_25                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285548749848|[140285551531544]                                                                                                                                                                                |140286249829120|Expression       |(Project names + Projection)             |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|ExpressionTransform                               |         0|                17846|                     0|         0|          0|          0|           0|ExpressionTransform_26                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285548751384|[140285551531544]                                                                                                                                                                                |140286249829120|Expression       |(Project names + Projection)             |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|ExpressionTransform                               |         0|                17846|                     0|         0|          0|          0|           0|ExpressionTransform_27                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285548751896|[140285551531544]                                                                                                                                                                                |140286249829120|Expression       |(Project names + Projection)             |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|ExpressionTransform                               |         0|                17846|                     0|         0|          0|          0|           0|ExpressionTransform_28                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285548779544|[140285551531544]                                                                                                                                                                                |140286249829120|Expression       |(Project names + Projection)             |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|ExpressionTransform                               |         0|                17847|                     0|         0|          0|          0|           0|ExpressionTransform_29                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285549115928|[140285551531544]                                                                                                                                                                                |140286249829120|Expression       |(Project names + Projection)             |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|ExpressionTransform                               |         0|                17847|                     0|         0|          0|          0|           0|ExpressionTransform_30                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285549116952|[140285551531544]                                                                                                                                                                                |140286249829120|Expression       |(Project names + Projection)             |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|ExpressionTransform                               |         0|                17848|                     0|         0|          0|          0|           0|ExpressionTransform_31                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285551045656|[140285551531544]                                                                                                                                                                                |140286249829120|Expression       |(Project names + Projection)             |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|ExpressionTransform                               |         0|                17849|                     0|         0|          0|          0|           0|ExpressionTransform_32                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285551531032|[140285551531544]                                                                                                                                                                                |140286249829120|Expression       |(Project names + Projection)             |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|ExpressionTransform                               |         0|                17849|                     0|         0|          0|          0|           0|ExpressionTransform_33                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140291553997848|[140285551531544]                                                                                                                                                                                |140286249829120|Expression       |(Project names + Projection)             |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|ExpressionTransform                               |         0|                17849|                     0|         0|          0|          0|           0|ExpressionTransform_34                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140286250072088|[140285636736536]                                                                                                                                                                                |140286248328832|Filter           |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|FilterTransform                                   |       942|                11902|                  4880|    243635|    8930714|     243512|     9169674|FilterTransform_10                                  |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140286250072856|[140285636737304]                                                                                                                                                                                |140286248328832|Filter           |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|FilterTransform                                   |       792|                 5302|                   969|     40456|    1482575|      40440|     1522451|FilterTransform_11                                  |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140286250073624|[140285636738072]                                                                                                                                                                                |140286248328832|Filter           |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|FilterTransform                                   |       588|                 5891|                  1213|     40275|    1475484|      40247|     1514694|FilterTransform_12                                  |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140286250076696|[140285635224728]                                                                                                                                                                                |140286248328832|Filter           |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|FilterTransform                                   |       159|                 7018|                  1050|     40362|    1522174|      40362|     1280002|FilterTransform_13                                  |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285636736536|[140285635225368]                                                                                                                                                                                |140286248328832|Filter           |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|FilterTransform                                   |       416|                12876|                  4435|    243512|    9169674|     243512|     7708602|FilterTransform_14                                  |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285636737304|[140285635226008]                                                                                                                                                                                |140286248328832|Filter           |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|FilterTransform                                   |        90|                 6108|                   867|     40440|    1522451|      40440|     1279811|FilterTransform_15                                  |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285636738072|[140285635226648]                                                                                                                                                                                |140286248328832|Filter           |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|FilterTransform                                   |       174|                 6496|                  1025|     40247|    1514694|      40247|     1273212|FilterTransform_16                                  |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140286249579544|[140286250070552]                                                                                                                                                                                |140286248328832|Filter           |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|FilterTransform                                   |      1326|                 4599|                  2293|     49152|    1853667|      40384|     1482616|FilterTransform_5                                   |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140286249580312|[140286250072088]                                                                                                                                                                                |140286248328832|Filter           |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|FilterTransform                                   |      1701|                10141|                  5858|    253952|    9562094|     243635|     8930714|FilterTransform_6                                   |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140286249581080|[140286250072856]                                                                                                                                                                                |140286248328832|Filter           |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|FilterTransform                                   |      1592|                 3682|                  1775|     49152|    1850046|      40456|     1482575|FilterTransform_7                                   |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140286249581848|[140286250073624]                                                                                                                                                                                |140286248328832|Filter           |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|FilterTransform                                   |      1175|                 4689|                  1818|     49152|    1850082|      40275|     1475484|FilterTransform_8                                   |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140286250070552|[140286250076696]                                                                                                                                                                                |140286248328832|Filter           |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|FilterTransform                                   |      1052|                 5947|                  1223|     40384|    1482616|      40362|     1522174|FilterTransform_9                                   |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285636285464|[140285636287000,140285636287512,140285636288024,140285548749848,140285548751384,140285548751896,140285548779544,140285549115928,140285549116952,140285551045656,140285551531032,140291553997848]|140286248330112|Limit            |preliminary LIMIT (without OFFSET)       |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|Limit                                             |         0|                17862|                     8|         9|        340|          9|         340|Limit_22                                            |Limit_6             |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285635227928|[140285481411544]                                                                                                                                                                                |              0|                 |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|LimitsCheckingTransform                           |         1|                17875|                    76|         9|        340|          9|         340|LimitsCheckingTransform_36                          |                    |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285481411544|[140285549650200]                                                                                                                                                                                |              0|                 |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|MaterializingTransform                            |         0|                17879|                    74|         9|        340|          9|         340|MaterializingTransform_40                           |                    |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285636286488|[140286249579544]                                                                                                                                                                                |140285635451904|ReadFromMergeTree|educational_organization_id_pk_projection|         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|      4473|                    0|                  3642|         0|          0|      49152|     1853667|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_1|ReadFromMergeTree_11|
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285548748824|[140286249580312]                                                                                                                                                                                |140285635451904|ReadFromMergeTree|educational_organization_id_pk_projection|         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|      9962|                    0|                  7635|         0|          0|     253952|     9562094|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_2|ReadFromMergeTree_11|
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285548749336|[140286249581080]                                                                                                                                                                                |140285635451904|ReadFromMergeTree|educational_organization_id_pk_projection|         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|      3534|                    0|                  3393|         0|          0|      49152|     1850046|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_3|ReadFromMergeTree_11|
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285548750360|[140286249581848]                                                                                                                                                                                |140285635451904|ReadFromMergeTree|educational_organization_id_pk_projection|         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|      4526|                    0|                  3020|         0|          0|      49152|     1850082|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_4|ReadFromMergeTree_11|
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285634867736|[140285549650200]                                                                                                                                                                                |              0|                 |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|NullSource                                        |         1|                    0|                     0|         0|          0|          0|           0|NullSource_41                                       |                    |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285551532056|[140285549650200]                                                                                                                                                                                |              0|                 |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|NullSource                                        |         0|                    0|                     0|         0|          0|          0|           0|NullSource_42                                       |                    |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285549650200|[]                                                                                                                                                                                               |              0|                 |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|ParallelFormattingOutputFormat                    |       412|                17889|                     0|         9|        340|          0|           0|ParallelFormattingOutputFormat_37                   |                    |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285634867224|[140285636285464,140285636285464,140285636285464,140285636285464,140285636285464,140285636285464,140285636285464,140285636285464,140285636285464,140285636285464,140285636285464,140285636285464]|140285636246016|Aggregating      |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|Resize                                            |         0|                17850|                     0|         9|        340|          9|         340|Resize_21                                           |Aggregating_4       |
19294dc11be6|2025-10-15|2025-10-15 22:44:02|2025-10-15 22:44:02.571297000|140285551531544|[140285635227928]                                                                                                                                                                                |              0|                 |                                         |         0|edfdc4be-8d38-4686-9267-461fd141c0c2|edfdc4be-8d38-4686-9267-461fd141c0c2|Resize                                            |         0|                17871|                     0|         9|        340|          9|         340|Resize_35                                           |                    |
```

```sql
SELECT SUM(elapsed_us) FROM system.processors_profile_log WHERE query_id = 'edfdc4be-8d38-4686-9267-461fd141c0c2';
```

Результат
```text
SUM(elapsed_us)|
---------------+
          40360|
```

### 10. Получим информацию о частях таблицы 

```sql
SELECT * FROM system.parts WHERE table = 'mart_student_lesson';
```

Результат
```text
partition|name        |uuid                                |part_type|active|marks|rows   |bytes_on_disk|data_compressed_bytes|data_uncompressed_bytes|primary_key_size|marks_bytes|secondary_indices_compressed_bytes|secondary_indices_uncompressed_bytes|secondary_indices_marks_bytes|modification_time  |remove_time        |refcount|min_date  |max_date  |min_time           |max_time           |partition_id|min_block_number|max_block_number|level|data_version|primary_key_bytes_in_memory|primary_key_bytes_in_memory_allocated|index_granularity_bytes_in_memory|index_granularity_bytes_in_memory_allocated|is_frozen|database|table              |engine   |disk_name|path                                                                            |hash_of_all_files               |hash_of_uncompressed_files      |uncompressed_hash_of_compressed_files|delete_ttl_info_min|delete_ttl_info_max|move_ttl_info.expression|move_ttl_info.min|move_ttl_info.max|default_compression_codec|recompression_ttl_info.expression|recompression_ttl_info.min|recompression_ttl_info.max|group_by_ttl_info.expression|group_by_ttl_info.min|group_by_ttl_info.max|rows_where_ttl_info.expression|rows_where_ttl_info.min|rows_where_ttl_info.max|projections                                  |visible|creation_tid                                      |removal_tid_lock|removal_tid                                       |creation_csn|removal_csn|has_lightweight_delete|last_removal_attempt_time|removal_state                           |
---------+------------+------------------------------------+---------+------+-----+-------+-------------+---------------------+-----------------------+----------------+-----------+----------------------------------+------------------------------------+-----------------------------+-------------------+-------------------+--------+----------+----------+-------------------+-------------------+------------+----------------+----------------+-----+------------+---------------------------+-------------------------------------+---------------------------------+-------------------------------------------+---------+--------+-------------------+---------+---------+--------------------------------------------------------------------------------+--------------------------------+--------------------------------+-------------------------------------+-------------------+-------------------+------------------------+-----------------+-----------------+-------------------------+---------------------------------+--------------------------+--------------------------+----------------------------+---------------------+---------------------+------------------------------+-----------------------+-----------------------+---------------------------------------------+-------+--------------------------------------------------+----------------+--------------------------------------------------+------------+-----------+----------------------+-------------------------+----------------------------------------+
tuple()  |all_1_6_1_10|00000000-0000-0000-0000-000000000000|Wide     |     1|  817|6671718|    219355646|            191556449|              545101748|            1737|      30420|                                 0|                                   0|                            0|2025-10-15 22:38:14|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               1|               6|    1|          10|                       4902|                                 5158|                             6536|                                       6536|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/4c6/4c6fb050-7352-4fd1-93ac-a3c1b348bbb2/all_1_6_1_10/|864b4b3197e2658d4e4b49d407f55be2|04842761452ee847d40b0d3eccd21649|91573e8e16d39c6eb6ddd8c75452bd5b     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |['educational_organization_id_pk_projection']|      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_7_7_0_10|00000000-0000-0000-0000-000000000000|Wide     |     1|  137|1111953|     37608049|             32937909|               90849526|             469|       6256|                                 0|                                   0|                            0|2025-10-15 22:38:11|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               7|               7|    0|          10|                        548|                                  676|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/4c6/4c6fb050-7352-4fd1-93ac-a3c1b348bbb2/all_7_7_0_10/|fd2fd39a2c51455f60f80e0d3fb2ed46|431f8d5eb3b26baaef5ec565540b4007|4e2e23bd97771e8eb3947a440cd4ca55     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |['educational_organization_id_pk_projection']|      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_8_8_0_10|00000000-0000-0000-0000-000000000000|Wide     |     1|  137|1111953|     37611318|             32939528|               90842679|             462|       6252|                                 0|                                   0|                            0|2025-10-15 22:38:11|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               8|               8|    0|          10|                        548|                                  676|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/4c6/4c6fb050-7352-4fd1-93ac-a3c1b348bbb2/all_8_8_0_10/|2e9e63be5c3e9766296b387fa4f5151a|8385017406faca767e2f62b4f9f1a895|623a68d06473ef55002a787a5848513b     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |['educational_organization_id_pk_projection']|      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
tuple()  |all_9_9_0_10|00000000-0000-0000-0000-000000000000|Wide     |     1|  136|1104376|     37359199|             32717017|               90224896|             464|       6270|                                 0|                                   0|                            0|2025-10-15 22:38:11|1970-01-01 03:00:00|       1|1970-01-01|1970-01-01|1970-01-01 03:00:00|1970-01-01 03:00:00|all         |               9|               9|    0|          10|                        544|                                  672|                               25|                                         25|        0|learn_db|mart_student_lesson|MergeTree|default  |/var/lib/clickhouse/store/4c6/4c6fb050-7352-4fd1-93ac-a3c1b348bbb2/all_9_9_0_10/|1887e8b5042f0285bb883f1c69d4452c|e5e2e68b39b3c685f40646a7e730660b|2e95434f8b0b6551962c5f5d319e119e     |1970-01-01 03:00:00|1970-01-01 03:00:00|[]                      |[]               |[]               |LZ4                      |[]                               |[]                        |[]                        |[]                          |[]                   |[]                   |[]                            |[]                     |[]                     |['educational_organization_id_pk_projection']|      1|{1:1, 2:1, 3:00000000-0000-0000-0000-000000000000}|               0|{1:0, 2:0, 3:00000000-0000-0000-0000-000000000000}|           0|          0|                     0|      1970-01-01 03:00:00|Cleanup thread hasn't seen this part yet|
```

Посмотрим внутрь каталога с данными.

```bash
cd /var/lib/clickhouse/store/4c6/4c6fb050-7352-4fd1-93ac-a3c1b348bbb2/all_1_6_1_10/
```

Результат
```text
root@19294dc11be6:/var/lib/clickhouse/store/4c6/4c6fb050-7352-4fd1-93ac-a3c1b348bbb2/all_1_6_1_10# ls
checksums.txt                                   lesson_date.bin            load_date.cmrk2       person_id.cmrk2           subject_name.bin
class_id.bin                                    lesson_date.cmrk2          mark.bin              person_id_int.bin         subject_name.cmrk2
class_id.cmrk2                                  lesson_month_digits.bin    mark.cmrk2            person_id_int.cmrk2       t.bin
columns.txt                                     lesson_month_digits.cmrk2  mark.null.bin         primary.cidx              t.cmrk2
count.txt                                       lesson_month_text.bin      mark.null.cmrk2       serialization.json        teacher_id.bin
default_compression_codec.txt                   lesson_month_text.cmrk2    metadata_version.txt  student_profile_id.bin    teacher_id.cmrk2
educational_organization_id.bin                 lesson_year.bin            parallel_id.bin       student_profile_id.cmrk2
educational_organization_id.cmrk2               lesson_year.cmrk2          parallel_id.cmrk2     subject_id.bin
educational_organization_id_pk_projection.proj  load_date.bin              person_id.bin         subject_id.cmrk2
```

Видим что есть дополнительный каталог educational_organization_id_pk_projection.proj

```bash
cd educational_organization_id_pk_projection.proj
```

Результат
```text
root@19294dc11be6:/var/lib/clickhouse/store/4c6/4c6fb050-7352-4fd1-93ac-a3c1b348bbb2/all_1_6_1_10/educational_organization_id_pk_projection.proj# ls
checksums.txt                      primary.cidx
columns.txt                        serialization.json
count.txt                          subject_name.bin
default_compression_codec.txt      subject_name.cmrk2
educational_organization_id.bin    tuple%28educational_organization_id%2C%20lesson_date%29%2E1.bin
educational_organization_id.cmrk2  tuple%28educational_organization_id%2C%20lesson_date%29%2E1.cmrk2
lesson_date.bin                    tuple%28educational_organization_id%2C%20lesson_date%29%2E1.sparse.idx.bin
lesson_date.cmrk2                  tuple%28educational_organization_id%2C%20lesson_date%29%2E1.sparse.idx.cmrk2
mark.bin                           tuple%28educational_organization_id%2C%20lesson_date%29%2E2.bin
mark.cmrk2                         tuple%28educational_organization_id%2C%20lesson_date%29%2E2.cmrk2
mark.null.bin                      tuple%28educational_organization_id%2C%20lesson_date%29%2E2.sparse.idx.bin
mark.null.cmrk2                    tuple%28educational_organization_id%2C%20lesson_date%29%2E2.sparse.idx.cmrk2
metadata_version.txt
```

В нем хранятся данные этой таблицы, но уже в другом порядке (сортировкой и первичным ключом). Но при этом все это носит одинаковое наименование.
И теперь когда выполняется запрос к таблице то CH выбирает откуда запрашивать данные: из оригинальной таблицы или из проекции. И если он решит, что эффективнее получить данные из проекции, то он так и сделает.

### 11. Вернемся к первому запросу

```sql
SELECT t1.subject_name AS res_0, avg(t1.mark) AS res_1
FROM learn_db.mart_student_lesson AS t1
WHERE t1.lesson_date BETWEEN toDate32('2025-10-09') AND toDate32('2025-10-15')
GROUP BY res_0
LIMIT 1000001
```

Результат
```text
Query id: 8f76a85d-5547-418a-aa17-4fd594b2fb70

   ┌─res_0───────────────┬──────────────res_1─┐
1. │ Физика              │  3.989075493233328 │
2. │ Биология            │  3.623186069009997 │
3. │ Химия               │  3.836993107869851 │
4. │ Русский язык        │  4.255012936610608 │
5. │ География           │ 3.7645456002564512 │
6. │ Литература          │  4.159743884419636 │
7. │ Математика          │  4.351738570508693 │
8. │ Информатика         │  3.601061639134215 │
9. │ Физическая культура │  3.522869523350987 │
   └─────────────────────┴────────────────────┘

9 rows in set. Elapsed: 0.019 sec. Processed 194.18 thousand rows, 6.93 MB (10.39 million rows/s., 370.52 MB/s.)
Peak memory usage: 139.40 KiB.
```

### 12. Получим план выполнения запроса

```sql
EXPLAIN indexes = 1
SELECT t1.subject_name AS res_0, avg(t1.mark) AS res_1
FROM learn_db.mart_student_lesson AS t1
WHERE t1.lesson_date BETWEEN toDate32('2025-10-09') AND toDate32('2025-10-15')
GROUP BY res_0
LIMIT 1000001
```

Результат
```text
explain                                                                                     |
--------------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                                   |
  Limit (preliminary LIMIT (without OFFSET))                                                |
    Aggregating                                                                             |
      Expression (Before GROUP BY)                                                          |
        Expression                                                                          |
          ReadFromMergeTree (learn_db.mart_student_lesson)                                  |
          Indexes:                                                                          |
            PrimaryKey                                                                      |
              Keys:                                                                         |
                lesson_date                                                                 |
              Condition: and((lesson_date in (-Inf, 20376]), (lesson_date in [20370, +Inf)))|
              Parts: 4/4                                                                    |
              Granules: 26/1223                                                             |                                                          |                                                                                                              |
```
### 13. Создадим новую проекцию

```sql
ALTER TABLE learn_db.mart_student_lesson
ADD PROJECTION avg_mark_subject_name_group_projection
(
    SELECT 
    	subject_name,
    	lesson_date,
    	avg(mark)
    GROUP BY subject_name,
    	lesson_date
);

-- Materialize it so existing data is built in the new order
ALTER TABLE learn_db.mart_student_lesson
MATERIALIZE PROJECTION avg_mark_subject_name_group_projection;
```

### 14. Получим план выполнения запроса

```sql
EXPLAIN indexes = 1
SELECT t1.subject_name AS res_0, avg(t1.mark) AS res_1
FROM learn_db.mart_student_lesson AS t1
WHERE t1.lesson_date BETWEEN toDate32('2025-10-09') AND toDate32('2025-10-15')
GROUP BY res_0
LIMIT 1000001
```

Результат
```text
explain                                                                                   |
------------------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                                 |
  Limit (preliminary LIMIT (without OFFSET))                                              |
    Aggregating                                                                           |
      Filter                                                                              |
        ReadFromMergeTree (avg_mark_subject_name_group_projection)                        |
        Indexes:                                                                          |
          PrimaryKey                                                                      |
            Keys:                                                                         |
              lesson_date                                                                 |
            Condition: and((lesson_date in (-Inf, 20376]), (lesson_date in [20370, +Inf)))|
            Parts: 4/4                                                                    |
            Granules: 4/1223                                                              |                                                                                                                 |
```

Запрос получил данные из сгруппированной проекции и было прочитано всего 4 гранулы.

### 15. Проверим скорость выполнения запроса.

```sql
SELECT * FROM system.query_log ORDER BY event_time DESC;
```

Результат
```text
query_id = 'dbafec13-4341-4f7d-99b9-3802b1d01e71'
```

```sql
SELECT * FROM system.processors_profile_log WHERE query_id = 'dbafec13-4341-4f7d-99b9-3802b1d01e71' order by processor_uniq_id;
```

Результат
```text
hostname    |event_date|event_time         |event_time_microseconds      |id             |parent_ids                                                                                                                                                                                       |plan_step      |plan_step_name   |plan_step_description                 |plan_group|initial_query_id                    |query_id                            |name                                              |elapsed_us|input_wait_elapsed_us|output_wait_elapsed_us|input_rows|input_bytes|output_rows|output_bytes|processor_uniq_id                                   |step_uniq_id        |
------------+----------+-------------------+-----------------------------+---------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+-----------------+--------------------------------------+----------+------------------------------------+------------------------------------+--------------------------------------------------+----------+---------------------+----------------------+----------+-----------+-----------+------------+----------------------------------------------------+--------------------+
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140286243360280|[140285467037720]                                                                                                                                                                                |140285574838528|Aggregating      |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|AggregatingTransform                              |        76|                  689|                     0|        63|       2317|          0|           0|AggregatingTransform_14                             |Aggregating_4       |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285469526808|[140285467037720]                                                                                                                                                                                |140285574838528|Aggregating      |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|AggregatingTransform                              |       121|                  630|                     0|        63|       2317|          0|           0|AggregatingTransform_15                             |Aggregating_4       |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285469527448|[140285467037720]                                                                                                                                                                                |140285574838528|Aggregating      |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|AggregatingTransform                              |       190|                  678|                     6|        72|       2657|          9|         340|AggregatingTransform_16                             |Aggregating_4       |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285261140632|[140285467037720]                                                                                                                                                                                |140285574838528|Aggregating      |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|AggregatingTransform                              |       158|                  616|                     0|        63|       2317|          0|           0|AggregatingTransform_17                             |Aggregating_4       |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140283726921304|[140285469527448]                                                                                                                                                                                |              0|                 |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|ConvertingAggregatedToChunksTransform             |        23|                    0|                     0|         0|          0|          9|         340|ConvertingAggregatedToChunksTransform_40            |                    |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285470905880|[140284947763224]                                                                                                                                                                                |140285277740544|Expression       |(Project names + Projection)          |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|ExpressionTransform                               |         2|                  893|                     1|         9|        340|          9|         340|ExpressionTransform_20                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285657677848|[140284947763224]                                                                                                                                                                                |140285277740544|Expression       |(Project names + Projection)          |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|ExpressionTransform                               |         0|                  886|                     0|         0|          0|          0|           0|ExpressionTransform_21                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285659636248|[140284947763224]                                                                                                                                                                                |140285277740544|Expression       |(Project names + Projection)          |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|ExpressionTransform                               |         0|                  886|                     0|         0|          0|          0|           0|ExpressionTransform_22                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285686003224|[140284947763224]                                                                                                                                                                                |140285277740544|Expression       |(Project names + Projection)          |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|ExpressionTransform                               |         0|                  886|                     0|         0|          0|          0|           0|ExpressionTransform_23                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285686689816|[140284947763224]                                                                                                                                                                                |140285277740544|Expression       |(Project names + Projection)          |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|ExpressionTransform                               |         0|                  886|                     0|         0|          0|          0|           0|ExpressionTransform_24                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285464643608|[140284947763224]                                                                                                                                                                                |140285277740544|Expression       |(Project names + Projection)          |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|ExpressionTransform                               |         0|                  887|                     0|         0|          0|          0|           0|ExpressionTransform_25                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285276861976|[140284947763224]                                                                                                                                                                                |140285277740544|Expression       |(Project names + Projection)          |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|ExpressionTransform                               |         0|                  887|                     0|         0|          0|          0|           0|ExpressionTransform_26                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285263238168|[140284947763224]                                                                                                                                                                                |140285277740544|Expression       |(Project names + Projection)          |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|ExpressionTransform                               |         0|                  887|                     0|         0|          0|          0|           0|ExpressionTransform_27                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285264806424|[140284947763224]                                                                                                                                                                                |140285277740544|Expression       |(Project names + Projection)          |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|ExpressionTransform                               |         0|                  887|                     0|         0|          0|          0|           0|ExpressionTransform_28                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285264806936|[140284947763224]                                                                                                                                                                                |140285277740544|Expression       |(Project names + Projection)          |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|ExpressionTransform                               |         0|                  887|                     0|         0|          0|          0|           0|ExpressionTransform_29                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285267405848|[140284947763224]                                                                                                                                                                                |140285277740544|Expression       |(Project names + Projection)          |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|ExpressionTransform                               |         0|                  887|                     0|         0|          0|          0|           0|ExpressionTransform_30                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285267406360|[140284947763224]                                                                                                                                                                                |140285277740544|Expression       |(Project names + Projection)          |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|ExpressionTransform                               |         0|                  888|                     0|         0|          0|          0|           0|ExpressionTransform_31                              |Expression_9        |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285574841624|[140286243360280]                                                                                                                                                                                |140287468649728|Filter           |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|FilterTransform                                   |         9|                  676|                    67|        63|       2632|         63|        2317|FilterTransform_10                                  |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285574842392|[140285469526808]                                                                                                                                                                                |140287468649728|Filter           |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|FilterTransform                                   |        11|                  614|                   115|        63|       2632|         63|        2317|FilterTransform_11                                  |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140286242300696|[140285469527448]                                                                                                                                                                                |140287468649728|Filter           |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|FilterTransform                                   |        11|                  636|                   137|        63|       2632|         63|        2317|FilterTransform_12                                  |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140286242303000|[140285261140632]                                                                                                                                                                                |140287468649728|Filter           |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|FilterTransform                                   |         9|                  604|                    52|        63|       2632|         63|        2317|FilterTransform_13                                  |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140286242307608|[140285574841624]                                                                                                                                                                                |140287468649728|Filter           |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|FilterTransform                                   |        22|                  650|                    79|      3294|     213378|         63|        2632|FilterTransform_6                                   |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140286242309912|[140285574842392]                                                                                                                                                                                |140287468649728|Filter           |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|FilterTransform                                   |        23|                  587|                   130|      3294|     213378|         63|        2632|FilterTransform_7                                   |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285574839320|[140286242300696]                                                                                                                                                                                |140287468649728|Filter           |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|FilterTransform                                   |        22|                  610|                   151|      3294|     213378|         63|        2632|FilterTransform_8                                   |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285574840088|[140286242303000]                                                                                                                                                                                |140287468649728|Filter           |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|FilterTransform                                   |        18|                  581|                    63|      3294|     213378|         63|        2632|FilterTransform_9                                   |Filter_12           |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285467041304|[140285470905880,140285657677848,140285659636248,140285686003224,140285686689816,140285464643608,140285276861976,140285263238168,140285264806424,140285264806936,140285267405848,140285267406360]|140287468638208|Limit            |preliminary LIMIT (without OFFSET)    |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|Limit                                             |         0|                  897|                     6|         9|        340|          9|         340|Limit_19                                            |Limit_6             |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285261141272|[140285464275416]                                                                                                                                                                                |              0|                 |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|LimitsCheckingTransform                           |         0|                  908|                    26|         9|        340|          9|         340|LimitsCheckingTransform_33                          |                    |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285464275416|[140284943105304]                                                                                                                                                                                |              0|                 |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|MaterializingTransform                            |         0|                  910|                    24|         9|        340|          9|         340|MaterializingTransform_37                           |                    |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140287468972056|[140286242307608]                                                                                                                                                                                |140286241082368|ReadFromMergeTree|avg_mark_subject_name_group_projection|         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|       557|                    0|                   105|         0|          0|       3294|      213378|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_2|ReadFromMergeTree_11|
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285467038744|[140286242309912]                                                                                                                                                                                |140286241082368|ReadFromMergeTree|avg_mark_subject_name_group_projection|         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|       494|                    0|                   156|         0|          0|       3294|      213378|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_3|ReadFromMergeTree_11|
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285467039256|[140285574839320]                                                                                                                                                                                |140286241082368|ReadFromMergeTree|avg_mark_subject_name_group_projection|         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|       484|                    0|                   176|         0|          0|       3294|      213378|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_4|ReadFromMergeTree_11|
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285467039768|[140285574840088]                                                                                                                                                                                |140286241082368|ReadFromMergeTree|avg_mark_subject_name_group_projection|         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|       455|                    0|                    87|         0|          0|       3294|      213378|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_5|ReadFromMergeTree_11|
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285467040792|[140284943105304]                                                                                                                                                                                |              0|                 |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|NullSource                                        |         0|                    0|                     0|         0|          0|          0|           0|NullSource_38                                       |                    |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285467038232|[140284943105304]                                                                                                                                                                                |              0|                 |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|NullSource                                        |         0|                    0|                     0|         0|          0|          0|           0|NullSource_39                                       |                    |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140284943105304|[]                                                                                                                                                                                               |              0|                 |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|ParallelFormattingOutputFormat                    |       296|                  917|                     0|         9|        340|          0|           0|ParallelFormattingOutputFormat_34                   |                    |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140285467037720|[140285467041304,140285467041304,140285467041304,140285467041304,140285467041304,140285467041304,140285467041304,140285467041304,140285467041304,140285467041304,140285467041304,140285467041304]|140285574838528|Aggregating      |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|Resize                                            |         0|                  889|                     0|         9|        340|          9|         340|Resize_18                                           |Aggregating_4       |
19294dc11be6|2025-10-15|2025-10-15 23:52:31|2025-10-15 23:52:31.241660000|140284947763224|[140285261141272]                                                                                                                                                                                |              0|                 |                                      |         0|dbafec13-4341-4f7d-99b9-3802b1d01e71|dbafec13-4341-4f7d-99b9-3802b1d01e71|Resize                                            |         0|                  905|                     0|         9|        340|          9|         340|Resize_32                                           |                    |
```

```sql
SELECT SUM(elapsed_us) FROM system.processors_profile_log WHERE query_id = 'dbafec13-4341-4f7d-99b9-3802b1d01e71';
```

Результат
```text
SUM(elapsed_us)|
---------------+
           2981|
```

Скорость выполнения увеличилась в 13 раз.