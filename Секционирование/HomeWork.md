***Установил БД***
```python

demo=# \l demo 
                             List of databases
 Name |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges 
------+----------+----------+-------------+-------------+-------------------
 demo | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 | 
(1 row)


```

***Какой диапазон данных(mix, max) есть в таблице flights***

```python

demo=# select min(scheduled_departure),max(scheduled_departure) from bookings.flights;
          min           |          max           
------------------------+------------------------

 2016-08-15 02:45:00+03 | 2017-09-14 20:55:00+03
(1 строка)


```

***Создаем партицированную таблицу flights_new, с ключом по диапазону range по полю scheduled_departure и партиции***

```python
demo=# create table flights_new (like flights) partition by range (scheduled_departure);
CREATE TABLE

demo=# create table flights_new_2016_08 partition of flights_new for values from ('2016-08-01') to ('2016-09-01');
CREATE TABLE

demo=# create table flights_new_2016_09 partition of flights_new for values from ('2016-09-01') to ('2016-10-01');
CREATE TABLE

demo=# create table flights_new_2016_10 partition of flights_new for values from ('2016-10-01') to ('2016-11-01');
CREATE TABLE

demo=# create table flights_new_2016_11 partition of flights_new for values from ('2016-11-01') to ('2016-12-01');
CREATE TABLE

demo=# create table flights_new_2016_12 partition of flights_new for values from ('2016-12-01') to ('2017-01-01');
CREATE TABLE

demo=# create table flights_new_2017_01 partition of flights_new for values from ('2017-01-01') to ('2017-02-01');
CREATE TABLE

demo=# create table flights_new_2017_02 partition of flights_new for values from ('2017-02-01') to ('2017-03-01');
CREATE TABLE

demo=# create table flights_new_2017_03 partition of flights_new for values from ('2017-03-01') to ('2017-04-01');
CREATE TABLE

demo=# create table flights_new_2017_04 partition of flights_new for values from ('2017-04-01') to ('2017-05-01');
CREATE TABLE

demo=# create table flights_new_2017_05 partition of flights_new for values from ('2017-05-01') to ('2017-06-01');
CREATE TABLE

demo=# create table flights_new_2017_06 partition of flights_new for values from ('2017-06-01') to ('2017-07-01');
CREATE TABLE

demo=# create table flights_new_2017_07 partition of flights_new for values from ('2017-07-01') to ('2017-08-01');
CREATE TABLE

demo=# create table flights_new_2017_08 partition of flights_new for values from ('2017-08-01') to ('2017-09-01');
CREATE TABLE

demo=# create table flights_new_2017_09 partition of flights_new for values from ('2017-09-01') to ('2017-10-01');
CREATE TABLE
```

```python
-- Смотрим, что из всего этого получилось.

                                              Секционированная таблица "bookings.flights_new"
       Столбец       |           Тип            | Правило сортировки | Допустимость NULL | По умолчанию | Хранилище | Сжатие | Цель для статистики | Описание 
---------------------+--------------------------+--------------------+-------------------+--------------+-----------+--------+---------------------+----------
 flight_id           | integer                  |                    | not null          |              | plain     |        |                     | 
 flight_no           | character(6)             |                    | not null          |              | extended  |        |                     | 
 scheduled_departure | timestamp with time zone |                    | not null          |              | plain     |        |                     | 
 scheduled_arrival   | timestamp with time zone |                    | not null          |              | plain     |        |                     | 
 departure_airport   | character(3)             |                    | not null          |              | extended  |        |                     | 
 arrival_airport     | character(3)             |                    | not null          |              | extended  |        |                     | 
 status              | character varying(20)    |                    | not null          |              | extended  |        |                     | 
 aircraft_code       | character(3)             |                    | not null          |              | extended  |        |                     | 
 actual_departure    | timestamp with time zone |                    |                   |              | plain     |        |                     | 
 actual_arrival      | timestamp with time zone |                    |                   |              | plain     |        |                     | 
Ключ разбиения: RANGE (scheduled_departure)
Секции: flights_new_2016_08 FOR VALUES FROM ('2016-08-01 00:00:00+03') TO ('2016-09-01 00:00:00+03'),
        flights_new_2016_09 FOR VALUES FROM ('2016-09-01 00:00:00+03') TO ('2016-10-01 00:00:00+03'),
        flights_new_2016_10 FOR VALUES FROM ('2016-10-01 00:00:00+03') TO ('2016-11-01 00:00:00+03'),
        flights_new_2016_11 FOR VALUES FROM ('2016-11-01 00:00:00+03') TO ('2016-12-01 00:00:00+03'),
        flights_new_2016_12 FOR VALUES FROM ('2016-12-01 00:00:00+03') TO ('2017-01-01 00:00:00+03'),
        flights_new_2017_01 FOR VALUES FROM ('2017-01-01 00:00:00+03') TO ('2017-02-01 00:00:00+03'),
        flights_new_2017_02 FOR VALUES FROM ('2017-02-01 00:00:00+03') TO ('2017-03-01 00:00:00+03'),
        flights_new_2017_03 FOR VALUES FROM ('2017-03-01 00:00:00+03') TO ('2017-04-01 00:00:00+03'),
        flights_new_2017_04 FOR VALUES FROM ('2017-04-01 00:00:00+03') TO ('2017-05-01 00:00:00+03'),
        flights_new_2017_05 FOR VALUES FROM ('2017-05-01 00:00:00+03') TO ('2017-06-01 00:00:00+03'),
        flights_new_2017_06 FOR VALUES FROM ('2017-06-01 00:00:00+03') TO ('2017-07-01 00:00:00+03'),
        flights_new_2017_07 FOR VALUES FROM ('2017-07-01 00:00:00+03') TO ('2017-08-01 00:00:00+03'),
        flights_new_2017_08 FOR VALUES FROM ('2017-08-01 00:00:00+03') TO ('2017-09-01 00:00:00+03'),
        flights_new_2017_09 FOR VALUES FROM ('2017-09-01 00:00:00+03') TO ('2017-10-01 00:00:00+03')
(END)

```

