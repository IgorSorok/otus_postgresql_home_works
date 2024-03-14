**Создал ВМ**
```python
+----------------------+---------+---------------+---------+----------------+--------------+
|          ID          |  NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP  |
+----------------------+---------+---------------+---------+----------------+--------------+
| fhmfe07d8ikfbn6v7010 | sber-vm | ru-central1-a | RUNNING | 130.193.37.161 | 192.168.0.32 |
+----------------------+---------+---------------+---------+----------------+--------------+


root@ubuntu:/home/igor# yc compute instance create \
    --name sber-vm \
    --hostname sber-vm \
    --cores 2 \  ## 2 ядра
    --memory 4 \  ## 4GB ОЗУ
    --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \ ## SSD 10GB
    --network-interface subnet-name=sber-subnet,nat-ip-version=ipv4 \
    --ssh-key ~/.ssh/yc_key.pub \

```
**Развернул кластер PostgreSQL 15 с дефолтными настройками.**
```python
yc-user@sber-vm:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

**Запускаю тест с дефолтными настройками**
```python
pgbench -c8 -P 6 -T 60 -U postgres postgres

#С8 - число одновременных сеансов БД
#P 6 - вывод отчёта о прогрессе через заданное число секунд
#T 60 - выполнение теста с ограничением по времени

postgres@sber-vm:/home/yc-user$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
starting vacuum...end.
progress: 6.0 s, 713.5 tps, lat 11.135 ms stddev 8.568, 0 failed
progress: 12.0 s, 765.0 tps, lat 10.427 ms stddev 7.959, 0 failed
progress: 18.0 s, 383.8 tps, lat 20.800 ms stddev 21.270, 0 failed
progress: 24.0 s, 765.2 tps, lat 10.431 ms stddev 8.264, 0 failed
progress: 30.0 s, 667.2 tps, lat 11.967 ms stddev 10.566, 0 failed
progress: 36.0 s, 735.8 tps, lat 10.845 ms stddev 9.271, 0 failed
progress: 42.0 s, 772.5 tps, lat 10.323 ms stddev 7.716, 0 failed
progress: 48.0 s, 437.2 tps, lat 18.270 ms stddev 18.449, 0 failed
progress: 54.0 s, 724.2 tps, lat 11.013 ms stddev 8.377, 0 failed
progress: 60.0 s, 699.8 tps, lat 11.405 ms stddev 9.394, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 39993
number of failed transactions: 0 (0.000%)
latency average = 11.972 ms
latency stddev = 11.141 ms
initial connection time = 18.484 ms
tps = 666.460572 (without initial connection time)
```

**Настраиваю кластер под рекомендованные параметры, для увеличения производительности.**
```python
Все параметры изменял в конфигурационном файле
etc/postgresql/15/main/postgresql.conf

сделал релоад

Перехожу в лог, тут видно, что некоторые параметры требуют рестарта

2024-03-14 08:15:43.830 UTC [585] LOG:  received SIGHUP, reloading configuration files
2024-03-14 08:15:43.830 UTC [585] LOG:  parameter "max_connections" cannot be changed without restarting the server
2024-03-14 08:15:43.830 UTC [585] LOG:  parameter "shared_buffers" cannot be changed without restarting the server
2024-03-14 08:15:43.830 UTC [585] LOG:  parameter "work_mem" changed to "6553kB"
2024-03-14 08:15:43.830 UTC [585] LOG:  parameter "maintenance_work_mem" changed to "512MB"
2024-03-14 08:15:43.830 UTC [585] LOG:  parameter "effective_io_concurrency" changed to "2"
2024-03-14 08:15:43.830 UTC [585] LOG:  parameter "wal_buffers" cannot be changed without restarting the server
2024-03-14 08:15:43.830 UTC [585] LOG:  parameter "max_wal_size" changed to "16GB"
2024-03-14 08:15:43.830 UTC [585] LOG:  parameter "min_wal_size" changed to "4GB"
2024-03-14 08:15:43.830 UTC [585] LOG:  parameter "effective_cache_size" changed to "3GB"
2024-03-14 08:15:43.830 UTC [585] LOG:  parameter "default_statistics_target" changed to "500"

Рестартую кластер:
yc-user@sber-vm:~$ sudo systemctl restart postgresql@15-main

