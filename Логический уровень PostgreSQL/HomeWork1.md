создайте новый кластер PostgresSQL 14
зайдите в созданный кластер под пользователем postgres
создайте новую базу данных testdb
зайдите в созданную базу данных под пользователем postgres
создайте новую схему testnm
создайте новую таблицу t1 с одной колонкой c1 типа integer
вставьте строку со значением c1=1
создайте новую роль readonly
дайте новой роли право на подключение к базе данных testdb
дайте новой роли право на использование схемы testnm
дайте новой роли право на select для всех таблиц схемы testnm
создайте пользователя testread с паролем test123
дайте роль readonly пользователю testread
зайдите под пользователем testread в базу данных testdb
сделайте select * from t1;
получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
напишите что именно произошло в тексте домашнего задания
у вас есть идеи почему? ведь права то дали?
посмотрите на список таблиц
подсказка в шпаргалке под пунктом 20
а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)
вернитесь в базу данных testdb под пользователем postgres
удалите таблицу t1
создайте ее заново но уже с явным указанием имени схемы testnm
вставьте строку со значением c1=1
зайдите под пользователем testread в базу данных testdb
сделайте select * from testnm.t1;
получилось?
есть идеи почему? если нет - смотрите шпаргалку
как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку
сделайте select * from testnm.t1;
получилось?
есть идеи почему? если нет - смотрите шпаргалку
сделайте select * from testnm.t1;
получилось?
ура!
теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
есть идеи как убрать эти права? если нет - смотрите шпаргалку
если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
расскажите что получилось и почему

SET search_path TO testnm, public, "$user", pg_catalog, pg_temp;

postgres=# SET search_path TO testnm, public, "$user", pg_catalog, pg_temp;
SET
Time: 0,256 ms
postgres=# show search_path ;
                 search_path                  
----------------------------------------------
 testnm, public, "$user", pg_catalog, pg_temp
(1 row)
Time: 0,210 ms

testdb=# revoke CREATE on SCHEMA public from PUBLIC ;
REVOKE
Time: 291,170 ms

testdb=# REVOKE ALL ON DATABASE testdb FROM public;
REVOKE

testdb=# \dn+
                          List of schemas
  Name  |  Owner   |  Access privileges   |      Description       
--------+----------+----------------------+------------------------
 public | postgres | postgres=UC/postgres+| standard public schema
        |          | =U/postgres          | 
 testnm | postgres |                      | 
(2 rows)



-- Создаю таблицу
testdb=# CREATE TABLE t1(c1 integer);
CREATE TABLE
Time: 207,801 ms

testdb=# INSERT INTO t1 VALUES (1);
INSERT 0 1
Time: 309,945 ms

 -- Создаю роль 
 testdb=# CREATE ROLE readonly;

testdb=# \du readonly 
            List of roles
 Role name |  Attributes  | Member of 
-----------+--------------+----------
 readonly  | Cannot login | {}


-- Даю гранты

testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;  -- грант на подключение к БД
GRANT
Time: 309,310 ms

testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;  -- грант на использование схемы
GRANT
Time: 316,109 ms

testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly; -- грант на чтение всех таблиц в схеме существующих на данный момент
GRANT
Time: 297,975 ms

testdb=# ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly;   -- Задаем права для роли по умолчанию
ALTER DEFAULT PRIVILEGES
Time: 369,273 ms




-- Создаю юзера и даю ему грант роли

testdb=# CREATE USER testread WITH PASSWORD '111';
testdb=# GRANT readonly TO testread;
GRANT ROLE
Time: 0,202 ms

testdb=# \du testread 
            List of roles
 Role name | Attributes | Member of  
-----------+------------+------------
 testread  |            | {readonly}



-- Подключаюсь к базе под юзером и даю селект:)


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


-- удалять я ее конечно же не буду, просто сделаю новую с указанием схемы

testdb=# CREATE TABLE testnm.t2(c1 integer);
CREATE TABLE
Time: 339,805 ms

testdb=# INSERT INTO testnm.t2 VALUES (2);
INSERT 0 1
Time: 297,589 ms


-- Подключаемся под юзером и проверяем селект

testdb=# \c testdb testread 
Password for user testread: 
You are now connected to database "testdb" as user "testread".

testdb=> select * from testnm.t2 ;
 c1 
----
  2
(1 row)
Time: 0,371 ms


Всё работает)

Потому-что:
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

-- по юзером testread пробуем создать табличку

testdb=> CREATE TABLE t3(c1 integer);
ERROR:  permission denied for schema testnm
Time: 0,316 ms
 
 
 Никаких результатов, но так и должно быть.. 
 Ведь никто не давал грантов на создание таблиц для роли readonly
 
 
 Для меня эта тема очень сложная.. Слишком много нюансов..:(
 
 В качестве обратной связи хотелось бы получить:
 1. Рекомендации по настройке search_path
 2. По возможности доплнительных материалов для изучения, на тему работы с пользователями и схемами.
 Спасибо:)

