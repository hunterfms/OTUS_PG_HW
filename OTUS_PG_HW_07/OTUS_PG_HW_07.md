# ДЗ на тему - Блокировки

1) Настроил журналирование Postgresql-15 на логирование блокировок живущих более 200 миллисекунд путем редактирования в файле конфигурации СУБД параметра deadlock_timeout.
```
fedor@pg15:/var/lib/postgresql/15$ sudo tail -n 15 /etc/postgresql/15/main/postgresql.conf
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

log_lock_waits = on
deadlock_timeout = 200ms
fedor@pg15:/var/lib/postgresql/15$ sudo systemctl restart postgresql@15-main
```
2) Подготовил таблицу для теста блокировок.
```
postgres=# create table forlocks (text varchar);
CREATE TABLE
postgres=# insert into forlocks select 'text1';
INSERT 0 1
```
3) Реализую в двух разных сессиях обновление данных одной и той же строки, что бы создать блокировку.

В первой сесии обновляю строку не завершая транзакции.
```
postgres=# BEGIN;
BEGIN
postgres=*# update forlocks set text = 'text2' where text = 'text1';
UPDATE 1
```

Во второй сесии пытаюсь обновить эту же строку другим значением.
```
postgres=# update forlocks set text = 'text3' where text = 'text1';
```

Блокировка удерживается. В третьей сесии смотрю наличие блокировки.
```
postgres=# SELECT pid, usename,
       pg_blocking_pids(pid) AS blocked_by,
       query AS blocked_query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
 pid  | usename  | blocked_by |                      blocked_query

------+----------+------------+-------------------------------------------------
---------
 1007 | postgres | {957}      | update forlocks set text = 'text3' where text =
'text1';
(1 row)

postgres=#
```

4) Завершаю транзакцию в первой сесии, что бы далее дать возможность транзакции во второй сессии обновить значение уже на 'text3'.

Первая сессия.
```
postgres=*# COMMIT;
COMMIT
```

Вторая сессия. Транзакция не обновила строку поскольку условие было fulse, так как первая транзакция изменила значение для where text = 'text1' на 'text2' до того как очередь дошла до второй транзакции.
```
postgres=# update forlocks set text = 'text3' where text = 'text1';
UPDATE 0
postgres=# select * from forlocks;
 text
-------
 text2
(1 row)

```
5) Смотрим лог блокировок с зафиксированной ситуацией болкировки. DETAIL: Process holding the lock: 957. Wait queue: 1007 while updating tuple (0,1) in relation "forlocks"
```
fedor@pg15:/var/lib/postgresql/15$ sudo tail /var/log/postgresql/postgresql-15-main.log
==> /var/log/postgresql/postgresql-15-main.log <==
2024-11-13 14:53:32.231 UTC [1007] postgres@postgres DETAIL:  Process holding the lock: 957. Wait queue: 1007.
2024-11-13 14:53:32.231 UTC [1007] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "forlocks"
2024-11-13 14:53:32.231 UTC [1007] postgres@postgres STATEMENT:  update forlocks set text = 'text3' where text = 'text1';
2024-11-13 14:58:04.755 UTC [907] LOG:  checkpoint starting: time
2024-11-13 14:58:04.893 UTC [907] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.106 s, sync=0.005 s, total=0.138 s; sync files=2, longest=0.004 s, average=0.003 s; distance=8 kB, estimate=90 kB
2024-11-13 14:58:40.756 UTC [1007] postgres@postgres LOG:  process 1007 acquired ShareLock on transaction 648204 after 308726.254 ms
2024-11-13 14:58:40.756 UTC [1007] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "forlocks"
2024-11-13 14:58:40.756 UTC [1007] postgres@postgres STATEMENT:  update forlocks set text = 'text3' where text = 'text1';
2024-11-13 15:03:04.990 UTC [907] LOG:  checkpoint starting: time
2024-11-13 15:03:05.345 UTC [907] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.106 s, sync=0.011 s, total=0.355 s; sync files=2, longest=0.010 s, average=0.006 s; distance=0 kB, estimate=81 kB
fedor@pg15:/var/lib/postgresql/15$

```

