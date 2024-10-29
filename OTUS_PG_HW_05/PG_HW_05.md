# ДЗ по теме - Настройка Postgresql

1) Установил на новую виртуальную машину с Ubuntu22.04_min пакет приложений с настройкой ssh.
```
sudo apt update
sudo apt install mc vim net-tools shh -y
```
2) Установил Postgresql 15. 
```
curl -fSsL https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | sudo tee /usr/share/keyrings/postgresql.gpg > /dev/null
echo deb [arch=amd64,arm64,ppc64el signed-by=/usr/share/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt/ jammy-pgdg main | sudo tee -a /etc/apt/sources.list.d/postgresql.list
sudo apt update
sudo apt install postgresql-client-15 postgresql-15 -y
...
fedor@pg15:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
fedor@pg15:~$
```
3) Создал базу loadtest_defaultconf для проведения нагрузочного тестирования с параметрами кластера по умолчанию. Запустил инициализацию базы для теста.
```
postgres=# create database loadtest_defaultconf;
CREATE DATABASE

postgres=# \q
fedor@pg15:~$ su postgres
Password:
postgres@pg15:/home/fedor$ pgbench -i -s 50 loadtest_defaultconf
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
5000000 of 5000000 tuples (100%) done (elapsed 12.39 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 39.65 s (drop tables 0.00 s, create tables 0.04 s, client-side generate 12.45 s, vacuum 7.78 s, primary keys 19.38 s).
postgres@pg15:/home/fedor$

```
4) Выполнил нагрузочное тестирование с параметрами кластера по умолчанию.
```
postgres@pg15:/home/fedor$ pgbench -c 100 -t 500 loadtest_defaultconf
pgbench (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 50
query mode: simple
number of clients: 100
number of threads: 1
maximum number of tries: 1
number of transactions per client: 500
number of transactions actually processed: 50000/50000
number of failed transactions: 0 (0.000%)
latency average = 877.574 ms
initial connection time = 194.667 ms
tps = 113.950547 (without initial connection time)
postgres@pg15:/home/fedor$
```

5) Приведение конфигурации парамтров СУБД выполнил по рекомендациям с https://pgtune.website.yandexcloud.net:
DB Version: 15
OS Type: linux
DB Type: dw
Total Memory (RAM): 1 GB
CPUs num: 1
Connections num: 100
Data Storage: hdd

