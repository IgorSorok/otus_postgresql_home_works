***Подготовительные процедуры***

```python
postgres@ubuntu:/home/igor$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

-- Создал БД

postgres=# \l locks 
                              List of databases
 Name  |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges 
-------+----------+----------+-------------+-------------+-------------------
 locks | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
(1 row)

-- Создал таблицу и наполнил данными

locks=# CREATE TABLE accounts(
  acc_no integer PRIMARY KEY,
  amount numeric
);
CREATE TABLE
Time: 319,286 ms

locks=# INSERT INTO accounts VALUES (1,1000.00), (2,2000.00), (3,3000.00), (4,4000.00), (5,5000.00), (6,6000.00);
INSERT 0 6
Time: 297,460 ms

locks=# select * from accounts;
 acc_no | amount  
--------+---------
      1 | 1000.00
      2 | 2000.00
      3 | 3000.00
      4 | 4000.00
      5 | 5000.00
      6 | 6000.00
(6 rows)
Time: 0,268 ms

locks=# 

```

**Задание 1**

```python

Для того чтобы блокировки выводились в лог необходимо включить параметр log_lock_waits 
Таймаут же в свою очередь выставляется в параметре deadlock_timeout

locks=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM
Time: 301,876 ms

locks=# ALTER SYSTEM SET deadlock_timeout = '200ms';
ALTER SYSTEM
Time: 305,677 ms

-- Перечитка

locks=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)



-- Проверка

locks=# show log_lock_waits ;
 log_lock_waits 
----------------
 on
(1 row)
Time: 0,191 ms

locks=# show deadlock_timeout ;
 deadlock_timeout 
------------------
 200ms
(1 row)

-- Воспроизведем ситуацию

*Сессия 1
locks=# begin ;
BEGIN
Time: 0,186 ms

locks=*# SELECT pg_backend_pid();
 pg_backend_pid 
----------------
          20742
(1 row)
Time: 0,355 ms

locks=*# UPDATE accounts SET amount = 900.00 WHERE acc_no = 1;
UPDATE 1
Time: 0,241 ms

*Сессия 2

locks=# begin ;
BEGIN
Time: 0,170 ms

locks=*# SELECT pg_backend_pid();
 pg_backend_pid 
----------------
          19980
(1 row)
Time: 0,232 ms

locks=*# UPDATE accounts SET amount = 150.00 WHERE acc_no = 1;
*повисла

-- Смотрим в лог (сообщения выводятся)

2024-03-26 16:06:41.669 MSK [19980] postgres@locks LOG:  process 19980 still waiting for ShareLock on transaction 801 after 206.821 ms
2024-03-26 16:06:41.669 MSK [19980] postgres@locks DETAIL:  Process holding the lock: 20742. Wait queue: 19980.
2024-03-26 16:06:41.669 MSK [19980] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "accounts"
2024-03-26 16:06:41.669 MSK [19980] postgres@locks STATEMENT:  UPDATE accounts SET amount = 150.00 WHERE acc_no = 1;


```

***Задание 2***

