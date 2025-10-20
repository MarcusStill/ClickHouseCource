# Движок таблиц типа Join

### 1. Пересоздаем таблицу learn_db.mart_student_lesson и наполняем ее данными

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
	`mark` Nullable(UInt8), -- Оценка
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

### 2. Получаем список школ с идентификаторами и названиями

```sql
SELECT DISTINCT
	educational_organization_id,
	'Школа № ' || educational_organization_id as educational_organization_name
FROM	
	learn_db.mart_student_lesson;
```

Результат
```text
Query id: 35b2838e-e375-4ad0-999c-6832b3b2603b

    ┌─educational_organization_id─┬─educational_organization_name─┐
 1. │                          18 │ Школа № 18                    │
 2. │                          12 │ Школа № 12                    │
 3. │                           6 │ Школа № 6                     │
 4. │                           3 │ Школа № 3                     │
 5. │                          27 │ Школа № 27                    │
 6. │                          17 │ Школа № 17                    │
 7. │                          16 │ Школа № 16                    │
 8. │                          14 │ Школа № 14                    │
 9. │                          20 │ Школа № 20                    │
10. │                          11 │ Школа № 11                    │
11. │                          26 │ Школа № 26                    │
12. │                           0 │ Школа № 0                     │
13. │                          23 │ Школа № 23                    │
14. │                           5 │ Школа № 5                     │
15. │                          10 │ Школа № 10                    │
16. │                           9 │ Школа № 9                     │
17. │                           2 │ Школа № 2                     │
18. │                          22 │ Школа № 22                    │
19. │                           4 │ Школа № 4                     │
20. │                           8 │ Школа № 8                     │
21. │                          25 │ Школа № 25                    │
22. │                           1 │ Школа № 1                     │
23. │                          13 │ Школа № 13                    │
24. │                          24 │ Школа № 24                    │
25. │                          21 │ Школа № 21                    │
26. │                          19 │ Школа № 19                    │
27. │                           7 │ Школа № 7                     │
28. │                          15 │ Школа № 15                    │
    └─────────────────────────────┴───────────────────────────────┘

28 rows in set. Elapsed: 0.224 sec. Processed 10.00 million rows, 20.00 MB (44.68 million rows/s., 89.36 MB/s.)
Peak memory usage: 12.50 MiB.
```

### 3. Создаем таблицу с движком Join

```sql
DROP TABLE IF EXISTS educational_organization_id_join;
CREATE TABLE educational_organization_id_join(
	`educational_organization_id` Int16, 
	`educational_organization_name` String
) ENGINE = Join(ANY, LEFT, educational_organization_id);
```

### 4. Наполняем таблицу с движком Join данными

```
INSERT INTO educational_organization_id_join
SELECT DISTINCT
	educational_organization_id,
	'Школа № ' || educational_organization_id as educational_organization_name
FROM	
	learn_db.mart_student_lesson;
```

### 5. Получаем данные из созданной таблицы

```sql
SELECT * FROM learn_db.educational_organization_id_join;
```

Результат
```text
Query id: 193d6ed9-a248-4976-9bf9-e307856a417e

    ┌─educational_organization_id─┬─educational_organization_name─┐
 1. │                           0 │ Школа № 0                     │
 2. │                           1 │ Школа № 1                     │
 3. │                           2 │ Школа № 2                     │
 4. │                           3 │ Школа № 3                     │
 5. │                           4 │ Школа № 4                     │
 6. │                           5 │ Школа № 5                     │
 7. │                           6 │ Школа № 6                     │
 8. │                           7 │ Школа № 7                     │
 9. │                           8 │ Школа № 8                     │
10. │                           9 │ Школа № 9                     │
11. │                          10 │ Школа № 10                    │
12. │                          11 │ Школа № 11                    │
13. │                          12 │ Школа № 12                    │
14. │                          13 │ Школа № 13                    │
15. │                          14 │ Школа № 14                    │
16. │                          15 │ Школа № 15                    │
17. │                          16 │ Школа № 16                    │
18. │                          17 │ Школа № 17                    │
19. │                          18 │ Школа № 18                    │
20. │                          19 │ Школа № 19                    │
21. │                          20 │ Школа № 20                    │
22. │                          21 │ Школа № 21                    │
23. │                          22 │ Школа № 22                    │
24. │                          23 │ Школа № 23                    │
25. │                          24 │ Школа № 24                    │
26. │                          25 │ Школа № 25                    │
27. │                          26 │ Школа № 26                    │
28. │                          27 │ Школа № 27                    │
    └─────────────────────────────┴───────────────────────────────┘

28 rows in set. Elapsed: 0.002 sec.
```

