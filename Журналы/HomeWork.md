***Создал VM***

```python
+----------------------+---------+---------------+---------+---------------+-------------+
|          ID          |  NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP  | INTERNAL IP |
+----------------------+---------+---------------+---------+---------------+-------------+
| fhmd8tot0ftuoqbfs31n | sber-vm | ru-central1-a | RUNNING | 51.250.95.155 | 192.168.0.9 |
+----------------------+---------+---------------+---------+---------------+-------------+
```

***Развернул кластер PostgreSQL 14***

```python
yc-user@sber-vm:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

```

***Настройка выполнения контрольной точки***

```python

-- По дефолту стоит это значение:
postgres=# show checkpoint_timeout ;
 checkpoint_timeout 
--------------------
 5min
(1 row)




-- Поменял параметр в конфиге postgresql.conf
-- Сделал перечитку конфига
-- Теперь значение парметра выглядит так:

postgres=# show checkpoint_timeout ;

 checkpoint_timeout 
--------------------
 30s
(1 row)


```

***Создаю новую базу test и запускаю pgbench***

```python
postgres=# create database test;
CREATE DATABASE

postgres=# \l test 
                             List of databases
 Name |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges 
------+----------+----------+-------------+-------------+-------------------
 test | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
(1 row)


-- В базе смотрю текущее положение LSN
test=# SELECT pg_current_wal_lsn(), pg_current_wal_insert_lsn(), pg_walfile_name(pg_current_wal_lsn()) file_current_wal_lsn, pg_walfile_name(pg_current_wal_insert_lsn()) file_current_wal_insert_lsn;

 pg_current_wal_lsn | pg_current_wal_insert_lsn |   file_current_wal_lsn   | file_current_wal_insert_lsn 
--------------------+---------------------------+--------------------------+-----------------------------
 0/16FB518          | 0/16FB518                 | 000000010000000000000001 | 000000010000000000000001
(1 row)


-- запускаю инициализацию pgbench

postgres@sber-vm:/etc/postgresql/14/main$ pgbench -i test

dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.09 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.22 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.88 s, vacuum 0.07 s, primary keys 0.25 s).

-- Теперь запускаю сам тест

postgres@sber-vm:/etc/postgresql/14/main$ pgbench -P 30 -T 600 -c 10 test
pgbench (14.11 (Ubuntu 14.11-1.pgdg20.04+1))

starting vacuum...end.
progress: 30.0 s, 378.1 tps, lat 26.369 ms stddev 27.296
progress: 60.0 s, 362.2 tps, lat 27.562 ms stddev 25.370
progress: 90.0 s, 405.2 tps, lat 24.634 ms stddev 24.526
progress: 120.0 s, 430.7 tps, lat 23.186 ms stddev 23.703
progress: 150.0 s, 375.3 tps, lat 26.607 ms stddev 26.661
progress: 180.0 s, 403.8 tps, lat 24.723 ms stddev 26.173
progress: 210.0 s, 425.6 tps, lat 23.452 ms stddev 22.025
progress: 240.0 s, 413.4 tps, lat 24.155 ms stddev 27.001
progress: 270.0 s, 435.6 tps, lat 22.900 ms stddev 23.967
progress: 300.0 s, 379.2 tps, lat 26.349 ms stddev 27.739
progress: 330.0 s, 355.3 tps, lat 28.102 ms stddev 28.494
progress: 360.0 s, 340.9 tps, lat 29.287 ms stddev 31.693
progress: 390.0 s, 337.8 tps, lat 29.582 ms stddev 27.445
progress: 420.0 s, 399.2 tps, lat 25.009 ms stddev 25.472
progress: 450.0 s, 377.1 tps, lat 26.472 ms stddev 27.736
progress: 480.0 s, 322.0 tps, lat 31.013 ms stddev 31.602
progress: 510.0 s, 389.9 tps, lat 25.615 ms stddev 25.819
progress: 540.0 s, 413.8 tps, lat 24.130 ms stddev 24.694
progress: 570.0 s, 391.4 tps, lat 25.506 ms stddev 25.822
progress: 600.0 s, 405.8 tps, lat 24.602 ms stddev 24.475
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
duration: 600 s
number of transactions actually processed: 232282
latency average = 25.791 ms
latency stddev = 26.411 ms
initial connection time = 26.482 ms
tps = 387.125637 (without initial connection time)

-- Положение LSN после теста

 pg_current_wal_lsn | pg_current_wal_insert_lsn |   file_current_wal_lsn   | file_current_wal_insert_lsn 
--------------------+---------------------------+--------------------------+-----------------------------
 0/1B3E0478         | 0/1B3E0478                | 00000001000000000000001B | 00000001000000000000001B
(1 row)


```

