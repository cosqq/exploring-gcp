# Introduction 

Introducting the use of Bigquery for analytical workloads. Below is a summarized notes for what i have learnt about Big query as a technology and how I have explored using it.

# Partitions 

# Arrays and Structs 

Arrays and Structs are one of the concepts I came across when exploring the uses of Big Query. It was introduced as sorta like an in between concept of normalization (slower with joins but higher integrity) and denormalization (faster without joins lower integrity). 

It proposes how it is efficient by storing tables in tables or by letting cells hold multiple values. This reduces the number of rows and lesser number of tables to maintain.  

## Arrays & UNNEST() Operator

Arrays are `REPEATED` field type.
 
 ---
### Creating Arrays 

Below is a simple example of how Arrays are like in tables.

```
Create or Replace TABLE "Students" as (
Select "Luffy" as name, ["English","Chinese","Tamil"] as languages 
UNION ALL
Select "Kid" as name, ["English","Chinese","Tamil"] as languages 
)
```
---
### Array Funcs 

#### ARRAY_AGG(): 
Aggregate our string values into an array. We usually would want to use this to make each row unique by concatenating string values and represent it with a group. For example, given `Students` table 

| name | language |
| --- |---|
| Kid | English |
| Kid | Chinese |
| Kid | Tamil |
| Kid | Chinese |


```
SELECT name,
ARRAY_AGG( DISTINCT language ORDER BY language LIMIT 5) AS students_language
ARRAY_LENGTH(ARRAY_AGG( DISTINCT language)) as students_no_languages
FROM Students 
GROUP BY name
```

| name | students_language | students_no_languages |
| --- |---| ---|
| Kid | [English,Chinese,Tamil] | 3

Notice : 
1. `DISTINCT` was used to remove duplicates for each repeating row 
2. `ARRAY_LENGTH` and `ARRAY_AGG` was used to combine rows pertaining to student name group into a single cell and to get the length of that array respectively 
3. `GROUP BY` was used in coherent with aggregate functions like `ARRAY_AGG`
4. Notice : `ARRAY_AGG( DISTINCT language ORDER BY language LIMIT 5)` This shows that ARRAY_AGG can also be used with `ORDER BY` and `LIMIT`

While array aggregation is straight forward, querying them is slight more complex.

---
#### UNNEST()

UNNEST() arrays to bring the array elements back into rows. How it is done in the arrays is similar to a `cross join` where the first table is cross joined to the next but can 'cut short the query'

Quoted by Google :
> UNNEST() always follows the table name in your FROM clause (think of it conceptually like a pre-joined table) 

Given a schema 

| column name | Type | 
| ---| ---|
name | STRING
students.language_arr | REPEATED

```
SELECT s.name, l.language
FROM Students AS s , 
UNNEST(students.language_arr) AS l
```
Notice : 
1. A `,` is used instead of a `cross join` 
2. `UNNEST()` first unpacks the array and treat the unpacked array as a table with an alias (its good practice to have).
3. Using the alias, we can treat it like another seperated table
4. To rename as column use `UNNEST(select * from unnest(x) as v)` this will name the unnested array column v


## STRUCT 

Is sorta way to create a sub table in a query. It is a `RECORD` type also referred to as nested fields.

Before starting, we should understand how does a struct look like in Big Query Tables. 

| class | students.name | students.grade |
| --- | --- | --- |
| A | Luffy | 80 |
| | Kid | 90
|B |Usopp| 100|
||Nami|80| 

Notice  
1. 

Example 
```
Select name
STRUCT(name, language) as student_info
from Students
```

### Querying Struct 

To query a struct, we can just select it. TO query a column in a struct would be as follow 

```
Select s.name, r.language
from Students as s, s.student_info as r
```

### More Complex Situations 

1. Unnest a struct 

Remember when unnesting a struct we are unnesting the table to be one for each row. To make a table looking normal, we can treat the `STRUCT` like a table to performa `Cross Join`. 

```
SELECT class, students.name
FROM Classes
CROSS JOIN
Students
```

| class | students.name | students.grade |
| --- | --- | --- |
| A | Luffy | 80 |
| A | Kid | 90
|B|Usopp| 100|
|B|Nami|80| 

Lets break it down even further. Lets take a query like 

```
#standardSQL
SELECT
  s.name,
    grade
FROM Class AS c
, UNNEST(c.Students) AS s
, UNNEST(s.grade) AS grade
WHERE grade = 80;
```
and a table example like 

| class | students.name | students.grade |
| --- | --- | --- |
| A | Luffy | 80 |
| A |  | 90
|B|Usopp| 100|
|B||80| 

To 

| class | UNNEST(students) | UNNEST(students.grade) |
| --- | --- | --- |
| A | Luffy | 80 |
| A | Luffy | 90
|B|Usopp |100|
|B|Usopp |80| 


Whats going on here is that students.grade is an array, in this case, we can drill down using the UNNEST operation. 