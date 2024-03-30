***Подготовительные процедуры***
```python
-- Создал ВМ в ЯК
root@ubuntu:/home/igor# yc compute instances list

+----------------------+---------+---------------+---------+----------------+--------------+
|          ID          |  NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP  |
+----------------------+---------+---------------+---------+----------------+--------------+
| fhm07f07lo6lmrs09hi0 | sber-vm | ru-central1-a | RUNNING | 158.160.52.150 | 192.168.0.10 |
+----------------------+---------+---------------+---------+----------------+--------------+

-- Параметры:

    --name sber-vm \
    --hostname sber-vm \
    --cores 4 \
    --memory 8 \
    --create-boot-disk size=20G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
    --network-interface subnet-name=sber-subnet,nat-ip-version=ipv4 \
    --ssh-key ~/.ssh/yc_key.pub \

-- Развернул кластер PG

yc-user@sber-vm:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log


-- Сразу проверяем показатели pgbench, для postgres из коробки

- Запуск инициализации
pgbench -i postgres

- Запуск теста

postgres@sber-vm:/home/yc-user$ pgbench -c 20 -j 2 -P 10 -T 60 postgres
pgbench (14.11 (Ubuntu 14.11-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 659.7 tps, lat 30.109 ms stddev 28.955
progress: 20.0 s, 722.2 tps, lat 27.584 ms stddev 25.451
progress: 30.0 s, 549.2 tps, lat 36.521 ms stddev 51.032
progress: 40.0 s, 665.4 tps, lat 30.132 ms stddev 32.825
progress: 50.0 s, 739.9 tps, lat 26.910 ms stddev 27.221
progress: 60.0 s, 596.0 tps, lat 33.714 ms stddev 44.401
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 2
duration: 60 s
number of transactions actually processed: 39344
latency average = 30.489 ms
latency stddev = 35.412 ms
initial connection time = 26.610 ms
tps = 655.688320 (without initial connection time)

```

***Тюнинг***

```python
-- Зашёл в pgtune и мне были предложены следующие параметры

# DB Version: 14
# OS Type: linux
# DB Type: oltp
# Total Memory (RAM): 8 GB
# CPUs num: 4
# Connections num: 50
# Data Storage: ssd

max_connections = 20
shared_buffers = 4GB
effective_cache_size = 6GB
maintenance_work_mem = 1GB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 26214kB
huge_pages = off
min_wal_size = 1GB
max_wal_size = 4GB
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_workers = 4
max_parallel_maintenance_workers = 2



-- Прописал параметры в конфигурационном файле postgresql.conf
-- Сделал рестарт
yc-user@sber-vm:~$ sudo pg_ctlcluster 14 main restart 


-- Тест с параметрами от pg-tune (Видим, что число tps немного уменьшилось)

postgres@sber-vm:/home/yc-user$ pgbench -c 20 -j 2 -P 10 -T 60 postgres
pgbench (14.11 (Ubuntu 14.11-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 538.2 tps, lat 36.792 ms stddev 47.505
progress: 20.0 s, 725.1 tps, lat 27.700 ms stddev 27.217
progress: 30.0 s, 674.9 tps, lat 29.645 ms stddev 30.243
progress: 40.0 s, 574.0 tps, lat 34.784 ms stddev 50.778
progress: 50.0 s, 748.5 tps, lat 26.714 ms stddev 25.554
progress: 60.0 s, 652.0 tps, lat 30.696 ms stddev 34.167
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 2
duration: 60 s
number of transactions actually processed: 39147
latency average = 30.644 ms
latency stddev = 36.274 ms
initial connection time = 27.392 ms
tps = 652.364236 (without initial connection time)

Что делать.. Жаль конечно, что на уроке об этом не было рассказано, но я начал эксперименты

- Для начала перевел кластер в асинхронный режим (нам же надо максимальную производительность)
Вот результаты теста (буду теперь прикладывать последние строки):

number of transactions actually processed: 194669
latency average = 6.147 ms
latency stddev = 5.385 ms
initial connection time = 27.567 ms
tps = 3244.637994 (without initial connection time)

Видим кратное увеличение производительности.

Начал думать, что еще можно подкрутить и решил:

Отключить режим синхронизации СУБД с ОС по дисковому вводу/выводу (есть риск потерять данные в случае отключения электричества)
fsync = off

-- Запускаю новый тест

number of transactions actually processed: 195801
latency average = 6.112 ms
latency stddev = 5.349 ms
initial connection time = 26.205 ms
tps = 3263.202558 (without initial connection time)

Видим, что результат еще более улучшился

Теперь надумал перевести wal_level в минимал (даёт меньшую нагрузку), ну и в след за ним max_wal_senders = 0

-- Еще раз запускаю тест

number of transactions actually processed: 198712
latency average = 6.023 ms
latency stddev = 5.136 ms
initial connection time = 26.681 ms
tps = 3311.899786 (without initial connection time)

Видим что результат стал еще лучше

Итог:
После проведенных манипуляций удалось увеличить производительность, более чем в 5 раз. Мощь

Если есть еще какие-то варианты увеличить производительность, то прошу сообщить.

```