### 5. Смотрим план выполнения запроса получения данных из таблицы с движком Join

```sql
EXPLAIN header = 1, actions = 1, indexes = 1
SELECT * FROM learn_db.educational_organization_id_join;
```

Результат
```text
explain                                                                                               |
------------------------------------------------------------------------------------------------------+
Expression ((Project names + (Projection + Change column names to column identifiers)))               |
Header: educational_organization_id Int16                                                             |
        educational_organization_name String                                                          |
Actions: INPUT : 0 -> educational_organization_id Int16 : 0                                           |
         INPUT : 1 -> educational_organization_name String : 1                                        |
         ALIAS educational_organization_id :: 0 -> __table1.educational_organization_id Int16 : 2     |
         ALIAS educational_organization_name :: 1 -> __table1.educational_organization_name String : 0|
         ALIAS __table1.educational_organization_id :: 2 -> educational_organization_id Int16 : 1     |
         ALIAS __table1.educational_organization_name :: 0 -> educational_organization_name String : 2|
Positions: 1 2                                                                                        |
  ReadFromStorage (Join)                                                                              |
  Header: educational_organization_id Int16                                                           |
          educational_organization_name String                                                        |
```

### 6. Выполняем запрос, в котором присоединяем таблицу с движком типа Join

```sql
SELECT 
	j.educational_organization_name,
	avg(mark) as avg_mark
FROM
	learn_db.mart_student_lesson l
	ANY LEFT JOIN learn_db.educational_organization_id_join j
		USING (educational_organization_id)
GROUP BY 
	j.educational_organization_name;
```

Результат
```text
Query id: df359d55-930e-49c1-a77d-a22f5f730716

    ┌─educational_organization_name─┬───────────avg_mark─┐
 1. │ Школа № 18                    │  3.785985740414207 │
 2. │ Школа № 7                     │  3.787128304645883 │
 3. │ Школа № 11                    │ 3.7816145234031184 │
 4. │ Школа № 24                    │  3.787242063166866 │
 5. │ Школа № 14                    │ 3.7869390693514258 │
 6. │ Школа № 20                    │  3.786731555682933 │
 7. │ Школа № 8                     │ 3.7873853866619758 │
 8. │ Школа № 10                    │  3.784560379903708 │
 9. │ Школа № 19                    │ 3.7886938094296947 │
10. │ Школа № 2                     │  3.790960979090061 │
11. │ Школа № 1                     │ 3.7866244133514626 │
12. │ Школа № 25                    │ 3.7819622463062856 │
13. │ Школа № 15                    │ 3.7840668792251475 │
14. │ Школа № 4                     │  3.784972584526465 │
15. │ Школа № 21                    │  3.788496154475536 │
16. │ Школа № 27                    │ 3.7812720116014087 │
17. │ Школа № 23                    │  3.785372439682993 │
18. │ Школа № 13                    │  3.786384604019721 │
19. │ Школа № 6                     │ 3.7845193781475803 │
20. │ Школа № 0                     │   3.78576120154572 │
21. │ Школа № 16                    │  3.788465226372398 │
22. │ Школа № 26                    │ 3.7834380279149102 │
23. │ Школа № 5                     │ 3.7865859308149705 │
24. │ Школа № 3                     │ 3.7884540663598405 │
25. │ Школа № 22                    │   3.78716170047612 │
26. │ Школа № 9                     │  3.782684163402965 │
27. │ Школа № 12                    │   3.78466515938665 │
28. │ Школа № 17                    │  3.786182176375626 │
    └───────────────────────────────┴────────────────────┘

28 rows in set. Elapsed: 0.169 sec. Processed 10.00 million rows, 40.00 MB (59.04 million rows/s., 236.17 MB/s.)
Peak memory usage: 101.15 MiB.
```

