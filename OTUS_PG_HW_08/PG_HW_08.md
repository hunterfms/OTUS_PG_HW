# ДЗ на тему - Журналы
1) Перенастроил Postgresql 15 на выполнение контрольной точки раз в 30 секунд checkpoint_timeout = 30.
```
fedor@pg15:/var/lib/postgresql$ sudo tail -n15 /etc/postgresql/15/main/postgresql.conf
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_timeout = 30
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
fedor@pg15:/var/lib/postgresql$
```
2)  Создал базу loadtest_wal для теста нагрузки pgbench
```
postgres=# create database loadtest_wal;
CREATE DATABASE
postgres=# \q
fedor@pg15:/var/lib/postgresql$ su postgres
Password:
postgres@pg15:~$ pgbench -i loadtest_wal
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.07 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.32 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.16 s, vacuum 0.07 s, primary keys 0.08 s).
```
3) Вывел текущий lsn для сравнения в будущем и провел нагрузочный тест в синхронном режиме 10 мин.
```
postgres=# select * from pg_current_wal_lsn ();
 pg_current_wal_lsn
--------------------
 3/8DD7D6D8
(1 row)
postgres=# select * from pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 3/8DD7D6D8
(1 row)
postgres@pg15:~$ pgbench -T 600 loadtest_wal
pgbench (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 179712
number of failed transactions: 0 (0.000%)
latency average = 3.339 ms
initial connection time = 2.872 ms
tps = 299.521219 (without initial connection time)
postgres@pg15:~$ psql
psql (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from pg_current_wal_lsn();
 pg_current_wal_lsn
--------------------
 3/A3B2FD50
(1 row)

postgres=# select * from pg_walfile_name('3/8DD7D6D8');
     pg_walfile_name
--------------------------
 00000001000000030000008D

postgres=# select * from pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 3/A3B2FD50
(1 row)

postgres=# select * from pg_walfile_name('3/A3B2FD50');
     pg_walfile_name
--------------------------
 0000000100000003000000A3
```
4) Вывел разницу между контрольными точками до и после наргрузки = 349,6 Mb.
```
postgres=# SELECT pg_wal_lsn_diff('3/8DD7D6D8', '3/A3B2FD50');
 pg_wal_lsn_diff
-----------------
      -366683768
(1 row)
```