Блокировка  без условия where возможны (задание *), если обновлять в двух транзакциях одно и то же поле. Столбец бцдет обновляться полность так,  как обновление будет проводиться без условия where и сследовательно наложиться эксклюзивная блокировка на всю таблицу.

6) Пересоздал таблицу с идентификатором поля и числовым значением и попробовал обновить весь столбец числового поля в открытой транзакции не завершаяя её и во второй сесии выполнить тоже самое.

первая сессия
```
postgres=# select * from forlocks;
 id | value
----+-------
  1 |    50
  2 |    75
  3 |   100
(3 rows)

postgres=# BEGIN;
BEGIN
postgres=*# update forlocks set value = value + 25;
UPDATE 3
postgres=*# commit;
COMMIT
```

вторая сессия
```
postgres=# update forlocks set value = value + 10;
UPDATE 3
postgres=# select * from forlocks;
 id | value
----+-------
  1 |    85
  2 |   110
  3 |   135
(3 rows)

postgres=#

```

в третьей сесиии смотрел блокировку
```
postgres=# select * from pg_locks;
   locktype    | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction | pid |       mode       | granted | fastpath |           waitstart
---------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+-----+------------------+---------+----------+-------------------------------
 relation      |        5 |    12073 |      |       |            |               |         |       |          | 5/17               | 901 | AccessShareLock  | t       | t        |
 virtualxid    |          |          |      |       | 5/17       |               |         |       |          | 5/17               | 901 | ExclusiveLock    | t       | t        |
 relation      |        5 |    27321 |      |       |            |               |         |       |          | 4/6                | 861 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 4/6        |               |         |       |          | 4/6                | 861 | ExclusiveLock    | t       | t        |
 relation      |        5 |    27321 |      |       |            |               |         |       |          | 3/11               | 827 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 3/11       |               |         |       |          | 3/11               | 827 | ExclusiveLock    | t       | t        |
 tuple         |        5 |    27321 |    0 |     1 |            |               |         |       |          | 4/6                | 861 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |        648221 |         |       |          | 4/6                | 861 | ShareLock        | f       | f        | 2024-11-13 16:01:46.368623+00
 transactionid |          |          |      |       |            |        648222 |         |       |          | 4/6                | 861 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |        648221 |         |       |          | 3/11               | 827 | ExclusiveLock    | t       | f        |
(10 rows)

postgres=#
```

7) В итоге, в журнале зарегистрирована блокировка на уровне всей таблицы где 827й pid блокировал 861го..
```
fedor@pg15:~$ sudo tail -n10 /var/log/postgresql/postgresql-15-main.log
2024-11-13 16:01:46.597 UTC [861] postgres@postgres DETAIL:  Process holding the lock: 827. Wait queue: 861.
2024-11-13 16:01:46.597 UTC [861] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "forlocks"
2024-11-13 16:01:46.597 UTC [861] postgres@postgres STATEMENT:  update forlocks set value = value + 10;
2024-11-13 16:03:00.912 UTC [713] LOG:  checkpoint starting: time
2024-11-13 16:03:01.145 UTC [713] LOG:  checkpoint complete: wrote 5 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.216 s, sync=0.006 s, total=0.234 s; sync files=4, longest=0.003 s, average=0.002 s; distance=0 kB, estimate=0 kB
2024-11-13 16:06:12.033 UTC [861] postgres@postgres LOG:  process 861 acquired ShareLock on transaction 648221 after 265664.427 ms
2024-11-13 16:06:12.033 UTC [861] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "forlocks"
2024-11-13 16:06:12.033 UTC [861] postgres@postgres STATEMENT:  update forlocks set value = value + 10;
2024-11-13 16:08:00.164 UTC [713] LOG:  checkpoint starting: time
2024-11-13 16:08:00.312 UTC [713] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.117 s, sync=0.005 s, total=0.148 s; sync files=2, longest=0.003 s, average=0.003 s; distance=0 kB, estimate=0 kB
```
