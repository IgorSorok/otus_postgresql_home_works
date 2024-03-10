
***На локальной ВМ развернул кластер PostgreSQL***
```python
postgres@ubuntu:/root$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

```
 **Создал БД testdb под пользователем postgres**
 ```python
postgres=# create database testdb;
CREATE DATABASE
Time: 827,426 ms

postgres=# \l testdb 
                              List of databases
  Name  |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges 
--------+----------+----------+-------------+-------------+-------------------
 testdb | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
(1 row)

```
**В целевой БД создаю схему testnm**

```python
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
Time: 3,650 ms

testdb=# \dn+
                          List of schemas
  Name  |  Owner   |  Access privileges   |      Description       
--------+----------+----------------------+------------------------
 public | postgres | postgres=UC/postgres+| standard public schema
        |          | =UC/postgres         | 
 testnm | postgres |                      | 
(2 rows)

```
***Создаю таблицу в схеме testnm и наполняю данными***
```python
testdb=# CREATE TABLE t1(c1 integer);
CREATE TABLE
Time: 207,801 ms

testdb=# INSERT INTO t1 VALUES (1);
INSERT 0 1
Time: 309,945 ms
```
 ***Создаю роль и раздаю гранты*** 
 ```python
testdb=# CREATE ROLE readonly;
testdb=# \du readonly 
            List of roles
 Role name |  Attributes  | Member of 
-----------+--------------+----------
 readonly  | Cannot login | {}

-- Даю гранты
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;  -грант на подключение к БД
GRANT
Time: 309,310 ms

testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;  -грант на использование схемы
GRANT
Time: 316,109 ms

testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;   -грант на чтение всех таблиц в схеме существующих на данный момент
GRANT
Time: 297,975 ms

testdb=# ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly;   -Задаем права для роли по умолчанию
ALTER DEFAULT PRIVILEGES
Time: 369,273 ms
```

***Создаю юзера и даю ему грант роли***
```python
testdb=# CREATE USER testread WITH PASSWORD '111';
testdb=# GRANT readonly TO testread;
GRANT ROLE
Time: 0,202 ms

testdb=# \du testread 
            List of roles
 Role name | Attributes | Member of  
-----------+------------+------------
 testread  |            | {readonly}
```

 ***Подключаюсь к базе под юзером и даю селект***
```python
testdb=# \c testdb testread
Password for user testread: 
You are now connected to database "testdb" as user "testread".

testdb=> \conninfo 
You are connected to database "testdb" as user "testread" via socket in "/var/run/postgresql" at port "5432".

testdb=> SELECT * FROM t1;
 c1 
----
  1
(1 row)
Time: 0,494 ms

Почему есть доступ? 
Будет чуть ниже)
```

***Удалять я ее конечно же не буду, просто сделаю новую с указанием схемы:***
```python
testdb=# CREATE TABLE testnm.t2(c1 integer);
CREATE TABLE
Time: 339,805 ms

testdb=# INSERT INTO testnm.t2 VALUES (2);
INSERT 0 1
Time: 297,589 ms
```

***Подключаюсь под юзером и проверяю селект***
```python
testdb=# \c testdb testread 
Password for user testread: 
You are now connected to database "testdb" as user "testread".

testdb=> select * from testnm.t2 ;
 c1 
----
  2
(1 row)
Time: 0,371 ms
```

***Всё работает 🙂. Потому-что:***
```python
testdb=# \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 testnm | t1   | table | postgres
 testnm | t2   | table | postgres
(3 rows)


Схемой public на проме не пользуются, хотя находятся и такие, т.к в каждой базе по умолчанию создается эта схема,
и роль public добавляется всем новым пользователям. И получается, что если у пользователя есть грант на коннект к базе, 
то он может создавать объекты в схеме public. В целях безопасности такого быть не должно.  Поэтому я ее отодвинул в поиске.

После создания схемы, я поменял search_path на кластерном уровне, в конфиге и привел к такому виду
testnm, public, "$user", pg_catalog, pg_temp;

Сомневаюсь в том, на сколько вообще это решение верное.. Буду признателен получить совет/рекомендацию.

Так же можно задать путь поиска на уровне транзакции. Выглядит это примерно так:
SET search_path TO testnm, public, "$user", pg_catalog, pg_temp;

Если есть еще какие-то способы, то также буду рад услышать/увидеть обратную связь.

Как недавно узнал в 15 PostgresSQL доступ к public прикрыли. Чтож, оно и к лучшему..
```
***Теперь по юзером testread пробую создать табличку***
```python
testdb=> CREATE TABLE t3(c1 integer);
ERROR:  permission denied for schema testnm
Time: 0,316 ms

Никаких результатов, но так и должно быть.. 
Ведь никто не давал грантов на создание таблиц для роли readonly
``` 
 
 Для меня эта тема очень сложная.. Слишком много нюансов..:(
 
 В качестве обратной связи хотелось бы получить:
 1. Рекомендации по настройке search_path
 2. По возможности доплнительных материалов для изучения, на тему работы с пользователями и схемами.
 Спасибо:)

