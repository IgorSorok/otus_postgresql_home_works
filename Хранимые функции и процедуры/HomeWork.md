
***Подготовительные мероприятия***
```python
-- Создал БД
postgres=# create database magazin;
CREATE DATABASE

magazin=# \l magazin 
                               List of databases
  Name   |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges 
---------+----------+----------+-------------+-------------+-------------------
 magazin | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 | 
(1 row)

-- Схема

magazin=# \dn+ pract_functions 
                       List of schemas
      Name       |  Owner   | Access privileges | Description 
-----------------+----------+-------------------+-------------
 pract_functions | postgres |                   | 
(1 row)

-- Путь поиска 
magazin=# SET search_path = pract_functions, public;
SET
magazin=# show search_path ;
       search_path       
-------------------------
 pract_functions, public
(1 row)

-- Создал табличку goods и наполнил данными
magazin=# CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);

INSERT INTO goods (goods_id, good_name, good_price)
VALUES  (1, 'Спички хозайственные', .50),
                (2, 'Автомобиль Ferrari FXX K', 185000000.01);
CREATE TABLE
INSERT 0 2

magazin=# \dt pract_functions.goods 
             List of relations
     Schema      | Name  | Type  |  Owner   
-----------------+-------+-------+----------
 pract_functions | goods | table | postgres
(1 row)

magazin=# select * from pract_functions.goods ;
 goods_id |        good_name         |  good_price  
----------+--------------------------+--------------
        1 | Спички хозайственные     |         0.50
        2 | Автомобиль Ferrari FXX K | 185000000.01
(2 rows)

-- Создал табличку sales

magazin=# CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);
CREATE TABLE

magazin=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
INSERT 0 4

magazin=# \dt pract_functions.sales 
             List of relations
     Schema      | Name  | Type  |  Owner   
-----------------+-------+-------+----------
 pract_functions | sales | table | postgres
(1 row)

magazin=# select * from pract_functions.sales
magazin-# ;

 sales_id | good_id |          sales_time           | sales_qty 
----------+---------+-------------------------------+-----------
        1 |       1 | 2024-05-05 16:13:11.599645+03 |        10
        2 |       1 | 2024-05-05 16:13:11.599645+03 |         1
        3 |       1 | 2024-05-05 16:13:11.599645+03 |       120
        4 |       2 | 2024-05-05 16:13:11.599645+03 |         1
(4 rows)

-- Отчёт

magazin=# SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
        good_name         |     sum      
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)


-- с увеличением объёма данных отчет стал создаваться медленно
-- Принято решение денормализовать БД, создать таблицу
CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL
);
```


***Создаю триггерную функцию***

```python

CREATE OR REPLACE FUNCTION add_mart_goods() RETURNS TRIGGER AS $$
DECLARE
        v_sales_qty int;
        v_good_id int;
        -- v_sum_sale numeric(16, 2);
BEGIN
        -- В зависимости от операции в начальной таблице, редактируем кол-во товара в витрине
        IF (TG_OP = 'INSERT') THEN
                v_sales_qty = NEW.sales_qty;
                v_good_id = NEW.good_id;
        ELSIF (TG_OP = 'UPDATE' and NEW.sales_qty != OLD.sales_qty) THEN
                v_sales_qty = NEW.sales_qty - OLD.sales_qty;
                v_good_id = OLD.good_id;
        ELSIF (TG_OP = 'DELETE') THEN
                v_sales_qty = -1 * OLD.sales_qty;
                v_good_id = OLD.good_id;
        END IF;
        -- отправляем данные на витрину
        INSERT INTO good_sum_mart
        SELECT good_name, good_price * v_sales_qty
        FROM goods WHERE goods_id = v_good_id;
        RETURN null;
END;
$$ LANGUAGE plpgsql;
CREATE FUNCTION

Создаю триггер

magazin=# CREATE TRIGGER good_sum_mart_tr
AFTER INSERT OR UPDATE OR DELETE ON sales
FOR EACH ROW EXECUTE FUNCTION add_mart_goods();


```

***Проверяем, как всё работает***

```python
-- Почистил таблицы и залил данные обратно

magazin=# truncate table sales;
TRUNCATE TABLE

magazin=# truncate table good_sum_mart;
TRUNCATE TABLE

magazin=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
INSERT 0 4


-- Сравним проверочный запрос с витриной

- Проврочный запрос

magazin=# SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

        good_name         |     sum      
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

- Витрина

magazin=# select good_name, sum(sum_sale)
from good_sum_mart
group by good_name;

        good_name         |     sum      
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

```

***Проведу несколько операций дабы проверить работу триггера***

```python
-- Будем продавать автомоьиль за сто лимонов :)
magazin=# update goods
set good_price = 100000000
where goods_id = 2;
UPDATE 1


-- Допустим, пришёл богатый айтишник и покупает оба спорткара (на работу ездить)

magazin=# INSERT INTO sales (good_id, sales_qty) VALUES (2, 2);
select good_name, sum(sum_sale)
from good_sum_mart
group by good_name;
INSERT 0 1

        good_name         |     sum      
--------------------------+--------------
 Автомобиль Ferrari FXX K | 385000000.01
 Спички хозайственные     |        65.50
(2 rows)

-- Но случилось так, что он грохнул дорогостоящий сервер и его штрафанули..
-- Говорит куплю один спорткар (он еще незнает, что его ждёт)

magazin=# update sales
set sales_qty = 1
where sales_id = 25;
UPDATE 0 1

select good_name, sum(sum_sale)
from good_sum_mart
group by good_name;

        good_name         |     sum      
--------------------------+--------------
 Автомобиль Ferrari FXX K | 285000000.01
 Спички хозайственные     |        65.50
(2 rows)


-- А на следующий день его поймали на том, что он слил БД клиентов в интернет
-- Тут уже не до спорткаров, деньги на адвокатов нужны

magazin=# delete from sales
where sales_id = 25;
DELETE 0 1


select good_name, sum(sum_sale)
from good_sum_mart
group by good_name;


        good_name         |     sum      
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)


```

***Чем такая схема (витрина+триггер) предпочтительнее отчета***

Преемущества схемы витрина + триггер в том что мы фиксируем цену на момент продажи, а не берем из справочника..
Незнаю что еще добавить. Но спасибо за внимание.
