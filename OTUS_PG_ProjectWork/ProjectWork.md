# Проектная работа по теме - Оптимизация запросов. 

### Цель: Разработать и в дальнейшем оптимизировать запросы с таблиц большого размера, для наиболее быстрой выдачи их результатов и наименьшего использования ими ресурсов сервера.  Оптимизация должна затронуть структуру базы.

Восстанавливаем на виртуальной машине PG15 DataSet "Employees database" с общим количеством строк в таблицах 3919015, с ресурса: https://raw.githubusercontent.com/neondatabase/postgres-sample-dbs/main/employees.sql.gz

Конфигурационный файл виртуальной СУБД Postgresql 15 настроен по рекомендациям PGTune
```
# DB Version: 15
# OS Type: linux
# DB Type: desktop
# Total Memory (RAM): 1 GB
# CPUs num: 1
# Connections num: 100
# Data Storage: hdd

max_connections = 100
shared_buffers = 64MB
effective_cache_size = 256MB
maintenance_work_mem = 64MB
checkpoint_completion_target = 0.9
wal_buffers = 1966kB
default_statistics_target = 100
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 273kB
huge_pages = off
min_wal_size = 100MB
max_wal_size = 2GB
```

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
6) Создаем резервную копию базы  employees без структуры и восстнавливаем её на этом же сервере еще в двух экземплярах employees_1 и employees_2. К защите планирую  восстановить три копии базы, что бы продемонстрировать производительность запросов Select и Update ДО оптимизации на базе employees и ПОСЛЕ оптимизации структуры на employees_1. На третьей базе employees_2 буду демонстрировать производительность третьего запроса уже с учетом нормализованной структуры данных. Команды подготовки бакапа и восстановления следующие: 
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

## Разработка запросов под текущую структуру.

Планирую выполнить обновление строк в базе одним запросом и оценить влияние индексов на оператор Update. Предполагаю - чем больше будет индексов улучшающих работу оператора Select, тем мене производительнее будет работать запрос Update ориентированный на теже данные. 

7) Разрабатывыаем свой запрос выборки данных Select.

![Minimum_2018](https://github.com/user-attachments/assets/d09da173-4b3c-4318-8713-b0425aa976fe)
   
```
-- Cотрудники компании зарплаты у которых на текущий момент меньше прожиточного минимума на 2025г.
-- Запрос учитывает только актуальных сотрудников, с их актуальными должностями в актуальных департаментах. 
select 
 e.first_name ||' '|| e.last_name AS "Имя сотрудника",
 t.title AS "Должность",
 s.amount AS "Зарплата на сегодня в $",
 67690 - s.amount AS "Недостаток в $"
 from employees.salary as s
Join employees.employee as e
 on s.employee_id = e.id
join employees.title as t
 on s.employee_id = t.employee_id 
join employees.department_employee as de
 on e.id = de.employee_id
join employees.department as d
 on d.id = de.department_id 
 where
  s.from_date >= (select max(from_date) from employees.salary)
   and s.to_date = (select max(to_date) from employees.salary)
   and t.to_date > (select current_date)
   and de.to_date > (select current_date)
   and s.amount < 67690 
ORDER BY s.amount;
```
Результат данного запроса из limit 10:
```
    Имя сотрудника    |     Должность      | Зарплата на сегодня в $ | Недостаток в $
----------------------+--------------------+-------------------------+----------------
 Woody Porenta        | Senior Engineer    |                   40960 |          26730
 Marc Molenaar        | Assistant Engineer |                   41325 |          26365
 Francoise Emden      | Engineer           |                   41595 |          26095
 Margo Gihr           | Senior Engineer    |                   42895 |          24795
 Neven Domenig        | Staff              |                   43305 |          24385
 Taiji Mullainathan   | Staff              |                   44354 |          23336
 Koldo Katzenelson    | Assistant Engineer |                   44775 |          22915
 Ioana Liedekerke     | Engineer           |                   44785 |          22905
 Gully Nittel         | Staff              |                   45363 |          22327
 Christoper Kropatsch | Engineer           |                   45457 |          22233
(10 rows)
```
8) Разрабатывыаем свои запрос обновления данных Update.
```
-- Изменям значение последней выданной зарплаты сотрудникам, у которых она меньше среднего прожиточного минимума, выше этого минимума на 33% от их текущей зарплаты.
-- Запрос строится на основе условий предыдущего запроса (только актуальные сотрудники, а не уволенные).
update employees.salary
 set amount = (67690 + (amount/3))
where employee_id in
 (select s.employee_id 
  from employees.salary as s
  Join employees.employee as e
   on s.employee_id = e.id
  join employees.title as t
   on s.employee_id = t.employee_id 
  join employees.department_employee as de
   on e.id = de.employee_id
  join employees.department as d
   on d.id = de.department_id 
  where
   s.from_date >= (select max(from_date) from employees.salary)
   and s.to_date = (select max(to_date) from employees.salary)
   and t.to_date > (select current_date)
   and de.to_date > (select current_date)
   and s.amount < 67690
)
and from_date >= (select max(from_date) from employees.salary)
and to_date = (select max(to_date) from employees.salary)
and amount < 67690;
```
9)  Проверка производительности запросов на неоптимизированной структуре базы с актуальным планом explain (analyze)
 а. Выводим актуальный план запроса Select через explain (analyze) который выводит сотрудников с зарплатами менее прожиточного минимума. Execution Time: 1260.088 ms. Первый запуск длился в пределах Execution Time: 411367.804 ms с той же структурой плана в резульаттах, скорее сего из за расчета первого плана запроса. Далее мной применялся ANALYSE;, что бы учесть статистику первого запуска и применить уже более оптимальный план выполнения.
