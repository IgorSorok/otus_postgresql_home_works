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
