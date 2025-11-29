# План выполнения запроса с помощью Explain

### 1. Пересоздаем таблицу и наполняем ее данными
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

### 2. Запрос для исследования

```sql
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

Результат
```text
subject_name       |avg_mark|
-------------------+--------+
Математика         |    4.35|
Русский язык       |    4.26|
Литература         |    4.15|
Физика             |     4.0|
Химия              |    3.85|
География          |    3.76|
Биология           |    3.64|
Информатика        |     3.6|
Физическая культура|    3.56|
```

## AST

### 3. Получаем абстрактное синтаксическое дерево

```sql
EXPLAIN AST 
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

Результат
```text
explain                                           |
--------------------------------------------------+
SelectWithUnionQuery (children 1)                 |
 ExpressionList (children 1)                      |
  SelectQuery (children 5)                        |
   ExpressionList (children 2)                    |
    Identifier subject_name                       |
    Function ROUND (alias avg_mark) (children 1)  |
     ExpressionList (children 2)                  |
      Function AVG (children 1)                   |
       ExpressionList (children 1)                |
        Identifier mark                           |
      Literal UInt64_2                            |
   TablesInSelectQuery (children 1)               |
    TablesInSelectQueryElement (children 1)       |
     TableExpression (children 1)                 |
      TableIdentifier learn_db.mart_student_lesson|
   Function greaterOrEquals (children 1)          |
    ExpressionList (children 2)                   |
     Identifier lesson_date                       |
     Function minus (children 1)                  |
      ExpressionList (children 2)                 |
       Function today (children 1)                |
        ExpressionList                            |
       Literal UInt64_10                          |
   ExpressionList (children 1)                    |
    Identifier subject_name                       |
   ExpressionList (children 1)                    |
    OrderByElement (children 1)                   |
     Identifier avg_mark                          |
```

### 4. Получаем абстрактное синтаксическое дерево в виде графа. [Страница для визуализации графа](https://dreampuf.github.io/GraphvizOnline/)

```sql
EXPLAIN AST graph = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

Результат
```text
explain                                                                    |
---------------------------------------------------------------------------+
digraph {                                                                  |
    rankdir="UD";                                                          |
    n140581367204376[label="SelectWithUnionQuery (children 1)"];           |
    n140581938508152[label="ExpressionList (children 1)"];                 |
    n140581371495704[label="SelectQuery (children 5)"];                    |
    n140582771867448[label="ExpressionList (children 2)"];                 |
    n140581365300632[label="Identifier subject_name"];                     |
    n140581367203992[label="Function ROUND (alias avg_mark) (children 1)"];|
    n140585259726744[label="ExpressionList (children 2)"];                 |
    n140581367203224[label="Function AVG (children 1)"];                   |
    n140581938506584[label="ExpressionList (children 1)"];                 |
    n140581940420056[label="Identifier mark"];                             |
    n140581938506584 -> n140581940420056;                                  |
    n140581367203224 -> n140581938506584;                                  |
    n140580239045720[label="Literal UInt64_2"];                            |
    n140585259726744 -> n140581367203224;                                  |
    n140585259726744 -> n140580239045720;                                  |
    n140581367203992 -> n140585259726744;                                  |
    n140582771867448 -> n140581365300632;                                  |
    n140582771867448 -> n140581367203992;                                  |
    n140581938047832[label="TablesInSelectQuery (children 1)"];            |
    n140581371497240[label="TablesInSelectQueryElement (children 1)"];     |
    n140581475251800[label="TableExpression (children 1)"];                |
    n140581367202840[label="TableIdentifier learn_db.mart_student_lesson"];|
    n140581475251800 -> n140581367202840;                                  |
    n140581371497240 -> n140581475251800;                                  |
    n140581938047832 -> n140581371497240;                                  |
    n140581367205144[label="Function greaterOrEquals (children 1)"];       |
    n140582771865656[label="ExpressionList (children 2)"];                 |
    n140581475254360[label="Identifier lesson_date"];                      |
    n140583053821464[label="Function minus (children 1)"];                 |
    n140582770787256[label="ExpressionList (children 2)"];                 |
    n140581369766296[label="Function today (children 1)"];                 |
    n140580238961048[label="ExpressionList"];                              |
    n140581369766296 -> n140580238961048;                                  |
    n140580239045272[label="Literal UInt64_10"];                           |
    n140582770787256 -> n140581369766296;                                  |
    n140582770787256 -> n140580239045272;                                  |
    n140583053821464 -> n140582770787256;                                  |
    n140582771865656 -> n140581475254360;                                  |
    n140582771865656 -> n140583053821464;                                  |
    n140581367205144 -> n140582771865656;                                  |
    n140582771864984[label="ExpressionList (children 1)"];                 |
    n140589100215512[label="Identifier subject_name"];                     |
    n140582771864984 -> n140589100215512;                                  |
    n140583894496184[label="ExpressionList (children 1)"];                 |
    n140581936368920[label="OrderByElement (children 1)"];                 |
    n140581365300312[label="Identifier avg_mark"];                         |
    n140581936368920 -> n140581365300312;                                  |
    n140583894496184 -> n140581936368920;                                  |
    n140581371495704 -> n140582771867448;                                  |
    n140581371495704 -> n140581938047832;                                  |
    n140581371495704 -> n140581367205144;                                  |
    n140581371495704 -> n140582771864984;                                  |
    n140581371495704 -> n140583894496184;                                  |
    n140581938508152 -> n140581371495704;                                  |
    n140581367204376 -> n140581938508152;                                  |
}                                                                          |
```

### 5. Получаем абстрактное синтаксическое дерево с флагом оптимизации (проверяет корректность указанных таблиц и колонок)

```sql
EXPLAIN AST optimize = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

Результат
```text
explain                                           |
--------------------------------------------------+
SelectWithUnionQuery (children 1)                 |
 ExpressionList (children 1)                      |
  SelectQuery (children 5)                        |
   ExpressionList (children 2)                    |
    Identifier subject_name                       |
    Function round (alias avg_mark) (children 1)  |
     ExpressionList (children 2)                  |
      Function avg (children 1)                   |
       ExpressionList (children 1)                |
        Identifier mark                           |
      Literal UInt64_2                            |
   TablesInSelectQuery (children 1)               |
    TablesInSelectQueryElement (children 1)       |
     TableExpression (children 1)                 |
      TableIdentifier learn_db.mart_student_lesson|
   Function greaterOrEquals (children 1)          |
    ExpressionList (children 2)                   |
     Identifier lesson_date                       |
     Function minus (children 1)                  |
      ExpressionList (children 2)                 |
       Function today (children 1)                |
        ExpressionList                            |
       Literal UInt64_10                          |
   ExpressionList (children 1)                    |
    Identifier subject_name                       |
   ExpressionList (children 1)                    |
    OrderByElement (children 1)                   |
     Function round (alias avg_mark) (children 1) |
      ExpressionList (children 2)                 |
       Function avg (children 1)                  |
        ExpressionList (children 1)               |
         Identifier mark                          |
       Literal UInt64_2                           |
```

## Query Tree

### 6. Получаем дерево запроса

```sql
EXPLAIN QUERY TREE 
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

Результат
```text
explain                                                                                                      |
-------------------------------------------------------------------------------------------------------------+
QUERY id: 0                                                                                                  |
  PROJECTION COLUMNS                                                                                         |
    subject_name String                                                                                      |
    avg_mark Nullable(Float64)                                                                               |
  PROJECTION                                                                                                 |
    LIST id: 1, nodes: 2                                                                                     |
      COLUMN id: 2, column_name: subject_name, result_type: String, source_id: 3                             |
      FUNCTION id: 4, function_name: round, function_type: ordinary, result_type: Nullable(Float64)          |
        ARGUMENTS                                                                                            |
          LIST id: 5, nodes: 2                                                                               |
            FUNCTION id: 6, function_name: avg, function_type: aggregate, result_type: Nullable(Float64)     |
              ARGUMENTS                                                                                      |
                LIST id: 7, nodes: 1                                                                         |
                  COLUMN id: 8, column_name: mark, result_type: Nullable(UInt8), source_id: 3                |
            CONSTANT id: 9, constant_value: UInt64_2, constant_value_type: UInt8                             |
  JOIN TREE                                                                                                  |
    TABLE id: 3, alias: __table1, table_name: learn_db.mart_student_lesson                                   |
  WHERE                                                                                                      |
    FUNCTION id: 10, function_name: greaterOrEquals, function_type: ordinary, result_type: UInt8             |
      ARGUMENTS                                                                                              |
        LIST id: 11, nodes: 2                                                                                |
          COLUMN id: 12, column_name: lesson_date, result_type: Date32, source_id: 3                         |
          CONSTANT id: 13, constant_value: UInt64_20364, constant_value_type: Date                           |
            EXPRESSION                                                                                       |
              FUNCTION id: 14, function_name: minus, function_type: ordinary, result_type: Date              |
                ARGUMENTS                                                                                    |
                  LIST id: 15, nodes: 2                                                                      |
                    CONSTANT id: 16, constant_value: UInt64_20374, constant_value_type: Date                 |
                      EXPRESSION                                                                             |
                        FUNCTION id: 17, function_name: today, function_type: ordinary, result_type: Date    |
                    CONSTANT id: 18, constant_value: UInt64_10, constant_value_type: UInt8                   |
  GROUP BY                                                                                                   |
    LIST id: 19, nodes: 1                                                                                    |
      COLUMN id: 20, column_name: subject_name, result_type: String, source_id: 3                            |
  ORDER BY                                                                                                   |
    LIST id: 21, nodes: 1                                                                                    |
      SORT id: 22, sort_direction: DESCENDING, with_fill: 0                                                  |
        EXPRESSION                                                                                           |
          FUNCTION id: 23, function_name: round, function_type: ordinary, result_type: Nullable(Float64)     |
            ARGUMENTS                                                                                        |
              LIST id: 24, nodes: 2                                                                          |
                FUNCTION id: 25, function_name: avg, function_type: aggregate, result_type: Nullable(Float64)|
                  ARGUMENTS                                                                                  |
                    LIST id: 26, nodes: 1                                                                    |
                      COLUMN id: 27, column_name: mark, result_type: Nullable(UInt8), source_id: 3           |
                CONSTANT id: 28, constant_value: UInt64_2, constant_value_type: UInt8                        |
```

### 7. Получаем дерево запроса без оптимизаций (проходов)

```sql
EXPLAIN QUERY TREE run_passes = 0
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

Результат
```text
explain                                                                               |
--------------------------------------------------------------------------------------+
QUERY id: 0                                                                           |
  PROJECTION                                                                          |
    LIST id: 1, nodes: 2                                                              |
      IDENTIFIER id: 2, identifier: subject_name                                      |
      FUNCTION id: 3, alias: avg_mark, function_name: ROUND, function_type: ordinary  |
        ARGUMENTS                                                                     |
          LIST id: 4, nodes: 2                                                        |
            FUNCTION id: 5, function_name: AVG, function_type: ordinary               |
              ARGUMENTS                                                               |
                LIST id: 6, nodes: 1                                                  |
                  IDENTIFIER id: 7, identifier: mark                                  |
            CONSTANT id: 8, constant_value: UInt64_2, constant_value_type: UInt8      |
  JOIN TREE                                                                           |
    IDENTIFIER id: 9, identifier: learn_db.mart_student_lesson                        |
  WHERE                                                                               |
    FUNCTION id: 10, function_name: greaterOrEquals, function_type: ordinary          |
      ARGUMENTS                                                                       |
        LIST id: 11, nodes: 2                                                         |
          IDENTIFIER id: 12, identifier: lesson_date                                  |
          FUNCTION id: 13, function_name: minus, function_type: ordinary              |
            ARGUMENTS                                                                 |
              LIST id: 14, nodes: 2                                                   |
                FUNCTION id: 15, function_name: today, function_type: ordinary        |
                CONSTANT id: 16, constant_value: UInt64_10, constant_value_type: UInt8|
  GROUP BY                                                                            |
    LIST id: 17, nodes: 1                                                             |
      IDENTIFIER id: 18, identifier: subject_name                                     |
  ORDER BY                                                                            |
    LIST id: 19, nodes: 1                                                             |
      SORT id: 20, sort_direction: DESCENDING, with_fill: 0                           |
        EXPRESSION                                                                    |
          IDENTIFIER id: 21, identifier: avg_mark                                     |
```