***Заливаем данные из таблицы flights в таблицу flights_new***

```python

demo=# insert into flights_new (select * from flights);
INSERT 0 214867

demo=# select 'select pg_size_pretty(pg_table_size('''||tablename||'''));' from pg_tables where tablename like 'flights_new%';
                           ?column?                           
--------------------------------------------------------------
 select pg_size_pretty(pg_table_size('flights_new'));
 select pg_size_pretty(pg_table_size('flights_new_2016_08'));
 select pg_size_pretty(pg_table_size('flights_new_2016_09'));
 select pg_size_pretty(pg_table_size('flights_new_2016_10'));
 select pg_size_pretty(pg_table_size('flights_new_2016_11'));
 select pg_size_pretty(pg_table_size('flights_new_2016_12'));
 select pg_size_pretty(pg_table_size('flights_new_2017_01'));
 select pg_size_pretty(pg_table_size('flights_new_2017_02'));
 select pg_size_pretty(pg_table_size('flights_new_2017_03'));
 select pg_size_pretty(pg_table_size('flights_new_2017_04'));
 select pg_size_pretty(pg_table_size('flights_new_2017_05'));
 select pg_size_pretty(pg_table_size('flights_new_2017_06'));
 select pg_size_pretty(pg_table_size('flights_new_2017_07'));
 select pg_size_pretty(pg_table_size('flights_new_2017_08'));
 select pg_size_pretty(pg_table_size('flights_new_2017_09'));
(15 строк)


-- Теперь этими селектами проверим размер таблиц

demo=# select pg_size_pretty(pg_table_size('flights_new'));
 pg_size_pretty 
----------------
 0 bytes
(1 строка)

demo=# select pg_size_pretty(pg_table_size('flights_new_2016_08'));
 pg_size_pretty 
----------------
 944 kB
(1 строка)

demo=# select pg_size_pretty(pg_table_size('flights_new_2016_09'));
 pg_size_pretty 
----------------
 1640 kB
(1 строка)

demo=# select pg_size_pretty(pg_table_size('flights_new_2016_10'));
 pg_size_pretty 
----------------
 1704 kB
(1 строка)

demo=# select pg_size_pretty(pg_table_size('flights_new_2016_11'));
 pg_size_pretty 
----------------
 1648 kB
(1 строка)

demo=# select pg_size_pretty(pg_table_size('flights_new_2016_12'));
 pg_size_pretty 
----------------
 1696 kB
(1 строка)

demo=# select pg_size_pretty(pg_table_size('flights_new_2017_01'));
 pg_size_pretty 
----------------
 1696 kB
(1 строка)

demo=# select pg_size_pretty(pg_table_size('flights_new_2017_02'));
 pg_size_pretty 
----------------
 1536 kB
(1 строка)

demo=# select pg_size_pretty(pg_table_size('flights_new_2017_03'));
 pg_size_pretty 
----------------
 1696 kB
(1 строка)

demo=# select pg_size_pretty(pg_table_size('flights_new_2017_04'));
 pg_size_pretty 
----------------
 1648 kB
(1 строка)

demo=# select pg_size_pretty(pg_table_size('flights_new_2017_05'));
 pg_size_pretty 
----------------
 1696 kB
(1 строка)

demo=# select pg_size_pretty(pg_table_size('flights_new_2017_06'));
 pg_size_pretty 
----------------
 1640 kB
(1 строка)

demo=# select pg_size_pretty(pg_table_size('flights_new_2017_07'));
 pg_size_pretty 
----------------
 1704 kB
(1 строка)

demo=# select pg_size_pretty(pg_table_size('flights_new_2017_08'));
 pg_size_pretty 
----------------
 1624 kB
(1 строка)

demo=# select pg_size_pretty(pg_table_size('flights_new_2017_09'));
 pg_size_pretty 
----------------
 728 kB
(1 строка)


Видим, что остновная таблица пустая, а партицированные таблицы содержут данные 

```
