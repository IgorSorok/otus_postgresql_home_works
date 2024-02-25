**1. Установлена и настроена ВМ в Yandex Cloud:**
```python
root@ubuntu:/home/igor# yc compute instances list
+----------------------+---------+---------------+---------+--------------+--------------+
|          ID          |  NAME   |    ZONE ID    | STATUS  | EXTERNAL IP  | INTERNAL IP  |
+----------------------+---------+---------------+---------+--------------+--------------+
| fhmiujk2h274n7uinrsu | sber-vm | ru-central1-a | RUNNING | 51.250.83.97 | 192.168.0.19 |
+----------------------+---------+---------------+---------+--------------+--------------+
```
**2. Установлен Docker Engine**
```python
Client: Docker Engine - Community
 Version:           25.0.3
 API version:       1.44
 Go version:        go1.21.6
 Git commit:        4debf41
 Built:             Tue Feb  6 21:14:17 2024
 OS/Arch:          linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          25.0.3
  API version:      1.44 (minimum version 1.24)
  Go version:       go1.21.6
  Git commit:       f417435
  Built:            Tue Feb  6 21:14:17 2024
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.28
  GitCommit:        ae07eda36dd25f8a1b98dfbf587313b99c0190bb
 runc:
  Version:          1.1.12
  GitCommit:        v1.1.12-0-g51d5e94
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```
**3. Создан каталог var/lib/postgres**
```python
sudo mkdir -p /var/lib/postgres
```
**4. Развернут контейнер с PostgreSQL 15:**
```python
yc-user@sber-vm:~$ sudo docker network create sber-net
67a4c35fa41d1d2aba217395f208adb91de9d58c693077ade62618ad9dd1120a
```
```python
yc-user@sber-vm:~$ sudo docker run --name sber-server --network sber-net -e POSTGRES_PASSWORD=postgres -d -p 5434:5434 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15

Unable to find image 'postgres:15' locally
15: Pulling from library/postgres
e1caac4eb9d2: Pull complete 
d615c5a045a4: Pull complete 
3b2141deb53c: Pull complete 
75e75606e68e: Pull complete 
413751f5b5b0: Pull complete 
9c890e547474: Pull complete 
42acbdd37ae0: Pull complete 
3564be86b294: Pull complete 
92ce85b89d40: Pull complete 
50ef3e06398b: Pull complete 
00d110c38cdf: Pull complete 
54abd977e238: Pull complete 
947d2ae74cb3: Pull complete 
a65b641a5e1e: Pull complete 
Digest: sha256:0f9282bc5b56b566d42061b80138a71c53cba907bc4f1fdbd91af24a773e3ed3
Status: Downloaded newer image for postgres:15
896413e533abee63adcbb77418617a071385ee399aa1b29daa9a9650fd31fb45
```
**5. Подключаемся к контейнеру с сервером: **
```python
yc-user@sber-vm:~$ sudo docker run -it --rm --network sber-net --name sber-client postgres:15 psql -h sber-server -U postgres
Password for user postgres: 
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.
```
**Создаем БД**
```sql
postgres=# create database sber;
CREATE DATABASE

postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges   
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | 
 sber      | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(4 rows)

```
**Создаем табличку и наполняем ее данными**
```sql
sber=#  CREATE TABLE sber (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL
);
CREATE TABLE

sber=# \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | sber | table | postgres
(1 row)

sber=# INSERT INTO sber(name) VALUES('igor');
INSERT INTO sber(name) VALUES('oleg');
INSERT INTO sber(name) VALUES('sacha');
INSERT INTO sber(name) VALUES('yri');
INSERT INTO sber(name) VALUES('liza');

INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1

sber=# select * from sber;
 id | name  
----+-------
  1 | igor
  2 | oleg
  3 | sacha
  4 | yri
  5 | liza
(5 rows)
```

**6. Удаляем контейнер:**
```sql
yc-user@sber-vm:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED       STATUS       PORTS                                                 NAMES
896413e533ab   postgres:15   "docker-entrypoint.s…"   4 hours ago   Up 4 hours   5432/tcp, 0.0.0.0:5434->5434/tcp, :::5434->5434/tcp   sber-server

yc-user@sber-vm:~$ sudo docker stop 896413e533ab
896413e533ab

yc-user@sber-vm:~$ sudo docker rm 896413e533ab
896413e533ab
```
**7. Устанавливаем снова:**
```sql
sudo docker run --name sber-server --network sber-net -e POSTGRES_PASSWORD=postgres -d -p 5434:5434 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
4967fbe77e0c06a0cc0042f16bfa7c3216eb8af91b98af2f921cdc5b0efd155e
yc-user@sber-vm:~$ sudo docker run -it --rm --network sber-net --name sber-client postgres:15 psql -h sber-server -U postgres
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

## Видим, что бвза осталась
postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges   
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | 
 sber      | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(4 rows)

## Подключимся к табличке и проверим даные, видими что они тоже остались на месте
postgres=# \c sber 
You are now connected to database "sber" as user "postgres".

sber=# select * from sber;
 id | sber  
----+-------
  1 | igor
  2 | oleg
  3 | sacha
  4 | yri
  5 | liza
(5 rows)


```