***Смотрю какой объем был сгенерирован*** 

```python

test=# SELECT pg_size_pretty('0/1B3E0478'::pg_lsn - '0/16FB518'::pg_lsn) wal_size;
 wal_size 
----------
 413 MB
(1 row)

Получается, что на одну точку приходится 20.65 MB (413/600*30)

-- В целом checkpoint срабатывал по расписанию

test=# select * from pg_stat_bgwriter\gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 162
checkpoints_req       | 2
checkpoint_write_time | 590247
checkpoint_sync_time  | 419
buffers_checkpoint    | 41932
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 3351
buffers_backend_fsync | 0
buffers_alloc         | 4102
stats_reset           | 2024-03-17 20:30:05.250032+00

```


***Настройка кластера в синхронный режим и запуск теста!***

```python
-- В конфиге поменял параметры и сделал перечитку

postgres=# show synchronous_commit ;

 synchronous_commit 
--------------------
 on
(1 row)

postgres=# show commit_delay ;
 commit_delay 
--------------
 0
(1 row)

postgres=# show commit_siblings ;
 commit_siblings 
-----------------
 5
(1 row)

-- Запускаю собственно быстренький тест

yc-user@sber-vm:~$ sudo -u postgres pgbench -P 1 -T 10 test
pgbench (14.11 (Ubuntu 14.11-1.pgdg20.04+1))
starting vacuum...end.
progress: 1.0 s, 333.0 tps, lat 2.977 ms stddev 1.960
progress: 2.0 s, 315.0 tps, lat 3.182 ms stddev 2.594
progress: 3.0 s, 305.0 tps, lat 3.278 ms stddev 2.574
progress: 4.0 s, 189.0 tps, lat 5.290 ms stddev 3.609
progress: 5.0 s, 74.0 tps, lat 13.407 ms stddev 9.689
progress: 6.0 s, 62.0 tps, lat 15.879 ms stddev 8.540
progress: 7.0 s, 107.0 tps, lat 9.495 ms stddev 5.570
progress: 8.0 s, 283.0 tps, lat 3.561 ms stddev 2.985
progress: 9.0 s, 232.0 tps, lat 4.296 ms stddev 3.842
progress: 10.0 s, 231.0 tps, lat 4.345 ms stddev 4.156
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 10 s
number of transactions actually processed: 2131
latency average = 4.690 ms
latency stddev = 4.904 ms
initial connection time = 4.111 ms
tps = 213.177725 (without initial connection time)

```

***Настройка кластера в асинхронный режим и тест***

```python

-- Применил параметры

postgres=# show synchronous_commit ;
 synchronous_commit 
--------------------
 off
(1 row)

postgres=# show wal_writer_delay ;
 wal_writer_delay 
------------------
 200ms
(1 row)

yc-user@sber-vm:/etc/postgresql/14/main$ sudo -u postgres pgbench -P 1 -T 10 test
pgbench (14.11 (Ubuntu 14.11-1.pgdg20.04+1))
starting vacuum...end.
progress: 1.0 s, 1587.0 tps, lat 0.627 ms stddev 0.052
progress: 2.0 s, 1612.0 tps, lat 0.620 ms stddev 0.082
progress: 3.0 s, 1641.9 tps, lat 0.609 ms stddev 0.031
progress: 4.0 s, 1646.1 tps, lat 0.607 ms stddev 0.032
progress: 5.0 s, 1661.0 tps, lat 0.602 ms stddev 0.032
progress: 6.0 s, 1648.0 tps, lat 0.607 ms stddev 0.030
progress: 7.0 s, 1666.9 tps, lat 0.600 ms stddev 0.030
progress: 8.0 s, 1652.9 tps, lat 0.604 ms stddev 0.032
progress: 9.0 s, 1646.0 tps, lat 0.607 ms stddev 0.038
progress: 10.0 s, 1640.1 tps, lat 0.610 ms stddev 0.051
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 10 s
number of transactions actually processed: 16403
latency average = 0.609 ms
latency stddev = 0.044 ms
initial connection time = 3.841 ms
tps = 1640.813575 (without initial connection time)

```

***Сравнение показателей***