Время выполнения запроса: 28 строк получено - 0.2s.

### 7. Получаем граф конвеера запроса и визуализируем в https://dreampuf.github.io/GraphvizOnline/

```sql
EXPLAIN pipeline graph = 1, compact = 0 
SELECT 
	j.educational_organization_name,
	avg(mark) as avg_mark
FROM
	learn_db.mart_student_lesson l
	ANY LEFT JOIN educational_organization_id_join j
		USING (educational_organization_id)
GROUP BY 
	j.educational_organization_name;
```

Результат
```text
digraph
{
  rankdir="LR";
  { node [shape = rect]
    n0[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_2"];
    n1[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_3"];
    n2[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_4"];
    n3[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_5"];
    n4[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_6"];
    n5[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_7"];
    n6[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_8"];
    n7[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_9"];
    n8[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_10"];
    n9[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_11"];
    n10[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_12"];
    n11[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_13"];
    n12[label="ExpressionTransform_14"];
    n13[label="ExpressionTransform_15"];
    n14[label="ExpressionTransform_16"];
    n15[label="ExpressionTransform_17"];
    n16[label="ExpressionTransform_18"];
    n17[label="ExpressionTransform_19"];
    n18[label="ExpressionTransform_20"];
    n19[label="ExpressionTransform_21"];
    n20[label="ExpressionTransform_22"];
    n21[label="ExpressionTransform_23"];
    n22[label="ExpressionTransform_24"];
    n23[label="ExpressionTransform_25"];
    n24[label="JoiningTransform_26"];
    n25[label="JoiningTransform_27"];
    n26[label="JoiningTransform_28"];
    n27[label="JoiningTransform_29"];
    n28[label="JoiningTransform_30"];
    n29[label="JoiningTransform_31"];
    n30[label="JoiningTransform_32"];
    n31[label="JoiningTransform_33"];
    n32[label="JoiningTransform_34"];
    n33[label="JoiningTransform_35"];
    n34[label="JoiningTransform_36"];
    n35[label="JoiningTransform_37"];
    n36[label="ExpressionTransform_38"];
    n37[label="ExpressionTransform_39"];
    n38[label="ExpressionTransform_40"];
    n39[label="ExpressionTransform_41"];
    n40[label="ExpressionTransform_42"];
    n41[label="ExpressionTransform_43"];
    n42[label="ExpressionTransform_44"];
    n43[label="ExpressionTransform_45"];
    n44[label="ExpressionTransform_46"];
    n45[label="ExpressionTransform_47"];
    n46[label="ExpressionTransform_48"];
    n47[label="ExpressionTransform_49"];
    n48[label="ExpressionTransform_50"];
    n49[label="ExpressionTransform_51"];
    n50[label="ExpressionTransform_52"];
    n51[label="ExpressionTransform_53"];
    n52[label="ExpressionTransform_54"];
    n53[label="ExpressionTransform_55"];
    n54[label="ExpressionTransform_56"];
    n55[label="ExpressionTransform_57"];
    n56[label="ExpressionTransform_58"];
    n57[label="ExpressionTransform_59"];
    n58[label="ExpressionTransform_60"];
    n59[label="ExpressionTransform_61"];
    n60[label="ExpressionTransform_62"];
    n61[label="ExpressionTransform_63"];
    n62[label="ExpressionTransform_64"];
    n63[label="ExpressionTransform_65"];
    n64[label="ExpressionTransform_66"];
    n65[label="ExpressionTransform_67"];
    n66[label="ExpressionTransform_68"];
    n67[label="ExpressionTransform_69"];
    n68[label="ExpressionTransform_70"];
    n69[label="ExpressionTransform_71"];
    n70[label="ExpressionTransform_72"];
    n71[label="ExpressionTransform_73"];
    n72[label="AggregatingTransform_74"];
    n73[label="AggregatingTransform_75"];
    n74[label="AggregatingTransform_76"];
    n75[label="AggregatingTransform_77"];
    n76[label="AggregatingTransform_78"];
    n77[label="AggregatingTransform_79"];
    n78[label="AggregatingTransform_80"];
    n79[label="AggregatingTransform_81"];
    n80[label="AggregatingTransform_82"];
    n81[label="AggregatingTransform_83"];
    n82[label="AggregatingTransform_84"];
    n83[label="AggregatingTransform_85"];
    n84[label="Resize_86"];
    n85[label="ExpressionTransform_87"];
    n86[label="ExpressionTransform_88"];
    n87[label="ExpressionTransform_89"];
    n88[label="ExpressionTransform_90"];
    n89[label="ExpressionTransform_91"];
    n90[label="ExpressionTransform_92"];
    n91[label="ExpressionTransform_93"];
    n92[label="ExpressionTransform_94"];
    n93[label="ExpressionTransform_95"];
    n94[label="ExpressionTransform_96"];
    n95[label="ExpressionTransform_97"];
    n96[label="ExpressionTransform_98"];
  }
  n0 -> n12;
  n1 -> n13;
  n2 -> n14;
  n3 -> n15;
  n4 -> n16;
  n5 -> n17;
  n6 -> n18;
  n7 -> n19;
  n8 -> n20;
  n9 -> n21;
  n10 -> n22;
  n11 -> n23;
  n12 -> n24;
  n13 -> n25;
  n14 -> n26;
  n15 -> n27;
  n16 -> n28;
  n17 -> n29;
  n18 -> n30;
  n19 -> n31;
  n20 -> n32;
  n21 -> n33;
  n22 -> n34;
  n23 -> n35;
  n24 -> n36;
  n25 -> n37;
  n26 -> n38;
  n27 -> n39;
  n28 -> n40;
  n29 -> n41;
  n30 -> n42;
  n31 -> n43;
  n32 -> n44;
  n33 -> n45;
  n34 -> n46;
  n35 -> n47;
  n36 -> n48;
  n37 -> n49;
  n38 -> n50;
  n39 -> n51;
  n40 -> n52;
  n41 -> n53;
  n42 -> n54;
  n43 -> n55;
  n44 -> n56;
  n45 -> n57;
  n46 -> n58;
  n47 -> n59;
  n48 -> n60;
  n49 -> n61;
  n50 -> n62;
  n51 -> n63;
  n52 -> n64;
  n53 -> n65;
  n54 -> n66;
  n55 -> n67;
  n56 -> n68;
  n57 -> n69;
  n58 -> n70;
  n59 -> n71;
  n60 -> n72;
  n61 -> n73;
  n62 -> n74;
  n63 -> n75;
  n64 -> n76;
  n65 -> n77;
  n66 -> n78;
  n67 -> n79;
  n68 -> n80;
  n69 -> n81;
  n70 -> n82;
  n71 -> n83;
  n72 -> n84;
  n73 -> n84;
  n74 -> n84;
  n75 -> n84;
  n76 -> n84;
  n77 -> n84;
  n78 -> n84;
  n79 -> n84;
  n80 -> n84;
  n81 -> n84;
  n82 -> n84;
  n83 -> n84;
  n84 -> n85;
  n84 -> n86;
  n84 -> n87;
  n84 -> n88;
  n84 -> n89;
  n84 -> n90;
  n84 -> n91;
  n84 -> n92;
  n84 -> n93;
  n84 -> n94;
  n84 -> n95;
  n84 -> n96;
}
```

