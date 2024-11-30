# ДЗ на тему - Журналы
1) Перенастроил Postgresql 15 на выполнение контрольной точки раз в 30 секунд (checkpoint_timeout = 30, log_checkpoints = on).
```
fedor@pg15:~$  sudo tail -n15 /etc/postgresql/15/main/postgresql.conf
[sudo] password for fedor:
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_timeout = 30
log_checkpoints = on
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 65536kB
min_wal_size = 4GB
max_wal_size = 16GB
log_lock_waits = on
deadlock_timeout = 200ms
fedor@pg15:~$
```
2)  Создал базу loadtest_wal для теста нагрузки pgbench
```
postgres=# create database loadtest_wal;
CREATE DATABASE
postgres=# \q
postgres@pg15:/home/fedor$ pgbench -i loadtest_wal
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.08 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.43 s (drop tables 0.00 s, create tables 0.05 s, client-side generate 0.24 s, vacuum 0.05 s, primary keys 0.09 s).
```
3) Вывел текущий lsn для сравнения в будущем и провел нагрузочный тест в синхронном режиме 10 мин.
```
postgres=# select * from pg_current_wal_lsn ();
 pg_current_wal_lsn
--------------------
 3/BF7944B8
(1 row)

postgres=# select * from pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 3/BF7944B8
(1 row)

postgres=# \q
postgres@pg15:/home/fedor$ pgbench -T 600 loadtest_wal
pgbench (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 215012
number of failed transactions: 0 (0.000%)
latency average = 2.791 ms
initial connection time = 2.227 ms
tps = 358.354436 (without initial connection time)
postgres@pg15:/home/fedor$ psql
could not change directory to "/home/fedor": Permission denied
psql (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from pg_current_wal_lsn();
 pg_current_wal_lsn
--------------------
 3/D5FB2148
(1 row)

postgres=# select * from pg_walfile_name('3/D5FB2148');
     pg_walfile_name
--------------------------
 0000000100000003000000D5
(1 row)
```
4) Для репликации разница между контрольными точками составила бы (lsn указал в обратном порядке) = 360,11 Mb.
```
postgres=# SELECT pg_wal_lsn_diff('3/D5FB2148', '3/BF7944B8');
 pg_wal_lsn_diff
-----------------
       377609360
(1 row)
```
5) За 10 минут должно было отработать 20 lsn (checkpoint_timeout = 30 сек). перед тестом для наглядности каталог pg_wal очистил от содержимого. Вид pg_wal на момент после нагрузочного теста. pg_wal накопил 4 файла 16мб (wal_buffers = 16MB).

![1](https://github.com/user-attachments/assets/6feaa097-4a95-4fda-8c82-037151ea7d4a)

по журналу наблюдается создание lsn (checkpoint starting: time) каждые 30 сек. 
2024-11-30 05:46:52.073 UTC [1593] LOG:  checkpoint starting: time
2024-11-30 05:47:22.054 UTC [1593] LOG:  checkpoint starting: time
2024-11-30 05:47:52.071 UTC [1593] LOG:  checkpoint starting: time
и т.д.
Наблюдаем завершение контрольной точки до начала следующей в пределах 3 сек., поскольку выставлено ограничение checkpoint_completion_target = 0.9 (на запись давать максимум 90% от checkpoint_timeout). С учетом постоянной интенсивности работы нагрузочного теста заполнение журнала записью  контрольных точек должно осуществляться примерно равными порциями за счет ограничений по времени записи.
```
fedor@pg15:~$ tail -n 10 /var/log/postgresql/postgresql-15-main.log
2024-11-30 05:46:52.073 UTC [1593] LOG:  checkpoint starting: time
2024-11-30 05:47:19.050 UTC [1593] LOG:  checkpoint complete: wrote 1763 buffers (1.3%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.901 s, sync=0.024 s, total=26.977 s; sync files=6, longest=0.011 s, average=0.004 s; distance=17409 kB, estimate=18459 kB
2024-11-30 05:47:22.054 UTC [1593] LOG:  checkpoint starting: time
2024-11-30 05:47:49.068 UTC [1593] LOG:  checkpoint complete: wrote 1987 buffers (1.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.925 s, sync=0.034 s, total=27.014 s; sync files=17, longest=0.010 s, average=0.002 s; distance=18353 kB, estimate=18448 kB
2024-11-30 05:47:52.071 UTC [1593] LOG:  checkpoint starting: time
2024-11-30 05:48:19.098 UTC [1593] LOG:  checkpoint complete: wrote 1708 buffers (1.3%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.969 s, sync=0.019 s, total=27.028 s; sync files=7, longest=0.014 s, average=0.003 s; distance=17470 kB, estimate=18350 kB
2024-11-30 05:49:22.307 UTC [1593] LOG:  checkpoint starting: time
2024-11-30 05:49:49.110 UTC [1593] LOG:  checkpoint complete: wrote 432 buffers (0.3%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.707 s, sync=0.049 s, total=26.803 s; sync files=15, longest=0.022 s, average=0.004 s; distance=9781 kB, estimate=17493 kB
2024-11-30 05:50:22.306 UTC [1593] LOG:  checkpoint starting: time
2024-11-30 05:50:23.098 UTC [1593] LOG:  checkpoint complete: wrote 6 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.739 s, sync=0.023 s, total=0.793 s; sync files=4, longest=0.021 s, average=0.006 s; distance=11 kB, estimate=15745 kB
fedor@pg15:~$
```
В среднем на точну (на физическом уровне) вышло 4 файла по 16мб на 20 точек = 3,27мб

## Проверка в асинхронном режиме.
6) Перенастроил сервер на ассинхронный режим (указав в файле настроек synchronous_commit = off) и провел идеинтичный нагрузочный тест.
```
fedor@pg15:/etc/postgresql/15/main$ tail -n 1 /etc/postgresql/15/main/postgresql.conf
synchronous_commit = off
fedor@pg15:~$ sudo systemctl restart postgresql@15-main
[sudo] password for fedor:
fedor@pg15:~$ su postgres
Password:
postgres@pg15:/home/fedor$ pgbench -T 600 loadtest_wal
pgbench (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1025434
number of failed transactions: 0 (0.000%)
latency average = 0.585 ms
initial connection time = 2.361 ms
tps = 1709.062270 (without initial connection time)
```
Сравниваем оба режима.
синхронный - tps = 358.354436
асинхронный - tps = 1709.062270
Выводы: В асинхронном режиме tps выше, поскольку отсутсвует ожидание подтверждения записи WAL на диск.

