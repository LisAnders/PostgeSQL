
Для начала проверим какие значения tps будут "из коробки": 

    root@denis-vm1:/etc/postgresql/14/main# sudo -u postgres pgbench -P 60 -T 600 postgres
    pgbench (14.5 (Ubuntu 14.5-1.pgdg22.04+1))
    starting vacuum...end.
    progress: 60.0 s, 529.4 tps, lat 1.888 ms stddev 1.854
    progress: 120.0 s, 611.4 tps, lat 1.635 ms stddev 1.375
    progress: 180.0 s, 611.4 tps, lat 1.635 ms stddev 1.370
    progress: 240.0 s, 655.9 tps, lat 1.524 ms stddev 1.204
    progress: 300.0 s, 570.9 tps, lat 1.751 ms stddev 1.492
    progress: 360.0 s, 630.2 tps, lat 1.586 ms stddev 1.316
    progress: 420.0 s, 644.4 tps, lat 1.552 ms stddev 1.424
    progress: 480.0 s, 710.8 tps, lat 1.407 ms stddev 0.948
    progress: 540.0 s, 582.1 tps, lat 1.718 ms stddev 1.497
    progress: 600.0 s, 492.9 tps, lat 2.029 ms stddev 1.619
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 1
    number of threads: 1
    duration: 600 s
    number of transactions actually processed: 362366
    latency average = 1.655 ms
    latency stddev = 1.418 ms
    initial connection time = 3.003 ms
    tps = 603.946008 (without initial connection time)

Теперь устанавливаем значения, которые предложит pgtune, в моем случае это: 

    # DB Version: 14
    # OS Type: linux
    # DB Type: mixed
    # Total Memory (RAM): 4 GB
    # CPUs num: 2
    # Connections num: 100
    # Data Storage: hdd

    max_connections = 100
    shared_buffers = 1GB
    effective_cache_size = 3GB
    maintenance_work_mem = 256MB
    checkpoint_completion_target = 0.9
    wal_buffers = 16MB
    default_statistics_target = 100
    random_page_cost = 4
    effective_io_concurrency = 2
    work_mem = 5242kB
    min_wal_size = 1GB
    max_wal_size = 4GB
    max_worker_processes = 2
    max_parallel_workers_per_gather = 1
    max_parallel_workers = 2
    max_parallel_maintenance_workers = 1

Данные значения поместил в отдельный файл в директории conf.d 
После перезапуска кластера убедился что значение подтянулись: 

    postgres=# show shared_buffers;
     shared_buffers
    ----------------
     1GB
    (1 row)

И теперь запускаем еще раз pgbench: 

    root@denis-vm1:/etc/postgresql/14/main/conf.d# sudo -u postgres pgbench -P 60 -T 600 postgres
    pgbench (14.5 (Ubuntu 14.5-1.pgdg22.04+1))
    starting vacuum...end.
    progress: 60.0 s, 629.7 tps, lat 1.588 ms stddev 1.078
    progress: 120.0 s, 609.6 tps, lat 1.640 ms stddev 1.184
    progress: 180.0 s, 568.4 tps, lat 1.759 ms stddev 1.382
    progress: 240.0 s, 665.2 tps, lat 1.503 ms stddev 0.933
    progress: 300.0 s, 612.7 tps, lat 1.632 ms stddev 1.146
    progress: 360.0 s, 623.7 tps, lat 1.603 ms stddev 1.117
    progress: 420.0 s, 619.7 tps, lat 1.614 ms stddev 1.133
    progress: 480.0 s, 597.7 tps, lat 1.673 ms stddev 1.170
    progress: 540.0 s, 636.3 tps, lat 1.571 ms stddev 1.076
    progress: 600.0 s, 647.8 tps, lat 1.543 ms stddev 1.040
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 1
    number of threads: 1
    duration: 600 s
    number of transactions actually processed: 372646
    latency average = 1.610 ms
    latency stddev = 1.129 ms
    initial connection time = 3.049 ms
    tps = 621.078649 (without initial connection time)