```
employees=# analyse;
ANALYZE
employees=# explain (analyze, buffers)
select
 e.first_name ||' '|| e.last_name AS "Имя сотрудника",
 t.title AS "Должность",
 s.amount AS "Зарплата на сегодня в $",
 67690 - s.amount AS "Недостаток в $"
 from employees.salary as s
Join employees.employee as e
 on s.employee_id = e.id
join employees.title as t
 on s.employee_id = t.employee_id
join employees.department_employee as de
 on e.id = de.employee_id
join employees.department as d
 on d.id = de.department_id
 where
  s.from_date >= (select max(from_date) from employees.salary)
   and s.to_date = (select max(to_date) from employees.salary)
   and t.to_date > (select current_date)
   and de.to_date > (select current_date)
   and s.amount < 67690
ORDER BY s.amount;
                                                                                QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Gather Merge  (cost=123837.40..123839.01 rows=14 width=59) (actual time=1257.061..1258.292 rows=278 loops=1)
   Workers Planned: 1
   Params Evaluated: $1, $3, $4, $5
   Workers Launched: 1
   Buffers: shared hit=8923 read=53289
   InitPlan 1 (returns $1)
     ->  Finalize Aggregate  (cost=33927.96..33927.97 rows=1 width=4) (actual time=435.822..435.866 rows=1 loops=1)
           Buffers: shared hit=256 read=17859
           ->  Gather  (cost=33927.75..33927.96 rows=2 width=4) (actual time=432.980..435.780 rows=3 loops=1)
                 Workers Planned: 2
                 Workers Launched: 2
                 Buffers: shared hit=256 read=17859
                 ->  Partial Aggregate  (cost=32927.75..32927.76 rows=1 width=4) (actual time=394.460..394.461 rows=1 loops=3)
                       Buffers: shared hit=256 read=17859
                       ->  Parallel Seq Scan on salary  (cost=0.00..29965.20 rows=1185020 width=4) (actual time=0.152..192.695 rows=948016 loops=3)
                             Buffers: shared hit=256 read=17859
   InitPlan 2 (returns $3)
     ->  Finalize Aggregate  (cost=33927.96..33927.97 rows=1 width=4) (actual time=433.605..433.627 rows=1 loops=1)
           Buffers: shared hit=352 read=17763
           ->  Gather  (cost=33927.75..33927.96 rows=2 width=4) (actual time=433.595..433.619 rows=3 loops=1)
                 Workers Planned: 2
                 Workers Launched: 2
                 Buffers: shared hit=352 read=17763
                 ->  Partial Aggregate  (cost=32927.75..32927.76 rows=1 width=4) (actual time=396.721..396.721 rows=1 loops=3)
                       Buffers: shared hit=352 read=17763
                       ->  Parallel Seq Scan on salary salary_1  (cost=0.00..29965.20 rows=1185020 width=4) (actual time=0.030..177.697 rows=948016 loops=3)
                             Buffers: shared hit=352 read=17763
   InitPlan 3 (returns $4)
     ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.004..0.004 rows=1 loops=1)
   InitPlan 4 (returns $5)
     ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=1)
   ->  Sort  (cost=54981.42..54981.46 rows=14 width=59) (actual time=362.411..362.424 rows=139 loops=2)
         Sort Key: s.amount
         Sort Method: quicksort  Memory: 38kB
         Buffers: shared hit=8315 read=17667
         Worker 0:  Sort Method: quicksort  Memory: 35kB
         ->  Hash Join  (cost=49852.51..54981.16 rows=14 width=59) (actual time=318.432..362.330 rows=139 loops=2)
               Hash Cond: (de.department_id = d.id)
               Buffers: shared hit=8278 read=17667
               ->  Parallel Hash Join  (cost=49851.31..54979.66 rows=14 width=39) (actual time=298.108..341.943 rows=139 loops=2)
                     Hash Cond: (e.id = s.employee_id)
                     Buffers: shared hit=8246 read=17667
                     ->  Parallel Seq Scan on employee e  (cost=0.00..4245.85 rows=176485 width=23) (actual time=0.005..14.608 rows=150012 loops=2)
                           Buffers: shared hit=2481
                     ->  Parallel Hash  (cost=49851.13..49851.13 rows=14 width=48) (actual time=296.165..296.168 rows=139 loops=2)
                           Buckets: 1024  Batches: 1  Memory Usage: 72kB
                           Buffers: shared hit=5753 read=17667
                           ->  Parallel Hash Join  (cost=44893.44..49851.13 rows=14 width=48) (actual time=249.874..296.076 rows=139 loops=2)
                                 Hash Cond: (de.employee_id = s.employee_id)
                                 Buffers: shared hit=5753 read=17667
                                 ->  Parallel Seq Scan on department_employee de  (cost=0.00..4551.26 rows=65020 width=13) (actual time=0.020..33.176 rows=120062 loops=2)
                                       Filter: (to_date > $5)
                                       Rows Removed by Filter: 45740
                                       Buffers: shared hit=2113
                                 ->  Parallel Hash  (cost=44893.14..44893.14 rows=24 width=35) (actual time=247.681..247.683 rows=139 loops=2)
                                       Buckets: 1024  Batches: 1  Memory Usage: 72kB
                                       Buffers: shared hit=3640 read=17667
                                       ->  Parallel Hash Join  (cost=38853.41..44893.14 rows=24 width=35) (actual time=189.142..247.568 rows=139 loops=2)
                                             Hash Cond: (t.employee_id = s.employee_id)
                                             Buffers: shared hit=3640 read=17667
                                             ->  Parallel Seq Scan on title t  (cost=0.00..5500.90 rows=61570 width=19) (actual time=0.022..37.508 rows=120062 loops=2)
                                                   Filter: (to_date > $4)
                                                   Rows Removed by Filter: 101592
                                                   Buffers: shared hit=3192
                                             ->  Parallel Hash  (cost=38852.84..38852.84 rows=45 width=16) (actual time=186.094..186.095 rows=139 loops=2)
                                                   Buckets: 1024  Batches: 1  Memory Usage: 72kB
                                                   Buffers: shared hit=448 read=17667
                                                   ->  Parallel Seq Scan on salary s  (cost=0.00..38852.84 rows=45 width=16) (actual time=1.581..185.915 rows=139 loops=2)
                                                         Filter: ((from_date >= $1) AND (amount < 67690) AND (to_date = $3))
                                                         Rows Removed by Filter: 1421884
                                                         Buffers: shared hit=448 read=17667
               ->  Hash  (cost=1.09..1.09 rows=9 width=5) (actual time=20.289..20.289 rows=9 loops=2)
                     Buckets: 1024  Batches: 1  Memory Usage: 9kB
                     Buffers: shared hit=2
                     ->  Seq Scan on department d  (cost=0.00..1.09 rows=9 width=5) (actual time=20.275..20.277 rows=9 loops=2)
                           Buffers: shared hit=2
 Planning:
   Buffers: shared hit=76
 Planning Time: 0.660 ms
 JIT:
   Functions: 108
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 4.111 ms, Inlining 0.000 ms, Optimization 2.504 ms, Emission 137.435 ms, Total 144.050 ms
 Execution Time: 1260.088 ms
(84 rows)
```
В выведенном плане наблюдаем в основном блоки Seq Scan из за отсутсвия индексов и соответсвенно высокую длительность обработки Execution Time.

 б. Выводим актуальный план запроса Update через explain (analyze, buffers) при котором реально обновляются данные таблицы зарплат employees.salary. Execution Execution Time: 2427.629 ms
