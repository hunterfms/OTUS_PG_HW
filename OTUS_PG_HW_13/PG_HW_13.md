# ДЗ на тему - Функции и процедуры (триггеры, поддержка заполнения витрин)

1) На машине PG15 отработал файл структуры проекта "витрина".
```
postgres@pg15:/home/fedor$ psql
could not change directory to "/home/fedor": Permission denied
psql (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
Type "help" for help.

postgres=# DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;
NOTICE:  schema "pract_functions" does not exist, skipping
DROP SCHEMA
CREATE SCHEMA
postgres=# SET search_path = pract_functions, publ;
SET
postgres=# CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
CREATE TABLE
postgres=# INSERT INTO goods (goods_id, good_name, good_price)
VALUES  (1, 'Спички хозайственные', .50),
                (2, 'Автомобиль Ferrari FXX K', 185000000.01);
INSERT 0 2
postgres=# CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);
CREATE TABLE
postgres=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
INSERT 0 4
```
2) Из скрипта выдал отчет
```
postgres=# SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
        good_name         |     sum
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
```
3) Далее по скрипту создал таблицу good_sum_mart (витрина). Дополнительно добавил ограничение уникальности для наименования товара.
```
postgres=# CREATE TABLE good_sum_mart
(
        good_name   varchar(63) NOT NULL,
        sum_sale        numeric(16, 2)NOT NULL
);
CREATE TABLE
postgres=# alter table good_sum_mart add constraint unq_good_name unique (good_name);
ALTER TABLE

```

При планировании функции триггера которая должна вычислять при каждой продаже сумму и записывать её в витрину вызовом триггера было принято следующее условие: требуется автоматически обновлять таблицу good_sum_mart с учетом любых измений (insert,update.delete) таблицы sales каждый раз учитывая актуальную цену товара из goods.

4) Подготавливаем тело функции.
```
create or replace function recalc_good_sum_mart()
  returns trigger as $$
   begin
     -- При добавлении новой записи в таблицу продаж определяем цену товара и его общую сумму. Далее либо добавляем строку по новому товару, либо наращиваем сумму.
     if (tg_op = 'insert') then
         insert into good_sum_mart (good_name, sum_sale)
         values (
             (select good_name from goods where goods_id = new.good_id),
             (select good_price * new.sales_qty from goods where goods_id = new.good_id)
         )
         on conflict (good_name) do update
         set sum_sale = good_sum_mart.sum_sale + excluded.sum_sale;
 
     -- При изменении записи в таблице продаж очищаем старую сумму и добавляем новую.
     elsif (tg_op = 'update') then
	   update good_sum_mart
         set sum_sale = sum_sale - (select good_price * old.sales_qty from goods where goods_id = old.good_id)
         where good_name = (select good_name from goods where goods_id = old.good_id);
         insert into good_sum_mart (good_name, sum_sale)
         values (
             (select good_name from goods where goods_id = new.good_id),
             (select good_price * new.sales_qty from goods where goods_id = new.good_id)
         )
         on conflict (good_name) do update
         set sum_sale = good_sum_mart.sum_sale + excluded.sum_sale;
 
     -- При удалении записи из таблицы продаж отнимаем новую сумму из старой и удаляем запись если результат обнулился.
     elsif (tg_op = 'delete') then
         update good_sum_mart
         set sum_sale = sum_sale - (select good_price * old.sales_qty from goods where goods_id = old.good_id)
         where good_name = (select good_name from goods where goods_id = old.good_id);
         delete from good_sum_mart where sum_sale <= 0;
   end if;
  return null;
 end;
 $$ language plpgsql;
```
5) Подготавливаем тело триггера.
```
 create trigger trg_recalc_good_sum_mart
 after insert or update or delete on sales
 for each row
 execute function recalc_good_sum_mart();
```
6) Проверка работы триггерной функции.

Insert
```
postgres=# insert into sales (good_id, sales_qty) values (1, 20);
INSERT 0 1
postgres=# select * from good_sum_mart;
      good_name       | sum_sale
----------------------+----------
 Спички хозайственные |    10.00
(1 row)
```
update
```
postgres=# update sales set sales_qty = 5 WHERE sales_id = 2;
UPDATE 1
postgres=# select * from good_sum_mart;
      good_name       | sum_sale
----------------------+----------
 Спички хозайственные |    12.00
(1 row)
```
delete
```
postgres=# delete from sales where sales_id = 1;
DELETE 1
postgres=# select * from good_sum_mart;
      good_name       | sum_sale
----------------------+----------
 Спички хозайственные |     7.00
(1 row)
```
теперь добавляем продажу автомобилей и далее удаление записей по ним из таблицы продаж.
```
postgres=# insert into sales (good_id, sales_qty) values (2, 2);
INSERT 0 1
postgres=# select * from good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Спички хозайственные     |         7.00
 Автомобиль Ferrari FXX K | 370000000.02
(2 rows)

postgres=# select * from sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        3 |       1 | 2024-12-21 11:44:38.271901+00 |       120
        4 |       2 | 2024-12-21 11:44:38.271901+00 |         1
        6 |       1 | 2024-12-21 11:53:56.244226+00 |        20
        2 |       1 | 2024-12-21 11:44:38.271901+00 |         5
        7 |       2 | 2024-12-21 11:57:26.626527+00 |         2
(5 rows)

postgres=# delete from sales where good_id = 2;
DELETE 2
postgres=# select * from good_sum_mart;
      good_name       | sum_sale
----------------------+----------
 Спички хозайственные |     7.00
(1 row)
```

## Приемущества тригегрной функции с таблицей "витрина" над кодом генерации стандартного для проета отчета:
В актуальности отчетных данных по суммам продаж каждого товара за счет перерасчета этих сумм, с учетом изменения цен, до момента их применения в таблице. 

Так же при больших объемах таблицы sales выдача отчета средствами приведенного в документе алгоритма весьма затратное мероприятие, поскольку он будет пересчитывать весь объем данных таблицы sales, а триггер к тому моменту уже подготовил отчет и нагрузка от его работы была только в рамках еденичных операций для отдельных строк, что должно было быть практически незаметно для пользователей. 

Плюс на момент нужды сформировать отчет он уже по сути готов и для его выдачи достаточно сделать select с витрины good_sum_mart).

