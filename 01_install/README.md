# Разворачиваем учебный проект

## 1. Установка однонодного Clickhouse в Docker версии 25.4
```console
docker run --name clickhouse-course -e CLICKHOUSE_DB=learn_db -e CLICKHOUSE_USER=username -e CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT=1 -e CLICKHOUSE_PASSWORD=password -p 8123:8123 -p 9000:9000/tcp -d -v clickhouse-logs:/var/log/clickhouse-server -v clickhouse-data:/var/lib/clickhouse clickhouse:25.4
```

#### HTTP Interface: 
* адрес дашборда Clickhouse: http://localhost:8123/dashboard
* адрес web интерфейса для выполнения запросов: http://localhost:8123/play

#### Параметры подключения к БД
* host: localhost
* port: 8123
* DB: learn_db
* user: username
* password: password

В свойствах драйвера устанавливаем у параметра socket_timeout значение 300000.

## 2. Создаем витрину данных в Clickhouse

```sql
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
	PRIMARY KEY(tuple())
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
FROM numbers(100000000);
```

Для генерации дополнительных данных, выполняем следующий запрос:
```sql
INSERT INTO learn_db.mart_student_lesson
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
	load_date,
	t,
	teacher_id,
	subject_id,
	subject_name,
	mark
)
SELECT
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
FROM numbers(100000000);
```


## 3. Разворачиваем Datalens в Docker
[Datalens](https://github.com/datalens-tech/datalens)

* Создаем новую папку для Datalens.
* Переход в созданный каталог.
* Запускаем контейнеры

```console
git clone https://github.com/datalens-tech/datalens
cd datalens
docker compose up -d
```

#### Параметры подключения к UI
* Host: http://localhost:8080/
* User: admin
* Password: admin


## 4. Организуем сетевую связность между Datalens и Clickhouse в Docker
```console
docker network connect datalens_default clickhouse-course
```


## 5. Создаем Workbook и добавляем подключение к БД

#### Параметры подключения:
* host: clickhouse-course
* port: 8123
* user: username
* password: password
* TSL: Off


## 6. Создаем чарты и дашборд
![2025-06-11_17-40](https://github.com/user-attachments/assets/3d47d638-9d90-4310-8e45-ff0dfa31a31c)


## 7. Определяем запросы, которые выполнялись во время открытия дашборда

#### 7.1 В DBeaver открываем редактор SQL и выполняем запрос:

```sql
SELECT *
FROM system.query_log
WHERE NOT query LIKE '%query_log%'
	AND 'QueryFinish' = type
	AND http_user_agent = 'DataLens'
ORDER BY event_time DESC
LIMIT 20
```

#### 7.2 Открываем Docker Desktop и заходим внутрь контейнера Clickhouse

Открываем вкладку "Exec" выполняем команду "bash". Сохраняем все запросы, выполняемые при открытии дашборда, в файл queries.tsv

```console
clickhouse-client --query="
SELECT query FROM system.query_log
WHERE NOT query LIKE '%query_log%' 
AND http_user_agent = 'DataLens' 
AND 'QueryFinish' = type 
ORDER BY event_time desc
LIMIT 9
" > queries.tsv
```

#### 7.3 С помощью утилиты clickhouse-client выполним запросы и посмотрим на затраченное время

Преобразуем `\n` в реальные newline:
```echo -e "$q" превращает \n в настоящие переводы строк.```

После этого clickhouse-client получит корректный запрос.

```console
time (while IFS= read -r q; do clickhouse-client --query="$(echo -e "$q")" > /dev/null; done < queries.tsv)
```

Результат
```commandline
real    0m12.116s
user    0m0.261s
sys     0m0.182s
```
