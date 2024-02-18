# Intro To Postgres

```Bash
docker run --name intropostgres -e POSTGRES_PASSWORD=postgresintro -d postgres
```
```Bash
docker exec -it intropostgres bash
```
```Bash
psql -d postgres -U postgres
```

__NOTE__: all commands starting with '\' are supported via "psql" only also you can type "\help" 
to get the full list of SQL commands


## DATABASES

- create a database
    ```SQL
   CREATE DATABASE test;
    ```
    - ';' is used to mark end of a sql query
- connect to test : `\c test`

## TABLES

- tables in RDBMS are used to present data in format of Rows and Columns.
- data in stored in rows and columns defines the structure of the table.
  - rows are not ordered
  - each column has a data type : integer , char , boolean etc

- Create a table for university courses 
```SQL
    CREATE TABLE courses(
    c_no text PRIMARY KEY,
    title text,
    hours integer
    );
```
- POSTGRES uses integrity constraints that will be checked automatically to prevent any invalid data being entered
  - PRIMARY KEY constraint allows only unique and not-null data inside the column i.e. no two or more rows 
  can have same data within that column

## Filling Tables with Data
- insert some rows in courses table
```SQL
INSERT INTO courses(c_no, title, hours)
VALUES ('CS301', 'Databases', 30),
('CS305', 'Networks', 60); 
```
    - to perfrom bulk data upload, use COPY command
- we need two more tables
```SQL
CREATE TABLE students(
s_id integer PRIMARY KEY,
name text,
start_year integer
);
```
- insert data
```SQL
INSERT INTO students(s_id, name, start_year)
VALUES (1451, 'Anna', 2014),
(1432, 'Victor', 2014),
(1556, 'Nina', 2015);
```
- another table
```SQL
CREATE TABLE exams(
s_id integer REFERENCES students(s_id),
c_no text REFERENCES courses(c_no),
score integer,
CONSTRAINT pk PRIMARY KEY(s_id, c_no)
); 
```
- fill data
```SQL
INSERT INTO exams(s_id, c_no, score)
VALUES (1451, 'CS301', 5),
(1556, 'CS301', 5),
(1451, 'CS305', 5),
(1432, 'CS305', 4);
```

## Data Retrieval
- use SELECT to read data , you can also rename the column of retrieved data for view 
```SQL
SELECT title AS course_title, hours
FROM courses;
``` 
- to display all columns
```SQL
SELECT * FROM courses;
```
> In professional development only required columns and only those rows that satisfy a certain condition
> should be specified for efficient queries and faster results

- use of DISTINCT to avoid repeated data in a column 
  - without DISTINCT 
  ```SQL
    SELECT start_year FROM students;
  ```
  - with 
  ```SQL
    SELECT DISTINCT start_year FROM students;
  ```
  
- You can use expressions as well with SELECT 
```SQL 
SELECT 2 + 3 AS result;
```

- use WHERE clause to apply conditions for filtering 
  - The condition must be of a logical type, can contain operators =, <> (or !=), >, >=, <, <=, as well as combine
  using logical operations AND, OR, NOT and parenthesis
  - comparisons and logical operations on NULL are not defines , use special operators like
    - IS NULL (IS NOT NULL) and IS DISTINCT FROM (IS NOT DISTINCT FROM).
```SQL
SELECT * FROM courses WHERE hours > 45 ; 
```

## JOINS
>  well-designed database should not contain redundant data.
> like exams table does not need student ID as it can be obtained from another.

- direct or Cartesian product of tables
  - Cartesian product of tables: like matrix multiplication , rows X x columns
```SQL
SELECT * FROM exams, courses; 
SELECT * from courses, exams;
```
- specify the join condition in the WHERE clause to can get a more useful and informative result
```SQL
SELECT courses.title, exams.s_id, exams.score
FROM courses, exams
WHERE courses.c_no = exams.c_no;
```

- JOIN keyword 
  -  the students that did not take the exam in this subject are excluded
```SQL
SELECT students.name, exams.score
FROM students
JOIN exams
    ON students.s_id = exams.s_id
    AND exams.c_no = 'CS305';   
```
- OUTER JOIN to include all the students
```SQL
SELECT students.name, exams.score
FROM students
LEFT JOIN exams
    ON students.s_id = exams.s_id
    AND exams.c_no = 'CS305'; 
```
> why left : the rows of the left table that don’t have a counterpart in the right table are added to the result 
> but if remove AND condition from JOIN and use WITH clause
```SQL
SELECT students.name, exams.score
FROM students
LEFT JOIN exams ON students.s_id = exams.s_id
WHERE exams.c_no = 'CS305';
```

> It better to join the table are using DB server as they very efficient at these operations, 
> instead of doing it application level


## Sub-queries 
- Nested SELECT command (The SELECT operation "returns" a table)
> A sub-query is used to return data that will be used in the main query as a condition to 
> further restrict the data to be retrieved.
>Sub-queries can be used with the SELECT, INSERT, UPDATE and DELETE statements along with the 
> operators like =, <, >, >=, <=, IN, etc.

