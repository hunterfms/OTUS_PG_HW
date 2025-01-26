# Проектная работа по теме - Оптимизация запросов. 

## Цель: Разработать и в дальнейшем оптимизировать запросы с таблиц большого размера, для наиболее быстрой выдачи их результатов и наименьшего использования ими ресурсов сервера.  Оптимизация должна затронуть сам код запроса и структуру базы.

Восстанавливаем на виртуальной машине PG15 DataSet "Employees database" с общим количеством строк в таблицах 3919015, с ресурса: https://raw.githubusercontent.com/neondatabase/postgres-sample-dbs/main/employees.sql.gz

DataSet включает в себя структуру таблиц с данными в виде справочников о сотрудниках компании, их должностях в отделах компании, уровню годовых зарплат и детальную информацию по выдаче зарплат за определенный период времени. Справочники содержат информацию с раскладкой по датам принятия на работу сотрудников и даты их увольнения. Данные регистрации выдачи зарплат также сопровождаются датами их выполнения. 
Характер данных по таблицам:

employees.employee – справочник сотрудников компании, наработанный в течение нескоих лет жизни компании представленный полями:
- Id – идентификатор строки.
- birth_date – день рождения сотрудника
- first_name – фамилия сотрудника
- last_name – имя сотрудника
- gender – пол.
- hire_date – дата устройства на работу

employees.department – справочник подразделений компании, наработанный в течение нескоих лет жизни компании представленный полями:
- Id – идент. строки.
- dept_name – наименование подразделения

employees.department_employee – таблица привязки сотрудника к департаменту по которой определяется в каком подразделении работает сотрудник, с данными наработанными в течение нескоих лет жизни компании представленна полями:
- employee_id – идент. сотрудника связанный с полем id таблицы employee.
- department_id – идент. подразделения связанный с полем id таблицы department.
- from_date – дата начала работы сотрудника в подразделении.
- to_date – дата окончания работы сотрудника в подразделении.

employees.department_manager – справочник руководителей подразделений
- employee_id – идент. сотрудника связанный с полем id таблицы employee.
- department_id – идент. подразделения связанный с полем id таблицы department.
- from_date – дата начала работы сотрудника в подразделении в качестве руководителя подразделения.
- to_date – дата окончания работы сотрудника в подразделении в качестве руководителя подразделения.

employees.title – таблица привязки сотрудника к должности подразделения в рамках которой ограничены его должностные инструкции, с данными наработанными в течение нескоих лет жизни компании представленна полями:
- employee_id – идент. сотрудника связанный с полем id таблицы employee.
- title– должность сотрудника.
- from_date – дата начала работы сотрудника в должности.
- to_date – дата окончания работы сотрудника в должности.

## Свои выводы и рекомендации по нормализации данных.

Было бы эффективнее упразднить таблицу справочник руководителей подразделений department_manager. Данные этой таблицы хоть и позволяют отслеживать историю смены руководителей но такая история уже есть совокупно в таблицах привязки сотрудника к департаменту с таблицей должностей. Иными словами можно выдать историю смены руководителей отдела связанным запросом, а не занимать под эти «результатирующие данные» диски (сорные данные).

## Приступим к реализации.

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
select count(*) from employees.department_employee;
select count(*) from employees.department;
select count(*) from employees.department_manager;
select count(*) from employees.salary;
select count(*) from employees.title;
 count
--------
 300024
(1 row)

 count
--------
 331603
(1 row)

 count
-------
     9
(1 row)

 count
-------
    24
(1 row)

  count
---------
 2844047
(1 row)

 count
--------
 443308
(1 row)

employees=#

```
3) Очищаем структуру таблиц оставляя только данные. Сначала удаляем все индексы, потом внешние ключи и первичные ключи.
```
DO
$do$
DECLARE
   _sql text;
BEGIN   
   SELECT 'DROP INDEX ' || string_agg(indexrelid::regclass::text, ', ')
   FROM   pg_index  i
   LEFT   JOIN pg_depend d ON d.objid = i.indexrelid
                          AND d.deptype = 'i'
   WHERE  i.indrelid = 'employees.employee'::regclass  -- для каждой таблицы в базе
   AND    d.objid IS NULL                      
   INTO   _sql;
   
   IF _sql IS NOT NULL THEN                    -- только если найдены index(es)
     EXECUTE _sql;
   END IF;
END
$do$;

-- удаление всех внешних ключей
ALTER TABLE employees.department_employee DROP CONSTRAINT dept_emp_ibfk_1;
ALTER TABLE employees.department_manager DROP CONSTRAINT dept_manager_ibfk_1;
ALTER TABLE employees.salary DROP CONSTRAINT salaries_ibfk_1;
ALTER TABLE employees.title DROP CONSTRAINT titles_ibfk_1;
ALTER TABLE employees.department_employee DROP CONSTRAINT dept_emp_ibfk_2;
ALTER TABLE employees.department_manager DROP CONSTRAINT dept_manager_ibfk_2;
-- удаление первичных ключей
ALTER TABLE employees.employee DROP CONSTRAINT idx_16988_primary;
ALTER TABLE employees.department_employee DROP CONSTRAINT idx_16982_primary;
ALTER TABLE employees.department DROP CONSTRAINT idx_16979_primary;
ALTER TABLE employees.department_manager DROP CONSTRAINT idx_16985_primary;
ALTER TABLE employees.salary DROP CONSTRAINT idx_16991_primary;
ALTER TABLE employees.title DROP CONSTRAINT idx_16994_primary;
```
5) Выдаем информацию по текущему состоянию структуры таблиц.
```
employees=# \d+ employees.employee
\d+ employees.department_employee
\d+ employees.department
\d+ employees.department_manager
\d+ employees.salary
\d+ employees.title
                                                                      Table "employees.employee"
   Column   |           Type            | Collation | Nullable |                    Default                     | Storage  | Compression | Stats target | Description