```python
-- Cоздаю представление

locks=# CREATE VIEW locks_v AS
SELECT pid,
       locktype,
       CASE locktype
         WHEN 'relation' THEN relation::regclass::text
         WHEN 'transactionid' THEN transactionid::text
         WHEN 'tuple' THEN relation::regclass::text||':'||tuple::text
       END AS lockid,
       mode,
       granted
FROM pg_locks
WHERE locktype in ('relation','transactionid','tuple')
AND (locktype != 'relation' OR relation = 'accounts'::regclass);
CREATE VIEW
Time: 308,234 ms

-- Запускаю 1-ю транзакцию

locks=# BEGIN;
SELECT txid_current(), pg_backend_pid();
BEGIN
Time: 0,093 ms
 txid_current | pg_backend_pid 
--------------+----------------
          804 |          20742
(1 row)
Time: 0,148 ms

locks=*# UPDATE accounts SET amount = amount + 350.00 WHERE acc_no = 1;
UPDATE 1
Time: 0,222 ms

-- Запускаю 2-ю транзакцию

locks=# BEGIN;
SELECT txid_current(), pg_backend_pid();
BEGIN
Time: 0,097 ms
 txid_current | pg_backend_pid 
--------------+----------------
          805 |          19980
(1 row)
Time: 0,190 ms

locks=*# UPDATE accounts SET amount = amount + 1111.00 WHERE acc_no = 1;
*висит

-- Запускаю 3-ю сессию

locks=# BEGIN;
SELECT txid_current(), pg_backend_pid();
BEGIN
Time: 0,095 ms
 txid_current | pg_backend_pid 
--------------+----------------
          806 |          21283
(1 row)
Time: 0,257 ms

locks=*# UPDATE accounts SET amount = amount - 120.00 WHERE acc_no = 1;
*также висит

-- Смотрим дерево (тут видно, кто кого блокирует и в каком режиме)

locks=*# SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for
FROM pg_locks WHERE relation = 'accounts'::regclass;

 locktype |       mode       | granted |  pid  | wait_for 
----------+------------------+---------+-------+----------
 relation | RowExclusiveLock | t       | 21283 | {19980}
 relation | RowExclusiveLock | t       | 20742 | {}
 relation | RowExclusiveLock | t       | 19980 | {20742}
 tuple    | ExclusiveLock    | f       | 21283 | {19980}
 tuple    | ExclusiveLock    | t       | 19980 | {20742}
(5 rows)
Time: 0,347 ms

-- Блокировки для первой сессии 20742

locks=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
  FROM pg_locks 
  WHERE pid = 20742;
   locktype    |   relation    | virtxid | xid |       mode       | granted 
---------------+---------------+---------+-----+------------------+---------
 relation      | pg_locks      |         |     | AccessShareLock  | t
 relation      | accounts_pkey |         |     | RowExclusiveLock | t
 relation      | accounts      |         |     | RowExclusiveLock | t
 virtualxid    |               | 4/803   |     | ExclusiveLock    | t
 transactionid |               |         | 808 | ExclusiveLock    | t
(5 rows)

relation для pg_locks и locks в режиме AccessShareLock — устанавливаются на чтение. (Классическая блокировка при SELECT из таблицы.)
relation для таблицы accounts и ее первичном ключе accounts_pkey в режиме RowExclusiveLock — устанавливается на изменение. (Блокировка апдейт)
virtualxid и transactionid в режиме ExclusiveLock — удерживаются каждой транзакцией для самой себя. transactionid показывает что сессия начала менять данные


-- Блокировки для второй сессии 19980
locks=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
  FROM pg_locks 
  WHERE pid = 19980;

   locktype    |   relation    | virtxid | xid |       mode       | granted 
---------------+---------------+---------+-----+------------------+---------
 relation      | accounts_pkey |         |     | RowExclusiveLock | t
 relation      | accounts      |         |     | RowExclusiveLock | t
 virtualxid    |               | 5/192   |     | ExclusiveLock    | t
 transactionid |               |         | 809 | ExclusiveLock    | t
 tuple         | accounts      |         |     | ExclusiveLock    | t
 transactionid |               |         | 808 | ShareLock        | f
(6 rows)
Time: 0,472 ms



- RowExclusiveLock в таблице accounts и ее первичном ключе accounts_pkey. Это блокировка UPDATE.
- ExclusiveLock на виртуальном идентификаторе транзакции (virtualxid), которая пока не меняет данные (virtxid=5/192). 
  В нашем случае она есть т.к. мы начали транзакцию через BEGIN
- ShareLock. Транзакция пытается наложить блокировку ShareLock на xid 809, но не может granted - false
- ExclusiveLock (tuple) возникает, когда несколько сессий пытаются изменить одни и те же данные.
Чтобы не ссылаться на одни и те же идентификаторы транзакций, формируется некий общий tuple с блокровкой на таблицу.
- ExclusiveLock исключительная блокировка настоящего номера транзакции (transactionid), которая начала менять данные (xid 808).
- Блокировки для pg_locks и locks отсутствуют, т.к вторая транзакция к ним не обращалась .


-- Блокировки для третьей сессии в целом аналогичная ситуация исходя первых двух (не прикладываю скрин, т.к вылетела виртуалка и всё пропало)

```

***Задание 3***