```
employees=# explain (analyze, buffers)
update employees.salary
 set amount = (67690 + (amount/3))
where employee_id in
 (select s.employee_id
  from employees.salary as s
  Join employees.employee as e
   on s.employee_id = e.id
  join employees.title as t
   on s.employee_id = t.employee_id
  join employees.department_employee as de
   on e.id = de.employee_id
  join employees.department as d
   on d.id = de.department_id
  where
   s.from_date >= (select max(from_date) from employees.salary)
   and s.to_date = (select max(to_date) from employees.salary)
   and t.to_date > (select current_date)
   and de.to_date > (select current_date)
   and s.amount < 67690
)
and from_date >= (select max(from_date) from employees.salary)
and to_date = (select max(to_date) from employees.salary)
and amount < 67690;
                                                                                        QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Update on salary  (cost=240763.57..376573.10 rows=0 width=0) (actual time=2405.081..2405.089 rows=0 loops=1)
   Buffers: shared hit=12463 read=105140 dirtied=284 written=2, temp read=6162 written=6162
   InitPlan 1 (returns $0)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=370.152..370.153 rows=1 loops=1)
           Buffers: shared hit=513 read=17602
           ->  Seq Scan on salary salary_1  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.010..181.774 rows=2844047 loops=1)
                 Buffers: shared hit=513 read=17602
   InitPlan 2 (returns $1)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=385.578..385.578 rows=1 loops=1)
           Buffers: shared hit=576 read=17539
           ->  Seq Scan on salary salary_2  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.003..184.261 rows=2844047 loops=1)
                 Buffers: shared hit=576 read=17539
   InitPlan 3 (returns $2)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=366.534..366.535 rows=1 loops=1)
           Buffers: shared hit=608 read=17507
           ->  Seq Scan on salary salary_3  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.001..181.322 rows=2844047 loops=1)
                 Buffers: shared hit=608 read=17507
   InitPlan 4 (returns $3)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=378.609..378.610 rows=1 loops=1)
           Buffers: shared hit=640 read=17475
           ->  Seq Scan on salary salary_4  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.004..179.681 rows=2844047 loops=1)
                 Buffers: shared hit=640 read=17475
   InitPlan 5 (returns $4)
     ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.004..0.004 rows=1 loops=1)
   InitPlan 6 (returns $5)
     ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=1)
   ->  Nested Loop Semi Join  (cost=26101.15..161910.68 rows=1 width=44) (actual time=2218.878..2400.552 rows=278 loops=1)
         Join Filter: (salary.employee_id = s.employee_id)
         Rows Removed by Join Filter: 38503
         Buffers: shared hit=11340 read=105137, temp read=6162 written=6162
         ->  Seq Scan on salary  (cost=0.00..67885.82 rows=107 width=22) (actual time=782.307..955.548 rows=278 loops=1)
               Filter: ((from_date >= $0) AND (amount < 67690) AND (to_date = $1))
               Rows Removed by Filter: 2843769
               Buffers: shared hit=1633 read=52712
         ->  Materialize  (cost=26101.15..93988.00 rows=23 width=62) (actual time=4.521..5.190 rows=140 loops=278)
               Buffers: shared hit=9707 read=52425, temp read=6162 written=6162
               ->  Hash Join  (cost=26101.15..93987.88 rows=23 width=62) (actual time=1256.933..1441.148 rows=278 loops=1)
                     Hash Cond: (de.department_id = d.id)
                     Buffers: shared hit=9707 read=52425, temp read=6162 written=6162
                     ->  Hash Join  (cost=26099.95..93986.36 rows=23 width=61) (actual time=1256.906..1440.987 rows=278 loops=1)
                           Hash Cond: (s.employee_id = e.id)
                           Buffers: shared hit=9706 read=52425, temp read=6162 written=6162
                           ->  Seq Scan on salary s  (cost=0.00..67885.82 rows=107 width=14) (actual time=745.223..919.776 rows=278 loops=1)
                                 Filter: ((from_date >= $2) AND (amount < 67690) AND (to_date = $3))
                                 Rows Removed by Filter: 2843769
                                 Buffers: shared hit=1920 read=52425
                           ->  Hash  (cost=25419.44..25419.44 rows=54441 width=47) (actual time=507.461..507.463 rows=240124 loops=1)
                                 Buckets: 131072 (originally 65536)  Batches: 4 (originally 1)  Memory Usage: 7169kB
                                 Buffers: shared hit=7786, temp read=4554 written=6156
                                 ->  Hash Join  (cost=18232.83..25419.44 rows=54441 width=47) (actual time=295.234..447.330 rows=240124 loops=1)
                                       Hash Cond: (e.id = t.employee_id)
                                       Buffers: shared hit=7786, temp read=4554 written=4554
                                       ->  Seq Scan on employee e  (cost=0.00..5481.24 rows=300024 width=14) (actual time=0.008..28.122 rows=300024 loops=1)
                                             Buffers: shared hit=2481
                                       ->  Hash  (cost=17507.48..17507.48 rows=58028 width=33) (actual time=295.169..295.171 rows=240124 loops=1)
                                             Buckets: 131072 (originally 65536)  Batches: 4 (originally 1)  Memory Usage: 7169kB
                                             Buffers: shared hit=5305, temp read=1991 written=3241
                                             ->  Hash Join  (cost=7639.71..17507.48 rows=58028 width=33) (actual time=85.441..237.000 rows=240124 loops=1)
                                                   Hash Cond: (t.employee_id = de.employee_id)
                                                   Buffers: shared hit=5305, temp read=1991 written=1991
                                                   ->  Seq Scan on title t  (cost=0.00..8733.35 rows=147769 width=14) (actual time=0.012..43.137 rows=240124 loops=1)
                                                         Filter: (to_date > $4)
                                                         Rows Removed by Filter: 203184
                                                         Buffers: shared hit=3192
                                                   ->  Hash  (cost=6258.04..6258.04 rows=110534 width=19) (actual time=85.053..85.054 rows=240124 loops=1)
                                                         Buckets: 262144 (originally 131072)  Batches: 4 (originally 1)  Memory Usage: 6785kB
                                                         Buffers: shared hit=2113, temp written=899
                                                         ->  Seq Scan on department_employee de  (cost=0.00..6258.04 rows=110534 width=19) (actual time=0.010..31.382 rows=240124 loops=1)
                                                               Filter: (to_date > $5)
                                                               Rows Removed by Filter: 91479
                                                               Buffers: shared hit=2113
                     ->  Hash  (cost=1.09..1.09 rows=9 width=11) (actual time=0.014..0.014 rows=9 loops=1)
                           Buckets: 1024  Batches: 1  Memory Usage: 9kB
                           Buffers: shared hit=1
                           ->  Seq Scan on department d  (cost=0.00..1.09 rows=9 width=11) (actual time=0.008..0.009 rows=9 loops=1)
                                 Buffers: shared hit=1
 Planning:
   Buffers: shared hit=3
 Planning Time: 0.435 ms
 JIT:
   Functions: 63
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 2.053 ms, Inlining 0.000 ms, Optimization 0.980 ms, Emission 24.981 ms, Total 28.015 ms
 Execution Time: 2427.629 ms
(84 rows)
```
### Доработка структуры таблиц для запроса Select.
10) Дорабатываем структуру базы employees_1 (база для проверки обоих запросов с доработанной структурой) - создаем структуру ключей и индексов под запрос Select . Выводим весь список индексов и ключей в таблицах схемы. Предполагается, что данные индексы будут использоваться оптимизатором для более быстрой выбоки данных по условиям запроса и для сборки данных в Join.
```
employees_1=# CREATE UNIQUE INDEX idx_employee_id_primary ON employees.employee USING btree (id);
CREATE UNIQUE INDEX idx_department_id_primary ON employees.department USING btree (id);
CREATE INDEX idx_title_todate ON employees.title USING btree (to_date);
CREATE INDEX idx_title_empid ON employees.title USING btree (employee_id);
CREATE INDEX idx_salary_empid ON employees.salary USING btree (employee_id);
CREATE INDEX idx_salary_todate ON employees.salary USING btree (to_date);
CREATE INDEX idx_salary_fromdate ON employees.salary USING btree (from_date);
CREATE INDEX idx_salary_amount ON employees.salary USING btree (amount);
CREATE INDEX idx_department_employee_to_date ON employees.department_employee USING btree (to_date);
CREATE INDEX idx_department_employee_depid ON employees.department_employee USING btree (department_id);
CREATE INDEX idx_department_employee_employeeid ON employees.department_employee USING btree (employee_id);
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
employees=# analyse;
ANALYZE
employees_1=# SELECT tablename, indexname, indexdef FROM pg_indexes where schemaname = 'employees';
      tablename      |             indexname              |                                                  indexdef
---------------------+------------------------------------+------------------------------------------------------------------------------------------------------------
 employee            | idx_employee_id_primary            | CREATE UNIQUE INDEX idx_employee_id_primary ON employees.employee USING btree (id)
 department          | idx_department_id_primary          | CREATE UNIQUE INDEX idx_department_id_primary ON employees.department USING btree (id)
 title               | idx_title_todate                   | CREATE INDEX idx_title_todate ON employees.title USING btree (to_date)
 title               | idx_title_empid                    | CREATE INDEX idx_title_empid ON employees.title USING btree (employee_id)
 salary              | idx_salary_empid                   | CREATE INDEX idx_salary_empid ON employees.salary USING btree (employee_id)
 salary              | idx_salary_todate                  | CREATE INDEX idx_salary_todate ON employees.salary USING btree (to_date)
 salary              | idx_salary_fromdate                | CREATE INDEX idx_salary_fromdate ON employees.salary USING btree (from_date)
 salary              | idx_salary_amount                  | CREATE INDEX idx_salary_amount ON employees.salary USING btree (amount)
 department_employee | idx_department_employee_to_date    | CREATE INDEX idx_department_employee_to_date ON employees.department_employee USING btree (to_date)
 department_employee | idx_department_employee_depid      | CREATE INDEX idx_department_employee_depid ON employees.department_employee USING btree (department_id)
 department_employee | idx_department_employee_employeeid | CREATE INDEX idx_department_employee_employeeid ON employees.department_employee USING btree (employee_id)
(11 rows)
```
### Проверка производительности запросов после доработки структуры. 
11) Проверка производительности запросов на оптимизированной структуре базы с актуальным планам explain (analyze, buffers) показала значительный прирост производительности запроса Select за счет индексов.

