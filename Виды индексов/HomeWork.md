
***Создать индекс к какой-либо из таблиц вашей БД***
```python
                                List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges 
-----------+----------+----------+-------------+-------------+-------------------
 dvdrental | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
(1 row)


dvdrental=# CREATE INDEX ind_rental ON rental(customer_id);
CREATE INDEX
Time: 331,615 ms

```

***Прислать результат команды explain, в которой используется данный индекс***

```python
dvdrental=# EXPLAIN analyze SELECT * FROM rental WHERE customer_id = 333 limit 3;
                                                         QUERY PLAN                                                          
-----------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.29..12.34 rows=3 width=36) (actual time=0.079..0.082 rows=3 loops=1)
   ->  Index Scan using ind_rental on rental  (cost=0.29..100.72 rows=25 width=36) (actual time=0.079..0.081 rows=3 loops=1)
         Index Cond: (customer_id = 333)
 Planning Time: 0.067 ms
 Execution Time: 0.094 ms
(5 rows)

Видим, что при поиске используется индекс

```

***Реализоватьиндекс для полнотекстового поиска***
```python

dvdrental=# CREATE INDEX idx_address_gin ON address USING GIN (to_tsvector('english', address));
CREATE INDEX
Time: 9,047 ms

dvdrental=# explain analyze SELECT count(*) FROM address
WHERE to_tsvector('english', address) @@ to_tsquery('english','Street');
                                                          QUERY PLAN                                                           
-------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=15.31..15.32 rows=1 width=8) (actual time=0.050..0.051 rows=1 loops=1)
   ->  Bitmap Heap Scan on address  (cost=8.02..15.30 rows=3 width=0) (actual time=0.024..0.043 rows=60 loops=1)
         Recheck Cond: (to_tsvector('english'::regconfig, (address)::text) @@ '''street'''::tsquery)
         Heap Blocks: exact=8
         ->  Bitmap Index Scan on idx_address_gin  (cost=0.00..8.02 rows=3 width=0) (actual time=0.013..0.014 rows=60 loops=1)
               Index Cond: (to_tsvector('english'::regconfig, (address)::text) @@ '''street'''::tsquery)
 Planning Time: 0.164 ms
 Execution Time: 0.070 ms
(8 rows)


```
 
***Реализовать индекс на часть таблицы или индекс на поле с функцией***

```python
dvdrental=# create index full_idx on city(upper(city));
CREATE INDEX
Time: 5,836 ms

dvdrental=# explain analyse                                  

select * from city where upper(city) = upper('London');

                                                   QUERY PLAN                                                    
-----------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on city  (cost=4.30..9.37 rows=3 width=23) (actual time=0.035..0.035 rows=2 loops=1)
   Recheck Cond: (upper((city)::text) = 'LONDON'::text)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on full_idx  (cost=0.00..4.30 rows=3 width=0) (actual time=0.032..0.032 rows=2 loops=1)
         Index Cond: (upper((city)::text) = 'LONDON'::text)
 Planning Time: 0.319 ms
 Execution Time: 0.063 ms

(7 rows)
Time: 0,718 ms

Видим что при поиске использзуется индекс full_idx

```
***Создать индекс на несколько полей***

```python

dvdrental=# create index many_fields on city(city_id,city);
CREATE INDEX
Time: 298,687 ms

dvdrental=# explain analyse
select * from city where city_id=10 and city = 'Alessandria';
                                                   QUERY PLAN                                                     
-------------------------------------------------------------------------------------------------------------------
 Index Scan using many_fields on city  (cost=0.28..8.29 rows=1 width=23) (actual time=0.025..0.025 rows=0 loops=1)
   Index Cond: ((city_id = 10) AND ((city)::text = 'Alessandria'::text))
 Planning Time: 0.255 ms
 Execution Time: 0.036 ms
(4 rows)

Time: 0,631 ms

Успешно реализован индекс  на несколько полей

```