```python
-- Первая сессия

locks=# begin ;
BEGIN
Time: 0,228 ms

locks=*# select pg_backend_pid();
 pg_backend_pid 
----------------
           4456
(1 row)
Time: 0,336 ms

locks=*# UPDATE accounts 
SET amount = amount + 150.00 WHERE acc_no = 1;
UPDATE 1
Time: 0,507 ms

-- Вторая сессия

locks=# begin ;
BEGIN
Time: 0,096 ms

locks=*# select pg_backend_pid();
 pg_backend_pid 
----------------
           4482
(1 row)
Time: 0,213 ms

locks=*# UPDATE accounts 
SET amount = amount + 200.00 WHERE acc_no = 2;
UPDATE 1
Time: 0,476 ms

-- Третья сессия

locks=# begin;
BEGIN
Time: 0,280 ms

locks=*# select pg_backend_pid();
 pg_backend_pid 
----------------
           4519
(1 row)
Time: 0,296 ms

locks=*# UPDATE accounts 
SET amount = amount + 300.00 WHERE acc_no = 3;
UPDATE 1
Time: 0,350 ms

Новый круг

-- Первая сессия

locks=*# UPDATE accounts 
SET amount = amount + 100.00 WHERE acc_no = 3;
повисла

-- Вторая

locks=*# UPDATE accounts 
SET amount = amount + 200.00 WHERE acc_no = 1;
повисла

-- Третья

locks=*# UPDATE accounts 
SET amount = amount + 300.00 WHERE acc_no = 2;

Появляется ошибка

ERROR:  deadlock detected
DETAIL:  Process 4519 waits for ShareLock on transaction 821; blocked by process 4482.
Process 4482 waits for ShareLock on transaction 820; blocked by process 4456.
Process 4456 waits for ShareLock on transaction 822; blocked by process 4519.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,2) in relation "accounts"
Time: 204,503 ms

-- Выдержка из лога

2024-03-27 05:32:28.284 MSK [4519] postgres@locks DETAIL:  Process holding the lock: 4482. Wait queue: .
2024-03-27 05:32:28.284 MSK [4519] postgres@locks CONTEXT:  while updating tuple (0,2) in relation "accounts"
2024-03-27 05:32:28.284 MSK [4519] postgres@locks STATEMENT:  UPDATE accounts 
        SET amount = amount + 300.00 WHERE acc_no = 2;
2024-03-27 05:32:28.284 MSK [4519] postgres@locks ERROR:  deadlock detected
2024-03-27 05:32:28.284 MSK [4519] postgres@locks DETAIL:  Process 4519 waits for ShareLock on transaction 821; blocked by process 4482.
        Process 4482 waits for ShareLock on transaction 820; blocked by process 4456.
        Process 4456 waits for ShareLock on transaction 822; blocked by process 4519.
        Process 4519: UPDATE accounts 
        SET amount = amount + 300.00 WHERE acc_no = 2;
        Process 4482: UPDATE accounts 
        SET amount = amount + 200.00 WHERE acc_no = 1;
        Process 4456: UPDATE accounts 
        SET amount = amount + 100.00 WHERE acc_no = 3;
2024-03-27 05:32:28.284 MSK [4519] postgres@locks HINT:  See server log for query details.
2024-03-27 05:32:28.284 MSK [4519] postgres@locks CONTEXT:  while updating tuple (0,2) in relation "accounts"
2024-03-27 05:32:28.284 MSK [4519] postgres@locks STATEMENT:  UPDATE accounts 
        SET amount = amount + 300.00 WHERE acc_no = 2;
2024-03-27 05:32:28.284 MSK [4456] postgres@locks LOG:  process 4456 acquired ShareLock on transaction 822 after 43593.647 ms
2024-03-27 05:32:28.284 MSK [4456] postgres@locks CONTEXT:  while updating tuple (0,3) in relation "accounts"
2024-03-27 05:32:28.284 MSK [4456] postgres@locks STATEMENT:  UPDATE accounts 
        SET amount = amount + 100.00 WHERE acc_no = 3;
		
Вывод: 
Благодаря тому, что мы включили логирование блокировок, стало можно разбираться в подобных ситуациях, не имеяя других средств для мониторинга 
(возможно пишу не связно, но у меня ночные смены).

```

***Задание 4***

```python
Две транзакции выполняющие UPDATE одной и той же таблицы без where, могут заблокировать друг друга.
Как это воспроизвести, я незнаю))
Думаю, что в жизни такое случается не часто..

```

Спасибо за внимание, я скоро спать 😄
