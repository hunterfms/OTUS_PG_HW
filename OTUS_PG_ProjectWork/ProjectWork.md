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
6) Разрабатывыаем свой запрос выборки данных Select.
```
-- Cотрудники компании зарплаты у которых на текущий момент меньше прожиточного минимума на 2025г.
-- Запрос учитывает только актуальных сотрудников, с их актуальными должностями в актуальных департаментах. 
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
7) Разрабатывыаем свои запрос обновления данных Update.
```
-- Устанавливаем зарплату сотрудникам, у которых она меньше среднего прожиточного минимума, выше этого минимума на 33% от их текущей зарплаты.
-- Запрос строится на основе условий предыдущего запроса (только актуальные сотрудники, а не уволенные).
update employees.salary
 set amount = (67690 + (amount/3))
where employee_id in 
(select 
 s.employee_id
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
   and t2.to_date > (select current_date))
and (from_date >= (select max(from_date) from employees.salary)
        or to_date = (select max(to_date) from employees.salary))
and amount < 67690;
```
9) Создаем резервную копию базы  employees без структуры и восстнавливаем её на этом же сервере под именем employees_2. К защите планирую  восстановить две копии базы, что бы продемонстрировать производительность ДО на одной базе и ПОСЛЕ доработки структуры на другой. Команды следующие: 
```
pg_dump --username=postgres --format=c --create --compress=9 employees > /1_temp/employees_empty.dump
...
postgres=# create database employees_2;
CREATE DATABASE
...
pg_restore -U postgres -d employees_2 < /1_temp/employees_empty.dump
```
11)  Проверка производительности запросов на неоптимизированной структуре базы с актуальным планом explain (analyze)
 а. Выводим актуальный план запроса Select через explain (analyze) который выводит 107728 сотрудников с зарплатами менее прожиточного минимума.
```
employees=# explain (analyze)
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
                                                                                                     QUERY PLAN