Ранее время обработки было - Execution Time: 1260.088 ms

После доработки структуры - Execution Time: 100.351 ms
```
employees_1=# explain (analyze, buffers)
select
 e.first_name ||' '|| e.last_name AS "Имя сотрудника",
 t.title AS "Должность",
 s.amount AS "Зарплата на сегодня в $",
 67690 - s.amount AS "Недостаток в $"
 from employees.salary as s
Join employees.employee as e
 on s.employee_id = e.id
join employees.title as t
 on s.employee_id = t.employee_id
join employees.department_employee as de
 on e.id = de.employee_id
join employees.department as d
 on d.id = de.department_id
 where
  s.from_date >= (select max(from_date) from employees.salary)
   and s.to_date = (select max(to_date) from employees.salary)
   and t.to_date > (select current_date)
   and de.to_date > (select current_date)
   and s.amount < 67690
ORDER BY s.amount;
                                                                                         QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=2681.57..2681.63 rows=23 width=59) (actual time=100.268..100.292 rows=278 loops=1)
   Sort Key: s.amount
   Sort Method: quicksort  Memory: 48kB
   Buffers: shared hit=1967 read=19698 written=8
   InitPlan 2 (returns $1)
     ->  Result  (cost=0.45..0.46 rows=1 width=4) (actual time=0.022..0.023 rows=1 loops=1)
           Buffers: shared hit=2 read=2
           InitPlan 1 (returns $0)
             ->  Limit  (cost=0.43..0.45 rows=1 width=4) (actual time=0.020..0.021 rows=1 loops=1)
                   Buffers: shared hit=2 read=2
                   ->  Index Only Scan Backward using idx_salary_fromdate on salary  (cost=0.43..59783.25 rows=2844047 width=4) (actual time=0.019..0.019 rows=1 loops=1)
                         Index Cond: (from_date IS NOT NULL)
                         Heap Fetches: 0
                         Buffers: shared hit=2 read=2
   InitPlan 4 (returns $3)
     ->  Result  (cost=0.45..0.46 rows=1 width=4) (actual time=0.019..0.020 rows=1 loops=1)
           Buffers: shared hit=2 read=2
           InitPlan 3 (returns $2)
             ->  Limit  (cost=0.43..0.45 rows=1 width=4) (actual time=0.017..0.018 rows=1 loops=1)
                   Buffers: shared hit=2 read=2
                   ->  Index Only Scan Backward using idx_salary_todate on salary salary_1  (cost=0.43..59731.25 rows=2844047 width=4) (actual time=0.016..0.017 rows=1 loops=1)
                         Index Cond: (to_date IS NOT NULL)
                         Heap Fetches: 0
                         Buffers: shared hit=2 read=2
   InitPlan 5 (returns $4)
     ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.000..0.001 rows=1 loops=1)
   InitPlan 6 (returns $5)
     ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=1)
   ->  Nested Loop  (cost=9.51..2680.10 rows=23 width=59) (actual time=12.141..100.137 rows=278 loops=1)
         Join Filter: (de.department_id = d.id)
         Rows Removed by Join Filter: 965
         Buffers: shared hit=1967 read=19698 written=8
         ->  Nested Loop  (cost=9.51..2676.04 rows=23 width=39) (actual time=12.131..99.808 rows=278 loops=1)
               Join Filter: (s.employee_id = e.id)
               Buffers: shared hit=1966 read=19698 written=8
               ->  Nested Loop  (cost=9.09..2663.57 rows=23 width=48) (actual time=12.121..98.152 rows=278 loops=1)
                     Join Filter: (s.employee_id = t.employee_id)
                     Buffers: shared hit=1310 read=19242 written=8
                     ->  Nested Loop  (cost=8.67..2637.04 rows=42 width=29) (actual time=12.108..96.258 rows=278 loops=1)
                           Buffers: shared hit=677 read=18763 written=7
                           ->  Bitmap Heap Scan on salary s  (cost=8.24..1764.62 rows=107 width=16) (actual time=12.084..94.276 rows=278 loops=1)
                                 Recheck Cond: (to_date = $3)
                                 Filter: ((from_date >= $1) AND (amount < 67690))
                                 Rows Removed by Filter: 239846
                                 Heap Blocks: exact=18115
                                 Buffers: shared hit=6 read=18322 written=7
                                 ->  Bitmap Index Scan on idx_salary_todate  (cost=0.00..8.22 rows=505 width=0) (actual time=7.836..7.836 rows=240124 loops=1)
                                       Index Cond: (to_date = $3)
                                       Buffers: shared hit=4 read=205
                           ->  Index Scan using idx_department_employee_employeeid on department_employee de  (cost=0.42..8.14 rows=1 width=13) (actual time=0.006..0.006 rows=1 loops=278)
                                 Index Cond: (employee_id = s.employee_id)
                                 Filter: (to_date > $5)
                                 Rows Removed by Filter: 0
                                 Buffers: shared hit=671 read=441
                     ->  Index Scan using idx_title_empid on title t  (cost=0.42..0.62 rows=1 width=19) (actual time=0.006..0.006 rows=1 loops=278)
                           Index Cond: (employee_id = de.employee_id)
                           Filter: (to_date > $4)
                           Rows Removed by Filter: 0
                           Buffers: shared hit=633 read=479 written=1
               ->  Index Scan using idx_employee_id_primary on employee e  (cost=0.42..0.53 rows=1 width=23) (actual time=0.005..0.005 rows=1 loops=278)
                     Index Cond: (id = t.employee_id)
                     Buffers: shared hit=656 read=456
         ->  Materialize  (cost=0.00..1.14 rows=9 width=5) (actual time=0.000..0.000 rows=4 loops=278)
               Buffers: shared hit=1
               ->  Seq Scan on department d  (cost=0.00..1.09 rows=9 width=5) (actual time=0.005..0.006 rows=9 loops=1)
                     Buffers: shared hit=1
 Planning:
   Buffers: shared hit=195 read=1
 Planning Time: 1.805 ms
 Execution Time: 100.351 ms
(70 rows)
```
Запрос по обновлению данных после доработки структуры базы отработал значительно быстрее даже с учетом обновления индексов которых ранее не было. 

