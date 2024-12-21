# ДЗ на тему - Репликациия
1) Подготовил две виртуальные машины PG1 и PG2, обе на версии (psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))). Настроил на обоих машинах конфигурацию pg_hba.conf в разделе ipv4, для двустороннего подключения по 5432. Добавил на обоих машинах в postgresql.conf listen_addresses = 'localhost, ip-подключающейся_машины' и wal_level = logical. перезагрузил СУБД на обоих машинах.

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
3) Добавил на PG2 таблицу для записи test2 и для чтения test. Заполнил test2 случайными 10ю значениями.
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
4) На всех инстанциях проверил режим репликации.
```
postgres=# show wal_level;
 wal_level
-----------
 logical
(1 row)
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
7) На первой машине PG1 создаю подписку для таблицы test2 для публикации одноименной таблицы на PG2
```
postgres=# CREATE SUBSCRIPTION logsub_test2
CONNECTION 'host=192.168.1.102 port=5432 user=postgres password=!Q@W3e4r5t dbname=postgres'
PUBLICATION logpubl_test2 WITH (copy_data = true);
NOTICE:  created replication slot "logsub_test2" on publisher
CREATE SUBSCRIPTION
postgres=#
```
8) На второй машине PG2 создаю подписку для таблицы test для публикации одноименной таблицы на PG1
```
postgres=# CREATE SUBSCRIPTION logsub_test
CONNECTION 'host=192.168.1.179 port=5432 user=postgres password=!Q@W3e4r5t dbname=postgres'
PUBLICATION logpubl_test WITH (copy_data = true);
NOTICE:  created replication slot "logsub_test" on publisher
CREATE SUBSCRIPTION
postgres=#
```
9) Проверяем работу репликации.  Сначала добавляем строку на PG1 в таблицу test и проверяем её применение на PG2 в одноименной таблице.
```
fedor@pg1:~$ sudo -u postgres psql
could not change directory to "/home/fedor": Permission denied
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# select * from test;
 id |         fio
----+----------------------
  1 | 43d216137d3adf02cb2c
  2 | 1ba1556c7b0fead74202
  3 | 1ca64bd1196e89db0607
  4 | 6fdd8592927c8a5de71d
  5 | ec1680a468460d45c3de
  6 | eb66d83685e99416daba
  7 | 108c4391ef473dd02340
  8 | acb0b29cd478bfede54f
  9 | b89cbc6c3a087a8e25c5
 10 | 9f969a9b3e76a59f9f9d
(10 rows)

postgres=# insert into test select 11, 'New_text_1';
INSERT 0 1
postgres=# ^
```
проверяем применение строки на PG2
```
fedor@pg2:~$ sudo -u postgres psql -h 192.168.1.179
could not change directory to "/home/fedor": Permission denied
Password for user postgres:
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# select * from test;
 id |         fio
----+----------------------
  1 | 43d216137d3adf02cb2c
  2 | 1ba1556c7b0fead74202
  3 | 1ca64bd1196e89db0607
  4 | 6fdd8592927c8a5de71d
  5 | ec1680a468460d45c3de
  6 | eb66d83685e99416daba
  7 | 108c4391ef473dd02340
  8 | acb0b29cd478bfede54f
  9 | b89cbc6c3a087a8e25c5
 10 | 9f969a9b3e76a59f9f9d
 11 | New_text_1
(11 rows)

postgres=#
```
10) Теперь добавляем строку на PG2 в таблицу test2 и проверяем её применение на PG1 в одноименной таблице.
```
fedor@pg2:~$ sudo -u postgres psql -h 192.168.1.179
could not change directory to "/home/fedor": Permission denied
Password for user postgres:
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# select * from test2;
 id |         fio
----+----------------------
  1 | 91bcce2cc6b14312a178
  2 | b54a6c7419e706b937af
  3 | 1c30fbf84b211755ff77
  4 | 28ebc212c7e9b987f4a7
  5 | cdb83cd9fd497d6a7009
  6 | c22a319ff196c30e91ce
  7 | 10f55963af8570050deb
  8 | 8c9c520dc2be8046c3d6
  9 | 05daee785f9416e94e26
 10 | 5ce8f7c9fd83da3e9bd9
