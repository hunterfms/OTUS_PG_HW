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

3) Для полнотекстового поиска объеденил столбец state с тексовм в новый столбец state_description с типом tsvector по которому будем выполнять полнотекстовый поиск. Далее проверил состав данных и сделал выборку по данному полю, что бы оценить производительность без индекса.
```
postgres=# ALTER TABLE public.index_test ADD "state_description" tsvector;
ALTER TABLE
postgres=# UPDATE public.index_test SET state_description = to_tsvector(state||'Description state collumn') WHERE state_description IS NULL;
UPDATE 50000
postgres=# select * from  public.index_test limit 3;                                                                                id | fk_id | state | amount |    item    |          created_at           |             state_description
----+-------+-------+--------+------------+-------------------------------+--------------------------------------------
  7 |     7 | otus7 |   2.12 | item_otus7 | 2024-12-02 11:43:21.17049+06  | 'collumn':3 'otus7description':1 'state':2
  8 |     8 | otus8 |   2.42 | item_otus8 | 2024-12-02 11:43:21.170493+06 | 'collumn':3 'otus8description':1 'state':2
  9 |     9 | otus9 |   2.73 | item_otus9 | 2024-12-02 11:43:21.170496+06 | 'collumn':3 'otus9description':1 'state':2
(3 строки)

postgres=# select * from public.index_test where state_description @@ to_tsquery('otus48888description');
  id   | fk_id |   state   |  amount  |      item      |          created_at          |               state_description
-------+-------+-----------+----------+----------------+------------------------------+------------------------------------------------
 48888 | 48888 | otus48888 | 14814.55 | item_otus48888 | 2024-12-02 11:43:21.34484+06 | 'collumn':3 'otus48888description':1 'state':2
(1 строка)

postgres=# explain (analyze, buffers) select * from public.index_test where state_description @@ to_tsquery('otus48888description');
                                                          QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..10609.59 rows=250 width=116) (actual time=140.027..142.613 rows=1 loops=1)
   Workers Planned: 1
   Workers Launched: 1
   Buffers: shared hit=2009
   ->  Parallel Seq Scan on index_test  (cost=0.00..9584.59 rows=147 width=116) (actual time=131.953..133.423 rows=0 loops=2)
         Filter: (state_description @@ to_tsquery('otus48888description'::text))
         Rows Removed by Filter: 25000
         Buffers: shared hit=2009
 Planning Time: 0.977 ms
 Execution Time: 142.638 ms
(10 строк)
```
Далее создал индекс для поля state_description и оценил результат полнотекстового поиска на производительность.

Без индекса производительность составила actual time=140.027..142.613

С индексовм производительность составила наиболее скорый результат actual time=0.040..0.041
```
postgres=# create index index_statedescr on public.index_test using gin(state_description);
CREATE INDEX

postgres=# explain (analyze, buffers) select * from public.index_test where state_description @@ to_tsquery('otus48888description');
                                                         QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on index_test  (cost=18.19..773.49 rows=250 width=116) (actual time=0.040..0.041 rows=1 loops=1)
   Recheck Cond: (state_description @@ to_tsquery('otus48888description'::text))
   Heap Blocks: exact=1
   Buffers: shared hit=5
   ->  Bitmap Index Scan on index_statedescr  (cost=0.00..18.12 rows=250 width=0) (actual time=0.035..0.035 rows=1 loops=1)
         Index Cond: (state_description @@ to_tsquery('otus48888description'::text))
         Buffers: shared hit=4
 Planning Time: 7.997 ms
 Execution Time: 0.224 ms
(9 строк)

postgres=#
```
кооментарий - Полнотекстовый поиск весьма производителен с индексом, но требует большего пространства на диске. Хранить данные в таком типе выгодно будет не полностью книги, а скорее заголовки и общее описание с ключевыми словами поиска. Впечатлил метод поиска по корням слов и уклада с учетом регионального аспекта текста.

4) Оценил выборку по числовому полю без индекса. Далее создал индекс на часть данного поля и проверил производительность.
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