### 8. Получаем информацию о времени выполнения шагов запроса с соединением таблиц

```sql
SELECT * FROM system.query_log ORDER BY event_time DESC;
```

Результат
```text
query_id = b96520e1-07b0-4b35-bbd4-f9c234893576
```

```sql
SELECT sum(elapsed_us) FROM system.processors_profile_log WHERE query_id = 'b96520e1-07b0-4b35-bbd4-f9c234893576' AND processor_uniq_id LIKE 'JoiningTransform_%';
```
Результат
```text
Name           |Value  |
---------------+-------+
SUM(elapsed_us)|764391|
```

### 9. Общее процессорное время выполнения всех шагов запроса

```sql
SELECT sum(elapsed_us) FROM system.processors_profile_log WHERE query_id = 'b96520e1-07b0-4b35-bbd4-f9c234893576' 
```

Результат
```text
Name           |Value  |
---------------+-------+
SUM(elapsed_us)|1854453|
```

### 10. Создаем второй вариант таблицы со списком школ с движком MergeTree

```sql
DROP TABLE IF EXISTS educational_organization_id_mergetree;
CREATE TABLE educational_organization_id_mergetree(
	`educational_organization_id` Int16, 
	`educational_organization_name` String
) ENGINE = MergeTree()
ORDER BY educational_organization_id;
```

