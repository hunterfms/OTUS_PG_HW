# ДЗ на тему - Секционирование
1) Скачал первую выданную гуглом "демо базы flights" базу среднего размера demo_medium.sql.
2) Для добавления скрипта базы на виртуалку PG15 воспользовался WinSCP.
3) Подключил базу к серверу
```
fedor@pg15:~$ sudo -u postgres psql -d postgres -f /var/lib/postgresql/15/demo_medium.sql -c 'demo'
```
![1](https://github.com/user-attachments/assets/1405eb6b-44f6-4d1d-8d93-eb0856ce1804)

5) Выдал limit 10 самых объемных таблиц.
```
postgres=# \c demo
You are now connected to database "demo" as user "postgres".
demo=# select schemaname as table_schema,
    relname as table_name,
    pg_size_pretty(pg_total_relation_size(relid)) as total_size,
    pg_size_pretty(pg_relation_size(relid)) as data_size,
    pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid))
      as external_size
from pg_catalog.pg_statio_user_tables
order by pg_total_relation_size(relid) desc,
         pg_relation_size(relid) desc
limit 10;
 table_schema |   table_name    | total_size | data_size  | external_size
--------------+-----------------+------------+------------+---------------
 bookings     | boarding_passes | 263 MB     | 109 MB     | 155 MB
 bookings     | ticket_flights  | 245 MB     | 154 MB     | 91 MB
 bookings     | tickets         | 134 MB     | 109 MB     | 25 MB
 bookings     | bookings        | 42 MB      | 30 MB      | 13 MB
 bookings     | flights         | 9872 kB    | 6336 kB    | 3536 kB
 bookings     | routes          | 152 kB     | 136 kB     | 16 kB
 bookings     | seats           | 144 kB     | 64 kB      | 80 kB
 bookings     | airports        | 64 kB      | 16 kB      | 48 kB
 bookings     | aircrafts       | 32 kB      | 8192 bytes | 24 kB
(9 rows)

demo=#
```
7) Выбираем таблицу для секционирования.  Рассмотрим состав данных первой наиболее размерной таблицы bookings.boarding_passes для примера секционирования.
```
demo=# select * from bookings.boarding_passes limit 5;
   ticket_no   | flight_id | boarding_no | seat_no
---------------+-----------+-------------+---------
 0005435208229 |     60731 |           1 | 1H
 0005435208224 |     60731 |           2 | 2A
 0005435208191 |     60731 |           3 | 2D
 0005435208233 |     60731 |           4 | 2H
 0005435208178 |     60731 |           5 | 3A
(5 rows)

demo=# SELECT column_name, column_default, data_type
FROM INFORMATION_SCHEMA.COLUMNS
WHERE table_name = 'boarding_passes';
 column_name | column_default |     data_type
-------------+----------------+-------------------
 ticket_no   |                | character
 flight_id   |                | integer
 boarding_no |                | integer
 seat_no     |                | character varying
(4 rows)

demo=# \d+ bookings.boarding_passes
                                                    Table "bookings.boarding_pas
ses"
   Column    |         Type         | Collation | Nullable | Default | Storage
| Compression | Stats target |       Description
-------------+----------------------+-----------+----------+---------+----------
+-------------+--------------+--------------------------
 ticket_no   | character(13)        |           | not null |         | extended
|             |              | Номер билета
 flight_id   | integer              |           | not null |         | plain
|             |              | Идентификатор рейса
 boarding_no | integer              |           | not null |         | plain
|             |              | Номер посадочного талона
 seat_no     | character varying(4) |           | not null |         | extended
|             |              | Номер места
Indexes:
    "boarding_passes_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
    "boarding_passes_flight_id_boarding_no_key" UNIQUE CONSTRAINT, btree (flight
_id, boarding_no)
    "boarding_passes_flight_id_seat_no_key" UNIQUE CONSTRAINT, btree (flight_id,
 seat_no)
Foreign-key constraints:
    "boarding_passes_ticket_no_fkey" FOREIGN KEY (ticket_no, flight_id) REFERENC
ES bookings.ticket_flights(ticket_no, flight_id)
Access method: heap

demo=#
```
9) Партицировать таблицу буду по полю flight_id поскольку данное поле имеет уникальный набор числовых значений характеризующий последовательность  возростания номеров авиарейсов. Секцыонирование позволит отдельно выбирать последний набор рейсов,и в итоге ускорит выборки по посленим рейсам. Всего около 65662 рейсов. Разделим таблицу на 13 партиций, что бы хранить по 5000 рейсов в каждой.
```
demo=# select min(flight_id), max(flight_id), count(*) from bookings.boarding_passes;
 min |  max  |  count
-----+-------+---------
   1 | 65662 | 1894295
(1 row)
```
10) Создаем отдельную таблицу для дальнейшего её секционирования.
```
demo=# CREATE TABLE bookings.boarding_passes_part (
   ticket_no bpchar(13),
   flight_id int,
   boarding_no int,
   seat_no character varying(4)
) PARTITION BY RANGE (flight_id);
CREATE TABLE
demo=#
```
11) Создаем разделы партиций. 14 партиций, что бы хранить по 5000 рейсов в каждой.
```
 demo=# CREATE TABLE boarding_passes_part0
 PARTITION OF bookings.boarding_passes_part
 FOR VALUES FROM (1) TO (5000);
 CREATE TABLE boarding_passes_part1
 PARTITION OF bookings.boarding_passes_part
 FOR VALUES FROM (5000) TO (10000);
 CREATE TABLE boarding_passes_part2
 PARTITION OF bookings.boarding_passes_part
 FOR VALUES FROM (10000) TO (15000);
 CREATE TABLE boarding_passes_part3
 PARTITION OF bookings.boarding_passes_part
 FOR VALUES FROM (15000) TO (20000);
 CREATE TABLE boarding_passes_part4
 PARTITION OF bookings.boarding_passes_part
 FOR VALUES FROM (20000) TO (25000);
 CREATE TABLE boarding_passes_part5
 PARTITION OF bookings.boarding_passes_part
 FOR VALUES FROM (25000) TO (30000);
 CREATE TABLE boarding_passes_part6
 PARTITION OF bookings.boarding_passes_part
 FOR VALUES FROM (30000) TO (35000);
 CREATE TABLE boarding_passes_part7
 PARTITION OF bookings.boarding_passes_part
 FOR VALUES FROM (35000) TO (40000);
 CREATE TABLE boarding_passes_part8
 PARTITION OF bookings.boarding_passes_part
 FOR VALUES FROM (40000) TO (45000);
 CREATE TABLE boarding_passes_part9
 PARTITION OF bookings.boarding_passes_part
 FOR VALUES FROM (45000) TO (50000);
 CREATE TABLE boarding_passes_part10
 PARTITION OF bookings.boarding_passes_part
 FOR VALUES FROM (50000) TO (55000);
 CREATE TABLE boarding_passes_part11
 PARTITION OF bookings.boarding_passes_part
 FOR VALUES FROM (55000) TO (60000);
 CREATE TABLE boarding_passes_part12
 PARTITION OF bookings.boarding_passes_part
 FOR VALUES FROM (60000) TO (65000);
 CREATE TABLE boarding_passes_part13
 PARTITION OF bookings.boarding_passes_part
 FOR VALUES FROM (65000) TO (70000);
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
demo=#
```
12) Заполняем партиции данными из изходной таблицы.
```
demo=# INSERT INTO bookings.boarding_passes_part SELECT * FROM bookings.boarding_passes;
INSERT 0 1894295
demo=#
```
13) Создаем индекс для секционированного поля по макету исходной таблицы. Нужно было создать индексы еще до заполнения данных.
```
demo=# create index boarding_passes_flight_id_boarding_no on bookings.boarding_passes_part (flight_id, boarding_no);
CREATE INDEX
demo=# create index boarding_passes_flight_id_seat_no on bookings.boarding_passes_part (flight_id, seat_no);
CREATE INDEX
demo=#

```
14)  Сравниваем производительность поиска из таблицы без партиций и с партициями.