### 8. Получаем дерево запроса со всеми оптимизациями и выводом списка примененных оптимизаций (проходов)

```sql
EXPLAIN QUERY TREE run_passes = 1, dump_passes = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

Результат
```text
explain                                                                                                                                                                    |
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
Pass 1 QueryAnalysis - Resolve type for each query expression. Replace identifiers, matchers with query expressions. Perform constant folding. Evaluate scalar subqueries. |
Pass 2 GroupingFunctionsResolvePass - Resolve GROUPING functions based on GROUP BY modifiers                                                                               |
Pass 3 AutoFinalOnQueryPass - Automatically applies final modifier to table expressions in queries if it is supported and if user level final setting is set               |
Pass 4 RemoveUnusedProjectionColumnsPass - Remove unused projection columns in subqueries.                                                                                 |
Pass 5 FunctionToSubcolumns - Rewrite function to subcolumns, for example tupleElement(column, subcolumn) into column.subcolumn                                            |
Pass 6 ConvertLogicalExpressionToCNFPass - Convert logical expression to CNF and apply optimizations using constraints                                                     |
Pass 7 RewriteSumFunctionWithSumAndCountPass - Rewrite sum(column +/- literal) into sum(column) and literal * count(column)                                                |
Pass 8 CountDistinct - Optimize single countDistinct into count over subquery                                                                                              |
Pass 9 UniqToCount - Rewrite uniq and its variants(except uniqUpTo) to count if subquery has distinct or group by clause.                                                  |
Pass 10 RewriteArrayExistsToHas - Rewrite arrayExists(func, arr) functions to has(arr, elem) when logically equivalent                                                     |
Pass 11 NormalizeCountVariants - Optimize count(literal), sum(1) into count().                                                                                             |
Pass 12 AggregateFunctionOfGroupByKeys - Eliminates min/max/any/anyLast aggregators of GROUP BY keys in SELECT section.                                                    |
Pass 13 AggregateFunctionsArithmericOperations - Extract arithmeric operations from aggregate functions.                                                                   |
Pass 14 UniqInjectiveFunctionsElimination - Remove injective functions from uniq functions arguments.                                                                      |
Pass 15 IfTransformStringsToEnumPass - Replaces string-type arguments in If and Transform to enum                                                                          |
Pass 16 OptimizeGroupByFunctionKeys - Eliminates functions of other keys in GROUP BY section.                                                                              |
Pass 17 OptimizeGroupByInjectiveFunctionsPass - Replaces injective functions by it's arguments in GROUP BY section.                                                        |
Pass 18 MultiIfToIf - Optimize multiIf with single condition to if.                                                                                                        |
Pass 19 IfConstantCondition - Optimize if, multiIf for constant condition.                                                                                                 |
Pass 20 IfChainToMultiIf - Optimize if chain to multiIf                                                                                                                    |
Pass 21 RewriteAggregateFunctionWithIf - Rewrite aggregate functions with if expression as argument when logically equivalent                                              |
Pass 22 SumIfToCountIf - Rewrite sum(if) and sumIf into countIf                                                                                                            |
Pass 23 ComparisonTupleEliminationPass - Rewrite tuples comparison into equivalent comparison of tuples arguments                                                          |
Pass 24 OptimizeRedundantFunctionsInOrderBy - If ORDER BY has argument x followed by f(x) transforms it to ORDER BY x.                                                     |
Pass 25 OrderByTupleElimination - Remove tuple from ORDER BY.                                                                                                              |
Pass 26 OrderByLimitByDuplicateElimination - Remove duplicate columns from ORDER BY, LIMIT BY.                                                                             |
Pass 27 FuseFunctionsPass - Replaces several calls of aggregate functions of the same family into one call                                                                 |
Pass 28 ConvertOrLikeChain - Replaces all the 'or's with {i}like to multiMatchAny                                                                                          |
Pass 29 LogicalExpressionOptimizer - Transforms chains of logical expressions if possible, i.e. replace chains of equality functions inside an OR with a single IN operator|
Pass 30 CrossToInnerJoin - Replace CROSS JOIN with INNER JOIN                                                                                                              |
Pass 31 ShardNumColumnToFunctionPass - Rewrite _shard_num column into shardNum() function                                                                                  |
Pass 32 OptimizeDateOrDateTimeConverterWithPreimagePass - Replace predicate having Date/DateTime converters with their preimages                                           |
                                                                                                                                                                           |
QUERY id: 0                                                                                                                                                                |
  PROJECTION COLUMNS                                                                                                                                                       |
    subject_name String                                                                                                                                                    |
    avg_mark Nullable(Float64)                                                                                                                                             |
  PROJECTION                                                                                                                                                               |
    LIST id: 1, nodes: 2                                                                                                                                                   |
      COLUMN id: 2, column_name: subject_name, result_type: String, source_id: 3                                                                                           |
      FUNCTION id: 4, function_name: round, function_type: ordinary, result_type: Nullable(Float64)                                                                        |
        ARGUMENTS                                                                                                                                                          |
          LIST id: 5, nodes: 2                                                                                                                                             |
            FUNCTION id: 6, function_name: avg, function_type: aggregate, result_type: Nullable(Float64)                                                                   |
              ARGUMENTS                                                                                                                                                    |
                LIST id: 7, nodes: 1                                                                                                                                       |
                  COLUMN id: 8, column_name: mark, result_type: Nullable(UInt8), source_id: 3                                                                              |
            CONSTANT id: 9, constant_value: UInt64_2, constant_value_type: UInt8                                                                                           |
  JOIN TREE                                                                                                                                                                |
    TABLE id: 3, alias: __table1, table_name: learn_db.mart_student_lesson                                                                                                 |
  WHERE                                                                                                                                                                    |
    FUNCTION id: 10, function_name: greaterOrEquals, function_type: ordinary, result_type: UInt8                                                                           |
      ARGUMENTS                                                                                                                                                            |
        LIST id: 11, nodes: 2                                                                                                                                              |
          COLUMN id: 12, column_name: lesson_date, result_type: Date32, source_id: 3                                                                                       |
          CONSTANT id: 13, constant_value: UInt64_20364, constant_value_type: Date                                                                                         |
            EXPRESSION                                                                                                                                                     |
              FUNCTION id: 14, function_name: minus, function_type: ordinary, result_type: Date                                                                            |
                ARGUMENTS                                                                                                                                                  |
                  LIST id: 15, nodes: 2                                                                                                                                    |
                    CONSTANT id: 16, constant_value: UInt64_20374, constant_value_type: Date                                                                               |
                      EXPRESSION                                                                                                                                           |
                        FUNCTION id: 17, function_name: today, function_type: ordinary, result_type: Date                                                                  |
                    CONSTANT id: 18, constant_value: UInt64_10, constant_value_type: UInt8                                                                                 |
  GROUP BY                                                                                                                                                                 |
    LIST id: 19, nodes: 1                                                                                                                                                  |
      COLUMN id: 20, column_name: subject_name, result_type: String, source_id: 3                                                                                          |
  ORDER BY                                                                                                                                                                 |
    LIST id: 21, nodes: 1                                                                                                                                                  |
      SORT id: 22, sort_direction: DESCENDING, with_fill: 0                                                                                                                |
        EXPRESSION                                                                                                                                                         |
          FUNCTION id: 23, function_name: round, function_type: ordinary, result_type: Nullable(Float64)                                                                   |
            ARGUMENTS                                                                                                                                                      |
              LIST id: 24, nodes: 2                                                                                                                                        |
                FUNCTION id: 25, function_name: avg, function_type: aggregate, result_type: Nullable(Float64)                                                              |
                  ARGUMENTS                                                                                                                                                |
                    LIST id: 26, nodes: 1                                                                                                                                  |
                      COLUMN id: 27, column_name: mark, result_type: Nullable(UInt8), source_id: 3                                                                         |
                CONSTANT id: 28, constant_value: UInt64_2, constant_value_type: UInt8                                                                                      |
```

### 9. Получаем дерево запроса со применением 2-х оптимизаций и выводом списка примененных оптимизаций (проходов)
```sql
EXPLAIN QUERY TREE dump_passes = 1, passes = 2
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```


Результат
```text
explain                                                                                                                                                                   |
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
Pass 1 QueryAnalysis - Resolve type for each query expression. Replace identifiers, matchers with query expressions. Perform constant folding. Evaluate scalar subqueries.|
Pass 2 GroupingFunctionsResolvePass - Resolve GROUPING functions based on GROUP BY modifiers                                                                              |
                                                                                                                                                                          |
QUERY id: 0                                                                                                                                                               |
  PROJECTION COLUMNS                                                                                                                                                      |
    subject_name String                                                                                                                                                   |
    avg_mark Nullable(Float64)                                                                                                                                            |
  PROJECTION                                                                                                                                                              |
    LIST id: 1, nodes: 2                                                                                                                                                  |
      COLUMN id: 2, column_name: subject_name, result_type: String, source_id: 3                                                                                          |
      FUNCTION id: 4, function_name: round, function_type: ordinary, result_type: Nullable(Float64)                                                                       |
        ARGUMENTS                                                                                                                                                         |
          LIST id: 5, nodes: 2                                                                                                                                            |
            FUNCTION id: 6, function_name: avg, function_type: aggregate, result_type: Nullable(Float64)                                                                  |
              ARGUMENTS                                                                                                                                                   |
                LIST id: 7, nodes: 1                                                                                                                                      |
                  COLUMN id: 8, column_name: mark, result_type: Nullable(UInt8), source_id: 3                                                                             |
            CONSTANT id: 9, constant_value: UInt64_2, constant_value_type: UInt8                                                                                          |
  JOIN TREE                                                                                                                                                               |
    TABLE id: 3, alias: __table1, table_name: learn_db.mart_student_lesson                                                                                                |
  WHERE                                                                                                                                                                   |
    FUNCTION id: 10, function_name: greaterOrEquals, function_type: ordinary, result_type: UInt8                                                                          |
      ARGUMENTS                                                                                                                                                           |
        LIST id: 11, nodes: 2                                                                                                                                             |
          COLUMN id: 12, column_name: lesson_date, result_type: Date32, source_id: 3                                                                                      |
          CONSTANT id: 13, constant_value: UInt64_20364, constant_value_type: Date                                                                                        |
            EXPRESSION                                                                                                                                                    |
              FUNCTION id: 14, function_name: minus, function_type: ordinary, result_type: Date                                                                           |
                ARGUMENTS                                                                                                                                                 |
                  LIST id: 15, nodes: 2                                                                                                                                   |
                    CONSTANT id: 16, constant_value: UInt64_20374, constant_value_type: Date                                                                              |
                      EXPRESSION                                                                                                                                          |
                        FUNCTION id: 17, function_name: today, function_type: ordinary, result_type: Date                                                                 |
                    CONSTANT id: 18, constant_value: UInt64_10, constant_value_type: UInt8                                                                                |
  GROUP BY                                                                                                                                                                |
    LIST id: 19, nodes: 1                                                                                                                                                 |
      COLUMN id: 20, column_name: subject_name, result_type: String, source_id: 3                                                                                         |
  ORDER BY                                                                                                                                                                |
    LIST id: 21, nodes: 1                                                                                                                                                 |
      SORT id: 22, sort_direction: DESCENDING, with_fill: 0                                                                                                               |
        EXPRESSION                                                                                                                                                        |
          FUNCTION id: 23, function_name: round, function_type: ordinary, result_type: Nullable(Float64)                                                                  |
            ARGUMENTS                                                                                                                                                     |
              LIST id: 24, nodes: 2                                                                                                                                       |
                FUNCTION id: 25, function_name: avg, function_type: aggregate, result_type: Nullable(Float64)                                                             |
                  ARGUMENTS                                                                                                                                               |
                    LIST id: 26, nodes: 1                                                                                                                                 |
                      COLUMN id: 27, column_name: mark, result_type: Nullable(UInt8), source_id: 3                                                                        |
                CONSTANT id: 28, constant_value: UInt64_2, constant_value_type: UInt8                                                                                     |