----------------------------------------------------------------------------------------------------------------------------------------------------------
-----------------------------------------------------------
 Sort  (cost=263426.03..264959.28 rows=613300 width=67) (actual time=3191.480..3209.480 rows=152067 loops=1)
   Sort Key: s.employee_id
   Sort Method: quicksort  Memory: 20145kB
   InitPlan 1 (returns $1)
     ->  Finalize Aggregate  (cost=33927.96..33927.97 rows=1 width=4) (actual time=517.932..517.992 rows=1 loops=1)
           ->  Gather  (cost=33927.75..33927.96 rows=2 width=4) (actual time=510.116..517.968 rows=3 loops=1)
                 Workers Planned: 2
                 Workers Launched: 2
                 ->  Partial Aggregate  (cost=32927.75..32927.76 rows=1 width=4) (actual time=462.270..462.271 rows=1 loops=3)
                       ->  Parallel Seq Scan on salary  (cost=0.00..29965.20 rows=1185020 width=4) (actual time=0.009..189.288 rows=948016 loops=3)
   InitPlan 2 (returns $3)
     ->  Finalize Aggregate  (cost=33927.96..33927.97 rows=1 width=4) (actual time=531.018..531.044 rows=1 loops=1)
           ->  Gather  (cost=33927.75..33927.96 rows=2 width=4) (actual time=530.891..531.033 rows=3 loops=1)
                 Workers Planned: 2
                 Workers Launched: 2
                 ->  Partial Aggregate  (cost=32927.75..32927.76 rows=1 width=4) (actual time=482.477..482.478 rows=1 loops=3)
                       ->  Parallel Seq Scan on salary salary_1  (cost=0.00..29965.20 rows=1185020 width=4) (actual time=0.009..232.334 rows=948016 loops=
3)
   ->  Hash Join  (cost=57204.62..136612.84 rows=613300 width=67) (actual time=2734.407..3148.735 rows=152067 loops=1)
         Hash Cond: (s.employee_id = e2.id)
         ->  Seq Scan on salary s  (cost=0.00..67885.82 rows=605023 width=16) (actual time=1106.139..1464.209 rows=107728 loops=1)
               Filter: ((amount < 67690) AND ((from_date >= $1) OR (to_date = $3)))
               Rows Removed by Filter: 2736319
         ->  Hash  (cost=54319.91..54319.91 rows=230777 width=50) (actual time=1628.098..1632.125 rows=371243 loops=1)
               Buckets: 524288 (originally 262144)  Batches: 1 (originally 1)  Memory Usage: 36412kB
               ->  Hash Join  (cost=42724.65..54319.91 rows=230777 width=50) (actual time=1119.529..1408.987 rows=371243 loops=1)
                     Hash Cond: (t2.employee_id = e2.id)
                     ->  Seq Scan on title t2  (cost=0.00..7625.08 rows=443308 width=19) (actual time=0.008..64.285 rows=443308 loops=1)
                     ->  Hash  (cost=40772.33..40772.33 rows=156186 width=31) (actual time=1118.902..1122.928 rows=240124 loops=1)
                           Buckets: 262144  Batches: 1  Memory Usage: 17636kB
                           ->  Hash Join  (cost=34503.52..40772.33 rows=156186 width=31) (actual time=847.749..1022.548 rows=240124 loops=1)
                                 Hash Cond: (e2.id = e.id)
                                 ->  Seq Scan on employee e2  (cost=0.00..5481.24 rows=300024 width=23) (actual time=0.005..36.294 rows=300024 loops=1)
                                 ->  Hash  (cost=32551.19..32551.19 rows=156186 width=8) (actual time=847.030..851.055 rows=240124 loops=1)
                                       Buckets: 262144  Batches: 1  Memory Usage: 11428kB
                                       ->  HashAggregate  (cost=29427.47..30989.33 rows=156186 width=8) (actual time=766.539..812.770 rows=240124 loops=1)
                                             Group Key: e.id
                                             Batches: 1  Memory Usage: 24593kB
                                             InitPlan 3 (returns $4)
                                               ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.004..0.004 rows=1 loops=1)
                                             InitPlan 4 (returns $5)
                                               ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=1)
                                             ->  Hash Join  (cost=12644.79..29036.98 rows=156186 width=8) (actual time=401.863..664.486 rows=240124 loops=1)
                                                   Hash Cond: (de.department_id = d.id)
                                                   ->  Gather  (cost=12622.42..23406.10 rows=56795 width=13) (actual time=401.836..617.211 rows=240124 loops=1)
                                                         Workers Planned: 1
                                                         Params Evaluated: $4, $5
                                                         Workers Launched: 1
                                                         ->  Parallel Hash Join  (cost=11622.42..16726.60 rows=33409 width=13) (actual time=370.531..442.911 rows=120062 loops=2)
                                                               Hash Cond: (e.id = t.employee_id)
                                                               ->  Parallel Seq Scan on employee e  (cost=0.00..4245.85 rows=176485 width=8) (actual time=0.008..11.062 rows=150012 loops=2)
                                                               ->  Parallel Hash  (cost=11204.80..11204.80 rows=33409 width=21) (actual time=368.377..368.380 rows=120062 loops=2)
                                                                     Buckets: 262144 (originally 65536)  Batches: 1 (originally 1)  Memory Usage: 16768kB
                                                                     ->  Parallel Hash Join  (cost=6270.52..11204.80 rows=33409 width=21) (actual time=146.201..311.975 rows=120062 loops=2)
                                                                           Hash Cond: (de.employee_id = t.employee_id)
                                                                           ->  Parallel Seq Scan on department_employee de  (cost=0.00..4551.26 rows=65020 width=13) (actual time=0.022..40.823 rows=120062 loops=2)
                                                                                 Filter: (to_date > $5)
                                                                                 Rows Removed by Filter: 45740
                                                                           ->  Parallel Hash  (cost=5500.90..5500.90 rows=61570 width=8) (actual time=143.707..143.707 rows=120062 loops=2)
                                                                                 Buckets: 262144  Batches: 1  Memory Usage: 11488kB
                                                                                 ->  Parallel Seq Scan on title t  (cost=0.00..5500.90 rows=61570 width=8) (actual time=4.913..82.264 rows=120062 loops=2)
                                                                                       Filter: (to_date > $4)
                                                                                       Rows Removed by Filter: 101592
                                                   ->  Hash  (cost=15.50..15.50 rows=550 width=20) (actual time=0.015..0.016 rows=9 loops=1)
                                                         Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                                         ->  Seq Scan on department d  (cost=0.00..15.50 rows=550 width=20) (actual time=0.009..0.010 rows=9 loops=1)
 Planning Time: 1.235 ms JIT: Functions: 113
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 4.922 ms, Inlining 0.000 ms, Optimization 2.750 ms, Emission 99.943 ms, Total 107.615 ms
 Execution Time: 3231.117 ms