Видим что прирост есть, но не особо большой.

Теперь попробуем увеличить пропускную способность в ушерб надежности данных.
Устанавливаем параметр synchronous_commit на значение off 

    root@denis-vm1:/etc/postgresql/14/main/conf.d# sudo -u postgres pgbench -P 60 -T 600 postgres
    pgbench (14.5 (Ubuntu 14.5-1.pgdg22.04+1))
    starting vacuum...end.
    progress: 60.0 s, 2422.7 tps, lat 0.412 ms stddev 0.049
    progress: 120.0 s, 2449.9 tps, lat 0.408 ms stddev 0.045
    progress: 180.0 s, 2435.4 tps, lat 0.410 ms stddev 0.051
    progress: 240.0 s, 2422.9 tps, lat 0.412 ms stddev 0.069
    progress: 300.0 s, 2425.3 tps, lat 0.412 ms stddev 0.051
    progress: 360.0 s, 2439.5 tps, lat 0.410 ms stddev 0.066
    progress: 420.0 s, 2432.4 tps, lat 0.411 ms stddev 0.063
    progress: 480.0 s, 2425.5 tps, lat 0.412 ms stddev 0.062
    progress: 540.0 s, 2415.7 tps, lat 0.414 ms stddev 0.066
    progress: 600.0 s, 2418.4 tps, lat 0.413 ms stddev 0.062
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 1
    number of threads: 1
    duration: 600 s
    number of transactions actually processed: 1457261
    latency average = 0.411 ms
    latency stddev = 0.059 ms
    initial connection time = 3.316 ms
    tps = 2428.781084 (without initial connection time)

Получаем значительный прирос в производительности, но риск утери данных, в случае если произойдет сбой сервера.

Так же, как один из методов ускорения производительности базы можно использовать параметр fsync.
Данный параметр отвечает за то, чтобы изменения в БД были физически записаны на диск, и его отключение может привести к безвозвратной порче данных в случае сбоя, но даст выйгрыш в скорости.

Проведем тест с выключенным fsync но включенным synchronous_commit:

    root@denis-vm1:~# sudo -u postgres pgbench -P 60 -T 600 postgres
    pgbench (14.5 (Ubuntu 14.5-1.pgdg22.04+1))
    starting vacuum...end.
    progress: 60.0 s, 2307.1 tps, lat 0.433 ms stddev 0.047
    progress: 120.0 s, 2292.9 tps, lat 0.436 ms stddev 0.042
    progress: 180.0 s, 2283.2 tps, lat 0.438 ms stddev 0.055
    progress: 240.0 s, 2272.3 tps, lat 0.440 ms stddev 0.047
    progress: 300.0 s, 2262.7 tps, lat 0.442 ms stddev 0.053
    progress: 360.0 s, 2287.4 tps, lat 0.437 ms stddev 0.045
    progress: 420.0 s, 2306.8 tps, lat 0.433 ms stddev 0.062
    progress: 480.0 s, 2291.0 tps, lat 0.436 ms stddev 0.055
    progress: 540.0 s, 2296.3 tps, lat 0.435 ms stddev 0.059
    progress: 600.0 s, 2294.1 tps, lat 0.436 ms stddev 0.054
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 1
    number of threads: 1
    duration: 600 s
    number of transactions actually processed: 1373626
    latency average = 0.437 ms
    latency stddev = 0.052 ms
    initial connection time = 3.394 ms
    tps = 2289.387961 (without initial connection time)

Видим что при отключении fsync получилось добиться количества транзакций в секнду практически сопостовимыми со значениями, полученными при выключении synchronous_commit

Отдельно отмечу что одновременное отключение fsync и synchronous_commit значимого прироста производительности уже не дало.

### Конец! ###

### Спасибо за внимание! ###

#### Автор - Кравченко Денис ####    
