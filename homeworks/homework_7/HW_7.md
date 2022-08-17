#### 1. Настройте выполнение контрольной точки раз в 30 секунд. ####

   Настройка частоты срабатывания контрольной точки задается параметром - chekpoint_timeout

   По умолчанию данный параметр = 5 минутам: 

        postgres=# show checkpoint_timeout;
        checkpoint_timeout
        --------------------
        5min
        (1 row)

   Задаем параметр = 30 секундам: 

        postgres=# alter system set checkpoint_timeout = '30s';
        ALTER SYSTEM

   Перезапускаем кластер и убеждаемся что данные приминились: 

        root@denis-vm1:/etc/postgresql/14/main# pg_ctlcluster 14 main restart
        root@denis-vm1:/etc/postgresql/14/main# sudo -u postgres psql
        psql (14.5 (Ubuntu 14.5-1.pgdg20.04+1))
        Type "help" for help.

        postgres=# show checkpoint_timeout;
        checkpoint_timeout
        --------------------
        30s
        (1 row)

   А так же включим логирование прохождения контрольных точек: 

        postgres=# show log_checkpoints;
        log_checkpoints
        -----------------
        off
        (1 row)

        postgres=# alter system set log_checkpoints = on;
        ALTER SYSTEM

   Проверяем что натройка применилась:

        postgres=# show log_checkpoints;
        log_checkpoints
        -----------------
        off
        (1 row)

   Не применилась....потому что надо перестянуть состояние конфигураций:

        postgres=# SELECT pg_reload_conf();
        pg_reload_conf
        ----------------
        t
        (1 row)

   Проверяем еще раз:

        postgres=# show log_checkpoints;
        log_checkpoints
        -----------------
        on
        (1 row)

   Готово, логирование чекпоинтов включено.


#### 2. 10 минут c помощью утилиты pgbench подавайте нагрузку. ####

   Подаем нагрузку на базу в течении 10 минут: 


        root@denis-vm1:~# sudo -u postgres pgbench -i test
        dropping old tables...
        NOTICE:  table "pgbench_accounts" does not exist, skipping
        NOTICE:  table "pgbench_branches" does not exist, skipping
        NOTICE:  table "pgbench_history" does not exist, skipping
        NOTICE:  table "pgbench_tellers" does not exist, skipping
        creating tables...
        generating data (client-side)...
        100000 of 100000 tuples (100%) done (elapsed 0.08 s, remaining 0.00 s)
        vacuuming...
        creating primary keys...
        done in 0.54 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.35 s, vacuum 0.07 s, primary keys 0.11 s).

        root@denis-vm1:~# sudo -u postgres pgbench -P 60 -T 600 test
        pgbench (14.5 (Ubuntu 14.5-1.pgdg20.04+1))
        starting vacuum...end.
        progress: 60.0 s, 501.2 tps, lat 1.995 ms stddev 1.643
        progress: 120.0 s, 480.2 tps, lat 2.082 ms stddev 1.764
        progress: 180.0 s, 494.1 tps, lat 2.023 ms stddev 1.586
        progress: 240.0 s, 581.9 tps, lat 1.718 ms stddev 1.228
        progress: 300.0 s, 606.2 tps, lat 1.649 ms stddev 1.167
        progress: 360.0 s, 621.7 tps, lat 1.608 ms stddev 1.099
        progress: 420.0 s, 591.6 tps, lat 1.690 ms stddev 1.200
        progress: 480.0 s, 569.8 tps, lat 1.755 ms stddev 1.322
        progress: 540.0 s, 636.9 tps, lat 1.570 ms stddev 0.968
        progress: 600.0 s, 550.2 tps, lat 1.817 ms stddev 1.367
        transaction type: <builtin: TPC-B (sort of)>
        scaling factor: 1
        query mode: simple
        number of clients: 1
        number of threads: 1
        duration: 600 s
        number of transactions actually processed: 338032
        latency average = 1.775 ms
        latency stddev = 1.344 ms
        initial connection time = 2.711 ms
        tps = 563.388303 (without initial connection time)


#### 3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку. ####

   Вот тут не совсем понял что пошло не так, или требование задания, но у меня по кругу перезаписывалось 4 wal файла по 16 МБ в каталоге /var/lib/postgresql/14/main/pg_wal

   Попробовал сменить настройку wal_keep_size на 1MB, но ничего не изменилось...

        -rw-------  1 postgres postgres 16777216 Aug 17 16:40 00000001000000000000007F
        -rw-------  1 postgres postgres 16777216 Aug 17 16:38 000000010000000000000080
        -rw-------  1 postgres postgres 16777216 Aug 17 16:39 000000010000000000000081
        -rw-------  1 postgres postgres 16777216 Aug 17 16:39 000000010000000000000082

