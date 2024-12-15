# ДЗ на тему - Резервное копирование
1) Создаем каталог для хранения резервных копий /media/backup и назначаем владельца postgres
```
fedor@pg15:/$ sudo mkdir /media/backup
[sudo] password for fedor:
fedor@pg15:/$ cd /media/backup
fedor@pg15:/media/backup$ sudo chown postgres /media/backup
fedor@pg15:/media/backup$ ls -al
total 8
drwxr-xr-x 2 postgres root 4096 Dec 15 09:16 .
drwxr-xr-x 3 root     root 4096 Dec 15 09:16 ..
fedor@pg15:/media/backup$
```
2) Создал базу для теста создания бакапа db_for_test_backup. Далее в этой базе создал новую схему и таблицу. Заполнил таблицу случайной сотней записей. 
```
postgres=# create database db_for_test_backup;
CREATE DATABASE

postgres=# \c db_for_test_backup
You are now connected to database "db_for_test_backup" as user "postgres".
db_for_test_backup=# create schema if not exists new_schema;
CREATE SCHEMA
db_for_test_backup=# create table  new_schema.test (id bigint, text_values text);
CREATE TABLE
db_for_test_backup=# insert into new_schema.test (id, text_values) select g.x, 'text'||g.x from generate_series(1, 100) as g(x);
INSERT 0 100
db_for_test_backup=# select * from new_schema.test limit 5;
 id | text_values
----+-------------
  1 | text1
  2 | text2
  3 | text3
  4 | text4
  5 | text5
(5 rows)

db_for_test_backup=#
```
3) c