Ранее время обработки было - Execution Time: 2427.629 ms

После доработки структуры - Execution Time: 897.711 ms

```
employees_1=# explain (analyze, buffers)
update employees.salary
 set amount = (67690 + (amount/3))
where employee_id in
 (select s.employee_id
  from employees.salary as s
  Join employees.employee as e
   on s.employee_id = e.id
  join employees.title as t
   on s.employee_id = t.employee_id
  join employees.department_employee as de
   on e.id = de.employee_id
  join employees.department as d
   on d.id = de.department_id
  where
   s.from_date >= (select max(from_date) from employees.salary)
   and s.to_date = (select max(to_date) from employees.salary)
   and t.to_date > (select current_date)
   and de.to_date > (select current_date)
   and s.amount < 67690
)
and from_date >= (select max(from_date) from employees.salary)
and to_date = (select max(to_date) from employees.salary)
and amount < 67690;
                                                                                         QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Update on salary  (cost=11.95..2874.74 rows=0 width=0) (actual time=686.314..686.321 rows=0 loops=1)
   Buffers: shared hit=6644 read=20117 dirtied=693 written=78
   InitPlan 2 (returns $1)
     ->  Result  (cost=0.45..0.46 rows=1 width=4) (actual time=0.017..0.018 rows=1 loops=1)
           Buffers: shared hit=2 read=2
           InitPlan 1 (returns $0)
             ->  Limit  (cost=0.43..0.45 rows=1 width=4) (actual time=0.015..0.016 rows=1 loops=1)
                   Buffers: shared hit=2 read=2
                   ->  Index Only Scan Backward using idx_salary_fromdate on salary salary_1  (cost=0.43..59783.25 rows=2844047 width=4) (actual time=0.014..0.015 rows=1 loops=1)
                         Index Cond: (from_date IS NOT NULL)
                         Heap Fetches: 0
                         Buffers: shared hit=2 read=2
   InitPlan 4 (returns $3)
     ->  Result  (cost=0.45..0.46 rows=1 width=4) (actual time=0.030..0.031 rows=1 loops=1)
           Buffers: shared hit=3 read=1
           InitPlan 3 (returns $2)
             ->  Limit  (cost=0.43..0.45 rows=1 width=4) (actual time=0.027..0.027 rows=1 loops=1)
                   Buffers: shared hit=3 read=1
                   ->  Index Only Scan Backward using idx_salary_todate on salary salary_2  (cost=0.43..59731.25 rows=2844047 width=4) (actual time=0.025..0.025 rows=1 loops=1)
                         Index Cond: (to_date IS NOT NULL)
                         Heap Fetches: 0
                         Buffers: shared hit=3 read=1
   InitPlan 6 (returns $5)
     ->  Result  (cost=0.45..0.46 rows=1 width=4) (actual time=0.009..0.010 rows=1 loops=1)
           Buffers: shared hit=4
           InitPlan 5 (returns $4)
             ->  Limit  (cost=0.43..0.45 rows=1 width=4) (actual time=0.008..0.008 rows=1 loops=1)
                   Buffers: shared hit=4
                   ->  Index Only Scan Backward using idx_salary_fromdate on salary salary_3  (cost=0.43..59783.25 rows=2844047 width=4) (actual time=0.007..0.007 rows=1 loops=1)
                         Index Cond: (from_date IS NOT NULL)
                         Heap Fetches: 0
                         Buffers: shared hit=4
   InitPlan 8 (returns $7)
     ->  Result  (cost=0.45..0.46 rows=1 width=4) (actual time=0.007..0.008 rows=1 loops=1)
           Buffers: shared hit=4
           InitPlan 7 (returns $6)
             ->  Limit  (cost=0.43..0.45 rows=1 width=4) (actual time=0.006..0.007 rows=1 loops=1)
                   Buffers: shared hit=4
                   ->  Index Only Scan Backward using idx_salary_todate on salary salary_4  (cost=0.43..59731.25 rows=2844047 width=4) (actual time=0.006..0.006 rows=1 loops=1)
                         Index Cond: (to_date IS NOT NULL)
                         Heap Fetches: 0
                         Buffers: shared hit=4
   InitPlan 9 (returns $8)
     ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=1)
   InitPlan 10 (returns $9)
     ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.000..0.000 rows=1 loops=1)
   ->  Nested Loop Semi Join  (cost=10.08..2872.87 rows=1 width=44) (actual time=29.098..672.303 rows=278 loops=1)
         Buffers: shared hit=3378 read=19973 written=75
         ->  Bitmap Heap Scan on salary  (cost=8.24..1764.62 rows=107 width=22) (actual time=16.495..102.261 rows=278 loops=1)
               Recheck Cond: (to_date = $3)
               Filter: ((from_date >= $1) AND (amount < 67690))
               Rows Removed by Filter: 239846
               Heap Blocks: exact=18115
               Buffers: shared hit=8 read=18320 written=68
               ->  Bitmap Index Scan on idx_salary_todate  (cost=0.00..8.22 rows=505 width=0) (actual time=12.799..12.799 rows=240124 loops=1)
                     Index Cond: (to_date = $3)
                     Buffers: shared hit=5 read=204
         ->  Nested Loop  (cost=1.83..10.35 rows=1 width=62) (actual time=2.049..2.049 rows=1 loops=278)
               Buffers: shared hit=3370 read=1653 written=7
               ->  Nested Loop  (cost=1.70..10.19 rows=1 width=61) (actual time=2.043..2.043 rows=1 loops=278)
                     Join Filter: (salary.employee_id = s.employee_id)
                     Buffers: shared hit=2814 read=1653 written=7
                     ->  Nested Loop  (cost=1.27..9.16 rows=1 width=47) (actual time=0.023..0.023 rows=1 loops=278)
                           Buffers: shared hit=1960 read=1376 written=6
                           ->  Nested Loop  (cost=0.84..8.67 rows=1 width=28) (actual time=0.015..0.015 rows=1 loops=278)
                                 Buffers: shared hit=1290 read=934 written=3
                                 ->  Index Scan using idx_employee_id_primary on employee e  (cost=0.42..8.14 rows=1 width=14) (actual time=0.007..0.007 rows=1 loops=278)
                                       Index Cond: (id = salary.employee_id)
                                       Buffers: shared hit=656 read=456 written=2
                                 ->  Index Scan using idx_title_empid on title t  (cost=0.42..0.52 rows=1 width=14) (actual time=0.007..0.007 rows=1 loops=278)
                                       Index Cond: (employee_id = e.id)
                                       Filter: (to_date > $8)
                                       Rows Removed by Filter: 0
                                       Buffers: shared hit=634 read=478 written=1
                           ->  Index Scan using idx_department_employee_employeeid on department_employee de  (cost=0.42..0.48 rows=1 width=19) (actual time=0.007..0.007 rows=1 loops=278)
                                 Index Cond: (employee_id = e.id)
                                 Filter: (to_date > $9)
                                 Rows Removed by Filter: 0
                                 Buffers: shared hit=670 read=442 written=3
                     ->  Index Scan using idx_salary_empid on salary s  (cost=0.43..1.02 rows=1 width=14) (actual time=2.019..2.019 rows=1 loops=278)
                           Index Cond: (employee_id = e.id)
                           Filter: ((from_date >= $5) AND (amount < 67690) AND (to_date = $7))
                           Rows Removed by Filter: 8
                           Buffers: shared hit=854 read=277 written=1
               ->  Index Scan using idx_department_id_primary on department d  (cost=0.14..0.15 rows=1 width=11) (actual time=0.003..0.003 rows=1 loops=278)
                     Index Cond: (id = de.department_id)
                     Buffers: shared hit=556
 Planning:
   Buffers: shared hit=215
 Planning Time: 1.898 ms
 Execution Time: 897.711 ms
(91 rows)
```
Выводы: В запросах массового изменения данных структура индексов может продлевает отработку поскольку требует дополнительного времени на синхронизацию  данных таблицы с индексами, для потдержания данных в индексах в актульаном состоянии, что положительно влияет на последующую работу оптимизатора  с ними в рамках будущих запросов.