### 11. Наполняем таблицу с движком MergeTree данными

```sql
INSERT INTO educational_organization_id_mergetree
SELECT DISTINCT
	educational_organization_id,
	'Школа № ' || educational_organization_id as educational_organization_name
FROM	
	learn_db.mart_student_lesson;
```

### 12. Смотрим план выполнения запроса соединения с таблицей с движком MergeTree

```sql
EXPLAIN header = 1, actions = 1, indexes = 1
SELECT * FROM educational_organization_id_mergetree;
```

Результат
```text
explain                                                                                               |
------------------------------------------------------------------------------------------------------+
Expression ((Project names + (Projection + Change column names to column identifiers)))               |
Header: educational_organization_id Int16                                                             |
        educational_organization_name String                                                          |
Actions: INPUT : 0 -> educational_organization_id Int16 : 0                                           |
         INPUT : 1 -> educational_organization_name String : 1                                        |
         ALIAS educational_organization_id :: 0 -> __table1.educational_organization_id Int16 : 2     |
         ALIAS educational_organization_name :: 1 -> __table1.educational_organization_name String : 0|
         ALIAS __table1.educational_organization_id :: 2 -> educational_organization_id Int16 : 1     |
         ALIAS __table1.educational_organization_name :: 0 -> educational_organization_name String : 2|
Positions: 1 2                                                                                        |
  ReadFromMergeTree (learn_db.educational_organization_id_mergetree)                                  |
  Header: educational_organization_id Int16                                                           |
          educational_organization_name String                                                        |
  ReadType: Default                                                                                   |
  Parts: 1                                                                                            |
  Granules: 1                                                                                         |
  Indexes:                                                                                            |
    PrimaryKey                                                                                        |
      Condition: true                                                                                 |
      Parts: 1/1                                                                                      |
      Granules: 1/1                                                                                   |
```

```sql
SELECT 
	j.educational_organization_name,
	avg(mark) as avg_mark
FROM
	learn_db.mart_student_lesson l
	ANY LEFT JOIN educational_organization_id_mergetree j
		USING (educational_organization_id)
GROUP BY 
	j.educational_organization_name;
```

