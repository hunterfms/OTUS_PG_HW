# Проектная работа по теме - Оптимизация запросов. 

### Цель: Разработать и в дальнейшем оптимизировать запросы с таблиц большого размера, для наиболее быстрой выдачи их результатов и наименьшего использования ими ресурсов сервера.  Оптимизация должна затронуть структуру базы.

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
fedor@pg15:~$ sudo wget https://raw.githubusercontent.com/neondatabase/postgres-sample-dbs/main/employees.sql.gz
--2025-01-16 16:40:01--  https://raw.githubusercontent.com/neondatabase/postgres-sample-dbs/main/employees.sql.gz
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.109.1 33, 185.199.110.133, 185.199.108.133, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.199.109.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 34871756 (33M) [application/octet-stream]
Saving to: ‘employees.sql.gz’

employees.sql.gz    100%[===================>]  33.26M  18.0MB/s    in 1.9s

2025-01-16 16:40:05 (18.0 MB/s) - ‘employees.sql.gz’ saved [34871756/34871756]

fedor@pg15:~$ mc
postgres@pg15:/1_temp$ pg_restore -d employees -Fc /1_temp/employees.sql.gz -c -v --no-owner --no-privileges

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

![Minimum_2018](https://github.com/user-attachments/assets/d09da173-4b3c-4318-8713-b0425aa976fe)
   
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
9) Создаем резервную копию базы  employees без структуры и восстнавливаем её на этом же сервере под именем employees_1 и employees_1. К защите планирую  восстановить три копии базы, что бы продемонстрировать производительность ДО на одной базе и ПОСЛЕ доработки структуры на другой и после нормализации структуры на третей базе. Команды следующие: 
```
pg_dump --username=postgres --format=c --create --compress=9 employees > /1_temp/employees_empty.dump
...
postgres=# create database employees_1;
CREATE DATABASE
postgres=# create database employees_2;
CREATE DATABASE
...
pg_restore -U postgres -d employees_1 < /1_temp/employees_empty.dump
pg_restore -U postgres -d employees_2 < /1_temp/employees_empty.dump
```
11)  Проверка производительности запросов на неоптимизированной структуре базы с актуальным планом explain (analyze)
 а. Выводим актуальный план запроса Select через explain (analyze) который выводит 107728 сотрудников с зарплатами менее прожиточного минимума. Execution Time: 399924.920 ms
