# ДЗ на тему - Репликациия
1) Подготовил две виртуальные машины PG1 и PG2, об на версии  
```
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
```
2) Добавил на PG1 таблицу для записи test и для чтения test2. Заполнил test случайными 10ю знпчениями. 
```
postgres=# create table test (id int, fio varchar(20));
CREATE TABLE
postgres=# create table test2 (id int, fio varchar(20));
CREATE TABLE
postgres=# insert into test
select
  generate_series(1,10) as id,
  md5(random()::text)::varchar(20) as fio;
INSERT 0 10
postgres=# select * from test limit 2;
 id |         fio
----+----------------------
  1 | 43d216137d3adf02cb2c
  2 | 1ba1556c7b0fead74202
(2 rows)

postgres=#
```
3) Добавил на PG2 таблицу для записи test2 и для чтения test. Заполнил test2 случайными 10ю знпчениями.
```
postgres=# create table test (id int, fio varchar(20));
CREATE TABLE
postgres=# create table test2 (id int, fio varchar(20));
CREATE TABLE
postgres=# insert into test2
select
  generate_series(1,10) as id,
  md5(random()::text)::varchar(20) as fio;
INSERT 0 10
postgres=# select * from test2 limit 2;
 id |         fio
----+----------------------
  1 | 91bcce2cc6b14312a178
  2 | b54a6c7419e706b937af
(2 rows)

postgres=#
```
4) На обоих инстанциях выставил режим репликации logical. далее перезагрузил СУБД для применения параметра (sudo pg_ctlcluster 14 main restart).
```
postgres=# postgres=# ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM
postgres=# show wal_level;
 wal_level
-----------
 replica
(1 row)

postgres=#
```
5) На первой машине PG1 создал публикация для таблицы test.
```
fedor@pg1:~$ sudo -u postgres psql
could not change directory to "/home/fedor": Permission denied
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# CREATE PUBLICATION logpubl_test FOR TABLE test;
CREATE PUBLICATION
postgres=# \dRp+
                          Publication logpubl_test
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test"

postgres=# \password
Enter new password for user "postgres":
Enter it again:
postgres=#
```
6) На второй машине PG2 создал публикация для таблицы test2.
```
fedor@pg2:~$ sudo -u postgres psql
could not change directory to "/home/fedor": Permission denied
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# CREATE PUBLICATION logpubl_test2 FOR TABLE test2;
CREATE PUBLICATION
postgres=# \dRp+
                          Publication logpubl_test2
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test2"

postgres=# \password
Enter new password for user "postgres":
Enter it again:
postgres=#
```
7) на первой машине PG1 создаю подписку для таблицы test2 для публикации одноименной таблицы на PG2
```
CREATE SUBSCRIPTION logsub_test2 
CONNECTION 'host=192.168.1.102 port=5432 user=postgres password=!Q@W3e4r5t dbname=postgres' 
PUBLICATION logpubl_test2 WITH (copy_data = true);
```
8) м
