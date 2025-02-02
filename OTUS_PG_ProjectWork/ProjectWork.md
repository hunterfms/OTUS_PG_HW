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
-- Изменям значение последней выданной зарплаты сотрудникам, у которых она меньше среднего прожиточного минимума, выше этого минимума на 33% от их текущей зарплаты.
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
8) Создаем резервную копию базы  employees без структуры и восстнавливаем её на этом же сервере под именем employees_1 и employees_1. К защите планирую  восстановить три копии базы, что бы продемонстрировать производительность ДО на одной базе и ПОСЛЕ доработки структуры на другой, и после нормализации структуры на третей базе. Команды следующие: 
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
9)  Проверка производительности запросов на неоптимизированной структуре базы с актуальным планом explain (analyze)
 а. Выводим актуальный план запроса Select через explain (analyze) который выводит 107728 сотрудников с зарплатами менее прожиточного минимума. Execution Time: 2565.833 ms (а вот первый запуск длился в пределах Execution Time: 411367.804 ms стой же структурой плана в резульаттах, скорее всего из за расчета первого плана запроса. Далее применялся ANALYSE;, что бы выровнить время отработки до постоянного значения с применением уже наработанного плана.)
```
employees=# vacuum;
VACUUM
employees=# analyse;
ANALYZE
employees=# explain (analyze, buffers)
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
ORDER BY s.employee_id; select max(to_date) from employees.salary)))
                                                                                                        QUERY PLAN

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------
 Sort  (cost=197927.75..198273.42 rows=138268 width=67) (actual time=2527.542..2548.613 rows=152067 loops=1)
   Sort Key: s.employee_id
   Sort Method: external merge  Disk: 9584kB
   Buffers: shared hit=15779 read=52039, temp read=9211 written=10088
   InitPlan 1 (returns $1)
     ->  Finalize Aggregate  (cost=33927.96..33927.97 rows=1 width=4) (actual time=424.734..424.793 rows=1 loops=1)
           Buffers: shared hit=673 read=17442
           ->  Gather  (cost=33927.75..33927.96 rows=2 width=4) (actual time=424.724..424.784 rows=3 loops=1)
                 Workers Planned: 2
                 Workers Launched: 2
                 Buffers: shared hit=673 read=17442
                 ->  Partial Aggregate  (cost=32927.75..32927.76 rows=1 width=4) (actual time=290.269..290.270 rows=1 loops=3)
                       Buffers: shared hit=673 read=17442
                       ->  Parallel Seq Scan on salary  (cost=0.00..29965.20 rows=1185020 width=4) (actual time=0.022..174.837 rows=948016 loops=3)
                             Buffers: shared hit=673 read=17442
   InitPlan 2 (returns $3)
     ->  Finalize Aggregate  (cost=33927.96..33927.97 rows=1 width=4) (actual time=433.484..433.510 rows=1 loops=1)
           Buffers: shared hit=769 read=17346
           ->  Gather  (cost=33927.75..33927.96 rows=2 width=4) (actual time=433.392..433.499 rows=3 loops=1)
                 Workers Planned: 2
                 Workers Launched: 2
                 Buffers: shared hit=769 read=17346
                 ->  Partial Aggregate  (cost=32927.75..32927.76 rows=1 width=4) (actual time=392.718..392.719 rows=1 loops=3)
                       Buffers: shared hit=769 read=17346
                       ->  Parallel Seq Scan on salary salary_1  (cost=0.00..29965.20 rows=1185020 width=4) (actual time=0.025..152.525 rows=948016 loops=3)
                             Buffers: shared hit=769 read=17346
   ->  Hash Join  (cost=40370.89..112592.22 rows=138268 width=67) (actual time=2182.850..2487.778 rows=152067 loops=1)
         Hash Cond: (s.employee_id = e2.id)
         Buffers: shared hit=15779 read=52039, temp read=8013 written=8886
         ->  Seq Scan on salary s  (cost=0.00..67885.82 rows=607224 width=16) (actual time=889.819..1104.157 rows=107728 loops=1)
               Filter: ((amount < 67690) AND ((from_date >= $1) OR (to_date = $3)))
               Rows Removed by Filter: 2736319
               Buffers: shared hit=2306 read=52039
         ->  Hash  (cost=39740.05..39740.05 rows=50467 width=50) (actual time=1292.889..1296.257 rows=371243 loops=1)
               Buckets: 131072 (originally 65536)  Batches: 2 (originally 1)  Memory Usage: 17213kB
               Buffers: shared hit=13473, temp read=5979 written=8596
               ->  Hash Join  (cost=29947.90..39740.05 rows=50467 width=50) (actual time=979.030..1203.403 rows=371243 loops=1)
                     Hash Cond: (t2.employee_id = e2.id)
                     Buffers: shared hit=13473, temp read=5979 written=6852
                     ->  Seq Scan on title t2  (cost=0.00..7625.08 rows=443308 width=19) (actual time=0.005..24.244 rows=443308 loops=1)
                           Buffers: shared hit=3192
                     ->  Hash  (cost=29520.96..29520.96 rows=34155 width=31) (actual time=978.934..982.301 rows=240124 loops=1)
                           Buckets: 131072 (originally 65536)  Batches: 4 (originally 1)  Memory Usage: 7169kB
                           Buffers: shared hit=10281, temp read=2447 written=4515
                           ->  Hash Join  (cost=23252.15..29520.96 rows=34155 width=31) (actual time=760.936..921.750 rows=240124 loops=1)
                                 Hash Cond: (e2.id = e.id)
                                 Buffers: shared hit=10281, temp read=2447 written=3320
                                 ->  Seq Scan on employee e2  (cost=0.00..5481.24 rows=300024 width=23) (actual time=0.002..17.036 rows=300024 loops=1)
                                       Buffers: shared hit=2481
                                 ->  Hash  (cost=22825.21..22825.21 rows=34155 width=8) (actual time=760.839..764.205 rows=240124 loops=1)
                                       Buckets: 262144 (originally 65536)  Batches: 2 (originally 1)  Memory Usage: 6736kB
                                       Buffers: shared hit=7800, temp read=994 written=2277
                                       ->  HashAggregate  (cost=22142.11..22483.66 rows=34155 width=8) (actual time=642.742..725.329 rows=240124 loops=1)
                                             Group Key: e.id
                                             Batches: 21  Memory Usage: 8249kB  Disk Usage: 7336kB
                                             Buffers: shared hit=7800, temp read=994 written=1867
                                             InitPlan 3 (returns $4)
                                               ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.007..0.008 rows=1 loops=1)
                                             InitPlan 4 (returns $5)
                                               ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.003 rows=1 loops=1)
                                             ->  Gather  (cost=18299.65..22056.70 rows=34155 width=8) (actual time=497.556..584.168 rows=240124 loops=1)
                                                   Workers Planned: 1
                                                   Params Evaluated: $4, $5
                                                   Workers Launched: 1
                                                   Buffers: shared hit=7800, temp read=124 written=263
                                                   ->  HashAggregate  (cost=17299.65..17641.20 rows=34155 width=8) (actual time=476.468..502.955 rows=120062 loops=2)
                                                         Group Key: e.id
                                                         Batches: 5  Memory Usage: 8241kB  Disk Usage: 720kB
                                                         Buffers: shared hit=7800, temp read=124 written=263
                                                         Worker 0:  Batches: 5  Memory Usage: 11057kB  Disk Usage: 680kB
                                                         ->  Hash Join  (cost=11636.05..17214.26 rows=34155 width=8) (actual time=254.970..421.572 rows=120062 loops=2)
                                                               Hash Cond: (de.department_id = d.id)
                                                               Buffers: shared hit=7800
                                                               ->  Parallel Hash Join  (cost=11634.85..16743.43 rows=34155 width=13) (actual time=242.318..363.883 rows=120062 loops=2)
                                                                     Hash Cond: (e.id = t.employee_id)
                                                                     Buffers: shared hit=7786
                                                                     ->  Parallel Seq Scan on employee e  (cost=0.00..4245.85 rows=176485 width=8) (actual time=0.004..12.903 rows=150012 loops=2)
                                                                           Buffers: shared hit=2481
                                                                     ->  Parallel Hash  (cost=11207.91..11207.91 rows=34155 width=21) (actual time=239.773..239.990 rows=120062 loops=2)
                                                                           Buckets: 262144 (originally 65536)  Batches: 1 (originally 1)  Memory Usage: 16768kB
                                                                           Buffers: shared hit=5305
                                                                           ->  Parallel Hash Join  (cost=6270.52..11207.91 rows=34155 width=21) (actual time=89.007..183.388rows=120062 loops=2)
                                                                                 Hash Cond: (de.employee_id = t.employee_id)
                                                                                 Buffers: shared hit=5305
                                                                                 ->  Parallel Seq Scan on department_employee de  (cost=0.00..4551.26 rows=65020 width=13) (actual time=0.022..28.381 rows=120062 loops=2)
                                                                                       Filter: (to_date > $5)
                                                                                       Rows Removed by Filter: 45740
                                                                                       Buffers: shared hit=2113
                                                                                 ->  Parallel Hash  (cost=5500.90..5500.90 rows=61570 width=8) (actual time=88.000..88.001 rows=120062 loops=2)
                                                                                       Buckets: 262144  Batches: 1  Memory Usage: 11456kB
                                                                                       Buffers: shared hit=3192
                                                                                       ->  Parallel Seq Scan on title t  (cost=0.00..5500.90 rows=61570 width=8) (actual time=0.017..38.282 rows=120062 loops=2)
                                                                                             Filter: (to_date > $4)
                                                                                             Rows Removed by Filter: 101592
                                                                                             Buffers: shared hit=3192
                                                               ->  Hash  (cost=1.09..1.09 rows=9 width=5) (actual time=12.624..12.625
rows=9 loops=2)
                                                                     Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                                                     Buffers: shared hit=2
                                                                     ->  Seq Scan on department d  (cost=0.00..1.09 rows=9 width=5) (actual time=12.611..12.613 rows=9 loops=2)
                                                                           Buffers: shared hit=2 Planning: Buffers: shared hit=48
 Planning Time: 0.638 ms
 JIT:
   Functions: 136
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 8.810 ms, Inlining 0.000 ms, Optimization 2.650 ms, Emission 78.325 ms, Total 89.786 ms
 Execution Time: 2565.833 ms
(108 rows)

employees=#
```
 б. Выводим актуальный план запроса Update через explain (analyze, buffers) при котором реально обновляются 107728 строки таблицы зарплат employees.salary. Execution Execution Time: 18231.000 ms
