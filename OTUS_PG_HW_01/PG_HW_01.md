## Подготовка виртуалки
1) Создал виртуалку на VMVirtualBox и установил с iso минимальную Server Ubuntu 22.04
2) + для удобства приложения - mc, vim, net-tools.
3) ufw не установлен по умолчанию в минимальной версии
``
fedor@pgindocker:~$ fedor@pgindocker:~$ sudo ufw status verbose
sudo: ufw: command not found
``
4) Установил ssh
``
sudo apt update
sudo apt install openssh-server
``
5) В конфиге /etc/ssh/sshd_config выставил порт 22 и IP где запускаю клиента PuTTY 
6) ``sudo systemctl daemon-reload``
7) ``sudo systemctl restart ssh.socket``
8) Создал каталог /var/lib/postgresql
``fedor@pgindocker:~$ sudo mkdir /var/lib/postgresql``
9) Установил докера как было показано в лекции
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
10) Запустил контейнер с пробросом каталога для хранения данных
```
fedor@pgindocker:~$ sudo docker run -p 5432:5432 -v /var/lib/postgresql/data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=password -d postgres
Unable to find image 'postgres:latest' locally
latest: Pulling from library/postgres
a480a496ba95: Pull complete
f5ece9c40e2b: Pull complete
241e5725184f: Pull complete
6832ae83547e: Pull complete
4db87ef10d0d: Pull complete
979fa3114f7b: Pull complete
f2bc6009bf64: Pull complete
c9097748b1df: Pull complete
9d5c934890a8: Pull complete
d14a7815879e: Pull complete
442a42d0b75a: Pull complete
82020414c082: Pull complete
b6ce4c941ce7: Pull complete
42e63a35cca7: Pull complete
Digest: sha256:8d3be35b184e70d81e54cbcbd3df3c0b47f37d06482c0dd1c140db5dbcc6a808
Status: Downloaded newer image for postgres:latest
9467377cd5f22eff3097efea86878e50496d1495fac43ab689e1182b8773b4c0
fedor@pgindocker:~$ sudo docker ps
CONTAINER ID   IMAGE      COMMAND                  CREATED         STATUS         PORTS                                       NAMES
2e172fd42655   postgres   "docker-entrypoint.s…"   3 minutes ago   Up 3 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   hungry_lehmann
```

11) Добавление рандомной таблицы с данными
```
fedor@pgindocker:~$ sudo docker exec -it 2e172fd42655 bash
root@2e172fd42655:/# psql -U postgres
psql (17.0 (Debian 17.0-1.pgdg120+1))
Type "help" for help.
postgres=# create table test1 (column1 int);
CREATE TABLE
postgres=# insert into test1 values (1);
INSERT 0 1
postgres=# insert into test1 values (2);
INSERT 0 1
postgres=# select * from test1;
 column1
---------
       1
       2
(2 rows)
postgres=# \q
root@2e172fd42655:/#
```

12) Запустил еще один контейнер для проверки удаленного подключения
```
fedor@pgindocker:~$ sudo docker run -p 5433:5432 -e POSTGRES_PASSWORD=mysecretpassword -d postgres
fedor@pgindocker:~$ sudo docker ps
CONTAINER ID   IMAGE      COMMAND                  CREATED          STATUS          PORTS                                       NAMES
611537301c70   postgres   "docker-entrypoint.s…"   11 seconds ago   Up 10 seconds   5432/tcp                                    some-postgres
2e172fd42655   postgres   "docker-entrypoint.s…"   28 minutes ago   Up 28 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   silly_moore
```

13) Проверил подключение из доп контейнера с выводом ранее созданной таблицы и добавил еще одну таблицу с данными
```
root@611537301c70:/# psql -h 192.168.1.224 -p 5432 -U postgres
Password for user postgres:
psql (17.0 (Debian 17.0-1.pgdg120+1))
Type "help" for help.

postgres=# select * from test1;
 column1
---------
       1
       2
(2 rows)

postgres=# create table test2 (column1 int);
CREATE TABLE
postgres=# insert into test2 values (1);
INSERT 0 1
postgres=# insert into test2 values (2);
INSERT 0 1
postgres=# select * from test2;
 column1
---------
       1
       2
(2 rows)

postgres=#
```

14) Остановил оба контейнера
```
fedor@pgindocker:~$ sudo docker stop 611537301c70
611537301c70
fedor@pgindocker:~$ sudo docker stop 2e172fd42655
2e172fd42655
fedor@pgindocker:~$ sudo docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

15) Запустил контейнер с СУБД данные которого проброшены в локальный каталог виртуалки
```
fedor@pgindocker:~$ sudo docker run -p 5432:5432 -v /var/lib/postgresql/data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=password -d postgres
e78b57dd05847d3678fd144afe308603c8918a9019ca6b8895718e6a37a53054
fedor@pgindocker:~$ sudo docker run -p 5433:5432 -e POSTGRES_PASSWORD=mysecretpassword -d postgres
5e137f095847867de3a1926efcb4a034ecf3df4dbc9456924b395f31ee2f85ed
fedor@pgindocker:~$ sudo docker ps
CONTAINER ID   IMAGE      COMMAND                  CREATED          STATUS          PORTS                                         NAMES
5e137f095847   postgres   "docker-entrypoint.s…"   2 seconds ago    Up 2 seconds    0.0.0.0:5433->5432/tcp, [::]:5433->5432/tcp   epic_feynman
e78b57dd0584   postgres   "docker-entrypoint.s…"   39 seconds ago   Up 38 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp     tender_khorana
fedor@pgindocker:~$ sudo docker exec -it ^Cash
fedor@pgindocker:~$ sudo docker exec -it 5e137f095847 bash
```

16) Проверил наличие ранее сохранённых данных в связанный каталог /var/lib/postgresql
```
root@5e137f095847:/# psql -h 192.168.1.224 -p 5432 -U postgres
Password for user postgres:
psql (17.0 (Debian 17.0-1.pgdg120+1))
Type "help" for help.

postgres=# select * from test1;
 column1
---------
       1
       2
(2 rows)

postgres=# select * from test2;
 column1
---------
       1
       2
(2 rows)

postgres=# \q
root@5e137f095847:/# exit
exit
fedor@pgindocker:~$
```

## Где были проблемы 
1) Пришлось искать в инете как правильно транслировать каталог в контейнер.
