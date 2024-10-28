# ДЗ по теме - Логический уровень Postgresql

1) Развернул схему на виртуальной машине (созданной еще в прошлых ДЗ), для проверки системы предоставления доступа в Postgresql по пунктам текущего ДЗ.
```
login as: fedor
fedor@192.168.1.224's password:
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-124-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
New release '24.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Sun Oct 27 13:16:00 2024
fedor@pgindocker:~$ sudo pg_lsclusters
[sudo] password for fedor:
Ver Cluster Port Status Owner    Data directory    Log file
14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log
fedor@pgindocker:~$ sudo -u postgres psql
could not change directory to "/home/fedor": Permission denied
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# create database testdb;
CREATE DATABASE
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# create schema testnm;
CREATE SCHEMA
testdb=# create table t1 (c1 int);
CREATE TABLE
testdb=# insert into t1 values('1');
INSERT 0 1
testdb=# create role readonly;
CREATE ROLE
testdb=# \du
 of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 readonly  | Cannot login                                               | {}

testdb=# grant connect on database testdb to readonly;
GRANT
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
testdb=# create user testread with login password 'test123';
CREATE ROLE
testdb=# grant readonly to testread;
GRANT ROLE
testdb=# \c testdb testread
connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
Previous connection kept
```
2) Выбрал вариант входа через пароль и не стал изменять настройки pg_hba.conf, поскольку в целях безопасности посчитал, что предпочтителен вход пользователя через явный ввод пароля. Дополнительно уяснил, что при указании пароля в командной строке он будет игнорироваться (видимо защита системы от логирования явных паролей).
```
fedor@pgindocker:~$ sudo psql -h 127.0.0.1 -U testread -d testdb -W test123
psql: warning: extra command-line argument "test123" ignored
Password:
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=>
```

3) Дальнейшая выборка данных из таблицы не была завершена успешно поскольку таблица была создана без указания конкретной схемы, да и к тому же при создании таблицы схемы testnm не существовало (таблица создалась по умолчанию в public).
```
testdb=> select * from testnm.t1;
ERROR:  relation "testnm.t1" does not exist
LINE 1: select * from testnm.t1;
testdb=> SELECT table_name FROM information_schema.tables WHERE table_schema='public';
 table_name
------------
(0 rows)
```

4) Выполнил вход под postgres и удалил t1. Далее создал таблицу заново.
```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# SELECT table_name FROM information_schema.tables WHERE table_schema='public';
 table_name
------------
 t1
(1 row)

testdb=# drop table t1;
DROP TABLE
testdb=# create table testnm.t1(c1 integer); insert into testnm.t1 values (1);
CREATE TABLE
INSERT 0 1
```

5) Выполнил вход под testread в базу testdb и выбрал данные из таблицы testnm.t1. Доступа к таблице нет поскольку ранее давался доступ только на те таблицы базы которые были созданы до команды предоставления доступа пользователю. Для доступа на таблицы с более поздней датой создания требуется заново предоставлять доступ (либо на каждую новую таблицу отдельно, либо на все таблицы в целом). Предполагаю что единая команда доступа на все таблицы базы GRANT SELECT сработала как курсор который по всем таблицам выполнял Grant к каждой существующей таблице отдельно и по этому последующие таблицы в его список естественно не войдут.
```
fedor@pgindocker:~$ sudo psql -h 127.0.0.1 -U testread -d testdb -W
Password:
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```

6) Ситуацию якобы должна была поменяет ALTER default privileges in SCHEMA но она дает доступ только для обьектов созданных (по дате) после её отработки.
```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly;
ALTER DEFAULT PRIVILEGES
testdb=# create table testnm.t3(c1 integer); insert into testnm.t3 values (1);
CREATE TABLE
INSERT 0 1
testdb=# \q
fedor@pgindocker:~$ sudo psql -h 127.0.0.1 -U testread -d testdb -W
Password:
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> select * from testnm.t3;
 c1
----
  1
(1 row)

testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```

7) Для доступа к testnm.t1 нужно снова отработать GRANT SELECT ON ALL TABLES IN SCHEMA либо отдельно GRANT на эту таблицу.
```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
testdb=# \q
fedor@pgindocker:~$ sudo psql -h 127.0.0.1 -U testread -d testdb -W
Password:
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)

testdb=>
```

8) Создание новой таблицы завершилось успешно поскольку явно не указывалась схема и по умолчанию она создалась в public, на которую по умолчанию есть доступ всем на создание обьектов.
```
testdb=> create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1
testdb=> SELECT table_name FROM information_schema.tables WHERE table_schema='public';
 table_name
------------
 t2
(1 row)
```
9) Исправляем явным запретом создания обьектов в схеме testdb.public
```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# REVOKE CREATE on SCHEMA public FROM public;
REVOKE
testdb=# REVOKE ALL on DATABASE testdb FROM public;
REVOKE
```
9) Далее проверяем под testread создание новой таблицы в схеме по умолчанию (public) в которой уже запрещено создавать объекты. В результате ошибка доутупа к изменениям в схеме. 
```
testdb=> create table t4(c1 integer); insert into t4 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t4(c1 integer);
                     ^
ERROR:  relation "t4" does not exist
LINE 1: insert into t4 values (2);

```

10) Бонусом попробовал REVOKE для схемы postgres.public под пользователем postgres и он отработал, но при рестарте СУБД пользователь postgres все еще создавал таблицы в базе postgres! Видимо для суперпользователя нет ограничений даже при REVOKE.
```
postgres=# select rolsuper from pg_roles where rolname = 'postgres';
 rolsuper
----------
 t
(1 row)
```