Результат
```text
Query id: 9187b1b1-6a62-4ba9-96c1-19b69592b458

    ┌─educational_organization_name─┬───────────avg_mark─┐
 1. │ Школа № 18                    │  3.785985740414207 │
 2. │ Школа № 20                    │  3.786731555682933 │
 3. │ Школа № 8                     │ 3.7873853866619758 │
 4. │ Школа № 14                    │ 3.7869390693514258 │
 5. │ Школа № 19                    │ 3.7886938094296947 │
 6. │ Школа № 10                    │  3.784560379903708 │
 7. │ Школа № 2                     │  3.790960979090061 │
 8. │ Школа № 25                    │ 3.7819622463062856 │
 9. │ Школа № 15                    │ 3.7840668792251475 │
10. │ Школа № 1                     │ 3.7866244133514626 │
11. │ Школа № 4                     │  3.784972584526465 │
12. │ Школа № 21                    │  3.788496154475536 │
13. │ Школа № 7                     │  3.787128304645883 │
14. │ Школа № 11                    │ 3.7816145234031184 │
15. │ Школа № 24                    │  3.787242063166866 │
16. │ Школа № 23                    │  3.785372439682993 │
17. │ Школа № 13                    │  3.786384604019721 │
18. │ Школа № 27                    │ 3.7812720116014087 │
19. │ Школа № 9                     │  3.782684163402965 │
20. │ Школа № 26                    │ 3.7834380279149102 │
21. │ Школа № 12                    │   3.78466515938665 │
22. │ Школа № 17                    │  3.786182176375626 │
23. │ Школа № 3                     │ 3.7884540663598405 │
24. │ Школа № 22                    │   3.78716170047612 │
25. │ Школа № 0                     │   3.78576120154572 │
26. │ Школа № 16                    │  3.788465226372398 │
27. │ Школа № 6                     │ 3.7845193781475803 │
28. │ Школа № 5                     │ 3.7865859308149705 │
    └───────────────────────────────┴────────────────────┘

28 rows in set. Elapsed: 0.218 sec. Processed 10.00 million rows, 40.00 MB (45.89 million rows/s., 183.58 MB/s.)
Peak memory usage: 149.38 MiB.
```

### 13. Получаем конвеер графа выполнения запроса с соединением с таблицой с движком MergeTree

```sql
EXPLAIN pipeline graph = 1, compact = 0
SELECT 
	j.educational_organization_name,
	avg(mark) as avg_mark
FROM
	learn_db.mart_student_lesson l
	ANY LEFT JOIN learn_db.educational_organization_id_mergetree j
		USING (educational_organization_id)
GROUP BY 
	j.educational_organization_name;
```