Далее будем тестировать запросы со сбором минимального набота данных при котором структура индексов, ключей и дополнительные изменения столбцов должны показать белее оптимальные рузультаты. Для тестирования будем использовать третью копию базы employees_2 без структуры.

### Личные рекомендации по нормализации данных.
а. Было бы эффективнее упразднить таблицу справочник руководителей подразделений department_manager. Данные этой таблицы хоть и позволяют отслеживать историю смены руководителей но такая история уже есть совокупно в таблицах привязки сотрудника к департаменту с таблицей должностей. Иными словами можно выдать историю смены руководителей отдела связанным запросом, а не занимать под эти «результатирующие данные» диски (эта таблица дублирует уже имеющиеся данные).

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
Теперь структура будет занимать меньше места на диске из за смены типов данных vatchar(50) на int в таблице 443 000 записей. 
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
### Разработка запроса к базе с минимальным результатирующим набором.
Разрабатываем запрос к базе с минимальным результатирующим набором, для проверки работы оптимизированной структуры БД. Проверять результаты оптмиизации будем на базе employees_2 без индексов и с индексами, где будет проведена работа по нормализации структуры данных в пунктах 12,13,14.

15) Разрабатан запрос.
```
-- Выдать информацию по последней выдаче заработной платы сотруднику с учетом его последней должности и департамента.
-- Поисковые данные по сотруднику вводим в условиях запроса

employees_2=# Select
 e.first_name AS "Фамилия",
 e.last_name AS "Имя",
 d.dept_name AS "Департамент",
 t.title AS "Должность",
 s.amount AS "Зарплата $",
 s.from_date AS "выдана от",
 s.to_date  AS "выдана до"
 from employees.employee as e
join employees.department_employee as de
 on e.id = de.employee_id
join employees.department as d
 on de.department_id = d.id
join employees.title as t
 on e.id = t.employee_id
join employees.salary as s
 on e.id = s.employee_id
where
 e.first_name like 'Almudena'
and e.last_name like 'Sur%'
and t.to_date = (select max(to_date) from employees.title)
and s.to_date = (select max(to_date) from employees.salary);
 Фамилия  | Имя  |    Департамент     |    Должность    | Зарплата $ | выдана от  | выдана до
----------+------+--------------------+-----------------+------------+------------+------------
 Almudena | Sury | Production         | Senior Engineer |      89888 | 2002-02-25 | 9999-01-01
 Almudena | Sury | Quality Management | Senior Engineer |      89888 | 2002-02-25 | 9999-01-01
(2 rows)
```

