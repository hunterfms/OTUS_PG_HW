# Дипломная работа по теме - Оптимизация запросов. 

## Цель: Разработать и в дальнейшем оптимизировать запросы с таблиц большого размера, для наиболее быстрой выдачи их результатов и наименьшего использования ими ресурсов сервера.  Оптимизация должна затронуть сам код запроса и структуру базы.

Восстанавливаем на виртуальной машине PG15 DataSet "Employees database" с общим количеством строк в таблицах 3919015, с ресурса: https://raw.githubusercontent.com/neondatabase/postgres-sample-dbs/main/employees.sql.gz
1) Подготавил базу employees и схему employees для проекта.  Далее скачал архив dataset-а, дал доступ postgres на каталог с фалом и восстановил его в базу employees через pg_restore.
```
postgres=# CREATE DATABASE employees;
CREATE DATABASE
postgres=# \c employees
You are now connected to database "employees" as user "postgres".
employees=# CREATE SCHEMA employees;
CREATE SCHEMA
employees-# \q
fedor@pg15:~$ wget https://raw.githubusercontent.com/neondatabase/postgres-sampl                               e-dbs/main/employees.sql.gz
--2025-01-16 16:40:01--  https://raw.githubusercontent.com/neondatabase/postgres                               -sample-dbs/main/employees.sql.gz
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.109.1                               33, 185.199.110.133, 185.199.108.133, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.199.109.                               133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 34871756 (33M) [application/octet-stream]
Saving to: ‘employees.sql.gz’

employees.sql.gz    100%[===================>]  33.26M  18.0MB/s    in 1.9s

2025-01-16 16:40:05 (18.0 MB/s) - ‘employees.sql.gz’ saved [34871756/34871756]

fedor@pg15:~$ mc
postgres@pg15:/1_temp$ pg_restore -d employees -Fc employees.sql.gz -c -v --no-owner --no-privileges

```
2) Выдаем размеры таблиц проекта в строках.
```
employees=# select * from pg_tables where schemaname = 'employees';
 schemaname |      tablename      | tableowner | tablespace | hasindexes | hasrules | hastriggers | rowsecurity

------------+---------------------+------------+------------+------------+----------+-------------+------------
-
 employees  | employee            | postgres   |            | t          | f        | t           | f
 employees  | department_employee | postgres   |            | t          | f        | t           | f
 employees  | department          | postgres   |            | t          | f        | t           | f
 employees  | department_manager  | postgres   |            | t          | f        | t           | f
 employees  | salary              | postgres   |            | t          | f        | t           | f
 employees  | title               | postgres   |            | t          | f        | t           | f
(6 rows)

employees=# select count(*) from employees.employee;
 count
--------
 300024
(1 row)

employees=# select count(*) from employees.department_employee;
 count
--------
 331603
(1 row)

employees=# select count(*) from employees.department;
 count
-------
     9
(1 row)

employees=# select count(*) from employees.department_manager;
 count
-------
    24
(1 row)

employees=# select count(*) from employees.salary;
  count
---------
 2844047
(1 row)

employees=# select count(*) from employees.title;
 count
--------
 443308
(1 row)
```
3) Очищаем структуру таблиц оставляя только ключи.
4) Выдаем информацию по текущей структуре таблиц.
```
\d+ employees.employee
```
6) отрабатываем запросы с параметром explane