```SQL
SELECT name,
(SELECT score
FROM exams
WHERE exams.s_id = students.s_id
AND exams.c_no = 'CS305')
FROM students;
```
- if the inner query is scalar(a regular SELECT query in parentheses that returns exactly one value: one row with
  one column.) but does not return any rows , NULL is returned.
  - OUTER JOIN can be used to replace such scalar queries

```SQL
SELECT *
FROM exams
WHERE (SELECT start_year FROM students
  WHERE students.s_id = exams.s_id) > 2014;
```

- IN checks whether the table returned by the sub-query contains the specified value.
  - display all the students who have any scores in the specified course

```SQL
SELECT name, start_year
FROM students
WHERE s_id IN (SELECT s_id FROM exams
      WHERE c_no = 'CS305'); 
```
- NOT IN, returns the opposite result.
  -  list of students who did not get any excellent scores
```SQL
SELECT name, start_year
FROM students
WHERE s_id NOT IN (SELECT s_id
                   FROM exams
                   WHERE score = 5);
```
## Alias
- here alias is given to nameless sub-query 
- here “s” is a table alias, while “ce” is a sub-query alias
```SQL 
SELECT s.name, ce.score
FROM students s
JOIN (SELECT exams.*
FROM courses, exams
WHERE courses.c_no = exams.c_no
AND courses.title = 'Databases') ce
ON s.s_id = ce.s_id;```
```
  - alternative query
```SQL
SELECT s.name, e.score
FROM students s, courses c, exams e
WHERE c.c_no = e.c_no
AND c.title = 'Databases'
AND s.s_id = e.s_id;
```

## Sorting

- ORDER BY : 
> [doc](https://postgrespro.com/doc/sql-select#SQL-ORDERBY)
```SQL
SELECT * FROM exams
ORDER BY score, s_id, c_no DESC; 
```

## Grouping

>the query returns a single row with, the value calculated based on several rows of data stored

-  total number of exams taken, the number of students who passed the exams, and the average score:
> count and avg are called aggregate functions
```SQL
SELECT count(*), count(DISTINCT s_id),
avg(score)
FROM exams;
```
  - using GROUP BY
```SQL
SELECT c_no, count(*),
count(DISTINCT s_id), avg(score)
FROM exams
GROUP BY c_no;
```

-  select the names of the students who got more than one excellent score (5), in any course:
  - WHERE and HAVING are used to filter the rows based on aggregation result.
    - WHERE is used before grouping allowing to use columns from original table 
    - HAVING is used after grouping thus can access  columns of the resulting table
```SQL
SELECT students.name
FROM students, exams
WHERE students.s_id = exams.s_id AND exams.score = 5
GROUP BY students.name
HAVING count(*) > 1;
```

## CHANGE/UPDATE

- let's double the lecture hours for a particular course
```SQL
UPDATE courses
SET hours = hours * 2
WHERE c_no = 'CS301';
```

- let's delete exam scores less than 5
```SQL
DELETE FROM exams WHERE score < 5; 
```

## Transactions

__NOTE__: you can use `\d students` to view all defined columns or just `\d` to view all tables present under the db.

-  add a new column
```SQL
ALTER TABLE students
ADD g_no text REFERENCES groups(g_no);
```
- create a table 
  - Each group must have a monitor (a student of the same group responsible for the students’ activities).
```SQL
CREATE TABLE groups(
g_no text PRIMARY KEY,
monitor integer NOT NULL REFERENCES students(s_id)
);
```

- let’s create a new group called “A-101,” move all the students into this group, and make Anna its monitor.
  - note that we cannot create a group without a monitor and a student cannot become a monitor unless it's part
  of the group.
    - A group of operations constituting an indivisible logical unit of work is called a transaction.
      - any transaction is executed either completely, or not at all.
      - if any commands results in an error, or we have aborted the transaction with the ROLLBACK command, the database stays 
      in the same state as before the 'BEGIN' command. This property is called atomicity
      - when a transaction is committed, all integrity constraints must hold true, 
      otherwise the transaction has to be aborted. This property is called consistency
      - other users will never see inconsistent data not yet committed by the transaction. 
      This property is called isolation. So database system can serve multiple sessions in parallel
        - Blocking occurs only if two different processes try changing the same row simultaneously.
      - durability is guaranteed: the committed data is never lost even in case of a failure 

- let’s start our transaction: type `BEGIN;`
  - asterisk in the prompt reminds us that the transaction is not yet completed.
-  add a new group, together with its monitor
```SQL
INSERT INTO groups(g_no, monitor)
SELECT 'A-101', s_id
FROM students
WHERE name = 'Anna';
```
- move all students
```SQL
UPDATE students SET g_no = 'A-101';
```

- end our transaction by typing `COMMIT;`

- To end the session : `\q`

___

