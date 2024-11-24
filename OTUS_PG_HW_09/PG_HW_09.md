# ДЗ на тему - Индексы
1) Создал таблицу для тестирования производительности индексов на базе примера из лекции.
```
postgres=# create table if not exists public.index_test (
        id bigint,
        fk_id bigint,
        state text,
        amount numeric,
        item text,
        created_at timestamp with time zone,
        constraint pgotus_pkey primary key (id)
);
CREATE TABLE
postgres=# insert into public.index_test (id, fk_id, state, amount, item, created_at)
select g.x, g.x, 'otus'||g.x, round(g.x / 3.3, 2), 'item_otus'||g.x, clock_timestamp()
from generate_series(1, 50000) as g(x);
INSERT 0 50000
postgres=# select * from public.index_test limit 10;
 id | fk_id | state  | amount |    item     |          created_at
----+-------+--------+--------+-------------+-------------------------------
  1 |     1 | otus1  |   0.30 | item_otus1  | 2024-11-24 14:13:27.960834+00
  2 |     2 | otus2  |   0.61 | item_otus2  | 2024-11-24 14:13:27.960986+00
  3 |     3 | otus3  |   0.91 | item_otus3  | 2024-11-24 14:13:27.96099+00
  4 |     4 | otus4  |   1.21 | item_otus4  | 2024-11-24 14:13:27.960992+00
  5 |     5 | otus5  |   1.52 | item_otus5  | 2024-11-24 14:13:27.961006+00
  6 |     6 | otus6  |   1.82 | item_otus6  | 2024-11-24 14:13:27.961008+00
  7 |     7 | otus7  |   2.12 | item_otus7  | 2024-11-24 14:13:27.96101+00
  8 |     8 | otus8  |   2.42 | item_otus8  | 2024-11-24 14:13:27.961011+00
  9 |     9 | otus9  |   2.73 | item_otus9  | 2024-11-24 14:13:27.961013+00
 10 |    10 | otus10 |   3.03 | item_otus10 | 2024-11-24 14:13:27.961014+00
(10 rows)
```
2) Выбрал данные id изначально индексированное по первичному ключу, для оценки производительности.
```
postgres=# explain (analyze, buffers)
select * from public.index_test where id = 49999 limit 1;
                                                          QUERY PLAN

------------------------------------------------------------------------------------------------------
-------------------------
 Limit  (cost=0.29..8.31 rows=1 width=54) (actual time=0.012..0.013 rows=1 loops=1)
   Buffers: shared hit=6
   ->  Index Scan using pgotus_pkey on index_test  (cost=0.29..8.31 rows=1 width=54) (actual time=0.01
1..0.012 rows=1 loops=1)
         Index Cond: (id = 49999)
         Buffers: shared hit=6
 Planning:
   Buffers: shared hit=24
 Planning Time: 0.172 ms
 Execution Time: 0.035 ms
(9 rows)

postgres=#
```
кооментарий - PK индекс (Index Scan using pgotus_pkey) нашел искомое значение за 6 преходов по страницам (Buffers: shared hit=6) и с затратами времени в пределах 12 миллисекунд (actual time=0.012...) хотя изначально планировал выполнить 24 перехода и отработать за 17 миллисекунд.

3) Оценил выборку по текстовому полю без индекса. Далее создал индекс для полнотекстового поиска и проверил производительность.
```
postgres=# explain (analyze, buffers)
select * from public.index_test where item like 'item_otus49999' limit 1;
                                                  QUERY PLAN

------------------------------------------------------------------------------------------------------
--------
 Limit  (cost=0.00..231.40 rows=1 width=54) (actual time=5.108..5.108 rows=1 loops=1)
   Buffers: shared hit=532
   ->  Seq Scan on index_test  (cost=0.00..1157.00 rows=5 width=54) (actual time=5.107..5.107 rows=1 l
oops=1)
         Filter: (item ~~ 'item_otus49999'::text)
         Rows Removed by Filter: 49998
         Buffers: shared hit=532
 Planning Time: 221.430 ms
 Execution Time: 5.123 ms
(8 rows)

postgres=#

```
кооментарий - 

4) Оценил выборку по числовому полю без индекса. Далее создал индекс на чать данного поля и проверил производительность.
```
postgres=# explain (analyze, buffers)
select * from public.index_test where fk_id = 49999 limit 1;
                                                  QUERY PLAN

------------------------------------------------------------------------------------------------------
--------
 Limit  (cost=0.00..1157.00 rows=1 width=54) (actual time=3.537..3.537 rows=1 loops=1)
   Buffers: shared hit=532
   ->  Seq Scan on index_test  (cost=0.00..1157.00 rows=1 width=54) (actual time=3.536..3.536 rows=1 l
oops=1)
         Filter: (fk_id = 49999)
         Rows Removed by Filter: 49998
         Buffers: shared hit=532
 Planning Time: 0.041 ms
 Execution Time: 3.547 ms
(8 rows)

postgres=#

```
кооментарий - 

5) Оценил выборку по числовому + текстовому полю без индекса. Создал составной индекс на два поля и проверил производительность.
```
postgres=# explain (analyze, buffers) select * from public.index_test where amount = 15151.21 and state like 'otus49%' limit 1;
                                                  QUERY PLAN

------------------------------------------------------------------------------------------------------
--------
 Limit  (cost=0.00..1282.00 rows=1 width=54) (actual time=4.617..4.618 rows=1 loops=1)
   Buffers: shared hit=532
   ->  Seq Scan on index_test  (cost=0.00..1282.00 rows=1 width=54) (actual time=4.616..4.616 rows=1 l
oops=1)
         Filter: ((state ~~ 'otus49%'::text) AND (amount = 15151.21))
         Rows Removed by Filter: 49998
         Buffers: shared hit=532
 Planning Time: 0.065 ms
 Execution Time: 4.628 ms
(8 rows)

postgres=#

```
кооментарий - 