#### 4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло? ####

   По статистике выполнения так же не понял где именно ей смотреть, нашел настройку log_checkpoints, включил её и в файле лога postgresql-14-main.log получил следующие данные: 

        2022-08-17 16:30:05.298 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:30:32.054 UTC [588] LOG:  checkpoint complete: wrote 1776 buffers (10.8%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.684 s, sync=0.029 s, total=26.756 s; sync files=16, longest=0.021 s, average=0.002 s; distance=16714 kB, estimate=18486 kB
        2022-08-17 16:30:35.057 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:31:02.149 UTC [588] LOG:  checkpoint complete: wrote 1695 buffers (10.3%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.987 s, sync=0.022 s, total=27.093 s; sync files=5, longest=0.012 s, average=0.005 s; distance=19118 kB, estimate=19118 kB
        2022-08-17 16:31:05.152 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:31:32.134 UTC [588] LOG:  checkpoint complete: wrote 1900 buffers (11.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.889 s, sync=0.020 s, total=26.982 s; sync files=11, longest=0.008 s, average=0.002 s; distance=18627 kB, estimate=19069 kB
        2022-08-17 16:31:35.137 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:32:02.111 UTC [588] LOG:  checkpoint complete: wrote 1770 buffers (10.8%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.883 s, sync=0.027 s, total=26.975 s; sync files=5, longest=0.014 s, average=0.006 s; distance=19435 kB, estimate=19435 kB
        2022-08-17 16:32:05.115 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:32:32.091 UTC [588] LOG:  checkpoint complete: wrote 1899 buffers (11.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.890 s, sync=0.021 s, total=26.977 s; sync files=10, longest=0.010 s, average=0.003 s; distance=18808 kB, estimate=19373 kB
        2022-08-17 16:32:35.094 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:33:02.102 UTC [588] LOG:  checkpoint complete: wrote 1767 buffers (10.8%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.888 s, sync=0.041 s, total=27.008 s; sync files=6, longest=0.029 s, average=0.007 s; distance=19412 kB, estimate=19412 kB
        2022-08-17 16:33:05.105 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:33:32.103 UTC [588] LOG:  checkpoint complete: wrote 1933 buffers (11.8%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.891 s, sync=0.031 s, total=26.998 s; sync files=9, longest=0.016 s, average=0.004 s; distance=20900 kB, estimate=20900 kB
        2022-08-17 16:33:35.106 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:34:02.060 UTC [588] LOG:  checkpoint complete: wrote 1771 buffers (10.8%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.879 s, sync=0.017 s, total=26.955 s; sync files=5, longest=0.011 s, average=0.004 s; distance=20106 kB, estimate=20821 kB
        2022-08-17 16:34:05.064 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:34:32.109 UTC [588] LOG:  checkpoint complete: wrote 1936 buffers (11.8%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.976 s, sync=0.028 s, total=27.046 s; sync files=9, longest=0.015 s, average=0.004 s; distance=20191 kB, estimate=20758 kB
        2022-08-17 16:34:35.112 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:35:02.100 UTC [588] LOG:  checkpoint complete: wrote 1808 buffers (11.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.902 s, sync=0.020 s, total=26.988 s; sync files=5, longest=0.020 s, average=0.004 s; distance=21355 kB, estimate=21355 kB
        2022-08-17 16:35:05.103 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:35:32.054 UTC [588] LOG:  checkpoint complete: wrote 1955 buffers (11.9%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.881 s, sync=0.014 s, total=26.951 s; sync files=9, longest=0.008 s, average=0.002 s; distance=20232 kB, estimate=21242 kB
        2022-08-17 16:35:35.054 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:36:02.111 UTC [588] LOG:  checkpoint complete: wrote 1810 buffers (11.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.990 s, sync=0.018 s, total=27.057 s; sync files=5, longest=0.010 s, average=0.004 s; distance=21137 kB, estimate=21232 kB
        2022-08-17 16:36:05.114 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:36:32.091 UTC [588] LOG:  checkpoint complete: wrote 1959 buffers (12.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.878 s, sync=0.029 s, total=26.978 s; sync files=10, longest=0.014 s, average=0.003 s; distance=20339 kB, estimate=21143 kB
        2022-08-17 16:36:35.094 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:37:02.039 UTC [588] LOG:  checkpoint complete: wrote 1790 buffers (10.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.882 s, sync=0.015 s, total=26.945 s; sync files=5, longest=0.008 s, average=0.003 s; distance=20297 kB, estimate=21058 kB
        2022-08-17 16:37:05.042 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:37:32.093 UTC [588] LOG:  checkpoint complete: wrote 1944 buffers (11.9%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.980 s, sync=0.014 s, total=27.051 s; sync files=8, longest=0.012 s, average=0.002 s; distance=20537 kB, estimate=21006 kB
        2022-08-17 16:37:35.096 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:38:02.115 UTC [588] LOG:  checkpoint complete: wrote 1778 buffers (10.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.899 s, sync=0.024 s, total=27.019 s; sync files=5, longest=0.013 s, average=0.005 s; distance=20421 kB, estimate=20947 kB
        2022-08-17 16:38:05.118 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:38:32.071 UTC [588] LOG:  checkpoint complete: wrote 2170 buffers (13.2%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.877 s, sync=0.035 s, total=26.953 s; sync files=9, longest=0.017 s, average=0.004 s; distance=20307 kB, estimate=20883 kB
        2022-08-17 16:38:35.074 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:39:02.128 UTC [588] LOG:  checkpoint complete: wrote 1824 buffers (11.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.981 s, sync=0.018 s, total=27.054 s; sync files=6, longest=0.018 s, average=0.003 s; distance=21438 kB, estimate=21438 kB
        2022-08-17 16:39:05.130 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:39:32.090 UTC [588] LOG:  checkpoint complete: wrote 1959 buffers (12.0%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.876 s, sync=0.025 s, total=26.960 s; sync files=8, longest=0.016 s, average=0.004 s; distance=20372 kB, estimate=21332 kB
        2022-08-17 16:39:35.093 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:40:02.056 UTC [588] LOG:  checkpoint complete: wrote 1773 buffers (10.8%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.907 s, sync=0.009 s, total=26.963 s; sync files=5, longest=0.006 s, average=0.002 s; distance=20449 kB, estimate=21243 kB
        2022-08-17 16:40:05.059 UTC [588] LOG:  checkpoint starting: time
        2022-08-17 16:40:32.098 UTC [588] LOG:  checkpoint complete: wrote 1751 buffers (10.7%); 0 WAL file(s) added, 0 removed, 1 recycled; write=27.000 s, sync=0.009 s, total=27.040 s; sync files=12, longest=0.008 s, average=0.001 s; distance=15280 kB, estimate=20647 kB   

   т.е. за время работы pgbench была создана 21 контрольная точка, и у всех время начала контрольной точки одинаковое с разницей в 30 секунд, 05 - 35.

#### 5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат. ####

   Проверяем текущий режим: 

        test=# show synchronous_commit;
        synchronous_commit
        --------------------
        on
        (1 row)

   Запускаем pgbench: 

        root@denis-vm1:~# sudo -u postgres pgbench -P 1 -T 10 test
        pgbench (14.5 (Ubuntu 14.5-1.pgdg20.04+1))
        starting vacuum...end.
        progress: 1.0 s, 677.0 tps, lat 1.472 ms stddev 0.468
        progress: 2.0 s, 686.0 tps, lat 1.458 ms stddev 0.514
        progress: 3.0 s, 712.0 tps, lat 1.404 ms stddev 0.439
        progress: 4.0 s, 710.0 tps, lat 1.407 ms stddev 0.420
        progress: 5.0 s, 517.0 tps, lat 1.933 ms stddev 1.367
        progress: 6.0 s, 342.0 tps, lat 2.913 ms stddev 1.954
        progress: 7.0 s, 337.0 tps, lat 2.982 ms stddev 1.953
        progress: 8.0 s, 468.0 tps, lat 2.136 ms stddev 1.496
        progress: 9.0 s, 697.0 tps, lat 1.434 ms stddev 0.436
        progress: 10.0 s, 687.0 tps, lat 1.455 ms stddev 0.534
        transaction type: <builtin: TPC-B (sort of)>
        scaling factor: 1
        query mode: simple
        number of clients: 1
        number of threads: 1
        duration: 10 s
        number of transactions actually processed: 5834
        latency average = 1.713 ms
        latency stddev = 1.095 ms
        initial connection time = 2.785 ms
        tps = 583.499195 (without initial connection time)   

   Теперь выключаем синхронный режим: 

        test=# ALTER SYSTEM SET synchronous_commit = off;
        ALTER SYSTEM
        test=# SELECT pg_reload_conf();
        pg_reload_conf
        ----------------
        t
        (1 row)

        test=# show synchronous_commit;
        synchronous_commit
        --------------------
        off
        (1 row)

   Повторно запускаем pgbench: 

        root@denis-vm1:~# sudo -u postgres pgbench -P 1 -T 10 test
        pgbench (14.5 (Ubuntu 14.5-1.pgdg20.04+1))
        starting vacuum...end.
        progress: 1.0 s, 2338.8 tps, lat 0.426 ms stddev 0.050
        progress: 2.0 s, 2328.1 tps, lat 0.429 ms stddev 0.084
        progress: 3.0 s, 2368.0 tps, lat 0.422 ms stddev 0.031
        progress: 4.0 s, 2395.1 tps, lat 0.417 ms stddev 0.029
        progress: 5.0 s, 2423.9 tps, lat 0.412 ms stddev 0.032
        progress: 6.0 s, 2413.1 tps, lat 0.414 ms stddev 0.030
        progress: 7.0 s, 2426.9 tps, lat 0.412 ms stddev 0.030
        progress: 8.0 s, 2419.1 tps, lat 0.413 ms stddev 0.033
        progress: 9.0 s, 2473.0 tps, lat 0.404 ms stddev 0.067
        progress: 10.0 s, 2488.0 tps, lat 0.402 ms stddev 0.028
        transaction type: <builtin: TPC-B (sort of)>
        scaling factor: 1
        query mode: simple
        number of clients: 1
        number of threads: 1
        duration: 10 s
        number of transactions actually processed: 24075
        latency average = 0.415 ms
        latency stddev = 0.046 ms
        initial connection time = 2.385 ms
        tps = 2407.950046 (without initial connection time)

   Видим прирост tps почти в 4 раза, это произошло потому, что в данном режиме (аминхронном) отсутствует ожидание подтверждения сброса данных из wal файлов на диск, т.е. мы получаем ответ о том что транзакция успешно завершена до фактической записи данных на диск, что может привести к тому, что в случае сбоя данные могут быть утеряны, но как видим из теста, этот метод позволяет значительно увеличить пропускную способность системы.


#### 6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу? ####

   Создаем кластер с включенной контрольной суммой: 

        pg_createcluster 14 main2 -- --data-checksums
    
   Запускаем и заходим в него: 

        root@denis-vm1:/etc/postgresql/14/main# pg_ctlcluster 14 main2 start
        root@denis-vm1:/etc/postgresql/14/main# pg_lsclusters
        Ver Cluster Port Status Owner    Data directory               Log file
        14  main    5432 online postgres /var/lib/postgresql/14/main  /var/log/postgresql/postgresql-14-main.log
        14  main2   5433 online postgres /var/lib/postgresql/14/main2 /var/log/postgresql/postgresql-14-main2.log

        root@denis-vm1:/etc/postgresql/14/main# sudo -u postgres psql port=5433
        psql (14.5 (Ubuntu 14.5-1.pgdg20.04+1))
        Type "help" for help.

        postgres=#

   Создаем базу данных и таблицу с данными: 

        postgres=# create database test2;
        CREATE DATABASE
        postgres=# \c test2
        You are now connected to database "test2" as user "postgres".
        test2=# create table t1 (i int);
        CREATE TABLE
        test2=# insert into t1 values(1),(2),(3);
        INSERT 0 3
        test2=# select * From t1;
        i
        ---
        1
        2
        3
        (3 rows)

   Смотрим по какому пути у нас находится таблица: 

        test2=# SELECT pg_relation_filepath('t1');
        pg_relation_filepath
        ----------------------
        base/16384/16385
        (1 row)

   Останавливаем кластер: 

        root@denis-vm1:/var/lib/postgresql/14/main2# pg_ctlcluster 14 main2 stop
        root@denis-vm1:/var/lib/postgresql/14/main2# pg_lsclusters
        Ver Cluster Port Status Owner    Data directory               Log file
        14  main    5432 online postgres /var/lib/postgresql/14/main  /var/log/postgresql/postgresql-14-main.log
        14  main2   5433 down   postgres /var/lib/postgresql/14/main2 /var/log/postgresql/postgresql-14-main2.log

   Меняем пару байт:

        root@denis-vm1:/var/lib/postgresql/14/main2# dd if=/dev/zero of=/var/lib/postgresql/14/main2/base/16384/16385 oflag=dsync conv=notrunc bs=1 count=8
        8+0 records in
        8+0 records out
        8 bytes copied, 0.00838023 s, 1.0 kB/s

   Запускаем кластер и пробуем выбрать данные: 

        test2=# select * from t1;
        WARNING:  page verification failed, calculated checksum 8640 but expected 14293
        ERROR:  invalid page in block 0 of relation base/16384/16385

   Получили ошибку.... 

   Для того, чтобы все же выбрать данные не смотря на ошибку, можно включить игнорирование ошибок по контрольным суммам: 

        test2=# alter system set ignore_checksum_failure = on;
        ALTER SYSTEM
        test2=# SELECT pg_reload_conf();
        pg_reload_conf
        ----------------
        t
        (1 row)

        test2=# show ignore_checksum_failure;
        ignore_checksum_failure
        -------------------------
        on
        (1 row)

   И теперь пробуем выбрать данные: 

        test2=# select * from t1;
        WARNING:  page verification failed, calculated checksum 8640 but expected 14293
        i
        ---
        1
        2
        3
        (3 rows)

   Видим что есть предупреждение по ошибке верификации, но при этом сами данные в таблице выводятся.

### Конец! ###

### Спасибо за внимание! ###

#### Автор - Кравченко Денис ####    