```
employees_1=# explain (analyze)
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
   and s.amount < 67690
ORDER BY s.employee_id;
                                                                                                        QUERY PLAN

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
-------------------------------------
 Sort  (cost=197916.99..198262.30 rows=138126 width=67) (actual time=399885.349..399906.434 rows=152067 loops=1)
   Sort Key: s.employee_id
   Sort Method: external merge  Disk: 9584kB
   InitPlan 1 (returns $1)
     ->  Finalize Aggregate  (cost=33927.96..33927.97 rows=1 width=4) (actual time=384.554..384.614 rows=1 loops=1)
           ->  Gather  (cost=33927.75..33927.96 rows=2 width=4) (actual time=379.655..384.590 rows=3 loops=1)
                 Workers Planned: 2
                 Workers Launched: 2
                 ->  Partial Aggregate  (cost=32927.75..32927.76 rows=1 width=4) (actual time=343.822..343.822 rows=1 loops=3)
                       ->  Parallel Seq Scan on salary  (cost=0.00..29965.20 rows=1185020 width=4) (actual time=0.024..164.645 rows=948016 loops=3)
   InitPlan 2 (returns $3)
     ->  Finalize Aggregate  (cost=33927.96..33927.97 rows=1 width=4) (actual time=399.981..400.007 rows=1 loops=1)
           ->  Gather  (cost=33927.75..33927.96 rows=2 width=4) (actual time=397.175..399.974 rows=3 loops=1)
                 Workers Planned: 2
                 Workers Launched: 2
                 ->  Partial Aggregate  (cost=32927.75..32927.76 rows=1 width=4) (actual time=361.938..361.939 rows=1 loops=3)
                       ->  Parallel Seq Scan on salary salary_1  (cost=0.00..29965.20 rows=1185020 width=4) (actual time=0.022..186.820 rows=948016 loops=3)
   ->  Hash Join  (cost=40399.35..112601.61 rows=138126 width=67) (actual time=399539.251..399845.665 rows=152067 loops=1)
         Hash Cond: (s.employee_id = e2.id)
         ->  Seq Scan on salary s  (cost=0.00..67885.82 rows=603231 width=16) (actual time=818.129..1030.598 rows=107728 loops=1)
               Filter: ((amount < 67690) AND ((from_date >= $1) OR (to_date = $3)))
               Rows Removed by Filter: 2736319
         ->  Hash  (cost=39766.23..39766.23 rows=50650 width=50) (actual time=398720.943..398723.740 rows=371243 loops=1)
               Buckets: 131072 (originally 65536)  Batches: 2 (originally 1)  Memory Usage: 17213kB
               ->  Hash Join  (cost=29972.24..39766.23 rows=50650 width=50) (actual time=398421.022..398635.562 rows=371243 loops=1)
                     Hash Cond: (t2.employee_id = e2.id)
                     ->  Seq Scan on title t2  (cost=0.00..7625.08 rows=443308 width=19) (actual time=0.022..23.443 rows=443308 loops=1)
                     ->  Hash  (cost=29543.76..29543.76 rows=34279 width=31) (actual time=398420.923..398423.719 rows=240124 loops=1)
                           Buckets: 131072 (originally 65536)  Batches: 4 (originally 1)  Memory Usage: 7169kB
                           ->  Hash Join  (cost=23274.94..29543.76 rows=34279 width=31) (actual time=398218.801..398368.104 rows=240124 loops=1)
                                 Hash Cond: (e2.id = e.id)
                                 ->  Seq Scan on employee e2  (cost=0.00..5481.24 rows=300024 width=23) (actual time=0.003..16.014 rows=300024 loops=1)
                                 ->  Hash  (cost=22846.46..22846.46 rows=34279 width=8) (actual time=398218.732..398221.527 rows=240124 loops=1)
                                       Buckets: 262144 (originally 65536)  Batches: 2 (originally 1)  Memory Usage: 6736kB
                                       ->  HashAggregate  (cost=22160.88..22503.67 rows=34279 width=8) (actual time=622.174..397961.053 rows=240124 loops=1)
                                             Group Key: e.id
                                             Batches: 202773  Memory Usage: 14129kB  Disk Usage: 10904kB
                                             InitPlan 3 (returns $4)
                                               ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.006..0.006 rows=1 loops=1)
                                             InitPlan 4 (returns $5)
                                               ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=1)
                                             ->  Gather  (cost=18304.46..22075.15 rows=34279 width=8) (actual time=488.772..568.047 rows=240124 loops=1)
                                                   Workers Planned: 1
                                                   Params Evaluated: $4, $5
                                                   Workers Launched: 1
                                                   ->  HashAggregate  (cost=17304.46..17647.25 rows=34279 width=8) (actual time=468.647..492.345 rows=120062 loops=2)
                                                         Group Key: e.id
                                                         Batches: 5  Memory Usage: 8241kB  Disk Usage: 696kB
                                                         Worker 0:  Batches: 5  Memory Usage: 11057kB  Disk Usage: 712kB
                                                         ->  Hash Join  (cost=11638.12..17218.77 rows=34279 width=8) (actual time=255.620..394.377 rows=120062 loops=2)
                                                               Hash Cond: (de.department_id = d.id)
                                                               ->  Parallel Hash Join  (cost=11636.92..16746.23 rows=34279 width=13) (actual time=242.585..348.348 rows=120062 loops=2)
                                                                     Hash Cond: (e.id = t.employee_id)
                                                                     ->  Parallel Seq Scan on employee e  (cost=0.00..4245.85 rows=176485 width=8) (actual time=0.002..15.120 rows=150012 loops=2)
                                                                     ->  Parallel Hash  (cost=11208.43..11208.43 rows=34279 width=21) (actual time=240.827..240.830 rows=120062 loops=2)
                                                                           Buckets: 262144 (originally 65536)  Batches: 1 (originally 1)  Memory Usage: 16768kB
                                                                           ->  Parallel Hash Join  (cost=6270.52..11208.43 rows=34279 width=21) (actual time=89.439..191.563 rows=120062 loops=2)
                                                                                 Hash Cond: (de.employee_id = t.employee_id)
                                                                                 ->  Parallel Seq Scan on department_employee de  (cost=0.00..4551.26 rows=65020 width=13) (actual time=0.021..29.254 rows=120062 loops=2)
                                                                                       Filter: (to_date > $5)
                                                                                       Rows Removed by Filter: 45740
                                                                                 ->  Parallel Hash  (cost=5500.90..5500.90 rows=61570 width=8) (actual time=86.216..86.217 rows=120062 loops=2)
                                                                                       Buckets: 262144  Batches: 1  Memory Usage: 11488kB
                                                                                       ->  Parallel Seq Scan on title t  (cost=0.00..5500.90 rows=61570 width=8) (actual time=0.014..40.049 rows=120062 loops=2)
                                                                                             Filter: (to_date > $4)
                                                                                             Rows Removed by Filter: 101592
                                                               ->  Hash  (cost=1.09..1.09 rows=9 width=5) (actual time=12.996..12.996 rows=9 loops=2)
                                                                     Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                                                     ->  Seq Scan on department d  (cost=0.00..1.09 rows=9 width=5) (actual time=12.983..12.985 rows=9 loops=2)
 Planning Time: 0.524 ms
 JIT:
   Functions: 136
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 8.830 ms, Inlining 0.000 ms, Optimization 6.720 ms, Emission 101.173 ms, Total 116.723 ms
 Execution Time: 399924.920 ms
(75 rows)

employees_1=#
```
 б. Выводим актуальный план запроса Update через explain (analyze)при котором реально обновляются 107728 строки таблицы зарплат employees.salary. Execution Time: 11172.971 ms