```
employees=# explain (analyze, buffers)
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
and amount < 67690;= (select max(to_date) from employees.salary)))))
                                                                                                          QUERY PLAN

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------
 Update on salary  (cost=332633.37..413697.95 rows=0 width=0) (actual time=18226.598..18226.611 rows=0 loops=1)
   Buffers: shared hit=343656 read=121150 dirtied=35292 written=15957, temp read=12637 written=12989
   InitPlan 1 (returns $0)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=392.170..392.171 rows=1 loops=1)
           Buffers: shared hit=897 read=17218
           ->  Seq Scan on salary salary_1  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.010..192.163 rows=2844047 loops=1)
                 Buffers: shared hit=897 read=17218
   InitPlan 2 (returns $1)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=390.128..390.128 rows=1 loops=1)
           Buffers: shared hit=929 read=17186
           ->  Seq Scan on salary salary_2  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.005..188.809 rows=2844047 loops=1)
                 Buffers: shared hit=929 read=17186
   InitPlan 3 (returns $2)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=373.878..373.879 rows=1 loops=1)
           Buffers: shared hit=961 read=17154
           ->  Seq Scan on salary salary_3  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.002..185.437 rows=2844047 loops=1)
                 Buffers: shared hit=961 read=17154
   InitPlan 4 (returns $3)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=542.888..542.888 rows=1 loops=1)
           Buffers: shared hit=993 read=17122
           ->  Seq Scan on salary salary_4  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.004..244.300 rows=2844047 loops=1)
                 Buffers: shared hit=993 read=17122
   InitPlan 5 (returns $4)
     ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.004..0.004 rows=1 loops=1)
   ->  Hash Semi Join  (cost=117970.97..199035.55 rows=195499 width=64) (actual time=3127.527..4045.607 rows=107728 loops=1)
         Hash Cond: (salary.employee_id = s.employee_id)
         Buffers: shared hit=18266 read=103884 written=7974, temp read=12637 written=12989
         ->  Seq Scan on salary  (cost=0.00..67885.82 rows=607224 width=22) (actual time=817.409..1657.324 rows=107728 loops=1)
               Filter: ((amount < 67690) AND ((from_date >= $0) OR (to_date = $1)))
               Rows Removed by Filter: 2736319
               Buffers: shared hit=1827 read=52518 written=7974
         ->  Hash  (cost=115919.58..115919.58 rows=78351 width=82) (actual time=2310.044..2310.051 rows=107728 loops=1)
               Buckets: 65536  Batches: 2  Memory Usage: 6847kB
               Buffers: shared hit=16439 read=51366, temp read=11640 written=12699
               ->  Hash Join  (cost=45177.89..115919.58 rows=78351 width=82) (actual time=2003.592..2277.994 rows=107728 loops=1)
                     Hash Cond: (s.employee_id = e2.id)
                     Buffers: shared hit=16439 read=51366, temp read=11640 written=11992
                     ->  Seq Scan on salary s  (cost=0.00..67885.82 rows=607224 width=14) (actual time=916.786..1127.951 rows=107728 loops=1)
                           Filter: ((amount < 67690) AND ((from_date >= $2) OR (to_date = $3)))
                           Rows Removed by Filter: 2736319
                           Buffers: shared hit=2979 read=51366
                     ->  Hash  (cost=44820.43..44820.43 rows=28597 width=68) (actual time=1086.653..1086.659 rows=240124 loops=1)
                           Buckets: 131072 (originally 32768)  Batches: 4 (originally 1)  Memory Usage: 7169kB
                           Buffers: shared hit=13460, temp read=9260 written=11632
                           ->  Hash Join  (cost=35246.97..44820.43 rows=28597 width=68) (actual time=879.583..1027.327 rows=240124 loops=1)
                                 Hash Cond: (t2.employee_id = e2.id)
                                 Buffers: shared hit=13460, temp read=9260 written=9612
                                 ->  Seq Scan on title t2  (cost=0.00..8733.35 rows=147769 width=14) (actual time=0.017..43.796 rows=240124 loops=1)
                                       Filter: (to_date > $4)
                                       Rows Removed by Filter: 203184
                                       Buffers: shared hit=3192
                                 ->  Hash  (cost=34521.19..34521.19 rows=58063 width=54) (actual time=879.510..879.516 rows=240124 loops=1)
                                       Buckets: 131072 (originally 65536)  Batches: 4 (originally 1)  Memory Usage: 7169kB
                                       Buffers: shared hit=10268, temp read=6796 written=8816
                                       ->  Hash Join  (cost=28252.37..34521.19 rows=58063 width=54) (actual time=668.640..824.852 rows=240124 loops=1)
                                             Hash Cond: (e2.id = "ANY_subquery".id)
                                             Buffers: shared hit=10268, temp read=6796 written=7148
                                             ->  Seq Scan on employee e2  (cost=0.00..5481.24 rows=300024 width=14) (actual time=0.005..28.035 rows=300024 loops=1)
                                                   Buffers: shared hit=2481
                                             ->  Hash  (cost=27526.59..27526.59 rows=58063 width=40) (actual time=668.576..668.581 rows=240124 loops=1)
                                                   Buckets: 131072 (originally 65536)  Batches: 4 (originally 1)  Memory Usage: 7169kB
                                                   Buffers: shared hit=7787, temp read=4165 written=5834
                                                   ->  Subquery Scan on "ANY_subquery"  (cost=26365.33..27526.59 rows=58063 width=40) (actual time=527.978..620.452 rows=240124 loops=1)
                                                         Buffers: shared hit=7787, temp read=4165 written=4517
                                                         ->  HashAggregate  (cost=26365.33..26945.96 rows=58063 width=8) (actual time=527.792..584.728 rows=240124 loops=1)
                                                               Group Key: e.id
                                                               Batches: 5  Memory Usage: 8241kB  Disk Usage: 3816kB
                                                               Buffers: shared hit=7787, temp read=4165 written=4517
                                                               InitPlan 6 (returns $5)
                                                                 ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=1)
                                                               InitPlan 7 (returns $6)
                                                                 ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=1)
                                                               ->  Hash Join  (cost=18234.82..26220.14 rows=58063 width=8) (actual time=245.983..466.339 rows=240124 loops=1)
                                                                     Hash Cond: (de.department_id = d.id)
                                                                     Buffers: shared hit=7787, temp read=3696 written=3696
                                                                     ->  Hash Join  (cost=18233.61..25420.57 rows=58063 width=13) (actual time=245.961..423.941 rows=240124 loops=1)
                                                                           Hash Cond: (e.id = t.employee_id)
                                                                           Buffers: shared hit=7786, temp read=3696 written=3696
                                                                           ->  Seq Scan on employee e  (cost=0.00..5481.24 rows=300024 width=8) (actual time=0.002..18.472 rows=300024 loops=1)
                                                                                 Buffers: shared hit=2481
                                                                           ->  Hash  (cost=17507.83..17507.83 rows=58063 width=21) (actual time=245.897..245.899 rows=240124 loops=1)
                                                                                 Buckets: 262144 (originally 65536)  Batches: 4 (originally 1)  Memory Usage: 8260kB
                                                                                 Buffers: shared hit=5305, temp read=1231 written=1832
                                                                                 ->  Hash Join  (cost=7639.71..17507.83 rows=58063 width=21) (actual time=70.261..204.481 rows=240124 loops=1)
                                                                                       Hash Cond: (t.employee_id = de.employee_id)
                                                                                       Buffers: shared hit=5305, temp read=1231 written=1231
                                                                                       ->  Seq Scan on title t  (cost=0.00..8733.35 rows=147769 width=8) (actual time=0.008..39.434 rows=240124 loops=1)
                                                                                             Filter: (to_date > $5)
                                                                                             Rows Removed by Filter: 203184
                                                                                             Buffers: shared hit=3192
                                                                                       ->  Hash  (cost=6258.04..6258.04 rows=110534 width=13) (actual time=70.155..70.156 rows=240124 loops=1)
                                                                                             Buckets: 262144 (originally 131072)  Batches: 2 (originally 1)  Memory Usage: 7322kB
                                                                                             Buffers: shared hit=2113, temp written=483
                                                                                             ->  Seq Scan on department_employee de  (cost=0.00..6258.04 rows=110534 width=13) (actual time=0.010..31.150 rows=240124 loops=1)
                                                                                                   Filter: (to_date > $6)
                                                                                                   Rows Removed by Filter: 91479
                                                                                                   Buffers: shared hit=2113
                                                                     ->  Hash  (cost=1.09..1.09 rows=9 width=5) (actual time=0.013..0.013 rows=9 loops=1)
                                                                           Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                                                           Buffers: shared hit=1
                                                                           ->  Seq Scan on department d  (cost=0.00..1.09 rows=9 width=5) (actual time=0.006..0.007 rows=9 loops=1)
                                                                                 Buffers: shared hit=1 Planning Time: 0.516 ms
 JIT:
   Functions: 93
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 2.958 ms, Inlining 0.000 ms, Optimization 1.727 ms, Emission 35.648 ms, Total 40.332 ms
 Execution Time: 18231.000 ms
(108 rows)

employees=#

```
### Доработка структуры таблиц для запроса.
10) Дорабатываем структуру базы - создаем структуру ключей и индексов под разработанные запросы на базе employees_1 (база для проверки обоих запросов с доработанной структурой). Выводим весь список индексов и ключей в таблицах схемы. Предполагается, что данные индексы буду пользоваться оптимизатором для более быстрой выбоки данных по условиям запросов.
```
employees=# \c employees_1
You are now connected to database "employees_1" as user "postgres".
employees_1=# CREATE UNIQUE INDEX idx_employee_id_primary ON employees.employee USING btree (id);
CREATE UNIQUE INDEX idx_department_id_primary ON employees.department USING btree (id);
CREATE INDEX idx_salary_amount ON employees.salary USING btree (amount);
CREATE UNIQUE INDEX idx_salary_empid_fromdate_primary ON employees.salary USING btree (employee_id, from_date, to_date);
CREATE INDEX idx_department_employee_depid ON employees.department_employee USING btree (department_id);
CREATE UNIQUE INDEX idx_departmen_temployee_empid_todate_primary ON employees.department_employee USING btree (employee_id, to_date);
CREATE INDEX idx_title_empid_todate ON employees.title USING btree (employee_id, to_date);
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
employees_1=# SELECT tablename, indexname, indexdef FROM pg_indexes where schemaname = 'employees';
      tablename      |                  indexname                   |                                                               indexdef

---------------------+----------------------------------------------+--------------------------------------------------------------------------------------------------------
------------------------------
 employee            | idx_employee_id_primary                      | CREATE UNIQUE INDEX idx_employee_id_primary ON employees.employee USING btree (id)
 department          | idx_department_id_primary                    | CREATE UNIQUE INDEX idx_department_id_primary ON employees.department USING btree (id)
 salary              | idx_salary_amount                            | CREATE INDEX idx_salary_amount ON employees.salary USING btree (amount)
 salary              | idx_salary_empid_fromdate_primary            | CREATE UNIQUE INDEX idx_salary_empid_fromdate_primary ON employees.salary USING btree (employee_id, fro
m_date, to_date)
 department_employee | idx_department_employee_depid                | CREATE INDEX idx_department_employee_depid ON employees.department_employee USING btree (department_id)
 department_employee | idx_departmen_temployee_empid_todate_primary | CREATE UNIQUE INDEX idx_departmen_temployee_empid_todate_primary ON employees.department_employee USING
 btree (employee_id, to_date)
 title               | idx_title_empid_todate                       | CREATE INDEX idx_title_empid_todate ON employees.title USING btree (employee_id, to_date)
(7 rows)
```
### Проверка производительности запросов после доработки структуры. 
11) Проверка производительности запросов на оптимизированной структуре базы с актуальным планам explain (analyze, buffers) показала примерно же время отработки по причине того, что оптимизатор запросов посчитал более выгодным где то использовать полное сканирование таблиц, а где то индексы гл без особых результатов, поскольку в запросах обрабатывалась значительная доля объёма данных базы.
```
employees_1=# explain (analyze, buffers)
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

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--------------------
 Merge Join  (cost=92122.00..167486.26 rows=142868 width=67) (actual time=1548.756..2706.769 rows=152067 loops=1)
   Merge Cond: (s.employee_id = t2.employee_id)
   Buffers: shared hit=969698 read=67190, temp read=498 written=637
   InitPlan 1 (returns $1)
     ->  Finalize Aggregate  (cost=33927.96..33927.97 rows=1 width=4) (actual time=406.749..406.807 rows=1 loops=1)
           Buffers: shared hit=6718 read=11397
           ->  Gather  (cost=33927.75..33927.96 rows=2 width=4) (actual time=406.511..406.795 rows=3 loops=1)
                 Workers Planned: 2
                 Workers Launched: 2
                 Buffers: shared hit=6718 read=11397
                 ->  Partial Aggregate  (cost=32927.75..32927.76 rows=1 width=4) (actual time=370.997..370.998 rows=1 loops=3)
                       Buffers: shared hit=6718 read=11397
                       ->  Parallel Seq Scan on salary  (cost=0.00..29965.20 rows=1185020 width=4) (actual time=0.014..164.922 rows=948016 loops=3)
                             Buffers: shared hit=6718 read=11397
   InitPlan 2 (returns $3)
     ->  Finalize Aggregate  (cost=33927.96..33927.97 rows=1 width=4) (actual time=443.333..443.357 rows=1 loops=1)
           Buffers: shared hit=6814 read=11301
           ->  Gather  (cost=33927.75..33927.96 rows=2 width=4) (actual time=439.656..443.335 rows=3 loops=1)
                 Workers Planned: 2
                 Workers Launched: 2
                 Buffers: shared hit=6814 read=11301
                 ->  Partial Aggregate  (cost=32927.75..32927.76 rows=1 width=4) (actual time=398.141..398.142 rows=1 loops=3)
                       Buffers: shared hit=6814 read=11301
                       ->  Parallel Seq Scan on salary salary_1  (cost=0.00..29965.20 rows=1185020 width=4) (actual time=0.017..182.330 rows=948016 loops=3)
                             Buffers: shared hit=6814 read=11301
   ->  Nested Loop  (cost=24264.20..74945.29 rows=85029 width=47) (actual time=1548.734..2533.618 rows=107728 loops=1)
         Buffers: shared hit=968837 read=62300, temp read=498 written=637
         ->  Merge Join  (cost=24263.77..36234.04 rows=34031 width=31) (actual time=698.588..910.044 rows=240124 loops=1)
               Merge Cond: (e2.id = e.id)
               Buffers: shared hit=579 read=10539, temp read=498 written=637
               ->  Index Scan using idx_employee_id_primary on employee e2  (cost=0.42..10284.78 rows=300024 width=23) (actual time=0.020..47.268 rows=300024 loops=1)
                     Buffers: shared hit=540 read=2778
               ->  Unique  (cost=24263.35..24433.50 rows=34031 width=8) (actual time=673.753..754.230 rows=240124 loops=1)
                     Buffers: shared hit=39 read=7761, temp read=498 written=637
                     InitPlan 3 (returns $4)
                       ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.008..0.008 rows=1 loops=1)
                     InitPlan 4 (returns $5)
                       ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=1)
                     ->  Sort  (cost=24263.32..24348.40 rows=34031 width=8) (actual time=673.751..701.777 rows=240124 loops=1)
                           Sort Key: e.id
                           Sort Method: external merge  Disk: 2824kB
                           Buffers: shared hit=39 read=7761, temp read=498 written=637
                           ->  Gather  (cost=17958.30..21701.71 rows=34031 width=8) (actual time=537.736..628.294 rows=240124 loops=1)
                                 Workers Planned: 1
                                 Params Evaluated: $4, $5
                                 Workers Launched: 1
                                 Buffers: shared hit=39 read=7761, temp read=145 written=283
                                 ->  HashAggregate  (cost=16958.30..17298.61 rows=34031 width=8) (actual time=519.053..546.016 rows=120062 loops=2)
                                       Group Key: e.id
                                       Batches: 5  Memory Usage: 10801kB  Disk Usage: 752kB
                                       Buffers: shared hit=39 read=7761, temp read=145 written=283
                                       Worker 0:  Batches: 5  Memory Usage: 11057kB  Disk Usage: 688kB
                                       ->  Hash Join  (cost=11633.98..16873.22 rows=34031 width=8) (actual time=287.004..461.583 rows=120062 loops=2)
                                             Hash Cond: (de.department_id = d.id)
                                             Buffers: shared hit=39 read=7761
                                             ->  Parallel Hash Join  (cost=11632.78..16740.63 rows=34031 width=13) (actual time=270.705..377.545 rows=120062 loops=2)
                                                   Hash Cond: (e.id = t.employee_id)
                                                   Buffers: shared hit=30 read=7756
                                                   ->  Parallel Seq Scan on employee e  (cost=0.00..4245.85 rows=176485 width=8) (actual time=0.006..21.531 rows=150012 loops=2)
                                                         Buffers: shared hit=1 read=2480
                                                   ->  Parallel Hash  (cost=11207.39..11207.39 rows=34031 width=21) (actual time=267.509..267.511 rows=120062 loops=2)
                                                         Buckets: 262144 (originally 65536)  Batches: 1 (originally 1)  Memory Usage: 16768kB
                                                         Buffers: shared hit=29 read=5276
                                                         ->  Parallel Hash Join  (cost=6270.52..11207.39 rows=34031 width=21) (actual time=106.925..223.483 rows=120062 loops=2)
                                                               Hash Cond: (de.employee_id = t.employee_id)
                                                               Buffers: shared hit=29 read=5276
                                                               ->  Parallel Seq Scan on department_employee de  (cost=0.00..4551.26 rows=65020 width=13) (actual time=0.031..37.957 rows=120062 loops=2)
                                                                     Filter: (to_date > $5)
                                                                     Rows Removed by Filter: 45740
                                                                     Buffers: shared read=2113
                                                               ->  Parallel Hash  (cost=5500.90..5500.90 rows=61570 width=8) (actual time=103.602..103.603 rows=120062 loops=2)
                                                                     Buckets: 262144  Batches: 1  Memory Usage: 11488kB
                                                                     Buffers: shared hit=29 read=3163
                                                                     ->  Parallel Seq Scan on title t  (cost=0.00..5500.90 rows=61570 width=8) (actual time=0.027..53.103 rows=120062 loops=2)
                                                                           Filter: (to_date > $4)
                                                                           Rows Removed by Filter: 101592
                                                                           Buffers: shared hit=29 read=3163
                                             ->  Hash  (cost=1.09..1.09 rows=9 width=5) (actual time=16.262..16.263 rows=9 loops=2)
                                                   Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                                   Buffers: shared hit=1 read=1
                                                   ->  Seq Scan on department d  (cost=0.00..1.09 rows=9 width=5) (actual time=16.248..16.251 rows=9 loops=2)
                                                         Buffers: shared hit=1 read=1
         ->  Index Scan using idx_salary_empid_fromdate_primary on salary s  (cost=0.43..1.12 rows=2 width=16) (actual time=0.006..0.007 rows=0 loops=240124)
               Index Cond: (employee_id = e2.id)
               Filter: ((amount < 67690) AND ((from_date >= $1) OR (to_date = $3)))
               Rows Removed by Filter: 10
               Buffers: shared hit=968258 read=51761
   ->  Index Scan using idx_title_empid_todate on title t2  (cost=0.42..20865.23 rows=443308 width=19) (actual time=0.013..97.227 rows=443308 loops=1)
         Buffers: shared hit=861 read=4890
 Planning:
   Buffers: shared hit=175 read=21 Planning Time: 1.690 ms
 JIT:
   Functions: 120
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 12.112 ms, Inlining 0.000 ms, Optimization 14.568 ms, Emission 73.030 ms, Total 99.710 ms
 Execution Time: 2723.204 ms
(97 rows)
```
Запрос по изменению данных со структурой ключей и ндексов отработал долще нежели без данной структуры.  Дополнительное время ушло на обновление структурированных индексов. 