```

### 10. Получаем AST со применением 2-х оптимизаций и выводом списка примененных оптимизаций (проходов)

```sql
EXPLAIN QUERY TREE dump_passes = 1, passes = 2, dump_tree = 0, dump_ast = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

Результат
```text
explain                                                                                                                                                                   |
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
Pass 1 QueryAnalysis - Resolve type for each query expression. Replace identifiers, matchers with query expressions. Perform constant folding. Evaluate scalar subqueries.|
Pass 2 GroupingFunctionsResolvePass - Resolve GROUPING functions based on GROUP BY modifiers                                                                              |
                                                                                                                                                                          |
SELECT                                                                                                                                                                    |
    __table1.subject_name AS subject_name,                                                                                                                                |
    round(avg(__table1.mark), 2) AS avg_mark                                                                                                                              |
FROM learn_db.mart_student_lesson AS __table1                                                                                                                             |
WHERE __table1.lesson_date >= _CAST('2025-10-03', 'Date')                                                                                                                 |
GROUP BY __table1.subject_name                                                                                                                                            |
ORDER BY round(avg(__table1.mark), 2) DESC                                                                                                                                |
```

## План выполнения запроса

### 11. Получение плана выполнения запроса

```sql
EXPLAIN PLAN -- или EXPLAIN
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

Результат
```text
explain                                                     |
------------------------------------------------------------+
Expression (Project names)                                  |
  Sorting (Sorting for ORDER BY)                            |
    Expression ((Before ORDER BY + Projection))             |
      Aggregating                                           |
        Expression (Before GROUP BY)                        |
          Expression                                        |
            ReadFromMergeTree (learn_db.mart_student_lesson)|
```

### 12. Получение плана выполнения запроса с описанием обрабатываемых колонок на каждом этапе

```sql
EXPLAIN PLAN header = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

Результат
```text
explain                                                         |
----------------------------------------------------------------+
Expression (Project names)                                      |
Header: subject_name String                                     |
        avg_mark Nullable(Float64)                              |
  Sorting (Sorting for ORDER BY)                                |
  Header: round(avg(__table1.mark), 2_UInt8) Nullable(Float64)  |
          __table1.subject_name String                          |
    Expression ((Before ORDER BY + Projection))                 |
    Header: round(avg(__table1.mark), 2_UInt8) Nullable(Float64)|
            __table1.subject_name String                        |
      Aggregating                                               |
      Header: __table1.subject_name String                      |
              avg(__table1.mark) Nullable(Float64)              |
        Expression (Before GROUP BY)                            |
        Header: __table1.subject_name String                    |
                __table1.mark Nullable(UInt8)                   |
          Expression                                            |
          Header: __table1.subject_name String                  |
                  __table1.mark Nullable(UInt8)                 |
            ReadFromMergeTree (learn_db.mart_student_lesson)    |
            Header: lesson_date Date32                          |
                    subject_name String                         |
                    mark Nullable(UInt8)                        |
```

### 13. Получение плана выполнения запроса с описанием каждого этапа

```sql
EXPLAIN PLAN actions = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

Результат
```text
explain                                                                                                                                                                                     |
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
Expression (Project names)                                                                                                                                                                  |
Actions: INPUT : 0 -> __table1.subject_name String : 0                                                                                                                                      |
         INPUT : 1 -> round(avg(__table1.mark), 2_UInt8) Nullable(Float64) : 1                                                                                                              |
         ALIAS __table1.subject_name :: 0 -> subject_name String : 2                                                                                                                        |
         ALIAS round(avg(__table1.mark), 2_UInt8) :: 1 -> avg_mark Nullable(Float64) : 0                                                                                                    |
Positions: 2 0                                                                                                                                                                              |
  Sorting (Sorting for ORDER BY)                                                                                                                                                            |
  Sort description: round(avg(__table1.mark), 2_UInt8) DESC                                                                                                                                 |
    Expression ((Before ORDER BY + Projection))                                                                                                                                             |
    Actions: INPUT : 0 -> avg(__table1.mark) Nullable(Float64) : 0                                                                                                                          |
             INPUT :: 1 -> __table1.subject_name String : 1                                                                                                                                 |
             COLUMN Const(UInt8) -> 2_UInt8 UInt8 : 2                                                                                                                                       |
             FUNCTION round(avg(__table1.mark) :: 0, 2_UInt8 :: 2) -> round(avg(__table1.mark), 2_UInt8) Nullable(Float64) : 3                                                              |
    Positions: 3 1                                                                                                                                                                          |
      Aggregating                                                                                                                                                                           |
      Keys: __table1.subject_name                                                                                                                                                           |
      Aggregates:                                                                                                                                                                           |
          avg(__table1.mark)                                                                                                                                                                |
            Function: avg(Nullable(UInt8)) → Nullable(Float64)                                                                                                                              |
            Arguments: __table1.mark                                                                                                                                                        |
      Skip merging: 0                                                                                                                                                                       |
        Expression (Before GROUP BY)                                                                                                                                                        |
        Actions: INPUT :: 0 -> __table1.subject_name String : 0                                                                                                                             |
                 INPUT :: 1 -> __table1.mark Nullable(UInt8) : 1                                                                                                                            |
        Positions: 0 1                                                                                                                                                                      |
          Expression                                                                                                                                                                        |
          Actions: INPUT : 0 -> subject_name String : 0                                                                                                                                     |
                   INPUT : 1 -> mark Nullable(UInt8) : 1                                                                                                                                    |
                   INPUT :: 2 -> lesson_date Date32 : 2                                                                                                                                     |
                   ALIAS subject_name :: 0 -> __table1.subject_name String : 3                                                                                                              |
                   ALIAS mark :: 1 -> __table1.mark Nullable(UInt8) : 0                                                                                                                     |
          Positions: 3 0                                                                                                                                                                    |
            ReadFromMergeTree (learn_db.mart_student_lesson)                                                                                                                                |
            ReadType: Default                                                                                                                                                               |
            Parts: 4                                                                                                                                                                        |
            Granules: 41                                                                                                                                                                    |
            Prewhere info                                                                                                                                                                   |
            Need filter: 1                                                                                                                                                                  |
              Prewhere filter                                                                                                                                                               |
              Prewhere filter column: greaterOrEquals(__table1.lesson_date, _CAST(20364_Date, 'Date'_String)) (removed)                                                                     |
              Actions: INPUT : 0 -> lesson_date Date32 : 0                                                                                                                                  |
                       COLUMN Const(UInt16) -> _CAST(20364_Date, 'Date'_String) Date : 1                                                                                                    |
                       FUNCTION greaterOrEquals(lesson_date : 0, _CAST(20364_Date, 'Date'_String) :: 1) -> greaterOrEquals(__table1.lesson_date, _CAST(20364_Date, 'Date'_String)) UInt8 : 2|
              Positions: 0 2                                                                                                                                                                |
```

### 14. Получение плана выполнения запроса с информацией о результатах применения индексов

```sql
EXPLAIN PLAN indexes = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

Результат
```text
explain                                                     |
------------------------------------------------------------+
Expression (Project names)                                  |
  Sorting (Sorting for ORDER BY)                            |
    Expression ((Before ORDER BY + Projection))             |
      Aggregating                                           |
        Expression (Before GROUP BY)                        |
          Expression                                        |
            ReadFromMergeTree (learn_db.mart_student_lesson)|
            Indexes:                                        |
              PrimaryKey                                    |
                Keys:                                       |
                  lesson_date                               |
                Condition: (lesson_date in [20364, +Inf))   |
                Parts: 4/4                                  |
                Granules: 41/1223                           |
```

### 15. Получение плана выполнения запроса в формате json