Выводим актуальный план запроса на базе без нормализации данных (еще нет справочника employees.titles) и индексов, после нескольких выполнений для сбора статистики, и после Analyse. По итогу Execution Time: 824.225 ms. Такая длительность обработки обусловлена отсутсвием индексов в таблицах что выражено присутсвием блоков Seq Scan.   
```
employees_2=# explain (analyze, buffers)
Select
 e.first_name AS "Фамилия",
 e.last_name AS "Имя",
 d.dept_name AS "Департамент",
 t.title AS "Должность",
 s.amount AS "Зарплата $",
 s.from_date AS "выдана от",
 s.to_date  AS "выдана до"
 from employees.employee as e
join employees.department_employee as de
 on e.id = de.employee_id
join employees.department as d
 on de.department_id = d.id
join employees.title as t
 on e.id = t.employee_id
join employees.salary as s
 on e.id = s.employee_id
where
 e.first_name like 'Almudena'
and e.last_name like 'Sur%'
and t.to_date = (select max(to_date) from employees.title)
and s.to_date = (select max(to_date) from employees.salary);
                                                                                   QUERY PLAN

------------------------------------------------------------------------------------------------------------------------------------------------------
---------------------------
 Nested Loop  (cost=56853.55..89783.40 rows=1 width=54) (actual time=823.968..824.183 rows=2 loops=1)
   Join Filter: (de.department_id = d.id)
   Rows Removed by Join Filter: 16
   Buffers: shared hit=14302 read=33062
   InitPlan 1 (returns $1)
     ->  Finalize Aggregate  (cost=6501.11..6501.12 rows=1 width=4) (actual time=69.732..69.811 rows=1 loops=1)
           Buffers: shared hit=3192
           ->  Gather  (cost=6500.90..6501.11 rows=2 width=4) (actual time=69.650..69.792 rows=3 loops=1)
                 Workers Planned: 2
                 Workers Launched: 2
                 Buffers: shared hit=3192
                 ->  Partial Aggregate  (cost=5500.90..5500.91 rows=1 width=4) (actual time=64.098..64.099 rows=1 loops=3)
                       Buffers: shared hit=3192
                       ->  Parallel Seq Scan on title  (cost=0.00..5039.12 rows=184712 width=4) (actual time=0.005..18.802 rows=147769 loops=3)
                             Buffers: shared hit=3192
   InitPlan 2 (returns $3)
     ->  Finalize Aggregate  (cost=33927.96..33927.97 rows=1 width=4) (actual time=390.929..390.952 rows=1 loops=1)
           Buffers: shared hit=1536 read=16579
           ->  Gather  (cost=33927.75..33927.96 rows=2 width=4) (actual time=390.924..390.949 rows=3 loops=1)
                 Workers Planned: 2
                 Workers Launched: 2
                 Buffers: shared hit=1536 read=16579
                 ->  Partial Aggregate  (cost=32927.75..32927.76 rows=1 width=4) (actual time=378.633..378.634 rows=1 loops=3)
                       Buffers: shared hit=1536 read=16579
                       ->  Parallel Seq Scan on salary  (cost=0.00..29965.20 rows=1185020 width=4) (actual time=0.035..224.678 rows=948016 loops=3)
                             Buffers: shared hit=1536 read=16579
   ->  Gather  (cost=16424.46..49353.10 rows=1 width=47) (actual time=823.943..824.051 rows=2 loops=1)
         Workers Planned: 2
         Params Evaluated: $1, $3
         Workers Launched: 2
         Buffers: shared hit=14300 read=33062
         ->  Parallel Hash Join  (cost=15424.46..48353.00 rows=1 width=47) (actual time=346.498..348.235 rows=1 loops=3)
               Hash Cond: (s.employee_id = e.id)
               Buffers: shared hit=9572 read=16483
               ->  Parallel Seq Scan on salary s  (cost=0.00..32927.74 rows=209 width=24) (actual time=0.037..214.114 rows=80041 loops=3)
                     Filter: (to_date = $3)
                     Rows Removed by Filter: 867974
                     Buffers: shared hit=1632 read=16483
               ->  Parallel Hash  (cost=15424.45..15424.45 rows=1 width=55) (actual time=120.864..120.867 rows=1 loops=3)
                     Buckets: 1024  Batches: 1  Memory Usage: 40kB
                     Buffers: shared hit=7786
                     ->  Parallel Hash Join  (cost=9923.39..15424.45 rows=1 width=55) (actual time=120.334..120.817 rows=1 loops=3)
                           Hash Cond: (t.employee_id = e.id)
                           Buffers: shared hit=7786
                           ->  Parallel Seq Scan on title t  (cost=0.00..5500.90 rows=41 width=19) (actual time=0.012..27.787 rows=80041 loops=3)
                                 Filter: (to_date = $1)
                                 Rows Removed by Filter: 67728
                                 Buffers: shared hit=3192
                           ->  Parallel Hash  (cost=9923.38..9923.38 rows=1 width=36) (actual time=62.536..62.538 rows=1 loops=3)
                                 Buckets: 1024  Batches: 1  Memory Usage: 40kB
                                 Buffers: shared hit=4594
                                 ->  Parallel Hash Join  (cost=5128.28..9923.38 rows=1 width=36) (actual time=62.127..62.491 rows=1 loops=3)
                                       Hash Cond: (de.employee_id = e.id)
                                       Buffers: shared hit=4594
                                       ->  Parallel Seq Scan on department_employee de  (cost=0.00..4063.61 rows=195061 width=13) (actual time=0.010..
6.803 rows=110534 loops=3)
                                             Buffers: shared hit=2113
                                       ->  Parallel Hash  (cost=5128.27..5128.27 rows=1 width=23) (actual time=17.550..17.551 rows=0 loops=3)
                                             Buckets: 1024  Batches: 1  Memory Usage: 40kB
                                             Buffers: shared hit=2481
                                             ->  Parallel Seq Scan on employee e  (cost=0.00..5128.27 rows=1 width=23) (actual time=17.293..17.511 row
s=0 loops=3)
                                                   Filter: (((first_name)::text ~~ 'Almudena'::text) AND ((last_name)::text ~~ 'Sur%'::text))
                                                   Rows Removed by Filter: 100008
                                                   Buffers: shared hit=2481
   ->  Seq Scan on department d  (cost=0.00..1.09 rows=9 width=17) (actual time=0.010..0.011 rows=9 loops=2)
         Buffers: shared hit=2
 Planning Time: 0.482 ms
 Execution Time: 824.225 ms
(67 rows)
```
16) Применяем доработку структуры базы которая описана в пунктах 12, 13, 14 (создание отдельного справочника employees.titles). Далее разрабатываем структуру индексов под запрос с учетом нового справочнка.  
```
employees_2=# CREATE UNIQUE INDEX idx_employee_id_primary ON employees.employee USING btree (id);
CREATE INDEX idx_employee_firstname_lastname ON employees.employee USING btree (first_name, last_name);
CREATE UNIQUE INDEX idx_department_id_primary ON employees.department USING btree (id);
CREATE INDEX idx_title_todate ON employees.title USING btree (to_date);
CREATE INDEX idx_title_empid ON employees.title USING btree (employee_id);
CREATE INDEX idx_title_titleid ON employees.title USING btree (title_id);
CREATE INDEX idx_salary_empid ON employees.salary USING btree (employee_id);
CREATE INDEX idx_salary_todate ON employees.salary USING btree (to_date);
CREATE INDEX idx_department_employee_depid ON employees.department_employee USING btree (department_id);
CREATE INDEX idx_department_employee_employeeid ON employees.department_employee USING btree (employee_id);
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
employees_2=# SELECT tablename, indexname, indexdef FROM pg_indexes where schemaname = 'employees';
      tablename      |             indexname              |                                                  indexdef

---------------------+------------------------------------+--------------------------------------------------------------------------------------------------
----------
 titles              | titles_pkey                        | CREATE UNIQUE INDEX titles_pkey ON employees.titles USING btree (title_id)
 employee            | idx_employee_id_primary            | CREATE UNIQUE INDEX idx_employee_id_primary ON employees.employee USING btree (id)
 employee            | idx_employee_firstname_lastname    | CREATE INDEX idx_employee_firstname_lastname ON employees.employee USING btree (first_name, last_
name)
 department          | idx_department_id_primary          | CREATE UNIQUE INDEX idx_department_id_primary ON employees.department USING btree (id)
 title               | idx_title_todate                   | CREATE INDEX idx_title_todate ON employees.title USING btree (to_date)
 title               | idx_title_empid                    | CREATE INDEX idx_title_empid ON employees.title USING btree (employee_id)
 title               | idx_title_titleid                  | CREATE INDEX idx_title_titleid ON employees.title USING btree (title_id)
 salary              | idx_salary_empid                   | CREATE INDEX idx_salary_empid ON employees.salary USING btree (employee_id)
 salary              | idx_salary_todate                  | CREATE INDEX idx_salary_todate ON employees.salary USING btree (to_date)
 department_employee | idx_department_employee_depid      | CREATE INDEX idx_department_employee_depid ON employees.department_employee USING btree (departme
nt_id)
 department_employee | idx_department_employee_employeeid | CREATE INDEX idx_department_employee_employeeid ON employees.department_employee USING btree (emp
loyee_id)
(11 rows)
```
17) Отрабатываем запрос на оптимизированной структуре данных с выводом плана, после нескольких его отработок для сбора статистики и применения ANALYSE;. В итоге время отработки снизилось в 2608 раз за счет изменения плана выполнения запроса с учетом индексов, применение которых наблюдаем в блоках "Index Scan".

