# Редактирование данных в Clickhouse

### 1. Пересоздаем таблицу learn_db.mart_student_lesson
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
FROM numbers(100000000);
```

### 2. Смотрим, в какое количество частей данных распределены данные таблицы learn_db.mart_student_lesson

```sql
SELECT * FROM system.parts WHERE table = 'mart_student_lesson';
```

### 3. Смотрим, в каком количестве частей данных попадают строки с предметом "Математика"
```
SELECT 
	DISTINCT _part
FROM 
	learn_db.mart_student_lesson
WHERE 
	subject_name = 'Математика';
```

Результат
```text
_part      |
-----------+
0_3_110_2  |
2_226_334_2|
1_116_221_2|
0_225_333_2|
3_109_213_2|
1_2_111_2  |
1_227_335_2|
2_338_359_1|
2_115_222_2|
0_113_224_2|
2_1_112_2  |
3_4_106_2  |
0_340_360_1|
3_219_321_2|
1_339_357_1|
3_325_345_1|
3_350_350_0|
3_355_355_0|
3_358_358_0|
```
       
### 4. Запускаем изменение названия предметов уроков с "Математика" на "Математическая наука"
```sql
ALTER TABLE learn_db.mart_student_lesson
UPDATE subject_name = 'Математическая наука' WHERE subject_name = 'Математика';
```

### 5. Выполняем запрос на получение названий предметов уроков
```sql
SELECT subject_name, count(*) as cnt FROM learn_db.mart_student_lesson GROUP BY subject_name;
```

Результат
```text
subject_name        |cnt     |
--------------------+--------+
Физика              | 6666872|
Биология            | 6672889|
Химия               | 6667011|
Русский язык        | 6662697|
География           | 6668558|
Литература          | 6664351|
Информатика         |46661462|
Математическая наука| 6667516|
Физическая культура | 6668644|
```

# Легковесное редактирование данных в Clickhouse

### 5. Запускаем контейнер с Clickhouse версии 25.5.3.75
```
docker run --name clickhouse-course2 -e CLICKHOUSE_DB=learn_db -e CLICKHOUSE_USER=username -e CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT=1 -e CLICKHOUSE_PASSWORD=password -p 8124:8123 -p 9001:9000/tcp -d clickhouse:25.5.3.75
```

### 7. Создаем таблицу learn_db.mart_student_lesson
```
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
FROM numbers(100000000);
```

### 8. Включение легковесного редактирования на уровне сессии
```
SET allow_experimental_lightweight_update = 1;
```

### 9. Вставляем данные
```sql
INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'successed', 100, 1),
('2025-06-01', 2, 'successed', 50, 2);

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'canceled', 110, 1),
('2025-06-01', 2, 'canceled', 60, 2);
```

### 10. Запускаем редактирование
```
ALTER TABLE learn_db.mart_student_lesson
UPDATE subject_name = 'Математическая наука' WHERE subject_name = 'Математика'
SETTINGS allow_experimental_lightweight_update = 1;
```

### 11. Выполняем запрос на получение названий предметов уроков
```
SELECT subject_name, count(*) as cnt FROM learn_db.mart_student_lesson GROUP BY subject_name;
```

Результат
```text 
sale_dt   |product_id|status   |amount|pcs|_part    |
----------+----------+---------+------+---+---------+
2025-06-01|         1|successed|210.00|  2|all_1_2_2|
2025-06-01|         2|successed|110.00|  4|all_1_2_2|
```
