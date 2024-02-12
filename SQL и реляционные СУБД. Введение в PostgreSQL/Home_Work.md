**1. Создан новый проект и инстанс виртуальной машины в Яндекс клауд:**

![image](https://github.com/IgorSorok/otus_postgresql_home_works/assets/159540313/d5c89018-b972-480d-b2ed-5c2f5b2a06fb)

**2. Добавлен SSH - ключ:**
```python
igor@ubuntu:~/.ssh$ cat id_ed25519.pub`
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEKqemsAzJ5GTRGNBX/QhsH6IC1FUlIqJ5UgtsG5DJzA igor@ubuntu
```
**3. Установлена 15-ая версия postgresql:**
```SQL
igor@igor-dba:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
**4. Заходим в  psql под пользователем postgres с двух сессий и выключаем AUTOCOMMIT:**
```SQL
igor@igor-dba:~$ sudo -u postgres psql
psql (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
Type "help" for help.
postgres=# \set AUTOCOMMIT off 
postgres=# \echo :AUTOCOMMIT 
off
```

**5. Создаем и наполням таблицу данными**
```SQL
postgres=# create table persons(id serial, first_name text, second_name text);   
CREATE TABLE
postgres=*# insert into persons(first_name, second_name) values('ivan', 'ivanov');  
INSERT 0 1
postgres=*# insert into persons(first_name, second_name) values('petr', 'petrov');  
INSERT 0 1
postgres=*# commit;
COMMIT
```
Данная таблица отоброжается с обеих сессий (мы же сделали commit);
```SQL
postgres(session2)=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
**6. Смотрим текущий уровень изоляции:**
```SQL
postgres=# show transaction isolation level;

 transaction_isolation 
-----------------------
 read committed
(1 row)
```
**7. Во второй транзакции, первой сессии добавляем еще данных (значение уровня изоляции не меням)**
```SQL
postgres=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```
***Выполняем запрос к таблице в этой же сессии и наблюдаем, что данные успешно добавились***
```SQL
postgres=*# select * from persons;

 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
***Во второй же сессии наблюдаем, что табличка осталась в предыдущем состоянии***
```SQL
postgres(session2)=# select * from persons;

 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

***Новая запись не отображается, потому-что установлен дефолтный уровень изоляции read committed и отключен автокомит
(вставка первой транзакцией не зафиксирована).***

***Фиксируем первую сессию.***
```SQL
postgres=*# commit;
COMMIT
```
***И теперь во второй сессии наблюдаем, что данные добавились***
```SQL
postgres(session2)=# select * from persons;

 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
***Новая запись отображается по причине того, что в первой транзакции был commit - данные зафиксированы***

***Завершаем транзакцию во второй сессии***
```SQL
postgres(session2)=*# commit;
WARNING:  there is no transaction in progress
COMMIT
```
**8. Начинаем новые, но уже repeatable read транзации:**
```SQL
postgres=# set transaction isolation level repeatable read;
SET
postgres=*# show transaction isolation level;
 transaction_isolation 
-----------------------
 repeatable read
(1 row)
```
***Во второй сессии выполняем запрос к табличке***
```SQL
postgres(session2)=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 rows)
```
***Новые данные не отображаются, т.к. в первой сессии не было коммита.
Коммитим в первой сессии***

***И повторно обращаемся к табличке во второй сессии***
***Видим, что данные не изменились***
```SQL
postgres(session2)=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 rows)
```
***Новые данные не отображаются, даже при коммите транзакции в первой сессии.
Потому что вторая транзакция видит снимок данных на момент своего начала
и не видит остальные завершенные изменения.
Для того, что-бы это исправить нужно завершить вторую транзакцию.***

***Коммитим транзакцию во второй сессии и видим изменения.***
```SQL
postgres=# select * from persons;

 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

Спасибо за внимание! Что-то я утомился с этим гитом)

