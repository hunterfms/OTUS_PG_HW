# ДЗ по теме - Vacuum and Autovacuum

1) Настроил виртуальную машину с двумя процами, 4гб ОЗУ и 10GB диском. Далее создал базу для теста производительности, инициализировал её и провел тест с параметрами СУБД по умолчанию.. 
```
login as: fedor
fedor@192.168.1.188's password:
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-124-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
New release '24.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Wed Oct 30 14:34:00 2024
fedor@pg15:~$ sudo -u postgres psql
could not change directory to "/home/fedor": Permission denied
psql (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
Type "help" for help.

postgres=# create database benchtest;
CREATE DATABASE
postgres=# \q
fedor@pg15:~$ su postgres
Password:
postgres@pg15:/home/fedor$ pgbench -i benchtest
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.11 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.27 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.14 s, vacuum 0.04 s, primary keys 0.07 s).
postgres@pg15:/home/fedor$ pgbench -c8 -P 6 -T 60 -U postgres benchtest
pgbench (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 591.1 tps, lat 13.449 ms stddev 10.247, 0 failed
progress: 12.0 s, 611.8 tps, lat 13.032 ms stddev 10.212, 0 failed
progress: 18.0 s, 363.3 tps, lat 21.961 ms stddev 16.825, 0 failed
progress: 24.0 s, 316.3 tps, lat 25.279 ms stddev 17.891, 0 failed
progress: 30.0 s, 604.5 tps, lat 13.207 ms stddev 10.500, 0 failed
progress: 36.0 s, 596.0 tps, lat 13.390 ms stddev 10.557, 0 failed
progress: 42.0 s, 699.0 tps, lat 11.415 ms stddev 8.872, 0 failed
progress: 48.0 s, 476.2 tps, lat 16.770 ms stddev 12.481, 0 failed
progress: 54.0 s, 346.8 tps, lat 22.998 ms stddev 16.378, 0 failed
progress: 60.0 s, 306.5 tps, lat 26.077 ms stddev 18.507, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 29478
number of failed transactions: 0 (0.000%)
latency average = 16.251 ms
latency stddev = 13.625 ms
initial connection time = 15.125 ms
tps = 491.160150 (without initial connection time)
postgres@pg15:/home/fedor$
```
2) Изменил настройки СУБД по рекомендации из ДЗ. Рестартанул службу Postgresql. Завел новую базу для теста и повторил тест снова.
```
fedor@pg15:~$ tail /etc/postgresql/15/main/postgresql.conf -n 15
#------------------------------------------------------------------------------

# Add settings for extensions here
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 65536kB
min_wal_size = 4GB
max_wal_size = 16GB

fedor@pg15:~$ sudo -u postgres psql
could not change directory to "/home/fedor": Permission denied
psql (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
Type "help" for help.

postgres=# create database benchtest2;
CREATE DATABASE
postgres=# \q
fedor@pg15:~$ sudo systemctl restart postgresql@15-main
fedor@pg15:~$ su postgres
Password:
postgres@pg15:/home/fedor$ pgbench -i benchtest2
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
done in 0.27 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.13 s, vacuum 0.05 s, primary keys 0.06 s).
postgres@pg15:/home/fedor$ pgbench -c8 -P 6 -T 60 -U postgres benchtest2
pgbench (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 588.5 tps, lat 13.494 ms stddev 10.061, 0 failed
progress: 12.0 s, 373.1 tps, lat 21.362 ms stddev 15.635, 0 failed
progress: 18.0 s, 321.6 tps, lat 24.827 ms stddev 18.412, 0 failed
progress: 24.0 s, 509.1 tps, lat 15.716 ms stddev 13.336, 0 failed
progress: 30.0 s, 338.3 tps, lat 23.591 ms stddev 16.098, 0 failed
progress: 36.0 s, 325.2 tps, lat 24.574 ms stddev 17.076, 0 failed
progress: 42.0 s, 343.4 tps, lat 23.263 ms stddev 17.201, 0 failed
progress: 48.0 s, 311.2 tps, lat 25.676 ms stddev 19.891, 0 failed
progress: 54.0 s, 354.5 tps, lat 22.536 ms stddev 16.680, 0 failed
progress: 60.0 s, 559.5 tps, lat 14.278 ms stddev 12.160, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 24154
number of failed transactions: 0 (0.000%)
latency average = 19.837 ms
latency stddev = 15.972 ms
initial connection time = 18.365 ms
tps = 402.480422 (without initial connection time)
postgres@pg15:/home/fedor$
```
По результатам сравнения после второго опыта производительность снизилась в пределах 24%.

тест1 tps = 491.160150

тест2 tps = 402.480422

3) Добавил таблицу и заполнил её миллионом строк текста.
```
postgres=# create table textdata (id INT, text varchar);
CREATE TABLE
postgres=# insert into textdata
SELECT generate_series(1,1000000) AS id, md5(random()::text) AS text;
INSERT 0 1000000
postgres=#
```
4) Выявлен размер таблицы
```
postgres=# SELECT schemaname,
       C.relname AS "relation",
       pg_size_pretty (pg_relation_size(C.oid)) as table,
       pg_size_pretty (pg_total_relation_size (C.oid)-pg_relation_size(C.oid)) as index,
       pg_size_pretty (pg_total_relation_size (C.oid)) as table_index,
       n_live_tup
FROM pg_class C
LEFT JOIN pg_namespace N ON (N.oid = C .relnamespace)
LEFT JOIN pg_stat_user_tables A ON C.relname = A.relname
WHERE nspname NOT IN ('pg_catalog', 'information_schema')
AND C.relkind <> 'i'
AND nspname !~ '^pg_toast'
ORDER BY pg_total_relation_size (C.oid) DESC;
 schemaname | relation | table | index | table_index | n_live_tup
------------+----------+-------+-------+-------------+------------
 public     | textdata | 65 MB | 56 kB | 65 MB       |    1000000
(1 row)

postgres=#
```
либо по id
```
postgres=# SELECT pg_relation_filepath('textdata');
 pg_relation_filepath
----------------------
 base/5/16673
(1 row)

postgres=#
```

![1](https://github.com/user-attachments/assets/de9859a4-aa07-4a4e-9399-991a68b38c19)

5)