(71 rows)

employees=#
```
 б. Выводим актуальный план запроса Update через explain (analyze)при котором реально обновляются 107728 строки таблицы зарплат employees.salary.
```
employees=# explain (analyze)
update employees.salary
 set amount = (67690 + (amount/3))
where employee_id in
(select
 s.employee_id
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
   and t2.to_date > (select current_date))
and (from_date >= (select max(from_date) from employees.salary)
        or to_date = (select max(to_date) from employees.salary))
and amount < 67690;
                                                                                                             QUERY PLAN

----------------------------------------------------------------------------------------------------------------------------------------------------------
---------------------------------------------------------------------------
 Update on salary  (cost=346571.57..424024.35 rows=0 width=0) (actual time=6794.243..6794.253 rows=0 loops=1)
   InitPlan 1 (returns $0)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=390.346..390.346 rows=1 loops=1)
           ->  Seq Scan on salary salary_1  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.011..225.966 rows=2844047 loops=1)
   InitPlan 2 (returns $1)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=344.333..344.333 rows=1 loops=1)
           ->  Seq Scan on salary salary_2  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.008..159.103 rows=2844047 loops=1)
   InitPlan 3 (returns $2)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=327.257..327.257 rows=1 loops=1)
           ->  Seq Scan on salary salary_3  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.003..156.626 rows=2844047 loops=1)
   InitPlan 4 (returns $3)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=336.225..336.225 rows=1 loops=1)
           ->  Seq Scan on salary salary_4  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.007..157.175 rows=2844047 loops=1)
   InitPlan 5 (returns $4)
     ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.005..0.005 rows=1 loops=1)
   ->  Hash Join  (cost=131909.17..209361.95 rows=494970 width=64) (actual time=3329.449..3582.656 rows=107728 loops=1)
         Hash Cond: (salary.employee_id = s.employee_id)
         ->  Seq Scan on salary  (cost=0.00..67885.82 rows=604026 width=22) (actual time=771.464..981.877 rows=107728 loops=1)
               Filter: ((amount < 67690) AND ((from_date >= $0) OR (to_date = $1)))
               Rows Removed by Filter: 2736319
         ->  Hash  (cost=129361.16..129361.16 rows=203841 width=82) (actual time=2557.215..2557.222 rows=107728 loops=1)
               Buckets: 262144  Batches: 1  Memory Usage: 14673kB
               ->  HashAggregate  (cost=127322.75..129361.16 rows=203841 width=82) (actual time=2502.209..2524.215 rows=107728 loops=1)
                     Group Key: s.employee_id
                     Batches: 1  Memory Usage: 22545kB
                     ->  Hash Join  (cost=55107.62..126813.14 rows=203841 width=82) (actual time=2222.803..2454.982 rows=107728 loops=1)
                           Hash Cond: (s.employee_id = e2.id)
                           ->  Seq Scan on salary s  (cost=0.00..67885.82 rows=604026 width=14) (actual time=663.509..858.166 rows=107728 loops=1)
                                 Filter: ((amount < 67690) AND ((from_date >= $2) OR (to_date = $3)))
                                 Rows Removed by Filter: 2736319
                           ->  Hash  (cost=54142.40..54142.40 rows=77218 width=68) (actual time=1558.851..1558.857 rows=240124 loops=1)
                                 Buckets: 262144 (originally 131072)  Batches: 1 (originally 1)  Memory Usage: 26436kB
                                 ->  Hash Join  (cost=46696.56..54142.40 rows=77218 width=68) (actual time=1372.819..1490.306 rows=240124 loops=1)
                                       Hash Cond: (e2.id = t2.employee_id)
                                       ->  Seq Scan on employee e2  (cost=0.00..5481.24 rows=300024 width=14) (actual time=0.062..33.247 rows=300024 loops=1)
                                       ->  Hash  (cost=45647.17..45647.17 rows=83951 width=54) (actual time=1372.160..1372.165 rows=240124 loops=1)
                                             Buckets: 262144 (originally 131072)  Batches: 1 (originally 1)  Memory Usage: 22684kB
                                             ->  Hash Join  (cost=36525.92..45647.17 rows=83951 width=54) (actual time=1186.114..1311.040 rows=240124 loops=1)
                                                   Hash Cond: (t2.employee_id = "ANY_subquery".id)
                                                   ->  Seq Scan on title t2  (cost=0.00..8733.35 rows=147769 width=14) (actual time=0.077..43.876 rows=240124 loops=1)
                                                         Filter: (to_date > $4)
                                                         Rows Removed by Filter: 203184
                                                   ->  Hash  (cost=34566.17..34566.17 rows=156780 width=40) (actual time=1184.989..1184.994 rows=240124 loops=1)
                                                         Buckets: 262144  Batches: 1  Memory Usage: 18932kB
                                                         ->  Subquery Scan on "ANY_subquery"  (cost=31430.57..34566.17 rows=156780 width=40) (actual time=1066.882..1136.887 rows=240124 loops=1)
                                                               ->  HashAggregate  (cost=31430.57..32998.37 rows=156780 width=8) (actual time=1066.873..1105.158 rows=240124 loops=1)
                                                                     Group Key: e.id
                                                                     Batches: 1  Memory Usage: 24593kB
                                                                     InitPlan 6 (returns $5)
                                                                       ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.003 rows=1 loops=1)
                                                                     InitPlan 7 (returns $6)
                                                                       ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=1)
                                                                     ->  Hash Join  (cost=18232.32..31038.59 rows=156780 width=8) (actual time=848.957..997.662 rows=240124 loops=1)
                                                                           Hash Cond: (de.department_id = d.id)
                                                                           ->  Hash Join  (cost=18209.94..25386.38 rows=57011 width=13) (actual time=848.908..960.441 rows=240124 loops=1)
                                                                                 Hash Cond: (e.id = t.employee_id)
                                                                                 ->  Seq Scan on employee e  (cost=0.00..5481.24 rows=300024 width=8) (actual time=0.004..24.860 rows=300024 loops=1)
                                                                                 ->  Hash  (cost=17497.31..17497.31 rows=57011 width=21) (actual time=848.702..848.704 rows=240124 loops=1)
                                                                                       Buckets: 262144 (originally 65536)  Batches: 1 (originally 1)  Memory Usage: 14477kB
                                                                                       ->  Hash Join  (cost=7639.71..17497.31 rows=57011 width=21) (actual time=70.230..801.014 rows=240124 loops=1)
                                                                                             Hash Cond: (t.employee_id = de.employee_id)
                                                                                             ->  Seq Scan on title t  (cost=0.00..8733.35 rows=147769 width=8) (actual time=0.010..660.078 rows=240124 loops=1)
                                                                                                   Filter: (to_date > $5)
                                                                                                   Rows Removed by Filter: 203184
                                                                                             ->  Hash  (cost=6258.04..6258.04 rows=110534 width=13) (actual time=69.865..69.866 rows=240124 loops=1)
                                                                                                   Buckets: 262144 (originally 131072)  Batches: 1 (originally 1)  Memory Usage: 12601kB
                                                                                                   ->  Seq Scan on department_employee de  (cost=0.00..6258.04 rows=110534 width=13) (actual time=0.026..30.428 rows=240124 loops=1)
                                                                                                         Filter: (to_date > $6)
                                                                                                         Rows Removed by Filter: 91479
                                                                           ->  Hash  (cost=15.50..15.50 rows=550 width=20) (actual time=0.034..0.035 rows=9 loops=1)
                                                                                 Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                                                                 ->  Seq Scan on department d  (cost=0.00..15.50 rows=550 width=20) (actual time=0.028..0.029 rows=9 loops=1)
 Planning Time: 1.929 ms
 JIT:
   Functions: 96
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 3.312 ms, Inlining 0.000 ms, Optimization 1.548 ms, Emission 35.962 ms, Total 40.822 ms
 Execution Time: 6822.470 ms
(78 rows)
employees=#
```
## Доработка структуры таблиц для запроса.
8) Дорабатываем структуру базы - создаем структуру ключей под разработанный запрос.
9) Дорабатываем структуру базы - создаем индексы под разработанный запрос.
10) ...
## Проверка производительности запросов после доработки структуры. 
11) Проверка производительности запроса на оптимизированной структуре базы с актуальным планом explain (analyze, buffers)
## Выводы

