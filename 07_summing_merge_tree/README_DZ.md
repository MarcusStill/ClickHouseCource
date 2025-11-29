## Задание 1.

Напишите скрипт создания таблицы page_views с движком SummingMergeTree с 4 колонками, содержащими следующие данные:
- дата;
- url;
- user_id;
- количество просмотров.
Ключ сортировки: дата и url.

Суммироваться должны только значения из колонки с количеством просмотров.

## Задание 2. 

Напишите скрипт вставки в таблицу следующих данных:

```sql
INSERT INTO page_views VALUES 
  ('2023-02-01', '/home', 101, 1),
  ('2023-02-01', '/home', 102, 1), 
  ('2023-02-01', '/about', 102, 1);
```

## Задание 3. 

Напишите скрипт получения информации по просмотрам страницы (сгруппированных по дате и url), который должен вывести 3 колонки:
- дата;
- url;
- количество просмотров.

### 1. Cкрипт создания таблицы

```sql
DROP TABLE IF EXISTS learn_db.page_views;
CREATE TABLE IF NOT EXISTS learn_db.page_views
(
	date Date,
	url String,
	user_id UInt32,
	views UInt32
) ENGINE = SummingMergeTree(views)
ORDER BY (date, url);
```

### 2. Cкрипт вставки в таблицу

```sql
INSERT INTO learn_db.page_views VALUES 
    ('2023-02-01', '/home', 101, 1),
    ('2023-02-01', '/home', 102, 1), 
    ('2023-02-01', '/about', 102, 1);
```

### 3. Скрипт получения информации

```sql
SELECT 
    date,
    url,
    sum(views) as count_views
FROM learn_db.page_views
GROUP BY date, url;
```

Результат:
```text
Результат

date      |url   |count_views|
----------+------+-----------+
2023-02-01|/about|          1|
2023-02-01|/home |          2|
```
