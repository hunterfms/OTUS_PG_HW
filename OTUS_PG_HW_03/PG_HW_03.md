##ДЗ по теме - Физический уровень Postgresql 

1) На виртуальной машине с сервером Ubuntu 22.04 через PuTYY установил Postgresql 14.
2) Добавил в Postgresql таблицу и строку в неё по инструкции в ДЗ
3) Остановил виртуальную машину и добавл новый диск в 10Гб.
4) Стартанул виртуалку и по инструкции в ДЗ монтировал диск в систему
```
fedor@pgindocker:~$ sudo su
root@pgindocker:/home/fedor# fdisk -l | grep 'Disk /dev/sd'
Disk /dev/sda: 10 GiB, 10737418240 bytes, 20971520 sectors
Disk /dev/sdb: 10 GiB, 10737418240 bytes, 20971520 sectors
root@pgindocker:/home/fedor# fdisk /dev/sdb

Welcome to fdisk (util-linux 2.37.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): p

Disk /dev/sdb: 10 GiB, 10737418240 bytes, 20971520 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 8068D9F6-17E2-442D-9D85-0E6AD0523AC9

Command (m for help): w
The partition table has been altered.
Syncing disks.

root@pgindocker:/home/fedor# mkdir /mnt/data
root@pgindocker:/home/fedor# mkfs.ext4 -L datapartition /dev/sdb1
mke2fs 1.46.5 (30-Dec-2021)
/dev/sdb1 is apparently in use by the system; will not make a filesystem here!
root@pgindocker:/home/fedor# lsblk --fs
NAME FSTYPE FSVER LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
sda
├─sda1
│
├─sda2
│    ext4   1.0         32c40a8e-3b99-44de-863b-2767c0858b0d      1.5G     7% /boot
└─sda3
     LVM2_m LVM2        5MBQ4u-s9Sc-a6gr-284G-ojQM-h7Bq-jtb2Wx
  └─ubuntu--vg-ubuntu--lv
     ext4   1.0         b0620a35-9a2d-42e9-ad62-05dc2e04f577        2G    69% /
sdb
└─sdb1

sr0
fedor@pgindocker:~$ sudo mkfs.ext4 -L datapartition /dev/sdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2621184 4k blocks and 655360 inodes
Filesystem UUID: 14eaa1e1-a487-4b21-9ba3-d22f1e4840fd
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

fedor@pgindocker:~$ sudo lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINT
NAME   FSTYPE    LABEL         UUID                                   MOUNTPOINT
sda
├─sda1
├─sda2 ext4                    32c40a8e-3b99-44de-863b-2767c0858b0d   /boot
└─sda3 LVM2_memb               5MBQ4u-s9Sc-a6gr-284G-ojQM-h7Bq-jtb2Wx
  └─ubuntu--vg-ubuntu--lv
       ext4                    b0620a35-9a2d-42e9-ad62-05dc2e04f577   /
sdb
└─sdb1 ext4      datapartition 14eaa1e1-a487-4b21-9ba3-d22f1e4840fd
sr0
```
5) Добавил в fstab строку для автоматического монтирования диска 
```
fedor@pgindocker:/etc$ sudo vim fstab
fedor@pgindocker:~$ sudo vim /etc/fstab
fedor@pgindocker:~$ cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
/dev/disk/by-id/dm-uuid-LVM-GGGZ1cnSIihxj25lArja19lHKcI2hTUaTccVq6G1MGgNPMyZh7h4uLMoiOsHDD40 / ext4 defaults 0 1
# /boot was on /dev/sda2 during curtin installation
/dev/disk/by-uuid/32c40a8e-3b99-44de-863b-2767c0858b0d /boot ext4 defaults 0 1
/swap.img       none    swap    sw      0       0
/dev/sdb1 /mnt/data ext4 defaults 0 2
```
6) Проверил текущую сборку дисков
```
fedor@pgindocker:~$ df -h -x tmpfs
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv  8.1G  5.7G  2.0G  75% /
/dev/sda2                          1.7G  125M  1.5G   8% /boot
/dev/sdb1                          9.8G   24K  9.3G   1% /mnt/data
```
7) Проверил запись данных и их удаление на диск по инструкции в ДЗ
```
fedor@pgindocker:~$ echo "success" | sudo tee /mnt/data/test_file
success
fedor@pgindocker:~$ cat /mnt/data/test_file
success
fedor@pgindocker:~$ sudo rm /mnt/data/test_file
```
8) Перезагрузил виртуалку и проверил активность монтированного диска.
```
login as: fedor
fedor@192.168.1.224's password:
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-124-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Thu Oct 24 18:13:42 2024
fedor@pgindocker:~$ df -h -x tmpfs
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv  8.1G  5.7G  2.0G  75% /
/dev/sda2                          1.7G  125M  1.5G   8% /boot
/dev/sdb1                          9.8G   24K  9.3G   1% /mnt/data
fedor@pgindocker:~$
```
9) Предоставил доступ к каталогу на диске для пользователя postgres
```
fedor@pgindocker:~$ sudo su
root@pgindocker:/home/fedor# chown -R postgres:postgres /mnt/data/
root@pgindocker:/home/fedor# exit
exit
fedor@pgindocker:~$
```
10) Перенос содержимого /var/lib/postgres/14 в /mnt/data
```
fedor@pgindocker:~$ sudo systemctl stop postgresql@14-main
fedor@pgindocker:~$ sudo mv /var/lib/postgresql/14 /mnt/data
```
11) Попытка старта СУБД, при которой не находится каталог с данными кластера, поскольку он был перенесен на новый диск.
```
fedor@pgindocker:~$ sudo systemctl start postgresql@14-main
Job for postgresql@14-main.service failed because the service did not take the s teps required by its unit configuration.
See "systemctl status postgresql@14-main.service" and "journalctl -xeu postgresq l@14-main.service" for details.
fedor@pgindocker:~$ sudo systemctl status postgresql@14-main
× postgresql@14-main.service - PostgreSQL Cluster 14-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled; vendor p>
     Active: failed (Result: protocol) since Thu 2024-10-24 18:31:20 UTC; 44s a>
    Process: 1205 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 14>
        CPU: 34ms

Oct 24 18:31:20 pgindocker systemd[1]: Starting PostgreSQL Cluster 14-main...
Oct 24 18:31:20 pgindocker postgresql@14-main[1205]: Error: /var/lib/postgresql>
```
12) Меняем в настройках /etc/postgresql/14/main/postgresql.conf параметр места расположения данных. В итоге имеем корректно стартованный Postgres с монтированного диска.
```
fedor@pgindocker:/etc/postgresql/14/main$ cat postgresql.conf | grep data_directory
data_directory = '/mnt/data/14/main'            # use data in another directory
fedor@pgindocker:/etc/postgresql/14/main$ cd /
fedor@pgindocker:/$ sudo systemctl start postgresql@14-main
fedor@pgindocker:/$ sudo systemctl status postgresql@14-main
● postgresql@14-main.service - PostgreSQL Cluster 14-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled; vendor pr>
     Active: active (running) since Thu 2024-10-24 18:45:12 UTC; 11s ago
    Process: 1282 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 14->
   Main PID: 1287 (postgres)
      Tasks: 7 (limit: 1019)
     Memory: 17.9M
        CPU: 107ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@14-main.service
             ├─1287 /usr/lib/postgresql/14/bin/postgres -D /mnt/data/14/main -c >
             ├─1289 "postgres: 14/main: checkpointer " "" "" "" "" "" "" "" "" ">
             ├─1290 "postgres: 14/main: background writer " "" "" "" "" "" "" "">
             ├─1291 "postgres: 14/main: walwriter " "" "" "" "" "" "" "" "" "" ">
             ├─1292 "postgres: 14/main: autovacuum launcher " "" "" "" "" "" "" >
             ├─1293 "postgres: 14/main: stats collector " "" "" "" "" "" "" "" ">
             └─1294 "postgres: 14/main: logical replication launcher " "" "" "" >

Oct 24 18:45:10 pgindocker systemd[1]: Starting PostgreSQL Cluster 14-main...
Oct 24 18:45:12 pgindocker systemd[1]: Started PostgreSQL Cluster 14-main.
lines 1-19/19 (END)
fedor@pgindocker:/$
```
13) Проверил наличие данных в таблице с монтированного диска.
```
fedor@pgindocker:/$ sudo -u postgres psql
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
(1 row)

postgres=# \q
fedor@pgindocker:/$
```