```
employees_1=# explain (analyze)
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

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
-----------------------------------------
 Update on salary  (cost=332636.96..413634.19 rows=0 width=0) (actual time=11153.297..11153.325 rows=0 loops=1)
   InitPlan 1 (returns $0)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=332.114..332.115 rows=1 loops=1)
           ->  Seq Scan on salary salary_1  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.009..172.791 rows=2844047 loops=1)
   InitPlan 2 (returns $1)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=353.460..353.461 rows=1 loops=1)
           ->  Seq Scan on salary salary_2  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.004..177.213 rows=2844047 loops=1)
   InitPlan 3 (returns $2)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=333.577..333.578 rows=1 loops=1)
           ->  Seq Scan on salary salary_3  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.004..175.745 rows=2844047 loops=1)
   InitPlan 4 (returns $3)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=346.100..346.101 rows=1 loops=1)
           ->  Seq Scan on salary salary_4  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.005..172.315 rows=2844047 loops=1)
   InitPlan 5 (returns $4)
     ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.007..0.008 rows=1 loops=1)
   ->  Hash Semi Join  (cost=117974.56..198971.79 rows=194931 width=64) (actual time=2760.476..3651.702 rows=107728 loops=1)
         Hash Cond: (salary.employee_id = s.employee_id)
         ->  Seq Scan on salary  (cost=0.00..67885.82 rows=603231 width=22) (actual time=720.248..1013.630 rows=107728 loops=1)
               Filter: ((amount < 67690) AND ((from_date >= $0) OR (to_date = $1)))
               Rows Removed by Filter: 2736319
         ->  Hash  (cost=115925.16..115925.16 rows=78272 width=82) (actual time=2040.013..2040.035 rows=107728 loops=1)
               Buckets: 65536  Batches: 2  Memory Usage: 6847kB
               ->  Hash Join  (cost=45200.13..115925.16 rows=78272 width=82) (actual time=1728.707..2001.055 rows=107728 loops=1)
                     Hash Cond: (s.employee_id = e2.id)
                     ->  Seq Scan on salary s  (cost=0.00..67885.82 rows=603231 width=14) (actual time=679.697..889.882 rows=107728 loops=1)
                           Filter: ((amount < 67690) AND ((from_date >= $2) OR (to_date = $3)))
                           Rows Removed by Filter: 2736319
                     ->  Hash  (cost=44841.35..44841.35 rows=28702 width=68) (actual time=1048.792..1048.813 rows=240124 loops=1)
                           Buckets: 131072 (originally 32768)  Batches: 4 (originally 1)  Memory Usage: 7169kB
                           ->  Hash Join  (cost=35266.85..44841.35 rows=28702 width=68) (actual time=852.310..993.174 rows=240124 loops=1)
                                 Hash Cond: (t2.employee_id = e2.id)
                                 ->  Seq Scan on title t2  (cost=0.00..8733.35 rows=147769 width=14) (actual time=0.042..42.666 rows=240124 loops=1)
                                       Filter: (to_date > $4)
                                       Rows Removed by Filter: 203184
                                 ->  Hash  (cost=34538.41..34538.41 rows=58275 width=54) (actual time=852.040..852.060 rows=240124 loops=1)
                                       Buckets: 131072 (originally 65536)  Batches: 4 (originally 1)  Memory Usage: 7169kB
                                       ->  Hash Join  (cost=28269.60..34538.41 rows=58275 width=54) (actual time=656.314..801.338 rows=240124 loops=1)
                                             Hash Cond: (e2.id = "ANY_subquery".id)
                                             ->  Seq Scan on employee e2  (cost=0.00..5481.24 rows=300024 width=14) (actual time=0.005..26.468 rows=300024 loops=1)
                                             ->  Hash  (cost=27541.16..27541.16 rows=58275 width=40) (actual time=656.120..656.139 rows=240124 loops=1)
                                                   Buckets: 131072 (originally 65536)  Batches: 4 (originally 1)  Memory Usage: 7169kB
                                                   ->  Subquery Scan on "ANY_subquery"  (cost=26375.66..27541.16 rows=58275 width=40) (actual time=522.842..611.429 rows=240124 loops=1)
                                                         ->  HashAggregate  (cost=26375.66..26958.41 rows=58275 width=8) (actual time=522.826..579.239 rows=240124 loops=1)
                                                               Group Key: e.id
                                                               Batches: 5  Memory Usage: 8241kB  Disk Usage: 3816kB
                                                               InitPlan 6 (returns $5)
                                                                 ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=1)
                                                               InitPlan 7 (returns $6)
                                                                 ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=1)
                                                               ->  Hash Join  (cost=18239.59..26229.95 rows=58275 width=8) (actual time=244.973..461.411 rows=240124 loops=1)
                                                                     Hash Cond: (de.department_id = d.id)
                                                                     ->  Hash Join  (cost=18238.38..25427.46 rows=58275 width=13) (actual time=244.945..422.474 rows=240124 loops=1)
                                                                           Hash Cond: (e.id = t.employee_id)
                                                                           ->  Seq Scan on employee e  (cost=0.00..5481.24 rows=300024 width=8) (actual time=0.003..18.489 rows=300024 loops=1)
                                                                           ->  Hash  (cost=17509.95..17509.95 rows=58275 width=21) (actual time=244.753..244.756 rows=240124 loops=1)
                                                                                 Buckets: 262144 (originally 65536)  Batches: 4 (originally 1)  Memory Usage: 8260kB
                                                                                 ->  Hash Join  (cost=7639.71..17509.95 rows=58275 width=21) (actual time=68.836..204.855 rows=240124 loops=1)
                                                                                       Hash Cond: (t.employee_id = de.employee_id)
                                                                                       ->  Seq Scan on title t  (cost=0.00..8733.35 rows=147769 width=8) (actual time=0.007..40.783 rows=240124 loops=1)
                                                                                             Filter: (to_date > $5)
                                                                                             Rows Removed by Filter: 203184
                                                                                       ->  Hash  (cost=6258.04..6258.04 rows=110534 width=13) (actual time=68.484..68.486 rows=240124 loops=1)
                                                                                             Buckets: 262144 (originally 131072)  Batches: 2 (originally 1)  Memory Usage: 7322kB
                                                                                             ->  Seq Scan on department_employee de  (cost=0.00..6258.04 rows=110534 width=13) (actual time=0.012..30.536 rows=240124 loops=1)
                                                                                                   Filter: (to_date > $6)
                                                                                                   Rows Removed by Filter: 91479
                                                                     ->  Hash  (cost=1.09..1.09 rows=9 width=5) (actual time=0.017..0.018 rows=9 loops=1)
                                                                           Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                                                           ->  Seq Scan on department d  (cost=0.00..1.09 rows=9 width=5) (actual time=0.011..0.012 rows=9 loops=1)
 Planning Time: 0.909 ms
 JIT:
   Functions: 93
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 3.043 ms, Inlining 0.000 ms, Optimization 1.483 ms, Emission 35.537 ms, Total 40.063 ms
 Execution Time: 11172.971 ms
(75 rows)

employees_1=#

```
### Доработка структуры таблиц для запроса.
8) Дорабатываем структуру базы - создаем структуру ключей под разработанный запрос.
9) Дорабатываем структуру базы - создаем индексы под разработанный запрос.
10) ...
### Проверка производительности запросов после доработки структуры. 
11) Проверка производительности запроса на оптимизированной структуре базы с актуальным планом explain (analyze, buffers)