![1](https://github.com/user-attachments/assets/e8434e23-1e75-4fe5-9f03-52b258370eb8)

Какие параметры сменились по рекомендации:
shared_buffers = 128MB на shared_buffers = 256MB  Рекомендуемое значение для shared_buffers — 25% доступной системной памяти.

#effective_cache_size = 4GB на effective_cache_size = 768MB  Рекомендуемое значение: примерно 50–75% от общего объёма оперативной памяти системы.

#maintenance_work_mem = 64MB на maintenance_work_mem = 128MB  Рекомендуется устанавливать более высокое значение, так как операции обслуживания очень требовательны к памяти и выполняются гораздо быстрее, если могут работать полностью в оперативной памяти. 

#wal_buffers = -1 sets based on shared_buffers на wal_buffers = 7864kB  Рекомендуется устанавливать следующее значение при доступной памяти 1–4 ГБ: 256–1024kb. Этот параметр стоит увеличивать в системах с большим количеством модификаций таблиц базы данных.

#default_statistics_target = 100 на default_statistics_target = 500  Значение по умолчанию — 100. Чем больше установленное значение, тем больше времени требуется для анализа ычисления статистики, но тем выше может быть качество оценок планировщика.

#effective_io_concurrency = 1 на effective_io_concurrency = 2  Чем больше это число, тем больше операций ввода/вывода будет пытаться выполнить параллельно в отдельном сеансе. для жёстких дисков рекомендовано выставлять — по количеству дисков. Видимо рекомендации с PGTune считают, что у меня установлено зеркало в RAID1 (хотя по факту всего один диск), поэтому теперь стоит учитывать эту специфику алгоритма с данного сайта. 

#work_mem = 4MB на work_mem = 655kB  Увеличивают значение work_mem для получения хороших результатов сложной сортировки. Сортировка в памяти происходит намного быстрее, чем сортировка данных на диске но PGTune ориентировался на характер клиенской среды приложений "DB Type: dw" который был выставлен при генерации значений, плюс на минимум ОЗУ в 1 GB и в итоге он выдал меньшее значение нежели по умолчанию.

#huge_pages = try на huge_pages = off Параметр контролирует, запрашиваются ли большие страницы для основной области общей памяти. При значении off большие страницы не будут запрошены. Видимо отключение параметра было рекомендовано в связи с текущим объемом ОЗУ (малый объем)

min_wal_size = 80MB на min_wal_size = 4GB  Это минимальный размер журнального сегмента, до которого должен «опуститься» WAL перед переиспользованием. Скорее всего рекомендовано выставление большего значения, чтобы справиться с резкими скачками использования WAL, например, при выполнении больших пакетных заданий.

max_wal_size = 1GB на max_wal_size = 16GB  Данный параметр, как и предыдущий видимо выставляется с учетом минимальных на сегодня объемов дискового пространства. Масимальный размер журнала в 16Гб будет реже вызывать операцию его пересоздания и в большинстве случаев на сегодня видимо оптимален с учетом его сканирования с целью поиска данных.
```
postgres@pg15:/home/fedor$ su fedor
Password:
fedor@pg15:~$ tail -n 15 /etc/postgresql/15/main/postgresql.conf

# Add settings for extensions here
max_connections = 100
shared_buffers = 256MB
effective_cache_size = 768MB
maintenance_work_mem = 128MB
checkpoint_completion_target = 0.9
wal_buffers = 7864kB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 655kB
huge_pages = off
min_wal_size = 4GB
max_wal_size = 16GB
fedor@pg15:~$

```
6) Рестартанул СУБД и создал новую базу loadtest_modifyconf для теста с рекомендуемой конфигурацией кластера.
```
fedor@pg15:~$ sudo systemctl restart postgresql@15-main
[sudo] password for fedor:
fedor@pg15:~$ sudo systemctl status postgresql@15-main
● postgresql@15-main.service - PostgreSQL Cluster 15-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Tue 2024-10-29 10:12:21 UTC; 13s ago
    Process: 5644 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 15-main start (code=exited, status=0/SUCCESS)
   Main PID: 5649 (postgres)
      Tasks: 6 (limit: 1019)
     Memory: 34.5M
        CPU: 138ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@15-main.service
             ├─5649 /usr/lib/postgresql/15/bin/postgres -D /var/lib/postgresql/15/main -c config_file=/etc/postgresql/15/main/postgresql.conf
             ├─5650 "postgres: 15/main: checkpointer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""
             ├─5651 "postgres: 15/main: background writer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""
             ├─5653 "postgres: 15/main: walwriter " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""
             ├─5654 "postgres: 15/main: autovacuum launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""
             └─5655 "postgres: 15/main: logical replication launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""

Oct 29 10:12:18 pg15 systemd[1]: Starting PostgreSQL Cluster 15-main...
Oct 29 10:12:21 pg15 systemd[1]: Started PostgreSQL Cluster 15-main.
fedor@pg15:~$

```
7) Запустил инициализацию базы для теста с рекомендованными параметрами конфигурации.
```
postgres@pg15:/home/fedor$ pgbench -i -s 50 loadtest_modifyconf
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
5000000 of 5000000 tuples (100%) done (elapsed 15.84 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 68.23 s (drop tables 0.00 s, create tables 0.05 s, client-side generate 15.92 s, vacuum 25.48 s, primary keys 26.76 s).
```
8) Выполнил нагрузочное тестирование с рекомендованными параметрами кластера.
```
postgres@pg15:/home/fedor$ pgbench -c 100 -t 500 loadtest_modifyconf
pgbench (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 50
query mode: simple
number of clients: 100
number of threads: 1
maximum number of tries: 1
number of transactions per client: 500
number of transactions actually processed: 50000/50000
number of failed transactions: 0 (0.000%)
latency average = 749.735 ms
initial connection time = 190.071 ms
tps = 133.380412 (without initial connection time)
postgres@pg15:/home/fedor$
```
## В итоге тестирование с рекомендованными параметрами конфигурации выдало больше tps нежели при стандартных настройках на 19,429865 
Default: tps = 113.950547
Modify: tps = 133.380412

## Считаю что в примере с виртуальной машиной на обычном компе с одним ядром и гигобайтом ОЗУ оценка производительности, даже с рекомендациями будет не особо эффективна. 
Более значимых результатов можно будет добиться на реальных серверах с множеством процессоров, реальной СХД и большим объёмом ОЗУ. Это касается и смены количества workers для вакуума и работы механизмов параллелизации в версионности строк, а так же при системе оценки выполнения планов запросов и их качества сборки оптимизатором СУБД.



