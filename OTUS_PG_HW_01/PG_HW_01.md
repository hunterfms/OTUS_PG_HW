# Установка ssh с авторизацией по ключам и проверка уровней изоляции транзакций в PG.
## Установка ssh с авторизацией по ключам
1) Подключился к двум виртуальным машинам pg1, pg2 через PuTYY используя SHH по паролю.
2) На виртуальной машине c ОС Ubuntu 22.04 сервер pg1 создал ssh ключи без пароля 
```
fedor@pg1:~$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/fedor/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/fedor/.ssh/id_rsa
Your public key has been saved in /home/fedor/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:KEJB22J9gjWF68M5qKDzMFAQAaULmp/mekF/sQT9cYc fedor@pg1
The key's randomart image is:
+---[RSA 3072]----+
|*=+ o+.    .     |
| o Bo.. . E .    |
|o B +o.. o .     |
|o*...oo..        |
|=..=.o.oS        |
|o.ooB.o          |
|=.+. +           |
|+=.              |
|.=o              |
+----[SHA256]-----+
```
3) Скопировал открытый ключ на другую машину c ОС Ubuntu 22.04 сервер pg2
```
fedor@pg1:~$ ssh-copy-id fedor@192.168.1.102
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/fedor/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
fedor@192.168.1.102's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'fedor@192.168.1.102'"
and check to make sure that only the key(s) you wanted were added.
```
4) Зашел с сессии первой машины pg1 на вторую pg2 используя авторизацию по ключу ssh (стандартную авторизацию на pg2 по паролю не выключал)
```
fedor@pg1:~$ ssh fedor@192.168.1.102
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-122-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
New release '24.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Tue Oct 15 12:00:01 2024 from 192.168.1.179
fedor@pg2:~$ 
```
5) Повторил операцию генерации ssh ключей и копирования открытого ключа со второй машины pg2 на первую машину pg1
```
fedor@pg2:~$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/fedor/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/fedor/.ssh/id_rsa
Your public key has been saved in /home/fedor/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:wWZl3UgOJZl864rfQSNV1tlfx7JZQXQH8mxYpt+6rrc fedor@pg2
The key's randomart image is:
+---[RSA 3072]----+
|         .=*+oOB*|
|       . o+++@o.X|
|        =  .=.+=+|
|       o . ..oo..|
|        S ..o . .|
|           o.. . |
|         . .. .  |
|        . .. ... |
|         .. o+E. |
+----[SHA256]-----+
fedor@pg2:~$ ssh-copy-id fedor@192.168.1.179
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/fedor/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
fedor@192.168.1.179's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'fedor@192.168.1.179'"
and check to make sure that only the key(s) you wanted were added.

fedor@pg2:~$
```
6) Зашел с сессии pg2 на pg1 используя авторизацию по ключу ssh (стандартную авторизацию на pg2 по паролю не выключал) 
```
fedor@pg2:~$ shh fedor@192.168.1.179
-bash: shh: command not found
fedor@pg2:~$ ssh fedor@192.168.1.179
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-122-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
New release '24.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Sun Oct 20 14:24:39 2024 from 192.168.1.3
fedor@pg1:~$
```
# Установка Postgresql-14
1) Установил Postgresql-14 на удаленный сервер pg2 через ssh с сессии pg1 и наоборот
```
fedor@pg2:~$ sudo apt update
fedor@pg2:~$ sudo apt -y install postgresql-14
```
2) В итоге Postgresql-14 установился на обоих машинах. Команда pg_lsclusters в сессиях выдали:
```
fedor@pg1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
fedor@pg1:~$ exit
logout
Connection to 192.168.1.179 closed.
fedor@pg2:~$
```
```
fedor@pg2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
fedor@pg2:~$ exit
logout
Connection to 192.168.1.102 closed.
fedor@pg1:~$
```
3) Перезагрузил машины что бы проверить активность Postgres после старта системы. Postgres стартовал автоматически.

# Проверка уровней изоляции транзакций.
1)  Подключился через PuTYY к машине pg1 и запустил на ней psql (первая сессия к СУБД на PG1).
```
login as: fedor
fedor@192.168.1.179's password:
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-122-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
New release '24.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Tue Oct 22 13:05:05 2024 from 192.168.1.102
fedor@pg1:~$ sudo -u postgres psql
[sudo] password for fedor:
could not change directory to "/home/fedor": Permission denied
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=#
```

2) Далее с машины pg2 подключился через shh к pg1 и запустил psql (вторая сессия к СУБД на PG1).
```
login as: fedor
fedor@192.168.1.102's password:
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-122-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
New release '24.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Tue Oct 22 13:03:23 2024 from 192.168.1.3
fedor@pg2:~$ ssh fedor@192.168.1.179
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-122-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
New release '24.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Tue Oct 22 13:02:58 2024 from 192.168.1.3
fedor@pg1:~$ sudo -u postgres psql
[sudo] password for fedor:
could not change directory to "/home/fedor": Permission denied
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=#
```
3) В первой сесиии подключенной с PuTYY отключил autocommit и заполнил таблицу.
```
fedor@pg1:~$ sudo -u postgres psql
[sudo] password for fedor:
could not change directory to "/home/fedor": Permission denied
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# \set autocommit off;
postgres=# \echo :autocommit;
off;;
postgres=# create table persons(id serial, first_name text, second_name text);
CREATE TABLE
postgres=# insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov');
INSERT 0 1
INSERT 0 1
postgres=# commit;
WARNING:  there is no transaction in progress
COMMIT
postgres=# SHOW TRANSACTION ISOLATION LEVEL;
 transaction_isolation
-----------------------
 read committed
(1 row)

postgres=# 
```
4) Тут же в первой сесии открыл новую транзакцию и добавил еще одну строку не завершая командой commit.
```
postgres=# BEGIN;
BEGIN
postgres=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```
5) Во второй сесии выбрал данные из таблицы и не увидел новую строку, поскольку транзакция в первой сесии все еще не была завершена.
```
postgres=# BEGIN;
BEGIN
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
6) Применил изменения в первой сесии.
```
postgres=*# commit;
COMMIT
```
7) Выбрал данные с таблицы во второй сесии и получил в результате все три строки, поскольку транзакция в первой сесии была завершена с применением изменений.
```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

postgres=*# commit;
COMMIT
postgres=#
```
8) Начал новые транзакции в сессиях repeatable read. Сначала добавил новую запись в певой сесии.
```
postgres=# BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```
9) Далее выбрал данные из таблицы во второй сесии не получив данных по новой строке. В данном случае уровень изоляции repeatable read предотвращает отображение изменений, сделанных другими транзакциями, до тех пор, пока транзакция не будет завершена.
```
postgres=# BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
10) Применил изменения в первой транзакции
```
postgres=*# commit;
COMMIT
```
11) Выбрал данные таблицы во второй сесии так и не получив четвертой строки. Вывод в repeatable read - вторая сессия сделала выборку не завершив транзакцию тем самым приостанавливая состояние данных на момент начала транзакции. В данной ситуации если первая сессия завершит свою транзакцию и зафиксирует изменения вторая сессия не увидит этих изменений, пока не завершит свою транзакцию.
```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
12) Завершил транзакцию во второй сесии и выбрал данные таблицы. В итоге получил результатом все строки включая четвертую, которую добавила первая сессия, поскольку завершение обеих транзакций высвободило сделанные в их рамках изменения для новых транзакций.
```
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)

postgres=# 
```

