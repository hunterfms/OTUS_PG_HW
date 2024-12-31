## ДЗ по теме - Сбор и использование статистики. 
(Хотя по содержанию вопросов в ДЗ оно видимо должно быть на тему  - Типы соединения таблиц.)

1) Подготавливаю структуру трех таблиц для выборок с разными типами соединений.

   Таблица Users будет содержать справочник учетных записей пользователей.

   Таблица categoryes будет содержать справочник категорий доступа к определенному функционалу предполагаемого ПО.

   Таблица categ_users будет содержать привязки пользователей к категориям по которым ПО будет определять категорию доступа для каждого пользователя.
```
postgres=# create database accessusers;
CREATE DATABASE
postgres=# \c accessusers
You are now connected to database "accessusers" as user "postgres".
accessusers=# create schema dbo;
CREATE SCHEMA
accessusers=# create table if not exists dbo.users (
        user_id bigint,
        user_name varchar(20) not null,
        full_name varchar(50) not null,
        status boolean not null,
        constraint users_pkey primary key (user_id));
CREATE TABLE
accessusers=# create table if not exists dbo.categoryes (
        categ_id bigint,
        categ_name varchar(20) not null,
        constraint categoryes_pkey primary key (categ_id));
CREATE TABLE
accessusers=# create table if not exists dbo.categ_users (
        id bigint,
        user_id bigint not null,
          categ_id bigint not null,
        constraint categ_users_pkey primary key (id));
CREATE TABLE
accessusers=#
```
2) Заполняю таблицы данными. В таблице dbo.categ_users для каждого пользователя определяем категорию доступа к ПО.
```
accessusers=# insert into dbo.users select 1, 'AIvanov', 'Иванов Александр', true;
insert into dbo.users select 2, 'AIvanov', 'Иванов Александр', true;
insert into dbo.users select 3, 'OLaptev', 'Лаптев Олег', true;
insert into dbo.users select 4, 'SPospelov', 'Поспелов Станислав', false;
insert into dbo.users select 5, 'LTsoy', 'Цой Елена', false;
insert into dbo.users select 6, 'IKrutoy', 'Крутой Игорь', true;
insert into dbo.users select 7, 'AZubarev', 'Зубарев Андрей', true;
insert into dbo.users select 8, 'FSnytkin', 'Сныткин Федор', true;
insert into dbo.users select 9, 'RCherny', 'Черный Роман', true;
insert into dbo.users select 10, 'ATroyan', 'Троян Анатолий', true;
insert into dbo.users select 11, 'ALeshyi', 'Леший Алексей', true;
INSERT 0 1
INSERT 0 1
...
accessusers=# insert into dbo.categoryes select 1, 'Administrator';
insert into dbo.categoryes select 2, 'ReadOnly';
insert into dbo.categoryes select 3, 'Operator';
insert into dbo.categoryes select 4, 'StuffOperator';
insert into dbo.categoryes select 5, 'Manager';
insert into dbo.categoryes select 6, 'Support';
INSERT 0 1
INSERT 0 1
...
accessusers=# insert into dbo.categ_users select 1, 6, 1;
insert into dbo.categ_users select 2, 1, 2;
insert into dbo.categ_users select 3, 2, 5;
insert into dbo.categ_users select 4, 3, 6;
insert into dbo.categ_users select 5, 4, 3;
insert into dbo.categ_users select 6, 5, 3;
insert into dbo.categ_users select 7, 7, 4;
insert into dbo.categ_users select 8, 8, 3;
INSERT 0 1
INSERT 0 1
...
accessusers=#
```
4) Реализуем запрос с соединением таблиц в рамках Inner Join.

Выбираем всех активных пользователей с категорией доступа "Operator".
```
accessusers=# select
 u.user_name,
 u.full_name,
 c.categ_name,
 u.status
from dbo.users as u
inner join dbo.categ_users as cu
 on u.user_id = cu.user_id
inner join dbo.categoryes as c
 on cu.categ_id = c.categ_id
where c.categ_name = 'Operator'
 and u.status = true;
 user_name |   full_name   | categ_name | status
-----------+---------------+------------+--------
 FSnytkin  | Сныткин Федор | Operator   | t
(1 row)

accessusers=#
```

Выбираем всех неактивных пользователей пользователей с выводом их категорией доступа.
```
accessusers=# select
 u.user_name,
 u.full_name,
 c.categ_name,
 u.status
from dbo.users as u
inner join dbo.categ_users as cu
 on u.user_id = cu.user_id
inner join dbo.categoryes as c
 on cu.categ_id = c.categ_id
where u.status = false;
 user_name |     full_name      | categ_name | status
-----------+--------------------+------------+--------
 SPospelov | Поспелов Станислав | Operator   | f
 LTsoy     | Цой Елена          | Operator   | f
(2 rows)

accessusers=#
```
5) Реализуем запрос с соединением таблиц в рамках left Join.