```
employees_1=# explain (analyze, buffers)
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

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
------------------------------------
 Update on salary  (cost=293420.51..374556.68 rows=0 width=0) (actual time=33650.770..33650.787 rows=0 loops=1)
   Buffers: shared hit=1965670 read=219267 dirtied=51960 written=26625, temp read=10257 written=10609
   InitPlan 1 (returns $0)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=359.248..359.250 rows=1 loops=1)
           Buffers: shared hit=8973 read=9142
           ->  Seq Scan on salary salary_1  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.010..169.609 rows=2844047 loops=1)
                 Buffers: shared hit=8973 read=9142
   InitPlan 2 (returns $1)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=367.484..367.485 rows=1 loops=1)
           Buffers: shared hit=9005 read=9110
           ->  Seq Scan on salary salary_2  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.005..168.446 rows=2844047 loops=1)
                 Buffers: shared hit=9005 read=9110
   InitPlan 3 (returns $2)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=376.183..376.184 rows=1 loops=1)
           Buffers: shared hit=6348 read=11767
           ->  Seq Scan on salary salary_3  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.004..184.307 rows=2844047 loops=1)
                 Buffers: shared hit=6348 read=11767
   InitPlan 4 (returns $3)
     ->  Aggregate  (cost=53665.59..53665.60 rows=1 width=4) (actual time=388.311..388.311 rows=1 loops=1)
           Buffers: shared hit=6380 read=11735
           ->  Seq Scan on salary salary_4  (cost=0.00..46555.47 rows=2844047 width=4) (actual time=0.006..183.410 rows=2844047 loops=1)
                 Buffers: shared hit=6380 read=11735
   InitPlan 5 (returns $4)
     ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.004..0.004 rows=1 loops=1)
   ->  Hash Semi Join  (cost=78758.10..159894.28 rows=202279 width=64) (actual time=3569.332..5993.969 rows=107728 loops=1)
         Hash Cond: (salary.employee_id = s.employee_id)
         Buffers: shared hit=902286 read=185538 written=5722, temp read=10257 written=10609
         ->  Seq Scan on salary  (cost=0.00..67885.82 rows=601431 width=22) (actual time=757.900..2961.549 rows=107728 loops=1)
               Filter: ((amount < 67690) AND ((from_date >= $0) OR (to_date = $1)))
               Rows Removed by Filter: 2736319
               Buffers: shared hit=17978 read=36367 written=5722
         ->  Hash  (cost=76639.13..76639.13 rows=80958 width=82) (actual time=2811.376..2811.388 rows=107728 loops=1)
               Buckets: 65536  Batches: 2  Memory Usage: 6847kB
               Buffers: shared hit=884308 read=149171, temp read=9260 written=10319
               ->  Nested Loop  (cost=34655.53..76639.13 rows=80958 width=82) (actual time=1636.565..2768.417 rows=107728 loops=1)
                     Buffers: shared hit=884308 read=149171, temp read=9260 written=9612
                     ->  Hash Join  (cost=34655.10..44227.51 rows=28493 width=68) (actual time=871.933..1050.547 rows=240124 loops=1)
                           Hash Cond: (t2.employee_id = e2.id)
                           Buffers: shared hit=2584 read=10876, temp read=9260 written=9612
                           ->  Seq Scan on title t2  (cost=0.00..8733.35 rows=147769 width=14) (actual time=0.024..46.405 rows=240124 loops=1)
                                 Filter: (to_date > $4)
                                 Rows Removed by Filter: 203184
                                 Buffers: shared hit=100 read=3092
                           ->  Hash  (cost=33931.95..33931.95 rows=57852 width=54) (actual time=871.852..871.862 rows=240124 loops=1)
                                 Buckets: 131072 (originally 65536)  Batches: 4 (originally 1)  Memory Usage: 7169kB
                                 Buffers: shared hit=2484 read=7784, temp read=6796 written=8816
                                 ->  Hash Join  (cost=27663.14..33931.95 rows=57852 width=54) (actual time=670.742..819.399 rows=240124 loops=1)
                                       Hash Cond: (e2.id = "ANY_subquery".id)
                                       Buffers: shared hit=2484 read=7784, temp read=6796 written=7148
                                       ->  Seq Scan on employee e2  (cost=0.00..5481.24 rows=300024 width=14) (actual time=0.009..25.822 rows=300024 loops=1)
                                             Buffers: shared hit=2480 read=1
                                       ->  Hash  (cost=26939.99..26939.99 rows=57852 width=40) (actual time=670.660..670.668 rows=240124 loops=1)
                                             Buckets: 131072 (originally 65536)  Batches: 4 (originally 1)  Memory Usage: 7169kB
                                             Buffers: shared hit=4 read=7783, temp read=4165 written=5834
                                             ->  Subquery Scan on "ANY_subquery"  (cost=25782.95..26939.99 rows=57852 width=40) (actual time=534.394..624.716 rows=240124 loops=1)
                                                   Buffers: shared hit=4 read=7783, temp read=4165 written=4517
                                                   ->  HashAggregate  (cost=25782.95..26361.47 rows=57852 width=8) (actual time=534.204..589.505 rows=240124 loops=1)
                                                         Group Key: e.id
                                                         Batches: 5  Memory Usage: 8241kB  Disk Usage: 3816kB
                                                         Buffers: shared hit=4 read=7783, temp read=4165 written=4517
                                                         InitPlan 6 (returns $5)
                                                           ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.003 rows=1 loops=1)
                                                         InitPlan 7 (returns $6)
                                                           ->  Result  (cost=0.00..0.01 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=1)
                                                         ->  Hash Join  (cost=18230.07..25638.29 rows=57852 width=8) (actual time=254.477..474.638 rows=240124 loops=1)
                                                               Hash Cond: (de.department_id = d.id)
                                                               Buffers: shared hit=4 read=7783, temp read=3696 written=3696
                                                               ->  Hash Join  (cost=18228.87..25413.72 rows=57852 width=13) (actual time=254.451..433.286 rows=240124 loops=1)
                                                                     Hash Cond: (e.id = t.employee_id)
                                                                     Buffers: shared hit=4 read=7782, temp read=3696 written=3696
                                                                     ->  Seq Scan on employee e  (cost=0.00..5481.24 rows=300024 width=8) (actual time=0.002..22.479 rows=300024 loop
s=1)
                                                                           Buffers: shared hit=1 read=2480
                                                                     ->  Hash  (cost=17505.72..17505.72 rows=57852 width=21) (actual time=254.370..254.373 rows=240124 loops=1)
                                                                           Buckets: 262144 (originally 65536)  Batches: 4 (originally 1)  Memory Usage: 8260kB
                                                                           Buffers: shared hit=3 read=5302, temp read=1231 written=1832
                                                                           ->  Hash Join  (cost=7639.71..17505.72 rows=57852 width=21) (actual time=73.464..212.503 rows=240124 loops
=1)
                                                                                 Hash Cond: (t.employee_id = de.employee_id)
                                                                                 Buffers: shared hit=3 read=5302, temp read=1231 written=1231
                                                                                 ->  Seq Scan on title t  (cost=0.00..8733.35 rows=147769 width=8) (actual time=0.008..42.550 rows=24
0124 loops=1)
                                                                                       Filter: (to_date > $5)
                                                                                       Rows Removed by Filter: 203184
                                                                                       Buffers: shared hit=3 read=3189
                                                                                 ->  Hash  (cost=6258.04..6258.04 rows=110534 width=13) (actual time=73.320..73.321 rows=240124 loops
=1)
                                                                                       Buckets: 262144 (originally 131072)  Batches: 2 (originally 1)  Memory Usage: 7322kB
                                                                                       Buffers: shared read=2113, temp written=483
                                                                                       ->  Seq Scan on department_employee de  (cost=0.00..6258.04 rows=110534 width=13) (actual time
=0.014..33.519 rows=240124 loops=1)
                                                                                             Filter: (to_date > $6)
                                                                                             Rows Removed by Filter: 91479
                                                                                             Buffers: shared read=2113
                                                               ->  Hash  (cost=1.09..1.09 rows=9 width=5) (actual time=0.015..0.016 rows=9 loops=1)
                                                                     Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                                                     Buffers: shared read=1
                                                                     ->  Seq Scan on department d  (cost=0.00..1.09 rows=9 width=5) (actual time=0.009..0.010 rows=9 loops=1)
                                                                           Buffers: shared read=1
                     ->  Index Scan using idx_salary_empid_fromdate_primary on salary s  (cost=0.43..1.12 rows=2 width=14) (actual time=0.007..0.007 rows=0 loops=240124)
                           Index Cond: (employee_id = e2.id)
                           Filter: ((amount < 67690) AND ((from_date >= $2) OR (to_date = $3)))
                           Rows Removed by Filter: 10
                           Buffers: shared hit=881724 read=138295
 Planning:
   Buffers: shared hit=191 read=24
 Planning Time: 1.574 ms
 JIT:
   Functions: 91
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 2.702 ms, Inlining 0.000 ms, Optimization 1.236 ms, Emission 32.274 ms, Total 36.212 ms
 Execution Time: 33654.836 ms
(107 rows)
```
Выводы: Для запросов которые выводит в результатирующий набор большую часть данных базы индексы не всегда приводят к росту производительности, поскольку оптимизатор в большенстве случаев принимает решение использовать полное сканирование таблицы как более выгодный способ. В запросов массового изменения данных структура индексов только усложняет отработку запроса коскольку требует дополнительного времени на изменение данных самих индексов.

Далее будем тестировать запросы со сбором минимального набота данных при котором структура индексов, ключей и дополнительные изменения столбцов должны показать лучшие рузультаты.

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
### Разработка запроса к базе с минимальным результатирующим набором.
Разрабатываем азпрос к базе с минимальным результатирующим набором, для проверки работы оптимизированной структуры БД. Проверять результаты оптмиизации будем на базе employees_2 без индексов и с индексами, где уже проведена работа по нормализации структуры данных в пунктах 12,13,14.
15) Разрабатываем азпрос и отрабатываем его на базе без индексов с выводом плана.
```
```
16) Отрабатываем запрос на базе синдексами и выводом плана после применения ANALYSE;.
```
```
Выводы по результатам: 

## Выводы