Теперь проверка:
postgres@sber-vm:/home/yc-user$ psql
psql (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
Type "help" for help.

postgres=# show max_connections ;
 max_connections 
-----------------
 40
(1 row)

postgres=# show shared_buffers ;
 shared_buffers 
----------------
 1GB
(1 row)

postgres=# show effective_cache_size ;
 effective_cache_size 
----------------------
 3GB
(1 row)

postgres=# show maintenance_work_mem ;
 maintenance_work_mem 
----------------------
 512MB
(1 row)

postgres=# show checkpoint_completion_target ;
 checkpoint_completion_target 
------------------------------
 0.9
(1 row)

postgres=# show wal_buffers ;
 wal_buffers 
-------------
 16MB
(1 row)

postgres=# show default_statistics_target ;
 default_statistics_target 
---------------------------
 500
(1 row)

postgres=# show random_page_cost ;
 random_page_cost 
------------------
 4
(1 row)

postgres=# show effective_io_concurrency ;
 effective_io_concurrency 
--------------------------
 2
(1 row)

postgres=# show work_mem ;
 work_mem 
----------
 6553kB
(1 row)

postgres=# show min_wal_size ;
 min_wal_size 
--------------
 4GB
(1 row)

postgres=# show max_wal_size ;
 max_wal_size 
--------------
 16GB
(1 row)
```

**Повторно запускаю тест**

```python
postgres@sber-vm:/home/yc-user$ pgbench -c8 -P 6 -T 60 -U postgres postgres

pgbench (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
starting vacuum...end.
progress: 6.0 s, 400.3 tps, lat 19.871 ms stddev 20.286, 0 failed
progress: 12.0 s, 792.2 tps, lat 10.066 ms stddev 7.793, 0 failed
progress: 18.0 s, 647.7 tps, lat 12.322 ms stddev 10.605, 0 failed
progress: 24.0 s, 759.0 tps, lat 10.513 ms stddev 7.777, 0 failed
progress: 30.0 s, 771.0 tps, lat 10.347 ms stddev 8.196, 0 failed
progress: 36.0 s, 482.0 tps, lat 16.561 ms stddev 18.003, 0 failed
progress: 42.0 s, 761.7 tps, lat 10.470 ms stddev 8.099, 0 failed
progress: 48.0 s, 722.2 tps, lat 11.049 ms stddev 8.933, 0 failed
progress: 54.0 s, 752.3 tps, lat 10.609 ms stddev 8.347, 0 failed
progress: 60.0 s, 709.0 tps, lat 11.248 ms stddev 8.262, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 40792
number of failed transactions: 0 (0.000%)
latency average = 11.735 ms
latency stddev = 10.804 ms
initial connection time = 19.653 ms
tps = 679.862848 (without initial connection time)
```
**Что изменилось?**
```python
В моём случае изменились:

Тест с дефолтными настройками кластера:
number of transactions actually processed: 39993
number of failed transactions: 0 (0.000%)
latency average = 11.972 ms
latency stddev = 11.141 ms
initial connection time = 18.484 ms
tps = 666.460572 (without initial connection time)

Тест с настроенным кластером:
number of transactions actually processed: 40792
number of failed transactions: 0 (0.000%)
latency average = 11.735 ms
latency stddev = 10.804 ms
initial connection time = 19.653 ms
tps = 679.862848 (without initial connection time)

Видно, что:
1. Увеличилось кол-во обработанных транзакций (number of transactions actually processed)
2. Немного уменьшилось время средней задержки (latency average)
3. Уменьшилось время отклика (latency stddev) ## Надеюсь я правильно это понимаю, если нет, то прошу меня поправить.
4. Увеличилось время начального соединения (initial connection time ) ## Хорошо это или плохо, не могу понять, просьба так же расшифровать
5. Увеличилась кол-во транзакций обработанных за секунду(tps)

На основе этих показателей, могу сделать вывод, что производительность кластера выросла.

Почему это произошло?
Однозначного ответа дать не могу. Понятно что увеличение производительности кластера вызвано оптимизацией настроек.
Но свести концы с концами немогу. Буду так же признателен за обратную свзяь по этому вопросу.

```

**Создаю таблицу и наполняю данными**
```python
postgres=# create table testik(c1 text);
CREATE TABLE
Time: 724,967 ms

postgres=# INSERT INTO testik(c1) SELECT 'privet' FROM generate_series(1,1000000);
INSERT 0 1000000
Time: 3687,043 ms (00:03,687)

postgres=# \dt
         List of relations
 Schema |  Name  | Type  |  Owner   
--------+--------+-------+----------
 public | testik | table | postgres

```
**Апдейт всех строк** 
```python
--Делаю пять апдейтов

postgres=# update testik set c1=c1||'f';
UPDATE 1000000
Time: 6023,624 ms (00:06,024)

postgres=# update testik set c1=c1||'!';
UPDATE 1000000
Time: 6727,234 ms (00:06,727)

postgres=# update testik set c1=c1||'u';
UPDATE 1000000
Time: 6261,103 ms (00:06,261)

postgres=# update testik set c1=c1||'&';
UPDATE 1000000
Time: 7151,069 ms (00:07,151)

postgres=# update testik set c1=c1||'M';
UPDATE 1000000
Time: 7078,692 ms (00:07,079)


Проверяю кол-во мертвых строк и время когда заходил в гости автовакуум:
Видно, что мертвых строк много, но автовакуум по всей видимости работает.

postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'testik';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 testik  |    1000000 |    1999853 |    199 | 2024-03-14 18:19:29.015117+00
(1 row)

-- Подождал пару минут

postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'testik';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 testik  |    1000000 |          0 |      0 | 2024-03-14 18:20:17.076148+00
(1 row)

Автовакуум отработал
```

**Снова делаю апдейт и смотрю размер таблицы**

```python

postgres=# update testik set c1=c1||'5';
UPDATE 1000000
Time: 4620,054 ms (00:04,620)

postgres=# update testik set c1=c1||'lol';
UPDATE 1000000
Time: 5051,854 ms (00:05,052)

postgres=# update testik set c1=c1||'umpalump';
UPDATE 1000000
Time: 6026,897 ms (00:06,027)

postgres=# update testik set c1=c1||'>';
UPDATE 1000000
Time: 5615,508 ms (00:05,616)

postgres=# update testik set c1=c1||'1';
UPDATE 1000000
Time: 8477,683 ms (00:08,478)

- Размер таблицы

postgres=# select pg_size_pretty(pg_total_relation_size('testik'));
 pg_size_pretty 
----------------
 207 MB
(1 row)

```
**Отключаю автоваккуум**
```python
postgres=# alter table testik set (autovacuum_enabled = off);
ALTER TABLE

--Делаю апдейт еще 10 раз
postgres=# update testik set c1=c1||'123';
UPDATE 1000000

postgres=# update testik set c1=c1||'12';
UPDATE 1000000

postgres=# update testik set c1=c1||'fuh';
UPDATE 1000000

postgres=# update testik set c1=c1||'MMM';
UPDATE 1000000

postgres=# update testik set c1=c1||'ZXC';
UPDATE 1000000
postgres=# update testik set c1=c1||'vbn';
UPDATE 1000000

postgres=# update testik set c1=c1||'m,.';
UPDATE 1000000

postgres=# update testik set c1=c1||'asd';
UPDATE 1000000

postgres=# update testik set c1=c1||'fgh';
UPDATE 1000000

postgres=# update testik set c1=c1||'qwe';
UPDATE 1000000

Колличество мертвых строк зашкаливает..

 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 testik  |    1000000 |    9997029 |    999 | 2024-03-14 18:27:17.223784+00

Размер таблицы увеличивается более чем в три раза.

postgres=# select pg_size_pretty(pg_total_relation_size('testik'));
 pg_size_pretty 
----------------
 755 MB
(1 row)


Незнаю, что тут можно объяснить..
Автовакуум отключили, апдейты сделали, апдейты в свою очередь наплодили кучу мертвых строк, таблица увеличилась в размерах.
Вывод:
Если хочешь спать спокойно, не отключай autovacuum..

--включил вакууи обратно
postgres=# alter table testik set (autovacuum_enabled = on);
ALTER TABLE

```

**В условии ДЗ этого вроде как нет, но как исправить ситуацию??**
```python
Решил сделать фульник, вот резултат:

postgres=# vacuum full testik ;
VACUUM

postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'testik';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 testik  |    1000000 |          0 |      0 | 2024-03-14 19:51:36.572265+00
(1 row)

postgres=# select pg_size_pretty(pg_total_relation_size('testik'));
 pg_size_pretty 
----------------
 81 MB
(1 row)

```
**Задание со *:**
```python
  _DO_
  _$do$_
  _BEGIN_
    _FOR i IN 1..10 LOOP_
      _RAISE NOTICE 'Step = %', i;_
      _update tab1 set c1=c1||i;_
    _END LOOP;_
  _END_
  _$do$;_
```

Спасибо за внимание!
Большая просьба дать развернутый комментарий по моим вопросам с параметрами.
Еще раз, большое спасибо 🙂.

```python
Видно, что:
1. Увеличилось кол-во обработанных транзакций (number of transactions actually processed)
2. Немного уменьшилось время средней задержки (latency average)
3. Уменьшилось время отклика (latency stddev) ## Надеюсь я правильно это понимаю, если нет, то прошу меня поправить.
4. Увеличилось время начального соединения (initial connection time ) ## Хорошо это или плохо, не могу понять, просьба так же расшифровать
5. Увеличилась кол-во транзакций обработанных за секунду(tps)

На основе этих показателей, могу сделать вывод, что производительность кластера выросла.

Почему это произошло?
Однозначного ответа дать не могу. Понятно что увеличение производительности кластера вызвано оптимизацией настроек.
Но свести концы с концами немогу.
Буду так же признателен за обратную свзяь по этому вопросу.
```
