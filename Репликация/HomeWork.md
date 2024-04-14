
***Для начала создаю первый экземпляр***

```python
Ver Cluster    Port Status Owner    Data directory                    Log file
14  main       5432 online postgres /var/lib/postgresql/14/main       /var/log/postgresql/postgresql-14-main.log
```

***Действия на первом инстансе***
```python

-- Применил необходимые параметры

postgres=# show wal_level ;
 wal_level 
-----------
 logical
(1 row)

-- Содал БД

postgres=# create database repa;
CREATE DATABASE

postgres=# \l+ repa 
                                                List of databases
 Name |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges |  Size   | Tablespace | Description 
------+----------+----------+-------------+-------------+-------------------+---------+------------+-------------
 repa | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                   | 8585 kB | pg_default | 
(1 row)


-- Создаем таблицу с которой будем реплицировать и наполняем данными

postgres=# \c repa ;
You are now connected to database "repa" as user "postgres".

repa=# create table test(id int primary key, name text);
CREATE TABLE

repa=# insert into test values (1, 'Value 1'), (2, 'Value 2');
INSERT 0 2

repa=# select * from test;
 id |  name   
----+---------
  1 | Value 1
  2 | Value 2
(2 rows)


-- Второя таблица без данных
repa=# create table test2(id int primary key, name text);
CREATE TABLE

-- Задаем пароль пользователю 
-- (в домашке будет postgres, а так конечно лучше использовать отдельного пользователя)

repa=# \password postgres 
Enter new password for user "postgres": 
Enter it again: 

-- Создаю публикацию в таблице test

repa=# \dt
         List of relations
 Schema | Name  | Type  |  Owner   
--------+-------+-------+----------
 public | test  | table | postgres
 public | test2 | table | postgres
(2 rows)

repa=# CREATE PUBLICATION testovaya FOR TABLE test;
CREATE PUBLICATION

repa=# \dRp[+]
                           Publication testovaya
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root 
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test"
```

***Второй экземпляр***
```python
До создания подписки, проделываем те же шаги, что и в первом экземпляре.
Либо снимаем дамп первого кластера и разворачиваем во втором

postgres@ubuntu:/home/igor$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
14  my      5433 online postgres /var/lib/postgresql/14/my   /var/log/postgresql/postgresql-14-my.log

postgres@ubuntu:/home/igor$ psql -p 5433
could not change directory to "/home/igor": Permission denied
Timing is on.
psql (14.11 (Ubuntu 14.11-1.pgdg22.04+1))

postgres=# show wal_level ;
 wal_level 
-----------
 logical
(1 row)

postgres=# \l+ repa 
                                                List of databases
 Name |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges |  Size   | Tablespace | Description 
------+----------+----------+-------------+-------------+-------------------+---------+------------+-------------
 repa | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                   | 8585 kB | pg_default | 
(1 row)

repa=# \dt
         List of relations
 Schema | Name  | Type  |  Owner   
--------+-------+-------+----------
 public | test  | table | postgres
 public | test2 | table | postgres
(2 rows)

-- Заполняем таблицу с которой будем реплицировать

-- Вторая таблица

repa=# create table test2(id int primary key, name text);
CREATE TABLE

repa=# insert into test2 values (1, 'Значение 1'), (2, 'Значение 2');
INSERT 0 2

repa=# select * from test2;
 id |    name    
----+------------
  1 | Значение 1
  2 | Значение 2
(2 rows)

Плюс задаем пароль пользователю postgres

-- Теперь создаем подписку на таблицу test

repa=# CREATE SUBSCRIPTION podpiska ## Коверкаем великий и могучий английский язык, как можем))
CONNECTION 'host=localhost port=5432 user=postgres password=111 dbname=repa' 
PUBLICATION testovaya WITH (copy_data = true);
NOTICE:  created replication slot "podpiska" on publisher
CREATE SUBSCRIPTION

repa=# \dRs
            List of subscriptions
   Name   |  Owner   | Enabled | Publication 
----------+----------+---------+-------------
 podpiska | postgres | t       | {testovaya}
(1 row)

repa=# SELECT * FROM pg_stat_subscription\gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16430
subname               | podpiska
pid                   | 34497
relid                 | 
received_lsn          | 0/1787600
last_msg_send_time    | 2024-04-14 10:23:09.541108+03
last_msg_receipt_time | 2024-04-14 10:23:09.541227+03
latest_end_lsn        | 0/1787600
latest_end_time       | 2024-04-14 10:23:09.541108+03

- Так как при создании подписки был включен параметр copy_data проверим, появились ли данные в табличке

repa=# select * from test;
 id |  name   
----+---------
  1 | Value 1
  2 | Value 2
(2 rows)
усё есть

-- И здесь же создаем публикацию для таблицы test2

repa=# CREATE PUBLICATION testovaya2 FOR TABLE test2;
CREATE PUBLICATION

repa=# \dRp[+]
                           Publication testovaya2
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root 
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test2"

```

