
***Cоздадим таблички и наполним их данными***

```python
Прошу строго не судить, я совсем не разработчик..:)

                                List of databases
    Name    |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges 
------------+----------+----------+-------------+-------------+-------------------
 test_joins | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 | 
(1 row)

-- Первая таблица айди и имя покупателя

test_joins=# create table tbl_names (pkey int, cname text);
CREATE TABLE

test_joins=# select * from tbl_names ;

 pkey |  cname   
------+----------
    1 | Mickle
    2 | Petr
    3 | Vanya
    4 | Igor
    5 | Ksuxha
    6 | Ashot
    7 | Mila
    8 | Umpalump
    9 | Frodo
(9 rows)


-- Вторая таблица, айди города и название города

test_joins=# create table tbl_city (city_id int, city_name text);
CREATE TABLE

test_joins=# select * from tbl_city ;
 city_id | city_name 
---------+-----------
       1 | Vladimir
       2 | Moscow
       3 | Kiev
       4 | Minsk
       5 | Belgrad
       6 | Derevnja
       7 | Erevan
(7 rows)

-- Третья таблица 


test_joins=# create table tbl_orders as
select trunc(generate_series(1,500)) as order_id,
       trunc((random() * 8+1)) as buyer_id,
       trunc((random() * 4+1)) as city_id,
       generate_series(10,2000)*(random()*8+1) as order_sum                                                                                                                                                                      
from generate_series(1, 1000);
SELECT 1991000

test_joins=# select * from tbl_orders limit 5;

 order_id | buyer_id | city_id |     order_sum      
----------+----------+---------+--------------------
        1 |        3 |       4 | 36.536194474050205
        2 |        7 |       1 |  87.75894987313166
        3 |        5 |       2 |  35.79141878842927
        4 |        2 |       2 |  84.12384193765757
        5 |        7 |       2 | 19.727460975512315
(5 rows)



```

***Прямое соединение двух или более таблиц***

```python
test_joins=# select o.order_id, c.city_name as city, b.cname as buyer, o.order_sum 
  from tbl_orders o, tbl_city c, tbl_names b 
  where c.city_id=o.city_id
        and b.pkey=o.buyer_id
  limit 15; 
 order_id |   city   | buyer  |     order_sum      
----------+----------+--------+--------------------
        1 | Minsk    | Vanya  | 36.536194474050205
        2 | Vladimir | Mila   |  87.75894987313166
        3 | Moscow   | Ksuxha |  35.79141878842927
        4 | Moscow   | Petr   |  84.12384193765757
        5 | Moscow   | Mila   | 19.727460975512315
        6 | Minsk    | Ashot  | 115.79335982738257
        7 | Moscow   | Vanya  | 133.07813888954706
        8 | Vladimir | Mila   | 103.49797308075762
        9 | Minsk    | Ashot  |  69.25531487008783
       10 | Moscow   | Mila   | 119.70301082065691
       11 | Vladimir | Ksuxha | 138.09211306689178
       12 | Moscow   | Mickle |  177.1086328620159
       13 | Minsk    | Mickle |  50.47637791555945
       14 | Minsk    | Petr   |  139.3719001935176
       15 | Kiev     | Mickle | 131.38816727337303
(15 rows)

```

***Левостороннее соединение***

```python
test_joins=# select *
    from tbl_city c left join tbl_orders o on c.city_id=o.city_id limit 15;
 city_id | city_name | order_id | buyer_id | city_id |     order_sum      
---------+-----------+----------+----------+---------+--------------------
       4 | Minsk     |        1 |        3 |       4 | 36.536194474050205
       1 | Vladimir  |        2 |        7 |       1 |  87.75894987313166
       2 | Moscow    |        3 |        5 |       2 |  35.79141878842927
       2 | Moscow    |        4 |        2 |       2 |  84.12384193765757
       2 | Moscow    |        5 |        7 |       2 | 19.727460975512315
       4 | Minsk     |        6 |        6 |       4 | 115.79335982738257
       2 | Moscow    |        7 |        3 |       2 | 133.07813888954706
       1 | Vladimir  |        8 |        7 |       1 | 103.49797308075762
       4 | Minsk     |        9 |        6 |       4 |  69.25531487008783
       2 | Moscow    |       10 |        7 |       2 | 119.70301082065691
       1 | Vladimir  |       11 |        5 |       1 | 138.09211306689178
       2 | Moscow    |       12 |        1 |       2 |  177.1086328620159
       4 | Minsk     |       13 |        1 |       4 |  50.47637791555945
       4 | Minsk     |       14 |        2 |       4 |  139.3719001935176
       3 | Kiev      |       15 |        1 |       3 | 131.38816727337303
(15 rows)

```