Ранее время обработки было - Execution Time: 824.225 ms

После доработки структуры - Execution Time: 0.316 ms

```
employees_2=# analyse;
ANALYZE
employees_2=# explain (analyze, buffers)
Select 
 e.first_name AS "Фамилия",
 e.last_name AS "Имя",
 d.dept_name AS "Департамент",
 ts.title AS "Должность",
 s.amount AS "Зарплата $",
 s.from_date AS "выдана от",
 s.to_date  AS "выдана до" 
 from employees.employee as e
join employees.department_employee as de
 on e.id = de.employee_id
join employees.department as d
 on de.department_id = d.id
join employees.title as t
 on e.id = t.employee_id
join employees.titles as ts
 on t.title_id = ts.title_id
join employees.salary as s
 on e.id = s.employee_id
where
 e.first_name like 'Almudena'
and e.last_name like 'Sur%'
and t.to_date = (select max(to_date) from employees.title)
and s.to_date = (select max(to_date) from employees.salary);
                                                                                        QUERY PLAN

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
---------
 Nested Loop  (cost=8.42..717.01 rows=1 width=55) (actual time=0.256..0.277 rows=2 loops=1)
   Join Filter: (e.id = s.employee_id)
   Buffers: shared hit=254
   InitPlan 2 (returns $1)
     ->  Result  (cost=0.47..0.48 rows=1 width=4) (actual time=0.008..0.009 rows=1 loops=1)
           Buffers: shared hit=4
           InitPlan 1 (returns $0)
             ->  Limit  (cost=0.42..0.47 rows=1 width=4) (actual time=0.007..0.007 rows=1 loops=1)
                   Buffers: shared hit=4
                   ->  Index Only Scan Backward using idx_title_todate on title  (cost=0.42..20524.88 rows=443308 width=4) (actual time=0.006..0.006 rows=1 loops=1)
                         Index Cond: (to_date IS NOT NULL)
                         Heap Fetches: 1
                         Buffers: shared hit=4
   InitPlan 4 (returns $3)
     ->  Result  (cost=0.48..0.49 rows=1 width=4) (actual time=0.007..0.007 rows=1 loops=1)
           Buffers: shared hit=4
           InitPlan 3 (returns $2)
             ->  Limit  (cost=0.43..0.48 rows=1 width=4) (actual time=0.007..0.007 rows=1 loops=1)
                   Buffers: shared hit=4
                   ->  Index Only Scan Backward using idx_salary_todate on salary  (cost=0.43..132190.66 rows=2844047 width=4) (actual time=0.006..0.007 rows=1 loops=1)
                         Index Cond: (to_date IS NOT NULL)
                         Heap Fetches: 1
                         Buffers: shared hit=4
   ->  Nested Loop  (cost=7.02..715.10 rows=1 width=63) (actual time=0.242..0.257 rows=2 loops=1)
         Join Filter: (t.title_id = ts.title_id)
         Rows Removed by Join Filter: 6
         Buffers: shared hit=240
         ->  Nested Loop  (cost=7.02..713.94 rows=1 width=55) (actual time=0.238..0.252 rows=2 loops=1)
               Join Filter: (e.id = t.employee_id)
               Buffers: shared hit=238
               ->  Nested Loop  (cost=6.60..713.42 rows=1 width=43) (actual time=0.224..0.233 rows=2 loops=1)
                     Join Filter: (de.department_id = d.id)
                     Rows Removed by Join Filter: 11
                     Buffers: shared hit=224
                     ->  Nested Loop  (cost=6.60..712.22 rows=1 width=36) (actual time=0.219..0.225 rows=2 loops=1)
                           Buffers: shared hit=222
                           ->  Bitmap Heap Scan on employee e  (cost=6.18..703.77 rows=1 width=23) (actual time=0.213..0.217 rows=1 loops=1)
                                 Filter: (((first_name)::text ~~ 'Almudena'::text) AND ((last_name)::text ~~ 'Sur%'::text))
                                 Rows Removed by Filter: 223
                                 Heap Blocks: exact=214
                                 Buffers: shared hit=218
                                 ->  Bitmap Index Scan on idx_employee_firstname_lastname  (cost=0.00..6.18 rows=234 width=0) (actual time=0.024..0.024 rows=224 loops=1)
                                       Index Cond: ((first_name)::text = 'Almudena'::text)
                                       Buffers: shared hit=4
                           ->  Index Scan using idx_department_employee_employeeid on department_employee de  (cost=0.42..8.44 rows=1 width=13) (actual time=0.004..0.005 rows=2
loops=1)
                                 Index Cond: (employee_id = e.id)
                                 Buffers: shared hit=4
                     ->  Seq Scan on department d  (cost=0.00..1.09 rows=9 width=17) (actual time=0.002..0.002 rows=6 loops=2)
                           Buffers: shared hit=2
               ->  Index Scan using idx_title_empid on title t  (cost=0.42..0.51 rows=1 width=12) (actual time=0.007..0.008 rows=1 loops=2)
                     Index Cond: (employee_id = de.employee_id)
                     Filter: (to_date = $1)
                     Rows Removed by Filter: 1
                     Buffers: shared hit=14
         ->  Seq Scan on titles ts  (cost=0.00..1.07 rows=7 width=16) (actual time=0.001..0.001 rows=4 loops=2)
               Buffers: shared hit=2
   ->  Index Scan using idx_salary_empid on salary s  (cost=0.43..0.93 rows=1 width=24) (actual time=0.009..0.009 rows=1 loops=2)
         Index Cond: (employee_id = de.employee_id)
         Filter: (to_date = $3)
         Rows Removed by Filter: 15
         Buffers: shared hit=14
 Planning:
   Buffers: shared hit=200
 Planning Time: 1.802 ms
 Execution Time: 0.316 ms
(65 rows)
```
## Цель достигнута!

## Выводы
При оптимизации структуру базы необходимо учитывать логигику всех запросов работающих совокупно с её данными, поскольку не всегда оптимизация структуры для одного запроса благотворно повлияет на производительность другого запроса работающего с этой структурой. 