***Возвращаемся на первый экземпляр***

```python
-- Подписываемся на публикацию таблицы test2

repa=# create subscription podpiska2 connection 'host=localhost port=5433 user=postgres password=111 dbname=repa' publication testovaya2 with (copy_data=true);
NOTICE:  created replication slot "podpiska2" on publisher
CREATE SUBSCRIPTION

-- Проверка
repa=# select * from test2;
 id |    name    
----+------------
  1 | Значение 1
  2 | Значение 2
(2 rows)

repa=# \dRs
             List of subscriptions
   Name    |  Owner   | Enabled | Publication  
-----------+----------+---------+--------------
 podpiska2 | postgres | t       | {testovaya2}
(1 row)

repa=# SELECT * FROM pg_stat_subscription\gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16422
subname               | podpiska2
pid                   | 34581
relid                 | 
received_lsn          | 0/179A730
last_msg_send_time    | 2024-04-14 10:39:20.767154+03
last_msg_receipt_time | 2024-04-14 10:39:20.767197+03
latest_end_lsn        | 0/179A730
latest_end_time       | 2024-04-14 10:39:20.767154+03

```

***Создаем третий экземпляр***
```python

root@ubuntu:/home/igor# pg_lsclusters 
Ver Cluster Port Status Owner    Data directory               Log file
14  main    5432 online postgres /var/lib/postgresql/14/main  /var/log/postgresql/postgresql-14-main.log
14  my      5433 online postgres /var/lib/postgresql/14/my    /var/log/postgresql/postgresql-14-my.log
14  node3   5434 online postgres /var/lib/postgresql/14/node3 /var/log/postgresql/postgresql-14-node3.log

-- wal_level оставляем по умолчанию

repa=# show wal_level ;
 wal_level 
-----------
 replica
(1 row)

-- Создаем БД
postgres=# \l repa 
                             List of databases
 Name |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges 
------+----------+----------+-------------+-------------+-------------------
 repa | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
(1 row)

-- Создаем две таблички без данных
repa=# create table test(id int primary key, name text);
CREATE TABLE
repa=# create table test2(id int primary key, name text);
CREATE TABLE

repa=# \dt
         List of relations
 Schema | Name  | Type  |  Owner   
--------+-------+-------+----------
 public | test  | table | postgres
 public | test2 | table | postgres
(2 rows)

-- Задаем пароль рользователю postgres

-- Пописываемся на таблицу test (первый кластер)

repa=# CREATE SUBSCRIPTION podpiska3
CONNECTION 'host=localhost port=5432 user=postgres password=111 dbname=repa' 
PUBLICATION testovaya WITH (copy_data = true);
NOTICE:  created replication slot "podpiska3" on publisher
CREATE SUBSCRIPTION

repa=# select * from test;
 id |  name   
----+---------
  1 | Value 1
  2 | Value 2
(2 rows)

repa=# \dRs
            List of subscriptions
   Name    |  Owner   | Enabled | Publication 
-----------+----------+---------+-------------
 podpiska3 | postgres | t       | {testovaya}
(1 row)

-- Подписываемся на таблицу test2 (второй кластер)

repa=# create subscription podpiska4 connection 'host=localhost port=5433 user=postgres password=111 dbname=repa' publication testovaya2 with (copy_data=true);
NOTICE:  created replication slot "podpiska4" on publisher
CREATE SUBSCRIPTION

repa=# select * from test2;
 id |    name    
----+------------
  1 | Значение 1
  2 | Значение 2
(2 rows)

repa=# \dRs
             List of subscriptions
   Name    |  Owner   | Enabled | Publication  
-----------+----------+---------+--------------
 podpiska3 | postgres | t       | {testovaya}
 podpiska4 | postgres | t       | {testovaya2}
(2 rows)

Что-ж пока как видим, всё работает, данные которые были в паблицируемых табличках, успешно реплицируются..

```
***Попробуем добавить новых данных***