(10 rows)

postgres=# insert into test2 select 11, 'New_text_2';
INSERT 0 1
postgres=#

```
проверяем применение строки на PG1
```
fedor@pg1:~$ sudo -u postgres psql
could not change directory to "/home/fedor": Permission denied
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# select * from test2;
 id |         fio
----+----------------------
  1 | 91bcce2cc6b14312a178
  2 | b54a6c7419e706b937af
  3 | 1c30fbf84b211755ff77
  4 | 28ebc212c7e9b987f4a7
  5 | cdb83cd9fd497d6a7009
  6 | c22a319ff196c30e91ce
  7 | 10f55963af8570050deb
  8 | 8c9c520dc2be8046c3d6
  9 | 05daee785f9416e94e26
 10 | 5ce8f7c9fd83da3e9bd9
 11 | New_text_2
(11 rows)

postgres=#
```
11) Далее запустил 3ю машину PG15 с добавлением её в конфиги pg_hba.conf на PG1, PG2 для подключения к ним с целью репликации таблиц на PG15. Плюс на обоих машинах (PG1, PG2) в postgresql.conf listen_addresses добавил IP машины PG15 и перезагрузил их кластера.
  
12)  На третьей машине PG15 создал пустые таблицы с идеинтичной структурой для репликации. Далее создал подписки на одноименные таблицы для записи с PG1 и PG2.  Спустя несколько секунд проверил записи на локальных таблицах (таблицы заполнились средствами репликации с PG1 и PG2)
```
fedor@pg15:~$ su postgres
Password:
postgres@pg15:/home/fedor$ psql
could not change directory to "/home/fedor": Permission denied
psql (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
Type "help" for help.

postgres=# create table test (id int, fio varchar(20));
CREATE TABLE
postgres=# create table test2 (id int, fio varchar(20));
CREATE TABLE
postgres=# CREATE SUBSCRIPTION logsub_test_pg1
CONNECTION 'host=192.168.1.179 port=5432 user=postgres password=!Q@W3e4r5t dbname=postgres'
PUBLICATION logpubl_test WITH (copy_data = true);
NOTICE:  created replication slot "logsub_test_pg1" on publisher
CREATE SUBSCRIPTION
postgres=# CREATE SUBSCRIPTION logsub_test2_pg2
CONNECTION 'host=192.168.1.102 port=5432 user=postgres password=!Q@W3e4r5t dbname=postgres'
PUBLICATION logpubl_test2 WITH (copy_data = true);
NOTICE:  created replication slot "logsub_test2_pg2" on publisher
CREATE SUBSCRIPTION
postgres=# select * from test;
 id |         fio
----+----------------------
  1 | 43d216137d3adf02cb2c
  2 | 1ba1556c7b0fead74202
  3 | 1ca64bd1196e89db0607
  4 | 6fdd8592927c8a5de71d
  5 | ec1680a468460d45c3de
  6 | eb66d83685e99416daba
  7 | 108c4391ef473dd02340
  8 | acb0b29cd478bfede54f
  9 | b89cbc6c3a087a8e25c5
 10 | 9f969a9b3e76a59f9f9d
 11 | New_text_1
(11 rows)

postgres=# select * from test2;
 id |         fio
----+----------------------
  1 | 91bcce2cc6b14312a178
  2 | b54a6c7419e706b937af
  3 | 1c30fbf84b211755ff77
  4 | 28ebc212c7e9b987f4a7
  5 | cdb83cd9fd497d6a7009
  6 | c22a319ff196c30e91ce
  7 | 10f55963af8570050deb
  8 | 8c9c520dc2be8046c3d6
  9 | 05daee785f9416e94e26
 10 | 5ce8f7c9fd83da3e9bd9
 11 | New_text_2
(11 rows)

postgres=#
```