## Повреждение данных таблицы в кластере с включенной контрольной суммой страниц.
7) Остановил кластер, удилил содержимое кластера main и взамен создал новый кластер с контрольной суммой страниц. далее стартанул службу СУБД.
```
postgres@pg15:/usr/lib/postgresql/15/bin$ /usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/main --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "C.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/15/main ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

initdb: warning: enabling "trust" authentication for local connections
initdb: hint: You can change this by editing pg_hba.conf or using the option -A, or --auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    /usr/lib/postgresql/15/bin/pg_ctl -D /var/lib/postgresql/15/main -l logfile start

postgres@pg15:/usr/lib/postgresql/15/bin$ exit
exit
fedor@pg15:/usr/lib/postgresql/15/bin$ sudo systemctl start postgresql@15-main
fedor@pg15:/usr/lib/postgresql/15/bin$ su postgres
Password:
postgres@pg15:/usr/lib/postgresql/15/bin$ psql
psql (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
Type "help" for help.

postgres=# \l
                                             List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+---------+---------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
(3 rows)

```

8) Подготовил таблицу
```
postgres=# create table test (id int, text varchar(5));
CREATE TABLE
postgres=# insert into test select 1, 'text1';
INSERT 0 1
postgres=# SELECT oid FROM pg_class WHERE relname = 'test';
  oid
-------
 16400
(1 row)
```

9) останови СУБД и изменил байты в файле таблице 16400 (изменил первые 4 символа на "edit").

![2](https://github.com/user-attachments/assets/b3170f2b-7913-4c6a-88e1-2b4fadf85ce5)

10) Стартанул СУБД и проверил содержимое таблицы.
```
fedor@pg15:/usr/lib/postgresql/15/bin$ sudo systemctl start postgresql@15-main
fedor@pg15:/usr/lib/postgresql/15/bin$ su postgres
Password:
postgres@pg15:/usr/lib/postgresql/15/bin$ psql
psql (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from test;
ERROR:  invalid page in block 0 of relation base/5/16400
postgres=#
```

11) Для избегания ошибки включил параметр zero_damaged_pages
```
postgres=# SET zero_damaged_pages = on;
SET
postgres=# select * from test;
WARNING:  invalid page in block 0 of relation base/5/16400; zeroing out page
 id | text
----+------
(0 rows)

postgres=#

```
После перезагрузки psql выборка таблицы выполнилась уже без предупреждения и можно было добавить в таблицу строку.
```
postgres=# \q
postgres@pg15:/usr/lib/postgresql/15/bin$ psql
psql (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from test;
 id | text
----+------
(0 rows)

postgres=# insert into test select 1, 'text1';
INSERT 0 1
postgres=# select * from test;
 id | text
----+-------
  1 | text1
(1 row)
```