```python
-- Топпаем в первый кластер и добавляем данные в таблицу test

repa=# insert into test values (3, 'Value 3'), (4, 'Value 4');
INSERT 0 2

-- Теперь галопом дуем во второй кластер и пишем новые данные в табличку test2

repa=# insert into test2 values (3, 'Значение 3'), (4, 'Значение 4');
INSERT 0 2

ПРОВЕРЯЕМ

-- в первом кластере (видим данные пришли)
repa=# select * from test2;
 id |    name    
----+------------
  1 | Значение 1
  2 | Значение 2
  3 | Значение 3
  4 | Значение 4
(4 rows)

-- во втором кластере (всё отлично)
 repa=# select * from test;
 id |  name   
----+---------
  1 | Value 1
  2 | Value 2
  3 | Value 3
  4 | Value 4
(4 rows)

-- Теперь в 3 кластере (красота да и только)

repa=# select * from test;
 id |  name   
----+---------
  1 | Value 1
  2 | Value 2
  3 | Value 3
  4 | Value 4
(4 rows)

repa=# select * from test2;
 id |    name    
----+------------
  1 | Значение 1
  2 | Значение 2
  3 | Значение 3
  4 | Значение 4
(4 rows)

Репликация работает. Пойду просить прибавку к ЗП, как бы не уволили..)
```

***Создаем четвертый экземпляр***

```python
root@ubuntu:/home/igor# pg_lsclusters 

Ver Cluster Port Status Owner    Data directory               Log file
14  main    5432 online postgres /var/lib/postgresql/14/main  /var/log/postgresql/postgresql-14-main.log
14  my      5433 online postgres /var/lib/postgresql/14/my    /var/log/postgresql/postgresql-14-my.log
14  node3   5434 online postgres /var/lib/postgresql/14/node3 /var/log/postgresql/postgresql-14-node3.log
14  node4   5435 online postgres /var/lib/postgresql/14/node4 /var/log/postgresql/postgresql-14-node4.log

-- Cносим директорию с данными
root@ubuntu:/home/igor# sudo rm -rf /var/lib/postgresql/14/node4

Ver Cluster Port Status Owner     Data directory               Log file
14  node4   5435 down   <unknown> /var/lib/postgresql/14/node4 /var/log/postgresql/postgresql-14-node4.log

-- Идём в ноду 3 и бекапируем в ноду 4

sudo -u postgres pg_basebackup -p 5434 -R -D /var/lib/postgresql/14/node4

Плюс в конфиге включаю параметр hot_standby чтобы реплика принимала запросы на чтение на кластере node4

-- Старт node 4
root@ubuntu:/etc/postgresql/14/node4# pg_ctlcluster 14 node4 start

----- Тестируем физическую реплику -------

postgres@ubuntu:/etc/postgresql/14/node4$ psql -p 5435

postgres=# \l repa 
                             List of databases
 Name |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges 
------+----------+----------+-------------+-------------+-------------------
 repa | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
(1 row)

postgres=# \c repa 
You are now connected to database "repa" as user "postgres".

repa=# show hot_standby;
 hot_standby 
-------------
 on
(1 row)

repa=# \dt+
                                   List of relations
 Schema | Name  | Type  |  Owner   | Persistence | Access method | Size  | Description 
--------+-------+-------+----------+-------------+---------------+-------+-------------
 public | test  | table | postgres | permanent   | heap          | 16 kB | 
 public | test2 | table | postgres | permanent   | heap          | 16 kB | 
(2 rows)

repa=# select * from test;
 id |  name   
----+---------
  1 | Value 1
  2 | Value 2
  3 | Value 3
  4 | Value 4
(4 rows)

repa=# select * from test2;
 id |    name    
----+------------
  1 | Значение 1
  2 | Значение 2
  3 | Значение 3
  4 | Значение 4
(4 rows)

Круто, все данные на месте

-- Теперь проверим, действительно ли реплика, работает как реплика
-- Попытаемся изменить данные на 4 node

repa=# insert into test values (5, 'Value 5');
ERROR:  cannot execute INSERT in a read-only transaction

Получаем типовую ошибку, которая говорит, уйди отсюда, на мастер раз хочешь что-то поменять..)

```
Спасибо за внимание!
Буду благодарен за ссылки на доп материалы для изучения.
Или может быть книги какие-то из своего опыта посоветуете.