### Выводы и рекомендации по нормализации данных.
а. Было бы эффективнее упразднить таблицу справочник руководителей подразделений department_manager. Данные этой таблицы хоть и позволяют отслеживать историю смены руководителей но такая история уже есть совокупно в таблицах привязки сотрудника к департаменту с таблицей должностей. Иными словами можно выдать историю смены руководителей отдела связанным запросом, а не занимать под эти «результатирующие данные» диски (сорные данные).
б. Поле title таблицы employees.title имеет повторяющиеся значения, поэтому целесообразным было бы вынести значения данного поля в отдельный справочник, а в employees.title оставить только id на каждую должность из нового справочника.

12) По пункту "б" создаем таблицу справочник titles для должностей м заполняем её данными.
```
-- какие данные будут внесены в справочник должностей
employees_2=# select distinct title from employees.title;
       title
--------------------
 Assistant Engineer
 Engineer
 Manager
 Senior Engineer
 Senior Staff
 Staff
 Technique Leader
(7 rows)

-- создаем таблицу справочник и заполняем её
employees_2=# create table employees.titles (title_id int PRIMARY KEY, title varchar(50) not null);
CREATE TABLE

employees_2=# INSERT INTO employees.titles (title_id ,title) values (1, 'Assistant Engineer');
INSERT INTO employees.titles (title_id ,title) values (2, 'Engineer');
INSERT INTO employees.titles (title_id ,title) values (3, 'Manager');
INSERT INTO employees.titles (title_id ,title) values (4, 'Senior Engineer');
INSERT INTO employees.titles (title_id ,title) values (5, 'Senior Staff');
INSERT INTO employees.titles (title_id ,title) values (6, 'Staff');
INSERT INTO employees.titles (title_id ,title) values (7, 'Technique Leader');
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
```
13) Добавляем в таблицу title поле title_id и заполняем его из titles.
```
employees_2=# alter table employees.title ADD COLUMN title_id integer;
ALTER TABLE

employees_2=# update employees.title set title_id = 1 where title = 'Assistant Engineer';
update employees.title set title_id = 2 where title = 'Engineer';
update employees.title set title_id = 3 where title = 'Manager';
update employees.title set title_id = 4 where title = 'Senior Engineer';
update employees.title set title_id = 5 where title = 'Senior Staff';
update employees.title set title_id = 6 where title = 'Staff';
update employees.title set title_id = 7 where title = 'Technique Leader';
UPDATE 15128
UPDATE 115003
UPDATE 24
UPDATE 97750
UPDATE 92853
UPDATE 107391
UPDATE 15159
employees_2=#
```
14) Удаляем избыточные данные из таблицы title.
```
employees_2=# alter table employees.title DROP COLUMN title;
ALTER TABLE
employees_2=# select * from employees.title limit 5;
 employee_id | from_date  |  to_date   | title_id
-------------+------------+------------+----------
       13283 | 1987-09-22 | 1995-09-22 |        2
       13652 | 1995-01-30 | 9999-01-01 |        2
       29161 | 1997-08-31 | 9999-01-01 |        2
       93144 | 1985-12-30 | 1994-12-30 |        2
       10001 | 1986-06-26 | 9999-01-01 |        4
(5 rows)
employees_2=# vacuum full;
VACUUM
employees_2=# select * from employees.titles;
 title_id |       title
----------+--------------------
        1 | Assistant Engineer
        2 | Engineer
        3 | Manager
        4 | Senior Engineer
        5 | Senior Staff
        6 | Staff
        7 | Technique Leader
(7 rows)
```
Теперь структура будет занимать меньше места на диске из за смены типов данных vatchar(50) на int в таблице 443 000 записей. Для новой структуры нужно будет доработать наши запросы с учетом изменений.
```
employees_2=# \d+ employees.title
\d+ employees.titles
                                           Table "employees.title"
   Column    |  Type   | Collation | Nullable | Default | Storage | Compression | Stats target | Description
-------------+---------+-----------+----------+---------+---------+-------------+--------------+-------------
 employee_id | bigint  |           | not null |         | plain   |             |              |
 from_date   | date    |           | not null |         | plain   |             |              |
 to_date     | date    |           |          |         | plain   |             |              |
 title_id    | integer |           |          |         | plain   |             |              |
Access method: heap

                                                Table "employees.titles"
  Column  |         Type          | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
----------+-----------------------+-----------+----------+---------+----------+-------------+--------------+-------------
 title_id | integer               |           | not null |         | plain    |             |              |
 title    | character varying(50) |           | not null |         | extended |             |              |
Indexes:
    "titles_pkey" PRIMARY KEY, btree (title_id)
Access method: heap

employees_2=#
```

Вид доработанных запросов с планами.
```
```

## Выводы