Результат
```text
digraph
{
  rankdir="LR";
  { node [shape = rect]
    n0[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_0"];
    n1[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_1"];
    n2[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_2"];
    n3[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_3"];
    n4[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_4"];
    n5[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_5"];
    n6[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_6"];
    n7[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_7"];
    n8[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_8"];
    n9[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_9"];
    n10[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_10"];
    n11[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_11"];
    n12[label="ExpressionTransform_12"];
    n13[label="ExpressionTransform_13"];
    n14[label="ExpressionTransform_14"];
    n15[label="ExpressionTransform_15"];
    n16[label="ExpressionTransform_16"];
    n17[label="ExpressionTransform_17"];
    n18[label="ExpressionTransform_18"];
    n19[label="ExpressionTransform_19"];
    n20[label="ExpressionTransform_20"];
    n21[label="ExpressionTransform_21"];
    n22[label="ExpressionTransform_22"];
    n23[label="ExpressionTransform_23"];
    n24[label="SimpleSquashingTransform_28"];
    n25[label="JoiningTransform_29"];
    n26[label="SimpleSquashingTransform_30"];
    n27[label="JoiningTransform_31"];
    n28[label="SimpleSquashingTransform_32"];
    n29[label="JoiningTransform_33"];
    n30[label="SimpleSquashingTransform_34"];
    n31[label="JoiningTransform_35"];
    n32[label="SimpleSquashingTransform_36"];
    n33[label="JoiningTransform_37"];
    n34[label="SimpleSquashingTransform_38"];
    n35[label="JoiningTransform_39"];
    n36[label="SimpleSquashingTransform_40"];
    n37[label="JoiningTransform_41"];
    n38[label="SimpleSquashingTransform_42"];
    n39[label="JoiningTransform_43"];
    n40[label="SimpleSquashingTransform_44"];
    n41[label="JoiningTransform_45"];
    n42[label="SimpleSquashingTransform_46"];
    n43[label="JoiningTransform_47"];
    n44[label="SimpleSquashingTransform_48"];
    n45[label="JoiningTransform_49"];
    n46[label="SimpleSquashingTransform_50"];
    n47[label="JoiningTransform_51"];
    n48[label="MergeTreeSelect(pool: ReadPoolInOrder, algorithm: InOrder)_24"];
    n49[label="ExpressionTransform_25"];
    n50[label="FillingRightJoinSide_26"];
    n51[label="Resize_27"];
    n52[label="ColumnPermuteTransform_52"];
    n53[label="ColumnPermuteTransform_53"];
    n54[label="ColumnPermuteTransform_54"];
    n55[label="ColumnPermuteTransform_55"];
    n56[label="ColumnPermuteTransform_56"];
    n57[label="ColumnPermuteTransform_57"];
    n58[label="ColumnPermuteTransform_58"];
    n59[label="ColumnPermuteTransform_59"];
    n60[label="ColumnPermuteTransform_60"];
    n61[label="ColumnPermuteTransform_61"];
    n62[label="ColumnPermuteTransform_62"];
    n63[label="ColumnPermuteTransform_63"];
    n64[label="ExpressionTransform_64"];
    n65[label="ExpressionTransform_65"];
    n66[label="ExpressionTransform_66"];
    n67[label="ExpressionTransform_67"];
    n68[label="ExpressionTransform_68"];
    n69[label="ExpressionTransform_69"];
    n70[label="ExpressionTransform_70"];
    n71[label="ExpressionTransform_71"];
    n72[label="ExpressionTransform_72"];
    n73[label="ExpressionTransform_73"];
    n74[label="ExpressionTransform_74"];
    n75[label="ExpressionTransform_75"];
    n76[label="ExpressionTransform_76"];
    n77[label="ExpressionTransform_77"];
    n78[label="ExpressionTransform_78"];
    n79[label="ExpressionTransform_79"];
    n80[label="ExpressionTransform_80"];
    n81[label="ExpressionTransform_81"];
    n82[label="ExpressionTransform_82"];
    n83[label="ExpressionTransform_83"];
    n84[label="ExpressionTransform_84"];
    n85[label="ExpressionTransform_85"];
    n86[label="ExpressionTransform_86"];
    n87[label="ExpressionTransform_87"];
    n88[label="AggregatingTransform_88"];
    n89[label="AggregatingTransform_89"];
    n90[label="AggregatingTransform_90"];
    n91[label="AggregatingTransform_91"];
    n92[label="AggregatingTransform_92"];
    n93[label="AggregatingTransform_93"];
    n94[label="AggregatingTransform_94"];
    n95[label="AggregatingTransform_95"];
    n96[label="AggregatingTransform_96"];
    n97[label="AggregatingTransform_97"];
    n98[label="AggregatingTransform_98"];
    n99[label="AggregatingTransform_99"];
    n100[label="Resize_100"];
    n101[label="ExpressionTransform_101"];
    n102[label="ExpressionTransform_102"];
    n103[label="ExpressionTransform_103"];
    n104[label="ExpressionTransform_104"];
    n105[label="ExpressionTransform_105"];
    n106[label="ExpressionTransform_106"];
    n107[label="ExpressionTransform_107"];
    n108[label="ExpressionTransform_108"];
    n109[label="ExpressionTransform_109"];
    n110[label="ExpressionTransform_110"];
    n111[label="ExpressionTransform_111"];
    n112[label="ExpressionTransform_112"];
  }
  n0 -> n12;
  n1 -> n13;
  n2 -> n14;
  n3 -> n15;
  n4 -> n16;
  n5 -> n17;
  n6 -> n18;
  n7 -> n19;
  n8 -> n20;
  n9 -> n21;
  n10 -> n22;
  n11 -> n23;
  n12 -> n24;
  n13 -> n26;
  n14 -> n28;
  n15 -> n30;
  n16 -> n32;
  n17 -> n34;
  n18 -> n36;
  n19 -> n38;
  n20 -> n40;
  n21 -> n42;
  n22 -> n44;
  n23 -> n46;
  n24 -> n25;
  n25 -> n52;
  n26 -> n27;
  n27 -> n53;
  n28 -> n29;
  n29 -> n54;
  n30 -> n31;
  n31 -> n55;
  n32 -> n33;
  n33 -> n56;
  n34 -> n35;
  n35 -> n57;
  n36 -> n37;
  n37 -> n58;
  n38 -> n39;
  n39 -> n59;
  n40 -> n41;
  n41 -> n60;
  n42 -> n43;
  n43 -> n61;
  n44 -> n45;
  n45 -> n62;
  n46 -> n47;
  n47 -> n63;
  n48 -> n49;
  n49 -> n50;
  n50 -> n51;
  n51 -> n25;
  n51 -> n27;
  n51 -> n29;
  n51 -> n31;
  n51 -> n33;
  n51 -> n35;
  n51 -> n37;
  n51 -> n39;
  n51 -> n41;
  n51 -> n43;
  n51 -> n45;
  n51 -> n47;
  n52 -> n64;
  n53 -> n65;
  n54 -> n66;
  n55 -> n67;
  n56 -> n68;
  n57 -> n69;
  n58 -> n70;
  n59 -> n71;
  n60 -> n72;
  n61 -> n73;
  n62 -> n74;
  n63 -> n75;
  n64 -> n76;
  n65 -> n77;
  n66 -> n78;
  n67 -> n79;
  n68 -> n80;
  n69 -> n81;
  n70 -> n82;
  n71 -> n83;
  n72 -> n84;
  n73 -> n85;
  n74 -> n86;
  n75 -> n87;
  n76 -> n88;
  n77 -> n89;
  n78 -> n90;
  n79 -> n91;
  n80 -> n92;
  n81 -> n93;
  n82 -> n94;
  n83 -> n95;
  n84 -> n96;
  n85 -> n97;
  n86 -> n98;
  n87 -> n99;
  n88 -> n100;
  n89 -> n100;
  n90 -> n100;
  n91 -> n100;
  n92 -> n100;
  n93 -> n100;
  n94 -> n100;
  n95 -> n100;
  n96 -> n100;
  n97 -> n100;
  n98 -> n100;
  n99 -> n100;
  n100 -> n101;
  n100 -> n102;
  n100 -> n103;
  n100 -> n104;
  n100 -> n105;
  n100 -> n106;
  n100 -> n107;
  n100 -> n108;
  n100 -> n109;
  n100 -> n110;
  n100 -> n111;
  n100 -> n112;
}
```