```sql
EXPLAIN PLAN json = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

Результат
```text
explain                                                                                                                                                                                                                                                        |
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
[¶  {¶    "Plan": {¶      "Node Type": "Expression",¶      "Node Id": "Expression_8",¶      "Description": "Project names",¶      "Plans": [¶        {¶          "Node Type": "Sorting",¶          "Node Id": "Sorting_7",¶          "Description": "Sorting fo|
```

### 16. Получение плана выполнения запроса в лаконичной форме без описаний

```sql
EXPLAIN PLAN description = 0
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

Результат
```text
explain                      |
-----------------------------+
Expression                   |
  Sorting                    |
    Expression               |
      Aggregating            |
        Expression           |
          Expression         |
            ReadFromMergeTree|
```

## PIPELINE

### 17. Получение pipeline запроса

```sql
EXPLAIN PIPELINE	
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

Результат
```text
explain                                                                         |
--------------------------------------------------------------------------------+
(Expression)                                                                    |
ExpressionTransform                                                             |
  (Sorting)                                                                     |
  MergingSortedTransform 12 → 1                                                 |
    MergeSortingTransform × 12                                                  |
      LimitsCheckingTransform × 12                                              |
        PartialSortingTransform × 12                                            |
          (Expression)                                                          |
          ExpressionTransform × 12                                              |
            (Aggregating)                                                       |
            Resize 4 → 12                                                       |
              AggregatingTransform × 4                                          |
                (Expression)                                                    |
                ExpressionTransform × 4                                         |
                  (Expression)                                                  |
                  ExpressionTransform × 4                                       |
                    (ReadFromMergeTree)                                         |
                    MergeTreeSelect(pool: ReadPool, algorithm: Thread) × 4 0 → 1|
```

### 18. Получение pipeline запроса для представления в виде графа

```sql
EXPLAIN PIPELINE graph = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

Результат
```text
explain                                                                     |
----------------------------------------------------------------------------+
digraph                                                                     |
{                                                                           |
  rankdir="LR";                                                             |
  { node [shape = rect]                                                     |
    subgraph cluster_0 {                                                    |
      label ="Aggregating";                                                 |
      style=filled;                                                         |
      color=lightgrey;                                                      |
      node [style=filled,color=white];                                      |
      { rank = same;                                                        |
        n4 [label="AggregatingTransform × 4"];                              |
        n5 [label="Resize"];                                                |
      }                                                                     |
    }                                                                       |
    subgraph cluster_1 {                                                    |
      label ="Sorting";                                                     |
      style=filled;                                                         |
      color=lightgrey;                                                      |
      node [style=filled,color=white];                                      |
      { rank = same;                                                        |
        n8 [label="LimitsCheckingTransform × 12"];                          |
        n9 [label="MergeSortingTransform × 12"];                            |
        n10 [label="MergingSortedTransform"];                               |
        n7 [label="PartialSortingTransform × 12"];                          |
      }                                                                     |
    }                                                                       |
    subgraph cluster_2 {                                                    |
      label ="ReadFromMergeTree";                                           |
      style=filled;                                                         |
      color=lightgrey;                                                      |
      node [style=filled,color=white];                                      |
      { rank = same;                                                        |
        n1 [label="MergeTreeSelect(pool: ReadPool, algorithm: Thread) × 4"];|
      }                                                                     |
    }                                                                       |
    subgraph cluster_3 {                                                    |
      label ="Expression";                                                  |
      style=filled;                                                         |
      color=lightgrey;                                                      |
      node [style=filled,color=white];                                      |
      { rank = same;                                                        |
        n3 [label="ExpressionTransform × 4"];                               |
      }                                                                     |
    }                                                                       |
    subgraph cluster_4 {                                                    |
      label ="Expression";                                                  |
      style=filled;                                                         |
      color=lightgrey;                                                      |
      node [style=filled,color=white];                                      |
      { rank = same;                                                        |
        n11 [label="ExpressionTransform"];                                  |
      }                                                                     |
    }                                                                       |
    subgraph cluster_5 {                                                    |
      label ="Expression";                                                  |
      style=filled;                                                         |
      color=lightgrey;                                                      |
      node [style=filled,color=white];                                      |
      { rank = same;                                                        |
        n6 [label="ExpressionTransform × 12"];                              |
      }                                                                     |
    }                                                                       |
    subgraph cluster_6 {                                                    |
      label ="Expression";                                                  |
      style=filled;                                                         |
      color=lightgrey;                                                      |
      node [style=filled,color=white];                                      |
      { rank = same;                                                        |
        n2 [label="ExpressionTransform × 4"];                               |
      }                                                                     |
    }                                                                       |
  }                                                                         |
  n4 -> n5 [label="× 4"];                                                   |
  n5 -> n6 [label="× 12"];                                                  |
  n8 -> n9 [label="× 12"];                                                  |
  n9 -> n10 [label="× 12"];                                                 |
  n10 -> n11 [label=""];                                                    |
  n7 -> n8 [label="× 12"];                                                  |
  n1 -> n2 [label="× 4"];                                                   |
  n3 -> n4 [label="× 4"];                                                   |
  n6 -> n7 [label="× 12"];                                                  |
  n2 -> n3 [label="× 4"];                                                   |
}                                                                           |
```

### 19. Получение pipeline запроса для представления в виде графа в расширенном формате

```sql
EXPLAIN PIPELINE graph = 1, compact = 0
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

Результат
```text
explain                                                              |
---------------------------------------------------------------------+
digraph                                                              |
{                                                                    |
  rankdir="LR";                                                      |
  { node [shape = rect]                                              |
    n0[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_1"];|
    n1[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_2"];|
    n2[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_3"];|
    n3[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_4"];|
    n4[label="ExpressionTransform_5"];                               |
    n5[label="ExpressionTransform_6"];                               |
    n6[label="ExpressionTransform_7"];                               |
    n7[label="ExpressionTransform_8"];                               |
    n8[label="ExpressionTransform_9"];                               |
    n9[label="ExpressionTransform_10"];                              |
    n10[label="ExpressionTransform_11"];                             |
    n11[label="ExpressionTransform_12"];                             |
    n12[label="AggregatingTransform_13"];                            |
    n13[label="AggregatingTransform_14"];                            |
    n14[label="AggregatingTransform_15"];                            |
    n15[label="AggregatingTransform_16"];                            |
    n16[label="Resize_17"];                                          |
    n17[label="ExpressionTransform_18"];                             |
    n18[label="ExpressionTransform_19"];                             |
    n19[label="ExpressionTransform_20"];                             |
    n20[label="ExpressionTransform_21"];                             |
    n21[label="ExpressionTransform_22"];                             |
    n22[label="ExpressionTransform_23"];                             |
    n23[label="ExpressionTransform_24"];                             |
    n24[label="ExpressionTransform_25"];                             |
    n25[label="ExpressionTransform_26"];                             |
    n26[label="ExpressionTransform_27"];                             |
    n27[label="ExpressionTransform_28"];                             |
    n28[label="ExpressionTransform_29"];                             |
    n29[label="PartialSortingTransform_30"];                         |
    n30[label="PartialSortingTransform_31"];                         |
    n31[label="PartialSortingTransform_32"];                         |
    n32[label="PartialSortingTransform_33"];                         |
    n33[label="PartialSortingTransform_34"];                         |
    n34[label="PartialSortingTransform_35"];                         |
    n35[label="PartialSortingTransform_36"];                         |
    n36[label="PartialSortingTransform_37"];                         |
    n37[label="PartialSortingTransform_38"];                         |
    n38[label="PartialSortingTransform_39"];                         |
    n39[label="PartialSortingTransform_40"];                         |
    n40[label="PartialSortingTransform_41"];                         |
    n41[label="LimitsCheckingTransform_42"];                         |
    n42[label="LimitsCheckingTransform_43"];                         |
    n43[label="LimitsCheckingTransform_44"];                         |
    n44[label="LimitsCheckingTransform_45"];                         |
    n45[label="LimitsCheckingTransform_46"];                         |
    n46[label="LimitsCheckingTransform_47"];                         |
    n47[label="LimitsCheckingTransform_48"];                         |
    n48[label="LimitsCheckingTransform_49"];                         |
    n49[label="LimitsCheckingTransform_50"];                         |
    n50[label="LimitsCheckingTransform_51"];                         |
    n51[label="LimitsCheckingTransform_52"];                         |
    n52[label="LimitsCheckingTransform_53"];                         |
    n53[label="MergeSortingTransform_54"];                           |
    n54[label="MergeSortingTransform_55"];                           |
    n55[label="MergeSortingTransform_56"];                           |
    n56[label="MergeSortingTransform_57"];                           |
    n57[label="MergeSortingTransform_58"];                           |
    n58[label="MergeSortingTransform_59"];                           |
    n59[label="MergeSortingTransform_60"];                           |
    n60[label="MergeSortingTransform_61"];                           |
    n61[label="MergeSortingTransform_62"];                           |
    n62[label="MergeSortingTransform_63"];                           |
    n63[label="MergeSortingTransform_64"];                           |
    n64[label="MergeSortingTransform_65"];                           |
    n65[label="MergingSortedTransform_66"];                          |
    n66[label="ExpressionTransform_67"];                             |
  }                                                                  |
  n0 -> n4;                                                          |
  n1 -> n5;                                                          |
  n2 -> n6;                                                          |
  n3 -> n7;                                                          |
  n4 -> n8;                                                          |
  n5 -> n9;                                                          |
  n6 -> n10;                                                         |
  n7 -> n11;                                                         |
  n8 -> n12;                                                         |
  n9 -> n13;                                                         |
  n10 -> n14;                                                        |
  n11 -> n15;                                                        |
  n12 -> n16;                                                        |
  n13 -> n16;                                                        |
  n14 -> n16;                                                        |
  n15 -> n16;                                                        |
  n16 -> n17;                                                        |
  n16 -> n18;                                                        |
  n16 -> n19;                                                        |
  n16 -> n20;                                                        |
  n16 -> n21;                                                        |
  n16 -> n22;                                                        |
  n16 -> n23;                                                        |
  n16 -> n24;                                                        |
  n16 -> n25;                                                        |
  n16 -> n26;                                                        |
  n16 -> n27;                                                        |
  n16 -> n28;                                                        |
  n17 -> n29;                                                        |
  n18 -> n30;                                                        |
  n19 -> n31;                                                        |
  n20 -> n32;                                                        |
  n21 -> n33;                                                        |
  n22 -> n34;                                                        |
  n23 -> n35;                                                        |
  n24 -> n36;                                                        |
  n25 -> n37;                                                        |
  n26 -> n38;                                                        |
  n27 -> n39;                                                        |
  n28 -> n40;                                                        |
  n29 -> n41;                                                        |
  n30 -> n42;                                                        |
  n31 -> n43;                                                        |
  n32 -> n44;                                                        |
  n33 -> n45;                                                        |
  n34 -> n46;                                                        |
  n35 -> n47;                                                        |
  n36 -> n48;                                                        |
  n37 -> n49;                                                        |
  n38 -> n50;                                                        |
  n39 -> n51;                                                        |
  n40 -> n52;                                                        |
  n41 -> n53;                                                        |
  n42 -> n54;                                                        |
  n43 -> n55;                                                        |
  n44 -> n56;                                                        |
  n45 -> n57;                                                        |
  n46 -> n58;                                                        |
  n47 -> n59;                                                        |
  n48 -> n60;                                                        |
  n49 -> n61;                                                        |
  n50 -> n62;                                                        |
  n51 -> n63;                                                        |
  n52 -> n64;                                                        |
  n53 -> n65;                                                        |
  n54 -> n65;                                                        |
  n55 -> n65;                                                        |
  n56 -> n65;                                                        |
  n57 -> n65;                                                        |
  n58 -> n65;                                                        |
  n59 -> n65;                                                        |
  n60 -> n65;                                                        |
  n61 -> n65;                                                        |
  n62 -> n65;                                                        |
  n63 -> n65;                                                        |
  n64 -> n65;                                                        |
  n65 -> n66;                                                        |
}                                                                    |
```

## Время выполнения этапов запроса

### 20. Выполняем запрос

```sql
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

Результат
```text
Query id: dabed898-4e56-4159-82a3-8f80d4a34b81

   ┌─subject_name────────┬─avg_mark─┐
1. │ Математика          │     4.35 │
2. │ Русский язык        │     4.26 │
3. │ Литература          │     4.15 │
4. │ Физика              │        4 │
5. │ Химия               │     3.85 │
6. │ География           │     3.76 │
7. │ Биология            │     3.64 │
8. │ Информатика         │      3.6 │
9. │ Физическая культура │     3.56 │
   └─────────────────────┴──────────┘

9 rows in set. Elapsed: 0.030 sec. Processed 317.06 thousand rows, 11.31 MB (10.72 million rows/s., 382.22 MB/s.)
Peak memory usage: 12.95 MiB.
```

### 21. Находим запись о выполнении запроса в query_log и берем его query_id

```sql
SELECT * FROM system.query_log ORDER BY event_time DESC;
```

Результат
```text
Query id: dabed898-4e56-4159-82a3-8f80d4a34b81
```

### 22. Подставляем query_id в запрос и получаем время выполнения и другую информацию по каждому этапу запроса

```sql
SELECT * FROM system.processors_profile_log WHERE query_id = 'dabed898-4e56-4159-82a3-8f80d4a34b81';
```

Результат
```text
hostname    |event_date|event_time         |event_time_microseconds      |id             |parent_ids                                                                                                                                                                                       |plan_step      |plan_step_name   |plan_step_description         |plan_group|initial_query_id                    |query_id                            |name                                              |elapsed_us|input_wait_elapsed_us|output_wait_elapsed_us|input_rows|input_bytes|output_rows|output_bytes|processor_uniq_id                                   |step_uniq_id       |
------------+----------+-------------------+-----------------------------+---------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+-----------------+------------------------------+----------+------------------------------------+------------------------------------+--------------------------------------------------+----------+---------------------+----------------------+----------+-----------+-----------+------------+----------------------------------------------------+-------------------+
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140580176279064|[140581919168024]                                                                                                                                                                                |140583421778944|ReadFromMergeTree|learn_db.mart_student_lesson  |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|      6601|                    0|                  1351|         0|          0|      33056|     1180268|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_1|ReadFromMergeTree_0|
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140580176280600|[140581919168536]                                                                                                                                                                                |140583421778944|ReadFromMergeTree|learn_db.mart_student_lesson  |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|     16558|                    0|                  6438|         0|          0|     197819|     7052486|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_2|ReadFromMergeTree_0|
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140580176281112|[140581919169048]                                                                                                                                                                                |140583421778944|ReadFromMergeTree|learn_db.mart_student_lesson  |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|      7382|                    0|                  1238|         0|          0|      32592|     1161958|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_3|ReadFromMergeTree_0|
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581919167512|[140581922656280]                                                                                                                                                                                |140583421778944|ReadFromMergeTree|learn_db.mart_student_lesson  |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|      6963|                    0|                  1264|         0|          0|      32703|     1166661|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_4|ReadFromMergeTree_0|
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581919168024|[140580725386264]                                                                                                                                                                                |140580164215040|Expression       |                              |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |        35|                 6870|                  1289|     33056|    1180268|      33056|     1048044|ExpressionTransform_5                               |Expression_11      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581919168536|[140580725386776]                                                                                                                                                                                |140580164215040|Expression       |                              |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |        97|                16865|                  6268|    197819|    7052486|     197819|     6261210|ExpressionTransform_6                               |Expression_11      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581919169048|[140581922054168]                                                                                                                                                                                |140580164215040|Expression       |                              |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |        16|                 7654|                  1206|     32592|    1161958|      32592|     1031590|ExpressionTransform_7                               |Expression_11      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922656280|[140581922054680]                                                                                                                                                                                |140580164215040|Expression       |                              |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |        23|                 7232|                  1220|     32703|    1166661|      32703|     1035849|ExpressionTransform_8                               |Expression_11      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140580725386264|[140582221975448]                                                                                                                                                                                |140580176648960|Expression       |Before GROUP BY               |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |        12|                 6926|                  1262|     33056|    1048044|      33056|     1048044|ExpressionTransform_9                               |Expression_3       |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140580725386776|[140582221976088]                                                                                                                                                                                |140580176648960|Expression       |Before GROUP BY               |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |        30|                17022|                  6198|    197819|    6261210|     197819|     6261210|ExpressionTransform_10                              |Expression_3       |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922054168|[140582221976728]                                                                                                                                                                                |140580176648960|Expression       |Before GROUP BY               |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |         4|                 7686|                  1195|     32592|    1031590|      32592|     1031590|ExpressionTransform_11                              |Expression_3       |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922054680|[140582221977368]                                                                                                                                                                                |140580176648960|Expression       |Before GROUP BY               |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |         8|                 7274|                  1203|     32703|    1035849|      32703|     1035849|ExpressionTransform_12                              |Expression_3       |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140582221975448|[140581922055192]                                                                                                                                                                                |140581445602816|Aggregating      |                              |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|AggregatingTransform                              |      1401|                 6957|                     0|     33056|    1048044|          0|           0|AggregatingTransform_13                             |Aggregating_4      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140582221976088|[140581922055192]                                                                                                                                                                                |140581445602816|Aggregating      |                              |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|AggregatingTransform                              |      6224|                17226|                    15|    197828|    6261550|          9|         340|AggregatingTransform_14                             |Aggregating_4      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140582221976728|[140581922055192]                                                                                                                                                                                |140581445602816|Aggregating      |                              |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|AggregatingTransform                              |      1200|                 7698|                     0|     32592|    1031590|          0|           0|AggregatingTransform_15                             |Aggregating_4      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140582221977368|[140581922055192]                                                                                                                                                                                |140581445602816|Aggregating      |                              |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|AggregatingTransform                              |      1207|                 7293|                     0|     32703|    1035849|          0|           0|AggregatingTransform_16                             |Aggregating_4      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922055192|[140581922056216,140581922056728,140581922057240,140581922057752,140581922058264,140581922058776,140581922059288,140581922059800,140581922060312,140581922060824,140581922061336,140581922061848]|140581445602816|Aggregating      |                              |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|Resize                                            |         0|                23543|                     0|         9|        340|          9|         340|Resize_17                                           |Aggregating_4      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922056216|[140583417066008]                                                                                                                                                                                |140580176649728|Expression       |(Before ORDER BY + Projection)|         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |        25|                23566|                    40|         9|        340|          9|         340|ExpressionTransform_18                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922056728|[140583417636888]                                                                                                                                                                                |140580176649728|Expression       |(Before ORDER BY + Projection)|         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |         0|                23503|                     0|         0|          0|          0|           0|ExpressionTransform_19                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922057240|[140583420820504]                                                                                                                                                                                |140580176649728|Expression       |(Before ORDER BY + Projection)|         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |         0|                23489|                     0|         0|          0|          0|           0|ExpressionTransform_20                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922057752|[140583420821272]                                                                                                                                                                                |140580176649728|Expression       |(Before ORDER BY + Projection)|         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |         0|                23490|                     0|         0|          0|          0|           0|ExpressionTransform_21                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922058264|[140583420823576]                                                                                                                                                                                |140580176649728|Expression       |(Before ORDER BY + Projection)|         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |         0|                23491|                     0|         0|          0|          0|           0|ExpressionTransform_22                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922058776|[140583420824344]                                                                                                                                                                                |140580176649728|Expression       |(Before ORDER BY + Projection)|         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |         0|                23491|                     0|         0|          0|          0|           0|ExpressionTransform_23                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922059288|[140583420825112]                                                                                                                                                                                |140580176649728|Expression       |(Before ORDER BY + Projection)|         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |         0|                23490|                     0|         0|          0|          0|           0|ExpressionTransform_24                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922059800|[140583420825880]                                                                                                                                                                                |140580176649728|Expression       |(Before ORDER BY + Projection)|         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |         0|                23491|                     0|         0|          0|          0|           0|ExpressionTransform_25                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922060312|[140583420826648]                                                                                                                                                                                |140580176649728|Expression       |(Before ORDER BY + Projection)|         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |         0|                23493|                     0|         0|          0|          0|           0|ExpressionTransform_26                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922060824|[140583420827416]                                                                                                                                                                                |140580176649728|Expression       |(Before ORDER BY + Projection)|         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |         0|                23494|                     0|         0|          0|          0|           0|ExpressionTransform_27                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922061336|[140583420828184]                                                                                                                                                                                |140580176649728|Expression       |(Before ORDER BY + Projection)|         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |         0|                23494|                     0|         0|          0|          0|           0|ExpressionTransform_28                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922061848|[140583420828952]                                                                                                                                                                                |140580176649728|Expression       |(Before ORDER BY + Projection)|         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |         0|                23495|                     0|         0|          0|          0|           0|ExpressionTransform_29                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583417066008|[140583883980184]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|PartialSortingTransform                           |        23|                23700|                    12|         9|        340|          9|         340|PartialSortingTransform_30                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583417636888|[140583885111320]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|PartialSortingTransform                           |         0|                23505|                     0|         0|          0|          0|           0|PartialSortingTransform_31                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583420820504|[140583885111960]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|PartialSortingTransform                           |         0|                23491|                     0|         0|          0|          0|           0|PartialSortingTransform_32                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583420821272|[140583885112600]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|PartialSortingTransform                           |         0|                23492|                     0|         0|          0|          0|           0|PartialSortingTransform_33                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583420823576|[140583885113240]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|PartialSortingTransform                           |         0|                23493|                     0|         0|          0|          0|           0|PartialSortingTransform_34                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583420824344|[140583885113880]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|PartialSortingTransform                           |         0|                23493|                     0|         0|          0|          0|           0|PartialSortingTransform_35                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583420825112|[140583885114520]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|PartialSortingTransform                           |         0|                23492|                     0|         0|          0|          0|           0|PartialSortingTransform_36                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583420825880|[140583885115160]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|PartialSortingTransform                           |         0|                23493|                     0|         0|          0|          0|           0|PartialSortingTransform_37                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583420826648|[140583885115800]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|PartialSortingTransform                           |         0|                23495|                     0|         0|          0|          0|           0|PartialSortingTransform_38                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583420827416|[140583885116440]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|PartialSortingTransform                           |         0|                23496|                     0|         0|          0|          0|           0|PartialSortingTransform_39                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583420828184|[140583885117080]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|PartialSortingTransform                           |         0|                23496|                     0|         0|          0|          0|           0|PartialSortingTransform_40                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583420828952|[140583885117720]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|PartialSortingTransform                           |         0|                23497|                     0|         0|          0|          0|           0|PartialSortingTransform_41                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583883980184|[140581917478936]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|LimitsCheckingTransform                           |         2|                23728|                     6|         9|        340|          9|         340|LimitsCheckingTransform_42                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583885111320|[140581917479704]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|LimitsCheckingTransform                           |         0|                23507|                     0|         0|          0|          0|           0|LimitsCheckingTransform_43                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583885111960|[140581917480472]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|LimitsCheckingTransform                           |         0|                23494|                     0|         0|          0|          0|           0|LimitsCheckingTransform_44                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583885112600|[140581917481240]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|LimitsCheckingTransform                           |         0|                23494|                     0|         0|          0|          0|           0|LimitsCheckingTransform_45                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583885113240|[140581917482008]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|LimitsCheckingTransform                           |         0|                23495|                     0|         0|          0|          0|           0|LimitsCheckingTransform_46                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583885113880|[140581917482776]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|LimitsCheckingTransform                           |         0|                23494|                     0|         0|          0|          0|           0|LimitsCheckingTransform_47                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583885114520|[140581917483544]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|LimitsCheckingTransform                           |         0|                23493|                     0|         0|          0|          0|           0|LimitsCheckingTransform_48                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583885115160|[140581917484312]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|LimitsCheckingTransform                           |         0|                23494|                     0|         0|          0|          0|           0|LimitsCheckingTransform_49                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583885115800|[140581917485848]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|LimitsCheckingTransform                           |         0|                23496|                     0|         0|          0|          0|           0|LimitsCheckingTransform_50                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583885116440|[140581917486616]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|LimitsCheckingTransform                           |         0|                23497|                     0|         0|          0|          0|           0|LimitsCheckingTransform_51                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583885117080|[140581917487384]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|LimitsCheckingTransform                           |         0|                23497|                     0|         0|          0|          0|           0|LimitsCheckingTransform_52                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583885117720|[140581917488152]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|LimitsCheckingTransform                           |         0|                23499|                     0|         0|          0|          0|           0|LimitsCheckingTransform_53                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581917478936|[140583883297816]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeSortingTransform                             |        16|                23736|                     6|         9|        340|          9|         340|MergeSortingTransform_54                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581917479704|[140583883297816]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeSortingTransform                             |        10|                23510|                     0|         0|          0|          0|           0|MergeSortingTransform_55                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581917480472|[140583883297816]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeSortingTransform                             |         4|                23511|                     0|         0|          0|          0|           0|MergeSortingTransform_56                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581917481240|[140583883297816]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeSortingTransform                             |         2|                23495|                     0|         0|          0|          0|           0|MergeSortingTransform_57                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581917482008|[140583883297816]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeSortingTransform                             |        20|                23496|                     0|         0|          0|          0|           0|MergeSortingTransform_58                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581917482776|[140583883297816]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeSortingTransform                             |         5|                23496|                     0|         0|          0|          0|           0|MergeSortingTransform_59                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581917483544|[140583883297816]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeSortingTransform                             |         3|                23494|                     0|         0|          0|          0|           0|MergeSortingTransform_60                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581917484312|[140583883297816]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeSortingTransform                             |        10|                23496|                     0|         0|          0|          0|           0|MergeSortingTransform_61                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581917485848|[140583883297816]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeSortingTransform                             |         9|                23498|                     0|         0|          0|          0|           0|MergeSortingTransform_62                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581917486616|[140583883297816]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeSortingTransform                             |         4|                23498|                     0|         0|          0|          0|           0|MergeSortingTransform_63                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581917487384|[140583883297816]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeSortingTransform                             |         3|                23498|                     0|         0|          0|          0|           0|MergeSortingTransform_64                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581917488152|[140583883297816]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergeSortingTransform                             |         3|                23500|                     0|         0|          0|          0|           0|MergeSortingTransform_65                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583883297816|[140581922055704]                                                                                                                                                                                |140580176279552|Sorting          |Sorting for ORDER BY          |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|MergingSortedTransform                            |        26|                23894|                    62|         9|        340|          9|         340|MergingSortedTransform_66                           |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140581922055704|[140583885118360]                                                                                                                                                                                |140580176649472|Expression       |Project names                 |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ExpressionTransform                               |        13|                23938|                    38|         9|        340|          9|         340|ExpressionTransform_67                              |Expression_8       |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140583885118360|[140580175841816]                                                                                                                                                                                |              0|                 |                              |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|LimitsCheckingTransform                           |         2|                23988|                    31|         9|        340|          9|         340|LimitsCheckingTransform_68                          |                   |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140580176279576|[140580175841816]                                                                                                                                                                                |              0|                 |                              |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|NullSource                                        |         1|                    0|                     0|         0|          0|          0|           0|NullSource_70                                       |                   |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140580176278040|[140580175841816]                                                                                                                                                                                |              0|                 |                              |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|NullSource                                        |         0|                    0|                     0|         0|          0|          0|           0|NullSource_71                                       |                   |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140580175841816|[]                                                                                                                                                                                               |              0|                 |                              |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|LazyOutputFormat                                  |        27|                24013|                     0|         9|        340|          0|           0|LazyOutputFormat_69                                 |                   |
19294dc11be6|2025-10-13|2025-10-13 22:22:35|2025-10-13 22:22:35.440830000|140588443545752|[140582221976088]                                                                                                                                                                                |              0|                 |                              |         0|dabed898-4e56-4159-82a3-8f80d4a34b81|dabed898-4e56-4159-82a3-8f80d4a34b81|ConvertingAggregatedToChunksTransform             |        87|                    0|                     0|         0|          0|          9|         340|ConvertingAggregatedToChunksTransform_72            |                   |
```

## Пример оптимизации

### 23. Оригинальный запрос

```sql
SELECT 
	educational_organization_id,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	subject_id = 3
GROUP BY
	educational_organization_id
ORDER BY 
	avg_mark desc;
```

Результат

```text
Query id: c455e2c6-107f-46b6-a2bc-abdb65f8b82c

    ┌─educational_organization_id─┬─avg_mark─┐
 1. │                          27 │     4.16 │
 2. │                           8 │     4.16 │
 3. │                          25 │     4.16 │
 4. │                           0 │     4.16 │
 5. │                           9 │     4.16 │
 6. │                           5 │     4.16 │
 7. │                           7 │     4.16 │
 8. │                           1 │     4.15 │
 9. │                           6 │     4.15 │
10. │                          10 │     4.15 │
11. │                          11 │     4.15 │
12. │                          15 │     4.15 │
13. │                          16 │     4.15 │
14. │                          18 │     4.15 │
15. │                          19 │     4.15 │
16. │                          20 │     4.15 │
17. │                          21 │     4.15 │
18. │                          14 │     4.15 │
19. │                          23 │     4.15 │
20. │                          24 │     4.15 │
21. │                           2 │     4.15 │
22. │                          22 │     4.15 │
23. │                           4 │     4.14 │
24. │                          13 │     4.14 │
25. │                          17 │     4.14 │
26. │                          26 │     4.14 │
27. │                          12 │     4.13 │
28. │                           3 │     4.13 │
    └─────────────────────────────┴──────────┘

28 rows in set. Elapsed: 0.086 sec. Processed 10.00 million rows, 60.00 MB (116.08 million rows/s., 696.50 MB/s.)
Peak memory usage: 5.53 MiB.
```

### 24. Проверяем эффективность применения индексов
```
EXPLAIN indexes = 1
SELECT 
	educational_organization_id,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	subject_id = 3
GROUP BY
	educational_organization_id
ORDER BY 
	avg_mark desc;
```

Результат

```text
explain                                                     |
------------------------------------------------------------+
Expression (Project names)                                  |
  Sorting (Sorting for ORDER BY)                            |
    Expression ((Before ORDER BY + Projection))             |
      Aggregating                                           |
        Expression (Before GROUP BY)                        |
          Expression                                        |
            ReadFromMergeTree (learn_db.mart_student_lesson)|
            Indexes:                                        |
              PrimaryKey                                    |
                Condition: true                             |
                Parts: 4/4                                  |
                Granules: 1223/1223                         |
```

```sql
explain indexes = 1
SELECT * FROM git.file_changes WHERE commit_message LIKE '%MYSQL%';
```

Результат
```text
explain                                      |
---------------------------------------------+
Expression ((Project names + Projection))    |
  Expression                                 |
    ReadFromMergeTree (git.file_changes)     |
    Indexes:                                 |
      PrimaryKey                             |
        Condition: true                      |
        Parts: 1/1                           |
        Granules: 33/33                      |
      Skip                                   |
        Name: commit_message_tokenbf_v1_index|
        Description: ngrambf_v1 GRANULARITY 1|
        Parts: 1/1                           |
        Granules: 4/33                       |
```

### 25. Смотрим, какие этапы выполнения запроса заняли больше всего времени

```sql
SELECT 
	educational_organization_id,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	subject_id = 3
GROUP BY
	educational_organization_id
ORDER BY 
	avg_mark desc;
```

Результат
```text
Query id: 67b939e6-8dbf-496c-85a0-167d3ce8b055

    ┌─educational_organization_id─┬─avg_mark─┐
 1. │                          27 │     4.16 │
 2. │                           8 │     4.16 │
 3. │                          25 │     4.16 │
 4. │                           0 │     4.16 │
 5. │                           9 │     4.16 │
 6. │                           5 │     4.16 │
 7. │                           7 │     4.16 │
 8. │                           1 │     4.15 │
 9. │                           6 │     4.15 │
10. │                          10 │     4.15 │
11. │                          11 │     4.15 │
12. │                          15 │     4.15 │
13. │                          16 │     4.15 │
14. │                          18 │     4.15 │
15. │                          19 │     4.15 │
16. │                          20 │     4.15 │
17. │                          21 │     4.15 │
18. │                          14 │     4.15 │
19. │                          23 │     4.15 │
20. │                          24 │     4.15 │
21. │                           2 │     4.15 │
22. │                          22 │     4.15 │
23. │                           4 │     4.14 │
24. │                          13 │     4.14 │
25. │                          17 │     4.14 │
26. │                          26 │     4.14 │
27. │                          12 │     4.13 │
28. │                           3 │     4.13 │
    └─────────────────────────────┴──────────┘

28 rows in set. Elapsed: 0.075 sec. Processed 10.00 million rows, 60.00 MB (132.86 million rows/s., 797.13 MB/s.)
Peak memory usage: 4.51 MiB.
```

```sql
SELECT * FROM system.query_log ORDER BY event_time DESC;
```

Результат
```text
Query id: f15406f7-a90e-49ca-8292-cd6727c3ca43
```

```sql
SELECT * FROM system.processors_profile_log WHERE query_id = 'f15406f7-a90e-49ca-8292-cd6727c3ca43' order by processor_uniq_id;
```

Результат
```text
hostname    |event_date|event_time         |event_time_microseconds      |id             |parent_ids       |plan_step|plan_step_name|plan_step_description|plan_group|initial_query_id                    |query_id                            |name                          |elapsed_us|input_wait_elapsed_us|output_wait_elapsed_us|input_rows|input_bytes|output_rows|output_bytes|processor_uniq_id               |step_uniq_id|
------------+----------+-------------------+-----------------------------+---------------+-----------------+---------+--------------+---------------------+----------+------------------------------------+------------------------------------+------------------------------+----------+---------------------+----------------------+----------+-----------+-----------+------------+--------------------------------+------------+
19294dc11be6|2025-10-13|2025-10-13 22:31:05|2025-10-13 22:31:05.397333000|140582768184472|[140582223126168]|        0|              |                     |         0|f15406f7-a90e-49ca-8292-cd6727c3ca43|f15406f7-a90e-49ca-8292-cd6727c3ca43|LimitsCheckingTransform       |         2|                   33|                    61|        12|        482|         12|         482|LimitsCheckingTransform_2       |            |
19294dc11be6|2025-10-13|2025-10-13 22:31:05|2025-10-13 22:31:05.397333000|140582223126168|[140580175962136]|        0|              |                     |         0|f15406f7-a90e-49ca-8292-cd6727c3ca43|f15406f7-a90e-49ca-8292-cd6727c3ca43|MaterializingTransform        |         0|                   40|                    59|        12|        482|         12|         482|MaterializingTransform_6        |            |
19294dc11be6|2025-10-13|2025-10-13 22:31:05|2025-10-13 22:31:05.397333000|140581942232088|[140580175962136]|        0|              |                     |         0|f15406f7-a90e-49ca-8292-cd6727c3ca43|f15406f7-a90e-49ca-8292-cd6727c3ca43|NullSource                    |         0|                    0|                     0|         0|          0|          0|           0|NullSource_7                    |            |
19294dc11be6|2025-10-13|2025-10-13 22:31:05|2025-10-13 22:31:05.397333000|140581942231576|[140580175962136]|        0|              |                     |         0|f15406f7-a90e-49ca-8292-cd6727c3ca43|f15406f7-a90e-49ca-8292-cd6727c3ca43|NullSource                    |         0|                    0|                     0|         0|          0|          0|           0|NullSource_8                    |            |
19294dc11be6|2025-10-13|2025-10-13 22:31:05|2025-10-13 22:31:05.397333000|140580175962136|[]               |        0|              |                     |         0|f15406f7-a90e-49ca-8292-cd6727c3ca43|f15406f7-a90e-49ca-8292-cd6727c3ca43|ParallelFormattingOutputFormat|       421|                   49|                     0|        12|        482|          0|           0|ParallelFormattingOutputFormat_3|            |
19294dc11be6|2025-10-13|2025-10-13 22:31:05|2025-10-13 22:31:05.397333000|140582221975448|[140582768184472]|        0|              |                     |         0|f15406f7-a90e-49ca-8292-cd6727c3ca43|f15406f7-a90e-49ca-8292-cd6727c3ca43|SourceFromSingleChunk         |        20|                    0|                    68|         0|          0|         12|         482|SourceFromSingleChunk_1         |            |
```

### 26. Получаем граф конвеера запроса.

```sql
EXPLAIN PIPELINE graph = 1, compact = 0
SELECT 
	educational_organization_id,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	subject_id = 3
GROUP BY
	educational_organization_id
ORDER BY 
	avg_mark desc; system.processors_profile_log WHERE query_id = 'f15406f7-a90e-49ca-8292-cd6727c3ca43' order by processor_uniq_id;
```

Результат
```text
explain                                                                |
-----------------------------------------------------------------------+
digraph                                                                |
{                                                                      |
  rankdir="LR";                                                        |
  { node [shape = rect]                                                |
    n0[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_1"];  |
    n1[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_2"];  |
    n2[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_3"];  |
    n3[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_4"];  |
    n4[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_5"];  |
    n5[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_6"];  |
    n6[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_7"];  |
    n7[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_8"];  |
    n8[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_9"];  |
    n9[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_10"]; |
    n10[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_11"];|
    n11[label="MergeTreeSelect(pool: ReadPool, algorithm: Thread)_12"];|
    n12[label="ExpressionTransform_13"];                               |
    n13[label="ExpressionTransform_14"];                               |
    n14[label="ExpressionTransform_15"];                               |
    n15[label="ExpressionTransform_16"];                               |
    n16[label="ExpressionTransform_17"];                               |
    n17[label="ExpressionTransform_18"];                               |
    n18[label="ExpressionTransform_19"];                               |
    n19[label="ExpressionTransform_20"];                               |
    n20[label="ExpressionTransform_21"];                               |
    n21[label="ExpressionTransform_22"];                               |
    n22[label="ExpressionTransform_23"];                               |
    n23[label="ExpressionTransform_24"];                               |
    n24[label="ExpressionTransform_25"];                               |
    n25[label="ExpressionTransform_26"];                               |
    n26[label="ExpressionTransform_27"];                               |
    n27[label="ExpressionTransform_28"];                               |
    n28[label="ExpressionTransform_29"];                               |
    n29[label="ExpressionTransform_30"];                               |
    n30[label="ExpressionTransform_31"];                               |
    n31[label="ExpressionTransform_32"];                               |
    n32[label="ExpressionTransform_33"];                               |
    n33[label="ExpressionTransform_34"];                               |
    n34[label="ExpressionTransform_35"];                               |
    n35[label="ExpressionTransform_36"];                               |
    n36[label="AggregatingTransform_37"];                              |
    n37[label="AggregatingTransform_38"];                              |
    n38[label="AggregatingTransform_39"];                              |
    n39[label="AggregatingTransform_40"];                              |
    n40[label="AggregatingTransform_41"];                              |
    n41[label="AggregatingTransform_42"];                              |
    n42[label="AggregatingTransform_43"];                              |
    n43[label="AggregatingTransform_44"];                              |
    n44[label="AggregatingTransform_45"];                              |
    n45[label="AggregatingTransform_46"];                              |
    n46[label="AggregatingTransform_47"];                              |
    n47[label="AggregatingTransform_48"];                              |
    n48[label="Resize_49"];                                            |
    n49[label="ExpressionTransform_50"];                               |
    n50[label="ExpressionTransform_51"];                               |
    n51[label="ExpressionTransform_52"];                               |
    n52[label="ExpressionTransform_53"];                               |
    n53[label="ExpressionTransform_54"];                               |
    n54[label="ExpressionTransform_55"];                               |
    n55[label="ExpressionTransform_56"];                               |
    n56[label="ExpressionTransform_57"];                               |
    n57[label="ExpressionTransform_58"];                               |
    n58[label="ExpressionTransform_59"];                               |
    n59[label="ExpressionTransform_60"];                               |
    n60[label="ExpressionTransform_61"];                               |
    n61[label="PartialSortingTransform_62"];                           |
    n62[label="PartialSortingTransform_63"];                           |
    n63[label="PartialSortingTransform_64"];                           |
    n64[label="PartialSortingTransform_65"];                           |
    n65[label="PartialSortingTransform_66"];                           |
    n66[label="PartialSortingTransform_67"];                           |
    n67[label="PartialSortingTransform_68"];                           |
    n68[label="PartialSortingTransform_69"];                           |
    n69[label="PartialSortingTransform_70"];                           |
    n70[label="PartialSortingTransform_71"];                           |
    n71[label="PartialSortingTransform_72"];                           |
    n72[label="PartialSortingTransform_73"];                           |
    n73[label="LimitsCheckingTransform_74"];                           |
    n74[label="LimitsCheckingTransform_75"];                           |
    n75[label="LimitsCheckingTransform_76"];                           |
    n76[label="LimitsCheckingTransform_77"];                           |
    n77[label="LimitsCheckingTransform_78"];                           |
    n78[label="LimitsCheckingTransform_79"];                           |
    n79[label="LimitsCheckingTransform_80"];                           |
    n80[label="LimitsCheckingTransform_81"];                           |
    n81[label="LimitsCheckingTransform_82"];                           |
    n82[label="LimitsCheckingTransform_83"];                           |
    n83[label="LimitsCheckingTransform_84"];                           |
    n84[label="LimitsCheckingTransform_85"];                           |
    n85[label="MergeSortingTransform_86"];                             |
    n86[label="MergeSortingTransform_87"];                             |
    n87[label="MergeSortingTransform_88"];                             |
    n88[label="MergeSortingTransform_89"];                             |
    n89[label="MergeSortingTransform_90"];                             |
    n90[label="MergeSortingTransform_91"];                             |
    n91[label="MergeSortingTransform_92"];                             |
    n92[label="MergeSortingTransform_93"];                             |
    n93[label="MergeSortingTransform_94"];                             |
    n94[label="MergeSortingTransform_95"];                             |
    n95[label="MergeSortingTransform_96"];                             |
    n96[label="MergeSortingTransform_97"];                             |
    n97[label="MergingSortedTransform_98"];                            |
    n98[label="ExpressionTransform_99"];                               |
  }                                                                    |
  n0 -> n12;                                                           |
  n1 -> n13;                                                           |
  n2 -> n14;                                                           |
  n3 -> n15;                                                           |
  n4 -> n16;                                                           |
  n5 -> n17;                                                           |
  n6 -> n18;                                                           |
  n7 -> n19;                                                           |
  n8 -> n20;                                                           |
  n9 -> n21;                                                           |
  n10 -> n22;                                                          |
  n11 -> n23;                                                          |
  n12 -> n24;                                                          |
  n13 -> n25;                                                          |
  n14 -> n26;                                                          |
  n15 -> n27;                                                          |
  n16 -> n28;                                                          |
  n17 -> n29;                                                          |
  n18 -> n30;                                                          |
  n19 -> n31;                                                          |
  n20 -> n32;                                                          |
  n21 -> n33;                                                          |
  n22 -> n34;                                                          |
  n23 -> n35;                                                          |
  n24 -> n36;                                                          |
  n25 -> n37;                                                          |
  n26 -> n38;                                                          |
  n27 -> n39;                                                          |
  n28 -> n40;                                                          |
  n29 -> n41;                                                          |
  n30 -> n42;                                                          |
  n31 -> n43;                                                          |
  n32 -> n44;                                                          |
  n33 -> n45;                                                          |
  n34 -> n46;                                                          |
  n35 -> n47;                                                          |
  n36 -> n48;                                                          |
  n37 -> n48;                                                          |
  n38 -> n48;                                                          |
  n39 -> n48;                                                          |
  n40 -> n48;                                                          |
  n41 -> n48;                                                          |
  n42 -> n48;                                                          |
  n43 -> n48;                                                          |
  n44 -> n48;                                                          |
  n45 -> n48;                                                          |
  n46 -> n48;                                                          |
  n47 -> n48;                                                          |
  n48 -> n49;                                                          |
  n48 -> n50;                                                          |
  n48 -> n51;                                                          |
  n48 -> n52;                                                          |
  n48 -> n53;                                                          |
  n48 -> n54;                                                          |
  n48 -> n55;                                                          |
  n48 -> n56;                                                          |
  n48 -> n57;                                                          |
  n48 -> n58;                                                          |
  n48 -> n59;                                                          |
  n48 -> n60;                                                          |
  n49 -> n61;                                                          |
  n50 -> n62;                                                          |
  n51 -> n63;                                                          |
  n52 -> n64;                                                          |
  n53 -> n65;                                                          |
  n54 -> n66;                                                          |
  n55 -> n67;                                                          |
  n56 -> n68;                                                          |
  n57 -> n69;                                                          |
  n58 -> n70;                                                          |
  n59 -> n71;                                                          |
  n60 -> n72;                                                          |
  n61 -> n73;                                                          |
  n62 -> n74;                                                          |
  n63 -> n75;                                                          |
  n64 -> n76;                                                          |
  n65 -> n77;                                                          |
  n66 -> n78;                                                          |
  n67 -> n79;                                                          |
  n68 -> n80;                                                          |
  n69 -> n81;                                                          |
  n70 -> n82;                                                          |
  n71 -> n83;                                                          |
  n72 -> n84;                                                          |
  n73 -> n85;                                                          |
  n74 -> n86;                                                          |
  n75 -> n87;                                                          |
  n76 -> n88;                                                          |
  n77 -> n89;                                                          |
  n78 -> n90;                                                          |
  n79 -> n91;                                                          |
  n80 -> n92;                                                          |
  n81 -> n93;                                                          |
  n82 -> n94;                                                          |
  n83 -> n95;                                                          |
  n84 -> n96;                                                          |
  n85 -> n97;                                                          |
  n86 -> n97;                                                          |
  n87 -> n97;                                                          |
  n88 -> n97;                                                          |
  n89 -> n97;                                                          |
  n90 -> n97;                                                          |
  n91 -> n97;                                                          |
  n92 -> n97;                                                          |
  n93 -> n97;                                                          |
  n94 -> n97;                                                          |
  n95 -> n97;                                                          |
  n96 -> n97;                                                          |
  n97 -> n98;                                                          |
}                                                                      |
```

### 27. Для оптимизации пересоздаем таблицу с другим первичным индексом

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
	PRIMARY KEY(subject_id, lesson_date)
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

### 28. Смотрим, как поменялось время выполнения этапов запроса

```sql
SELECT 
	educational_organization_id,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	subject_id = 3
GROUP BY
	educational_organization_id
ORDER BY 
	avg_mark desc;
```

Результат
```text
Query id: 0f5388a4-da94-46d5-8685-588f70119099

    ┌─educational_organization_id─┬─avg_mark─┐
 1. │                           0 │     4.16 │
 2. │                          14 │     4.16 │
 3. │                           2 │     4.16 │
 4. │                           3 │     4.16 │
 5. │                          25 │     4.16 │
 6. │                          23 │     4.16 │
 7. │                          27 │     4.16 │
 8. │                          16 │     4.16 │
 9. │                           7 │     4.15 │
10. │                           9 │     4.15 │
11. │                          10 │     4.15 │
12. │                          11 │     4.15 │
13. │                          12 │     4.15 │
14. │                          13 │     4.15 │
15. │                           8 │     4.15 │
16. │                           1 │     4.15 │
17. │                          17 │     4.15 │
18. │                          18 │     4.15 │
19. │                          19 │     4.15 │
20. │                          20 │     4.15 │
21. │                          21 │     4.15 │
22. │                           4 │     4.15 │
23. │                          15 │     4.14 │
24. │                           6 │     4.14 │
25. │                           5 │     4.14 │
26. │                          24 │     4.14 │
27. │                          26 │     4.14 │
28. │                          22 │     4.14 │
    └─────────────────────────────┴──────────┘

28 rows in set. Elapsed: 0.016 sec. Processed 688.13 thousand rows, 4.13 MB (41.77 million rows/s., 250.59 MB/s.)
Peak memory usage: 1.03 MiB.
```

```sql
SELECT * FROM system.query_log ORDER BY event_time DESC;
```

Результат
```text
Query id: 0f5388a4-da94-46d5-8685-588f70119099
```

```sql
SELECT * FROM system.processors_profile_log WHERE query_id = '0f5388a4-da94-46d5-8685-588f70119099' order by processor_uniq_id;
```

Результат
```text
hostname    |event_date|event_time         |event_time_microseconds      |id             |parent_ids                                                                                                                                                                                       |plan_step      |plan_step_name   |plan_step_description         |plan_group|initial_query_id                    |query_id                            |name                                              |elapsed_us|input_wait_elapsed_us|output_wait_elapsed_us|input_rows|input_bytes|output_rows|output_bytes|processor_uniq_id                                   |step_uniq_id       |
------------+----------+-------------------+-----------------------------+---------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+-----------------+------------------------------+----------+------------------------------------+------------------------------------+--------------------------------------------------+----------+---------------------+----------------------+----------+-----------+-----------+------------+----------------------------------------------------+-------------------+
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588445253144|[140582146299416]                                                                                                                                                                                |140580173165824|Aggregating      |                              |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|AggregatingTransform                              |      3219|                 7499|                     0|    147851|     591404|          0|           0|AggregatingTransform_13                             |Aggregating_4      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588445253784|[140582146299416]                                                                                                                                                                                |140580173165824|Aggregating      |                              |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|AggregatingTransform                              |      4156|                 6556|                     0|    190979|     763916|          0|           0|AggregatingTransform_14                             |Aggregating_4      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588445254424|[140582146299416]                                                                                                                                                                                |140580173165824|Aggregating      |                              |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|AggregatingTransform                              |      3298|                 8071|                    12|    253319|    1013472|         28|         308|AggregatingTransform_15                             |Aggregating_4      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588445255064|[140582146299416]                                                                                                                                                                                |140580173165824|Aggregating      |                              |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|AggregatingTransform                              |      1251|                 5334|                     0|     73721|     294884|          0|           0|AggregatingTransform_16                             |Aggregating_4      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140581924406232|[140588445254424]                                                                                                                                                                                |              0|                 |                              |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ConvertingAggregatedToChunksTransform             |        47|                    0|                     0|         0|          0|         28|         308|ConvertingAggregatedToChunksTransform_72            |                   |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588444921880|[140588445253784]                                                                                                                                                                                |140588457785088|Expression       |Before GROUP BY               |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |        15|                 6509|                  4139|    190979|     763916|     190979|      763916|ExpressionTransform_10                              |Expression_3       |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140581935794200|[140588445254424]                                                                                                                                                                                |140588457785088|Expression       |Before GROUP BY               |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |        21|                 7968|                  3301|    253291|    1013164|     253291|     1013164|ExpressionTransform_11                              |Expression_3       |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140581935794712|[140588445255064]                                                                                                                                                                                |140588457785088|Expression       |Before GROUP BY               |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |        10|                 5311|                  1208|     73721|     294884|      73721|      294884|ExpressionTransform_12                              |Expression_3       |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140583901311000|[140582218978072]                                                                                                                                                                                |140582156361472|Expression       |(Before ORDER BY + Projection)|         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |        12|                11429|                    18|        28|        308|         28|         308|ExpressionTransform_18                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140583901311512|[140582218978840]                                                                                                                                                                                |140582156361472|Expression       |(Before ORDER BY + Projection)|         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |         0|                11413|                     0|         0|          0|          0|           0|ExpressionTransform_19                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140583901312024|[140589048092696]                                                                                                                                                                                |140582156361472|Expression       |(Before ORDER BY + Projection)|         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |         0|                11414|                     0|         0|          0|          0|           0|ExpressionTransform_20                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140583901312536|[140589048095000]                                                                                                                                                                                |140582156361472|Expression       |(Before ORDER BY + Projection)|         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |         0|                11413|                     0|         0|          0|          0|           0|ExpressionTransform_21                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140583901313048|[140589048096536]                                                                                                                                                                                |140582156361472|Expression       |(Before ORDER BY + Projection)|         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |         0|                11414|                     0|         0|          0|          0|           0|ExpressionTransform_22                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140583901313560|[140589048097304]                                                                                                                                                                                |140582156361472|Expression       |(Before ORDER BY + Projection)|         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |         0|                11414|                     0|         0|          0|          0|           0|ExpressionTransform_23                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140583901314072|[140589048098072]                                                                                                                                                                                |140582156361472|Expression       |(Before ORDER BY + Projection)|         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |         0|                11414|                     0|         0|          0|          0|           0|ExpressionTransform_24                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140583901314584|[140588458117144]                                                                                                                                                                                |140582156361472|Expression       |(Before ORDER BY + Projection)|         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |         0|                11414|                     0|         0|          0|          0|           0|ExpressionTransform_25                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140583901315096|[140588458117912]                                                                                                                                                                                |140582156361472|Expression       |(Before ORDER BY + Projection)|         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |         0|                11415|                     0|         0|          0|          0|           0|ExpressionTransform_26                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140583901315608|[140588458118680]                                                                                                                                                                                |140582156361472|Expression       |(Before ORDER BY + Projection)|         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |         0|                11415|                     0|         0|          0|          0|           0|ExpressionTransform_27                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140583901316120|[140588458119448]                                                                                                                                                                                |140582156361472|Expression       |(Before ORDER BY + Projection)|         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |         0|                11414|                     0|         0|          0|          0|           0|ExpressionTransform_28                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140583901316632|[140588458120216]                                                                                                                                                                                |140582156361472|Expression       |(Before ORDER BY + Projection)|         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |         0|                11414|                     0|         0|          0|          0|           0|ExpressionTransform_29                              |Expression_10      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588457743384|[140588457745944]                                                                                                                                                                                |140582764875520|Expression       |                              |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |        99|                 7286|                  3278|    147851|     887106|     147851|      591404|ExpressionTransform_5                               |Expression_11      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588457743896|[140588444921880]                                                                                                                                                                                |140582764875520|Expression       |                              |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |        36|                 6435|                  4180|    190979|    1145874|     190979|      763916|ExpressionTransform_6                               |Expression_11      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582110449176|[140582767248280]                                                                                                                                                                                |140580176649216|Expression       |Project names                 |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |         5|                11640|                    22|        28|        308|         28|         308|ExpressionTransform_67                              |Expression_8       |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588457744920|[140581935794200]                                                                                                                                                                                |140582764875520|Expression       |                              |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |        77|                 7844|                  3347|    253291|    1519746|     253291|     1013164|ExpressionTransform_7                               |Expression_11      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588457745432|[140581935794712]                                                                                                                                                                                |140582764875520|Expression       |                              |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |        25|                 5252|                  1232|     73721|     442326|      73721|      294884|ExpressionTransform_8                               |Expression_11      |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588457745944|[140588445253144]                                                                                                                                                                                |140588457785088|Expression       |Before GROUP BY               |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|ExpressionTransform                               |        32|                 7438|                  3222|    147851|     591404|     147851|      591404|ExpressionTransform_9                               |Expression_3       |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582094664472|[]                                                                                                                                                                                               |              0|                 |                              |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|LazyOutputFormat                                  |        16|                11679|                     0|        28|        308|          0|           0|LazyOutputFormat_69                                 |                   |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588445255704|[140588458120984]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|LimitsCheckingTransform                           |         0|                11510|                     2|        28|        308|         28|         308|LimitsCheckingTransform_42                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588445256344|[140582495988248]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|LimitsCheckingTransform                           |         0|                11415|                     0|         0|          0|          0|           0|LimitsCheckingTransform_43                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588445256984|[140582146043160]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|LimitsCheckingTransform                           |         0|                11416|                     0|         0|          0|          0|           0|LimitsCheckingTransform_44                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588445257624|[140582091978264]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|LimitsCheckingTransform                           |         0|                11417|                     0|         0|          0|          0|           0|LimitsCheckingTransform_45                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582226952216|[140582091979032]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|LimitsCheckingTransform                           |         0|                11417|                     0|         0|          0|          0|           0|LimitsCheckingTransform_46                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582221976088|[140582091979800]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|LimitsCheckingTransform                           |         0|                11417|                     0|         0|          0|          0|           0|LimitsCheckingTransform_47                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582767233560|[140582091980568]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|LimitsCheckingTransform                           |         0|                11416|                     0|         0|          0|          0|           0|LimitsCheckingTransform_48                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582767234840|[140582091981336]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|LimitsCheckingTransform                           |         0|                11417|                     0|         0|          0|          0|           0|LimitsCheckingTransform_49                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582767240600|[140582091982104]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|LimitsCheckingTransform                           |         0|                11417|                     0|         0|          0|          0|           0|LimitsCheckingTransform_50                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582767243160|[140582496260120]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|LimitsCheckingTransform                           |         0|                11416|                     0|         0|          0|          0|           0|LimitsCheckingTransform_51                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582767247000|[140582496260888]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|LimitsCheckingTransform                           |         0|                11416|                     0|         0|          0|          0|           0|LimitsCheckingTransform_52                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582767247640|[140582496261656]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|LimitsCheckingTransform                           |         0|                11416|                     0|         0|          0|          0|           0|LimitsCheckingTransform_53                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582767248280|[140582094664472]                                                                                                                                                                                |              0|                 |                              |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|LimitsCheckingTransform                           |         1|                11668|                    19|        28|        308|         28|         308|LimitsCheckingTransform_68                          |                   |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588458120984|[140583883297816]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeSortingTransform                             |         5|                11513|                     1|        28|        308|         28|         308|MergeSortingTransform_54                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582495988248|[140583883297816]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeSortingTransform                             |         8|                11417|                     0|         0|          0|          0|           0|MergeSortingTransform_55                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582146043160|[140583883297816]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeSortingTransform                             |         2|                11417|                     0|         0|          0|          0|           0|MergeSortingTransform_56                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582091978264|[140583883297816]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeSortingTransform                             |         2|                11418|                     0|         0|          0|          0|           0|MergeSortingTransform_57                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582091979032|[140583883297816]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeSortingTransform                             |         2|                11418|                     0|         0|          0|          0|           0|MergeSortingTransform_58                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582091979800|[140583883297816]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeSortingTransform                             |         3|                11418|                     0|         0|          0|          0|           0|MergeSortingTransform_59                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582091980568|[140583883297816]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeSortingTransform                             |         5|                11417|                     0|         0|          0|          0|           0|MergeSortingTransform_60                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582091981336|[140583883297816]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeSortingTransform                             |         2|                11418|                     0|         0|          0|          0|           0|MergeSortingTransform_61                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582091982104|[140583883297816]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeSortingTransform                             |         2|                11418|                     0|         0|          0|          0|           0|MergeSortingTransform_62                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582496260120|[140583883297816]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeSortingTransform                             |         6|                11417|                     0|         0|          0|          0|           0|MergeSortingTransform_63                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582496260888|[140583883297816]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeSortingTransform                             |         4|                11417|                     0|         0|          0|          0|           0|MergeSortingTransform_64                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582496261656|[140583883297816]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeSortingTransform                             |         1|                11416|                     0|         0|          0|          0|           0|MergeSortingTransform_65                            |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140580176278552|[140588457743384]                                                                                                                                                                                |140588447588352|ReadFromMergeTree|learn_db.mart_student_lesson  |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|      7030|                    0|                  3449|         0|          0|     147851|      887106|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_1|ReadFromMergeTree_0|
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140580176279064|[140588457743896]                                                                                                                                                                                |140588447588352|ReadFromMergeTree|learn_db.mart_student_lesson  |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|      6194|                    0|                  4257|         0|          0|     190979|     1145874|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_2|ReadFromMergeTree_0|
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588457712152|[140588457744920]                                                                                                                                                                                |140588447588352|ReadFromMergeTree|learn_db.mart_student_lesson  |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|      7580|                    0|                  3482|         0|          0|     253291|     1519746|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_3|ReadFromMergeTree_0|
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588457713176|[140588457745432]                                                                                                                                                                                |140588447588352|ReadFromMergeTree|learn_db.mart_student_lesson  |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergeTreeSelect(pool: ReadPool, algorithm: Thread)|      4989|                    0|                  1311|         0|          0|      73721|      442326|MergeTreeSelect(pool: ReadPool, algorithm: Thread)_4|ReadFromMergeTree_0|
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140583883297816|[140582110449176]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|MergingSortedTransform                            |        15|                11611|                    31|        28|        308|         28|         308|MergingSortedTransform_66                           |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582168741400|[140582094664472]                                                                                                                                                                                |              0|                 |                              |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|NullSource                                        |         0|                    0|                     0|         0|          0|          0|           0|NullSource_70                                       |                   |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582168740888|[140582094664472]                                                                                                                                                                                |              0|                 |                              |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|NullSource                                        |         0|                    0|                     0|         0|          0|          0|           0|NullSource_71                                       |                   |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582218978072|[140588445255704]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|PartialSortingTransform                           |        10|                11496|                     4|        28|        308|         28|         308|PartialSortingTransform_30                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582218978840|[140588445256344]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|PartialSortingTransform                           |         0|                11414|                     0|         0|          0|          0|           0|PartialSortingTransform_31                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140589048092696|[140588445256984]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|PartialSortingTransform                           |         0|                11415|                     0|         0|          0|          0|           0|PartialSortingTransform_32                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140589048095000|[140588445257624]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|PartialSortingTransform                           |         0|                11415|                     0|         0|          0|          0|           0|PartialSortingTransform_33                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140589048096536|[140582226952216]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|PartialSortingTransform                           |         0|                11415|                     0|         0|          0|          0|           0|PartialSortingTransform_34                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140589048097304|[140582221976088]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|PartialSortingTransform                           |         0|                11415|                     0|         0|          0|          0|           0|PartialSortingTransform_35                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140589048098072|[140582767233560]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|PartialSortingTransform                           |         0|                11415|                     0|         0|          0|          0|           0|PartialSortingTransform_36                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588458117144|[140582767234840]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|PartialSortingTransform                           |         0|                11416|                     0|         0|          0|          0|           0|PartialSortingTransform_37                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588458117912|[140582767240600]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|PartialSortingTransform                           |         0|                11416|                     0|         0|          0|          0|           0|PartialSortingTransform_38                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588458118680|[140582767243160]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|PartialSortingTransform                           |         0|                11416|                     0|         0|          0|          0|           0|PartialSortingTransform_39                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588458119448|[140582767247000]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|PartialSortingTransform                           |         0|                11415|                     0|         0|          0|          0|           0|PartialSortingTransform_40                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140588458120216|[140582767247640]                                                                                                                                                                                |140582168741376|Sorting          |Sorting for ORDER BY          |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|PartialSortingTransform                           |         0|                11415|                     0|         0|          0|          0|           0|PartialSortingTransform_41                          |Sorting_7          |
19294dc11be6|2025-10-13|2025-10-13 22:39:19|2025-10-13 22:39:19.158231000|140582146299416|[140583901311000,140583901311512,140583901312024,140583901312536,140583901313048,140583901313560,140583901314072,140583901314584,140583901315096,140583901315608,140583901316120,140583901316632]|140580173165824|Aggregating      |                              |         0|0f5388a4-da94-46d5-8685-588f70119099|0f5388a4-da94-46d5-8685-588f70119099|Resize                                            |         0|                11423|                     0|        28|        308|         28|         308|Resize_17                                           |Aggregating_4      |
```

### 29. Проверяем, как поменялась эффективность применения индекса

```sql
EXPLAIN indexes = 1
SELECT 
	educational_organization_id,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	subject_id = 3
GROUP BY
	educational_organization_id
ORDER BY 
	avg_mark desc;
```

Результат
```text
explain                                                     |
------------------------------------------------------------+
Expression (Project names)                                  |
  Sorting (Sorting for ORDER BY)                            |
    Expression ((Before ORDER BY + Projection))             |
      Aggregating                                           |
        Expression (Before GROUP BY)                        |
          Expression                                        |
            ReadFromMergeTree (learn_db.mart_student_lesson)|
            Indexes:                                        |
              PrimaryKey                                    |
                Keys:                                       |
                  subject_id                                |
                Condition: (subject_id in [3, 3])           |
                Parts: 4/4                                  |
                Granules: 84/1223                           |
```