Выбираем всех пользователей не имеющих привязки к катериям доступа.
```
accessusers=# select
 u.user_name,
 u.full_name,
 u.status
from dbo.users as u
left join dbo.categ_users as cu
 on u.user_id = cu.user_id
 where cu.user_id is null;
 user_name |   full_name    | status
-----------+----------------+--------
 RCherny   | Черный Роман   | t
 ATroyan   | Троян Анатолий | t
 ALeshyi   | Леший Алексей  | t
(3 rows)

accessusers=#
```
6) Реализуем запрос с соединением таблиц в рамках inner + cross Join.
   
Выводим список всех возможных категорий которые можно дать каждому активному пользователю.
```
accessusers=# select
 u.user_name,
 u.full_name,
 c.categ_name,
 u.status
from dbo.users as u
inner join dbo.categ_users as cu
 on u.user_id = cu.user_id
cross join dbo.categoryes as c
 where u.status = true;
 user_name |    full_name     |  categ_name   | status
-----------+------------------+---------------+--------
 IKrutoy   | Крутой Игорь     | Administrator | t
 ISidorov  | Сидоров Иван     | Administrator | t
 AIvanov   | Иванов Александр | Administrator | t
 OLaptev   | Лаптев Олег      | Administrator | t
 AZubarev  | Зубарев Андрей   | Administrator | t
 FSnytkin  | Сныткин Федор    | Administrator | t
 IKrutoy   | Крутой Игорь     | ReadOnly      | t
 ISidorov  | Сидоров Иван     | ReadOnly      | t
 AIvanov   | Иванов Александр | ReadOnly      | t
 OLaptev   | Лаптев Олег      | ReadOnly      | t
 AZubarev  | Зубарев Андрей   | ReadOnly      | t
 FSnytkin  | Сныткин Федор    | ReadOnly      | t
 IKrutoy   | Крутой Игорь     | Operator      | t
 ISidorov  | Сидоров Иван     | Operator      | t
 AIvanov   | Иванов Александр | Operator      | t
 OLaptev   | Лаптев Олег      | Operator      | t
 AZubarev  | Зубарев Андрей   | Operator      | t
 FSnytkin  | Сныткин Федор    | Operator      | t
 IKrutoy   | Крутой Игорь     | StuffOperator | t
 ISidorov  | Сидоров Иван     | StuffOperator | t
 AIvanov   | Иванов Александр | StuffOperator | t
 OLaptev   | Лаптев Олег      | StuffOperator | t
 AZubarev  | Зубарев Андрей   | StuffOperator | t
 FSnytkin  | Сныткин Федор    | StuffOperator | t
 IKrutoy   | Крутой Игорь     | Manager       | t
 ISidorov  | Сидоров Иван     | Manager       | t
 AIvanov   | Иванов Александр | Manager       | t
 OLaptev   | Лаптев Олег      | Manager       | t
 AZubarev  | Зубарев Андрей   | Manager       | t
 FSnytkin  | Сныткин Федор    | Manager       | t
 IKrutoy   | Крутой Игорь     | Support       | t
 ISidorov  | Сидоров Иван     | Support       | t
 AIvanov   | Иванов Александр | Support       | t
 OLaptev   | Лаптев Олег      | Support       | t
 AZubarev  | Зубарев Андрей   | Support       | t
 FSnytkin  | Сныткин Федор    | Support       | t
(36 rows)

accessusers=#
```
7) Реализуем запрос с соединением таблиц в рамках Full Join.

Выводим в единый список всех пользователей и категорий не имеющих привязки друг к другу (незадействованные пользователи и категории).  Такой список может быть полезен при аудите, для выявления некорректно распределенного доступа администратором.  

Для начала добавим еще пару категорий к которым не будут привязаны пользователи.  Непривязанные к категориям пользователи у нас уже есть в количестве = 3, поэтому их добавлять не потребуется.
```
accessusers=# insert into dbo.categoryes select 7, 'Analitic';
insert into dbo.categoryes select 8, 'Student';
INSERT 0 1
INSERT 0 1
accessusers=#
```

Выбираем список всех пользователей и категорий не имеющих привязки друг к другу (незадействованные пользователи и категории).
```
accessusers=# select
 u.user_name,
 u.full_name,
 c.categ_name,
 u.status
from dbo.users as u
full join dbo.categ_users as cu
 on u.user_id = cu.user_id
full join dbo.categoryes as c
 on cu.categ_id = c.categ_id
where cu.categ_id is null and cu.user_id is null;
 user_name |   full_name    | categ_name | status
-----------+----------------+------------+--------
 ALeshyi   | Леший Алексей  |            | t
 ATroyan   | Троян Анатолий |            | t
 RCherny   | Черный Роман   |            | t
           |                | Student    |
           |                | Analitic   |
(5 rows)

accessusers=#
```