***Кросс соединение***

```python

test_joins=# select *
    from tbl_city c cross join tbl_orders o
        limit 10;     
 city_id | city_name | order_id | buyer_id | city_id |     order_sum      
---------+-----------+----------+----------+---------+--------------------
       1 | Vladimir  |        1 |        3 |       4 | 36.536194474050205
       2 | Moscow    |        1 |        3 |       4 | 36.536194474050205
       3 | Kiev      |        1 |        3 |       4 | 36.536194474050205
       4 | Minsk     |        1 |        3 |       4 | 36.536194474050205
       5 | Belgrad   |        1 |        3 |       4 | 36.536194474050205
       6 | Derevnja  |        1 |        3 |       4 | 36.536194474050205
       7 | Erevan    |        1 |        3 |       4 | 36.536194474050205
       1 | Vladimir  |        2 |        7 |       1 |  87.75894987313166
       2 | Moscow    |        2 |        7 |       1 |  87.75894987313166
       3 | Kiev      |        2 |        7 |       1 |  87.75894987313166
(10 rows)


```

***Полное соединение***
```python

test_joins=# select *
    from tbl_city c full outer join tbl_orders o on c.city_id=o.city_id
        limit 10;

 city_id | city_name | order_id | buyer_id | city_id |     order_sum      
---------+-----------+----------+----------+---------+--------------------
       4 | Minsk     |        1 |        3 |       4 | 36.536194474050205
       1 | Vladimir  |        2 |        7 |       1 |  87.75894987313166
       2 | Moscow    |        3 |        5 |       2 |  35.79141878842927
       2 | Moscow    |        4 |        2 |       2 |  84.12384193765757
       2 | Moscow    |        5 |        7 |       2 | 19.727460975512315
       4 | Minsk     |        6 |        6 |       4 | 115.79335982738257
       2 | Moscow    |        7 |        3 |       2 | 133.07813888954706
       1 | Vladimir  |        8 |        7 |       1 | 103.49797308075762
       4 | Minsk     |        9 |        6 |       4 |  69.25531487008783
       2 | Moscow    |       10 |        7 |       2 | 119.70301082065691
(10 rows)


```

***Запрос с разными типами соединений***

```python

test_joins=# select *
  from tbl_orders o left join tbl_city c on c.city_id=o.city_id right join tbl_names b on b.pkey=o.buyer_id  
  limit 10;     

 order_id | buyer_id | city_id |     order_sum      | city_id | city_name | pkey | cname  
----------+----------+---------+--------------------+---------+-----------+------+--------
       12 |        1 |       2 |  177.1086328620159 |       2 | Moscow    |    1 | Mickle
       13 |        1 |       4 |  50.47637791555945 |       4 | Minsk     |    1 | Mickle
       15 |        1 |       3 | 131.38816727337303 |       3 | Kiev      |    1 | Mickle
       20 |        1 |       1 |  68.14868995986154 |       1 | Vladimir  |    1 | Mickle
       29 |        1 |       2 | 153.16033822361516 |       2 | Moscow    |    1 | Mickle
       36 |        1 |       4 |   272.079839476244 |       4 | Minsk     |    1 | Mickle
       39 |        1 |       3 | 121.34525655414063 |       3 | Kiev      |    1 | Mickle
       41 |        1 |       4 | 166.47725065090953 |       4 | Minsk     |    1 | Mickle
       46 |        1 |       1 |  406.5139879137985 |       1 | Vladimir  |    1 | Mickle
       52 |        1 |       4 |  89.20086587959878 |       4 | Minsk     |    1 | Mickle
(10 rows)


```

***Метрики на основе представлений***

```python
-- Данная метрика показывает список SQL запросов, которые создают временные файлы
-- Сделана на основе представления pg_stat_database

select datname, temp_files, pg_size_pretty(temp_bytes) as temp_file_size
FROM pg_stat_database
WHERE temp_files > 0
ORDER BY temp_bytes DESC;

```

```python
-- Активные процессы автовакуума на основе представления pg_stat_activity

SELECT (clock_timestamp() - xact_start) AS ts_age,
state, pid, query FROM pg_stat_activity
WHERE query ilike '%autovacuum%' AND NOT pid=pg_backend_pid();

```

```python
Данная метрика показывает индексы которые пылятся в шкафу..


SELECT schemaname, relname, indexrelname
FROM pg_stat_all_indexes
WHERE idx_scan = 0 and schemaname <> 'pg_toast' and  schemaname <> 'pg_catalog';
```