```python
-- Тест 1
number of transactions actually processed: 2131
latency average = 4.690 ms
latency stddev = 4.904 ms
initial connection time = 4.111 ms
tps = 213.177725 (without initial connection time)

-- Тест 2

number of transactions actually processed: 16403
latency average = 0.609 ms
latency stddev = 0.044 ms
initial connection time = 3.841 ms
tps = 1640.813575 (without initial connection time)


  * Результат:
    * Количество транзакций в секунду (tps), увеличилось более чем в 7 раз с 213.177725  до 1640.813575
    * Уменьшилось среднее время задержки (latency average) с 4.690 ms до latency average = 0.609 ms
    * Уменьшилось время задержки с дисковым устройством (latency stddev) с 4.904 ms до 0.044 ms
    * За любую сверхпроизводительность, приходится платить надежностью, такова правда жизни, к сожалению.
	* В целом считаю, что асинхронный режим имеет место быть, особенно в тестовых средах. На проде всё таки лучше не использовать
```

***Создание нового кластера с включенной контрольной суммой страниц***

```python
-- Разворачиваю кластер
root@ubuntu:/home/igor# pg_createcluster 14 my_cluster -- --data-checksums
Creating new PostgreSQL cluster 14/test ...
/usr/lib/postgresql/14/bin/initdb -D /var/lib/postgresql/14/test --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

root@ubuntu:/home/igor# pg_lsclusters 

Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
14  my      5433 online postgres /var/lib/postgresql/14/my   /var/log/postgresql/postgresql-14-my.log

-- Проверка парматера
postgres=# show data_checksums ;
 data_checksums 
----------------
 on
(1 row)
Time: 0,201 ms

-- Создал БД
postgres=# create database otus;
CREATE DATABASE
Time: 1267,733 ms (00:01,268)

postgres=# \l otus 
                             List of databases
 Name |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges 
------+----------+----------+-------------+-------------+-------------------
 otus | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
(1 row)

-- Создал табличку и наполнил данными

postgres=# \c otus 
You are now connected to database "otus" as user "postgres".
otus=# CREATE TABLE test_text(t text);
CREATE TABLE

otus=# insert into test_text select 'нестрока '||s.id from generate_series(1, 500) as s(id); 
INSERT 0 500
Time: 3,545 ms

otus=# select * from test_text limit 5;
     t      
------------
 нестрока 1
 нестрока 2
 нестрока 3
 нестрока 4
 нестрока 5
(5 rows)

-- Где лежит таблица

otus=# select pg_relation_filepath('test_text');
 pg_relation_filepath 
----------------------
 base/16384/16395
(1 row)
Time: 0,432 ms

dd if=/dev/zero of=/var/lib/postgresql/14/my/base/16384/16395 oflag=dsync conv=notrunc bs=1 count=8

-- Стопаю кластер
root@ubuntu:/home/igor# pg_ctlcluster 14 my stop
root@ubuntu:/home/igor# pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
14  my      5433 down   postgres /var/lib/postgresql/14/my   /var/log/postgresql/postgresql-14-my.log

-- Совершаю диверсию

root@ubuntu:/home/igor# dd if=/dev/zero of=/var/lib/postgresql/14/my/base/16384/16395 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0,0559798 s, 0,1 kB/s

-- Стартую кластер

root@ubuntu:/home/igor# pg_ctlcluster 14 my start
root@ubuntu:/home/igor# pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
14  my      5433 online postgres /var/lib/postgresql/14/my   /var/log/postgresql/postgresql-14-my.log

-- делаю выборку из таблицы

postgres=# \c otus ;
You are now connected to database "otus" as user "postgres".
otus=# select * from test_text limit 5;
WARNING:  page verification failed, calculated checksum 56313 but expected 9582
ERROR:  invalid page in block 0 of relation base/16384/16395

Поздравлю, вы научили меня ломать ПРОМ.. 😄

Ну а если серьезно, то:
- При обращении к таблице обнаружились поврежденные данные

Попробуем исправить:

-- В конфиге прописываю и прменяю парамтер

otus=# show ignore_checksum_failure ;
 ignore_checksum_failure 
-------------------------
 on
(1 row)

-- Теперь пробуем снова обратиться к таблице

postgres@ubuntu:/etc/postgresql/14/my$ psql -p 5433
Timing is on.
psql (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# \c otus 
You are now connected to database "otus" as user "postgres".
otus=# select * from test_text limit 5;
WARNING:  page verification failed, calculated checksum 56313 but expected 9582

     t      
------------
 нестрока 1
 нестрока 2
 нестрока 3
 нестрока 4
 нестрока 5
(5 rows)


Всё работает, но не понятна корректность данных, нужна проверка. 
Работать с включенным ignore_checksum_failure - плохая практика.

```


Спасибо за внимание. Буду благодарен, если подскажете, как можно полностью исправить эту ситуацию
WARNING:  page verification failed, calculated checksum 56313 but expected 9582
И восстановить данные.  Сам не очень разобрался.