Видим, что шаг ReadPoolInOrder был выделен отдельно. для него была сделана предобработка (ExpressionTransform_25, FillingRightJoinSide_26, Resize_27) и потом шаг Resize_27 присоединяется к шагам JoiningTransform_*.

### 14. Получаем информацию о времени выполнения шагов запроса с соединением таблиц

```sql
SELECT * FROM system.query_log ORDER BY event_time DESC; 
```

Результат
```text
query_id = d3ee830d-5ea5-4671-a199-15d6aedd5897
```

```sql
SELECT sum(elapsed_us) FROM system.processors_profile_log WHERE query_id = '85a2cec4-d757-4132-9e6c-7d90ccbe4806' AND processor_uniq_id LIKE 'JoiningTransform_%';
```

Результат
```text
sum(elapsed_us)|
---------------+
         819213|
```

### 15. Общее процессорное время выполнения всех шагов запроса

```sql
SELECT sum(elapsed_us) FROM system.processors_profile_log WHERE query_id = '85a2cec4-d757-4132-9e6c-7d90ccbe4806' 
```

Результат
```text
Name           |Value  |
---------------+-------+
SUM(elapsed_us)|1907073|
```

**Вывод:** 
* Потребление памяти: Join экономичнее на 32%
* Скорость обработки: Join быстрее на 29%
* Время JoiningTransform: Join эффективнее на 7%
* Общее процессорное время: Join эффективнее на 3%