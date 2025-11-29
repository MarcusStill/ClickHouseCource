## Задание

Нужно создать 2 таблицы для хранения пользователей системы. В каждой таблице должны быть поля:
- user_id - уникальный идентификатор пользователя;
- active - поле, в котором хранится 0 или 1 (0 - пользователь не активен, 1 - активен);
- rank - поле с целом числом в диапазоне от 0 до 5. Означает внутреннюю градацию пользователей. Чем выше ранк у пользователя, тем больше привилегий он имеет.

1. Создайте таблицу user_collapsing с движком CollapsingMergeTree. 

Напишите запрос добавления в таблицу пользователя, у которого rank = 0, а в поле active cтоит 0.
Напишите запрос изменения значения в поле active с 0 на 1.
Напишите запрос изменения значения в поле rank с 0 до 5.
Напишите запрос, который вернет строку с актуальными данными пользователя.

2. Создайте таблицу user_replacing с движком ReplacingMergeTree.

Напишите запрос добавления в таблицу пользователя, у которого rank = 0, а в поле active cтоит 0.
Напишите запрос изменения значения в поле active с 0 на 1.
Напишите запрос изменения значения в поле rank с 0 до 5.
Напишите запрос, который вернет строку с актуальными данными пользователя.

### Задание 1

#### 1.1. Создание таблицы user_collapsing с движком CollapsingMergeTree

```sql
DROP TABLE IF EXISTS user_collapsing_merge_tree;

CREATE TABLE user_collapsing_merge_tree
(
    user_id UInt32,
    active UInt8,
    rank UInt8,
    sign Int8
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY (user_id);
```

#### 1.2. Добавление пользователя с rank = 0 и active = 0

```sql
INSERT INTO user_collapsing_merge_tree
(user_id, active, rank, sign)
VALUES
(1, 0, 0, 1);
```

#### 1.3. Изменение значения active с 0 на 1

```sql
INSERT INTO user_collapsing_merge_tree
(user_id, active, rank, sign)
VALUES
(1, 0, 0, -1),  
(1, 1, 0, 1);
```

#### 1.4. Получение актуальных данных пользователя

```sql
SELECT
    user_id,
    active,
    rank,
    _part
FROM user_collapsing_merge_tree
FINAL;
```

### Задание 2

#### 2.1. Создание таблицы user_replacing с движком ReplacingMergeTree

```sql
DROP TABLE IF EXISTS user_replacing_merge_tree;

CREATE TABLE user_replacing_merge_tree
(
    user_id UInt32,
    active UInt8,
    rank UInt8,
    version UInt32
)
ENGINE = ReplacingMergeTree(version)
ORDER BY user_id;

```

#### 2.2. Добавление пользователя с rank = 0 и active = 0

```sql
INSERT INTO user_replacing_merge_tree 
(user_id, active, rank, version)
VALUES 
(1, 0, 0, 1);
```

#### 2.3. Изменение значения active с 0 на 1

```sql
INSERT INTO user_replacing_merge_tree 
(user_id, active, rank, version)
VALUES 
(1, 1, 0, 2);  -- Увеличиваем версию
```

#### 2.4. Изменение значения rank с 0 до 5

```sql
INSERT INTO user_replacing_merge_tree 
(user_id, active, rank, version)
VALUES 
(1, 1, 5, 3);  -- Снова увеличиваем версию
```

#### 2.5. Получение актуальных данных пользователя

```sql
SELECT 
    user_id,
    active,
    rank,
    version,
    _part
FROM user_replacing_merge_tree
FINAL;
```