------------+---------------------------+-----------+----------+------------------------------------------------+----------+-------------+--------------+-------------
 id         | bigint                    |           | not null | nextval('employees.id_employee_seq'::regclass) | plain    |             |              |
 birth_date | date                      |           | not null |                                                | plain    |             |              |
 first_name | character varying(14)     |           | not null |                                                | extended |             |              |
 last_name  | character varying(16)     |           | not null |                                                | extended |             |              |
 gender     | employees.employee_gender |           | not null |                                                | plain    |             |              |
 hire_date  | date                      |           | not null |                                                | plain    |             |              |
Access method: heap

                                        Table "employees.department_employee"
    Column     |     Type     | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
---------------+--------------+-----------+----------+---------+----------+-------------+--------------+-------------
 employee_id   | bigint       |           | not null |         | plain    |             |              |
 department_id | character(4) |           | not null |         | extended |             |              |
 from_date     | date         |           | not null |         | plain    |             |              |
 to_date       | date         |           | not null |         | plain    |             |              |
Access method: heap

                                               Table "employees.department"
  Column   |         Type          | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
-----------+-----------------------+-----------+----------+---------+----------+-------------+--------------+-------------
 id        | character(4)          |           | not null |         | extended |             |              |
 dept_name | character varying(40) |           | not null |         | extended |             |              |
Access method: heap

                                        Table "employees.department_manager"
    Column     |     Type     | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
---------------+--------------+-----------+----------+---------+----------+-------------+--------------+-------------
 employee_id   | bigint       |           | not null |         | plain    |             |              |
 department_id | character(4) |           | not null |         | extended |             |              |
 from_date     | date         |           | not null |         | plain    |             |              |
 to_date       | date         |           | not null |         | plain    |             |              |
Access method: heap

                                          Table "employees.salary"
   Column    |  Type  | Collation | Nullable | Default | Storage | Compression | Stats target | Description
-------------+--------+-----------+----------+---------+---------+-------------+--------------+-------------
 employee_id | bigint |           | not null |         | plain   |             |              |
 amount      | bigint |           | not null |         | plain   |             |              |
 from_date   | date   |           | not null |         | plain   |             |              |
 to_date     | date   |           | not null |         | plain   |             |              |
Access method: heap

                                                  Table "employees.title"
   Column    |         Type          | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
-------------+-----------------------+-----------+----------+---------+----------+-------------+--------------+-------------
 employee_id | bigint                |           | not null |         | plain    |             |              |
 title       | character varying(50) |           | not null |         | extended |             |              |
 from_date   | date                  |           | not null |         | plain    |             |              |
 to_date     | date                  |           |          |         | plain    |             |              |
Access method: heap

employees=#
```
## Разработка запросов.
6) Разрабатывыаем свои запрос выборки данных.
```
-- Cотрудники компании зарплаты которых на текущий момент меньше прожиточного минимума на 2025г.
-- Запрос учитывает только актуальных сотрудников, с их актуальными должностями в актуальных департаментах. 
explain (analyze, buffers)
select 
 e2.first_name ||' '|| e2.last_name AS "Имя сотрудника",
 t2.title AS "Должность",
 s.amount AS "Зарплата на сегодня в $",
 67690 - s.amount AS "Недостаток в $" 
 from employees.salary as s
Join employees.employee as e2
 on s.employee_id = e2.id
join employees.title as t2
 on s.employee_id = t2.employee_id 
  where s.employee_id in  
(
select distinct e.id
 from employees.title as t
  join employees.employee as e
   on t.employee_id = e.id
  join employees.department_employee as de
   on t.employee_id = de.employee_id
  join employees.department as d
   on de.department_id = d.id  
  where 
   t.to_date > (select current_date)
   and de.to_date > (select current_date)
)  and (s.from_date >= (select max(from_date) from employees.salary)
        or s.to_date = (select max(to_date) from employees.salary))
   and s.amount < 67690 -- средний прожиточный минимум в США
ORDER BY s.employee_id;
```
7) Разрабатывыаем свои запрос обновления данных.
```
-- Устанавливаем зарплату сотрудникам, у которых она меньше среднего прожиточного минимума, выше этого минимума на 33% от их текущей зарплаты.
-- Запрос строится на основе условий предыдущего запроса (только актуальные сотрудники, а не уже уволенные).
```
9) Проверка производительности запроса на неоптимизированной структуре базы с актуальным планом explain (analyze, buffers)
## Доработка структуры таблиц для запроса.
8) Дорабатываем структуру базы - создаем структуру ключей под разработанный запрос.
9) Дорабатываем структуру базы - создаем индексы под разработанный запрос.
10) ...
## Проверка производительности 
11) Проверка производительности запроса на оптимизированной структуре базы с актуальным планом explain (analyze, buffers)
## Выводы

