**1. Установил и настроил ВМ в ЯО**
```python
root@ubuntu:/home/igor# yc compute instances list
+----------------------+---------+---------------+---------+---------------+-------------+
|          ID          |  NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP  | INTERNAL IP |
+----------------------+---------+---------------+---------+---------------+-------------+
| fhmi0avn4asu024v0vld | sber-vm | ru-central1-a | RUNNING | 51.250.14.130 | 192.168.0.9 |
+----------------------+---------+---------------+---------+---------------+-------------+
```
**2. Развернул кластер PostgreSQL 14:**
```python
yc-user@sber-vm:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
**3. Зашел в psql под пользователем postgres:**
```python
postgres@sber-vm:/home/yc-user$ psql
psql (14.11 (Ubuntu 14.11-1.pgdg20.04+1))
Type "help" for help.
postgres=# \conninfo
You are connected to database "postgres" as user "postgres" via socket in "/var/run/postgresql" at port "5432
```

**4. Создал таблцу и наполнил данными**
```python
postgres=# create table poc (id int);
CREATE TABLE
postgres=# \d
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | poc  | table | postgres
(1 row)

postgres=# insert into poc select g.x from generate_series(1, 1000) as g(x);
INSERT 0 1000
```
**5. Cтопаю кластер**
```python
yc-user@sber-vm:~$ sudo pg_ctlcluster 14 main stop
Cluster is not running.
yc-user@sber-vm:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

```
**6. Добавляю и подключаю новый диск HDD:**
```python
root@ubuntu:/home/igor# yc compute disk create \
    --name new-disk \
    --type network-hdd \
    --size 1 \
    --description "second disk for sber-vm"
done (7s)

root@ubuntu:/home/igor# yc compute disk list

+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
|          ID          |   NAME   |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP |       DESCRIPTION       |
+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
| fhmo0p0s9qj3aeuhok32 |          | 16106127360 | ru-central1-a | READY  | fhmi0avn4asu024v0vld |                 |                         |
| fhmtk1fp38le1ego95cg | new-disk |  1073741824 | ru-central1-a | READY  |                      |                 | second disk for sber-vm |
+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------------------+

** Подключение

+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
|          ID          |   NAME   |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP |       DESCRIPTION       |
+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------------------+
| fhmo0p0s9qj3aeuhok32 |          | 16106127360 | ru-central1-a | READY  | fhmi0avn4asu024v0vld |                 |                         |
| fhmtk1fp38le1ego95cg | new-disk |  1073741824 | ru-central1-a | READY  | fhmi0avn4asu024v0vld |                 | second disk for sber-vm |
+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------------------+

vda            15G            
├─vda1          1M            
└─vda2 ext4    15G /          
vdb             1G


** Провожу инициализацию:

Device     Boot Start     End Sectors  Size Id Type
/dev/vdb1        2048 2097151 2095104 1023M 83 Linux


** Проверяю результат после перезагрузки:

NAME   FSTYPE  SIZE MOUNTPOINT LABEL
vda             15G            
├─vda1           1M            
└─vda2 ext4     15G /          
vdb              1G            
└─vdb1 ext4   1023M   

```

**7. Меняем овнера на директории:**

```python
yc-user@sber-vm:~$ sudo mkdir /mnt/data
yc-user@sber-vm:~$ sudo mount /dev/vdb1 /mnt/data
yc-user@sber-vm:~$ sudo chmod a+w /mnt/data
yc-user@sber-vm:~$ sudo chown -R postgres:postgres /mnt/data/

udev            1.9G     0  1.9G   0% /dev
tmpfs           392M  832K  392M   1% /run
/dev/vda2        15G  2.8G   12G  20% /
tmpfs           2.0G   28K  2.0G   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
tmpfs           392M     0  392M   0% /run/user/1000
/dev/vdb1       989M   24K  922M   1% /mnt/data

```

**8. Перемещаю данные из /var/lib/postgres/15 в /mnt/data**

 ```python
yc-user@sber-vm:~$ sudo mv /var/lib/postgresql/14 /mnt/data

```

**9. Пытаюсь стартануть..**

```python
yc-user@sber-vm:~$ sudo pg_ctlcluster 14 main start
Error: /var/lib/postgresql/14/main is not accessible or does not exist

** Что-ж видим ошибку
** Для того что-бы стартануть кластер нужно поменять путь к директории с базой в конф файле:
Для этого идем в postgresql.conf и в разделе data directory меняем путь

** Вэри гуд, всё удалось..

yc-user@sber-vm:~$ sudo pg_ctlcluster 14 main start
yc-user@sber-vm:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory    Log file
14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log


```
**10. Захожу в psql и проверяю таблицу:**

```python
yc-user@sber-vm:~$ sudo su postgres
postgres@sber-vm:/home/yc-user$ psql

psql (14.11 (Ubuntu 14.11-1.pgdg20.04+1))
Type "help" for help.

** Всё на месте, всё хорошо..

postgres=# \d
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | poc  | table | postgres
(1 row)

postgres=# select * from poc ;

  id  
------
    1
    2
    ....
    1000
...
(1000 rows)

```


