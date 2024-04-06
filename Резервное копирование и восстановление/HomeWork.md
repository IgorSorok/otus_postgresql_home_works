
***Создал ВМ и развернул кластер PG***
```python
postgres@ubuntu:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

```

***БД, схема, таблица***

```python
-- Создал БД

postgres=# create database parilka;
CREATE DATABASE
Time: 383,642 ms
postgres=# \l parilka 
                              List of databases
  Name   |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges 
---------+----------+----------+-------------+-------------+-------------------
 parilka | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
(1 row)

-- Создал схему

postgres=# \c parilka 
You are now connected to database "parilka" as user "postgres".

parilka=# create schema kurilka;
CREATE SCHEMA
Time: 340,482 ms

parilka=# \dn kurilka 
  List of schemas
  Name   |  Owner   
---------+----------
 kurilka | postgres
(1 row)


-- Создал таблицу в схеме и заполонил ее данными

parilka=# create table kurilka.inflave as select generate_series(1, 10000) as id, md5(random()::text)::char(10) as name;
SELECT 10000
Time: 52,461 ms

parilka=# \dt kurilka.inflave 
          List of relations
 Schema  |  Name   | Type  |  Owner   
---------+---------+-------+----------
 kurilka | inflave | table | postgres
(1 row)

parilka=# select count(*) from kurilka.inflave ;
 count 
-------
 10000
(1 row)
Time: 1,457 ms

```

***Каталог для бэкапов***

```python

postgres@ubuntu:/home/igor$ cd $HOME
postgres@ubuntu:~$ mkdir pg_backpup
postgres@ubuntu:~$ ls --color
13  14  15  pg_backpup


```

***Снимаю логическую копию***

```python
postgres=# \c parilka 
You are now connected to database "parilka" as user "postgres".

parilka=# \copy kurilka.inflave to /var/lib/postgresql/pg_backpup/kurilka_dump.sql
COPY 10000
Time: 4,452 ms
parilka=# \q

postgres@ubuntu:~$ cd pg_backpup/
postgres@ubuntu:~/pg_backpup$ ls
kurilka_dump.sql


```

***Восттановление***

```python
-- Создаю вторую таблицу

parilka=# create table kurilka.inflave2 (id int, name text)
parilka-# ;
CREATE TABLE
Time: 6,972 ms

-- Восстанавливаюсь во вторую таблицу
parilka=# \copy kurilka.inflave2 from /var/lib/postgresql/pg_backpup/kurilka_dump.sql 
COPY 10000
Time: 338,965 ms

parilka=# \dt kurilka.*
           List of relations
 Schema  |   Name   | Type  |  Owner   
---------+----------+-------+----------
 kurilka | inflave  | table | postgres
 kurilka | inflave2 | table | postgres
(2 rows)

-- Проверка

parilka=# select count(*) from kurilka.inflave2;
 count 
-------
 10000
(1 row)
Time: 1,516 ms


```

***Используя утилиту pg_dump создадим бэкап***

```python

-- снимаю дамп
postgres@ubuntu:~/pg_backpup$ pg_dump -d parilka --create -Fc > /var/lib/postgresql/pg_backpup/parilka.gz

postgres@ubuntu:~/pg_backpup$ ls -lh
total 328K
-rw-rw-r-- 1 postgres postgres 156K апр  6 12:46 kurilka_dump.sql
-rw-rw-r-- 1 postgres postgres 172K апр  6 13:03 parilka.gz

```

***Восстановление при помощи pg_restore***

```python

-- Создал новую БД

postgres@ubuntu:~/pg_backpup$ psql
Timing is on.
psql (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# create database ne_kuri_igor;
CREATE DATABASE
Time: 767,235 ms

postgres=# \l ne_kuri_igor 
                                 List of databases
     Name     |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges 
--------------+----------+----------+-------------+-------------+-------------------
 ne_kuri_igor | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
(1 row)

-- Создал схему

ne_kuri_igor=# create schema kurilka;
CREATE SCHEMA
Time: 304,608 ms

ne_kuri_igor=# \dn
  List of schemas
  Name   |  Owner   
---------+----------
 kurilka | postgres
 public  | postgres
(2 rows)

-- Запускаю рестор и проверка

postgres@ubuntu:~/pg_backpup$ pg_restore -n kurilka -t inflave2 /var/lib/postgresql/pg_backpup/parilka.gz -d ne_kuri_igor

postgres@ubuntu:~/pg_backpup$ psql
Timing is on.
psql (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1))
Type "help" for help.
             
postgres=# \c ne_kuri_igor 
You are now connected to database "ne_kuri_igor" as user "postgres".
         
ne_kuri_igor=# \dt kurilka.inflave2 
           List of relations
 Schema  |   Name   | Type  |  Owner   
---------+----------+-------+----------
 kurilka | inflave2 | table | postgres
(1 row)

ne_kuri_igor=# select count(*) from kurilka.inflave2 ;
 count 
-------
 10000
(1 row)

Time: 1,295 ms


```