без партиций actual time=29.893..34.868
```
demo=# explain (analyze, buffers) select * from bookings.boarding_passes where flight_id between 31000 and 37000 order by flight_id;
                                                                          QUERY
PLAN
--------------------------------------------------------------------------------
-------------------------------------------------------------------------------
 Sort  (cost=24746.29..24974.22 rows=91171 width=25) (actual time=29.893..34.868
 rows=92035 loops=1)
   Sort Key: flight_id
   Sort Method: quicksort  Memory: 9544kB
   Buffers: shared hit=2809 read=255
   ->  Bitmap Heap Scan on boarding_passes  (cost=1938.93..17235.50 rows=91171 w
idth=25) (actual time=6.368..15.838 rows=92035 loops=1)
         Recheck Cond: ((flight_id >= 31000) AND (flight_id <= 37000))
         Heap Blocks: exact=2809
         Buffers: shared hit=2809 read=255
         ->  Bitmap Index Scan on boarding_passes_flight_id_seat_no_key  (cost=0
.00..1916.14 rows=91171 width=0) (actual time=6.002..6.002 rows=92035 loops=1)
               Index Cond: ((flight_id >= 31000) AND (flight_id <= 37000))
               Buffers: shared read=255
 Planning:
   Buffers: shared hit=15
 Planning Time: 0.116 ms
 Execution Time: 38.239 ms
(15 rows)

demo=#
```

с партициями actual time=0.031..24.515
```
demo=# explain (analyze, buffers) select * from bookings.boarding_passes_part where flight_id between 31000 and 37000 order by flight_id;
                                                                                                 QUERY PLAN

----------------------------------------------------------------------------------------------------------------------------------------------------------------
---------------------------------------------
 Append  (cost=0.58..8133.84 rows=92053 width=25) (actual time=0.031..24.515 rows=92035 loops=1)
   Buffers: shared hit=10181 read=251
   ->  Index Scan using boarding_passes_part6_flight_id_seat_no_idx on boarding_passes_part6 boarding_passes_part_1  (cost=0.29..4580.82 rows=60630 width=25) (a
ctual time=0.031..12.007 rows=60607 loops=1)
         Index Cond: ((flight_id >= 31000) AND (flight_id <= 37000))
         Buffers: shared hit=7255 read=166
   ->  Index Scan using boarding_passes_part7_flight_id_seat_no_idx on boarding_passes_part7 boarding_passes_part_2  (cost=0.29..3092.75 rows=31423 width=25) (a
ctual time=0.018..6.228 rows=31428 loops=1)
         Index Cond: ((flight_id >= 31000) AND (flight_id <= 37000))
         Buffers: shared hit=2926 read=85
 Planning:
   Buffers: shared hit=84 read=8 dirtied=3
 Planning Time: 0.439 ms
 Execution Time: 27.912 ms
(12 rows)

demo=#
```

# Вывод - Выборка с партицированной таблицы и со схожей структурой индексов по искомому полю производится в моём случае в 964 раза быстрее
