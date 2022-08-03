1. Развернута ВМ в Янедкс Облако 

2. Установлен PjstgreSQL 14

3. Применены настройки: 

    Изменения внесены в файле postgresql.conf при остановленном кластере:

        max_connections = 40
        shared_buffers = 1GB
        effective_cache_size = 3GB
        maintenance_work_mem = 512MB
        checkpoint_completion_target = 0.9
        wal_buffers = 16MB
        default_statistics_target = 500
        random_page_cost = 4
        effective_io_concurrency = 2
        work_mem = 6553kB
        min_wal_size = 4GB
        max_wal_size = 16GB

    После внесения изменений кластер был запущен

4. Выполнена команда pgbench -i postgres

         sudo -u postgres pgbench -i postgres

        dropping old tables...
        NOTICE:  table "pgbench_accounts" does not exist, skipping
        NOTICE:  table "pgbench_branches" does not exist, skipping
        NOTICE:  table "pgbench_history" does not exist, skipping
        NOTICE:  table "pgbench_tellers" does not exist, skipping
        creating tables...
        generating data (client-side)...
        100000 of 100000 tuples (100%) done (elapsed 0.06 s, remaining 0.00 s)
        vacuuming...
        creating primary keys...
        done in 0.44 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.29 s, vacuum 0.06 s, primary keys 0.08 s).

5. Запустили pgbench -c8 -P 60 -T 3600 -U postgres postgres

        sudo -u postgres pgbench -c 8 -P 60 -T 3600 -U postgres postgres
        pgbench (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
        starting vacuum...end.
        progress: 60.0 s, 647.4 tps, lat 12.328 ms stddev 10.742
        progress: 120.0 s, 720.1 tps, lat 11.089 ms stddev 9.556
        progress: 180.0 s, 533.0 tps, lat 14.991 ms stddev 13.223
        progress: 240.0 s, 706.4 tps, lat 11.300 ms stddev 9.205
        progress: 300.0 s, 687.6 tps, lat 11.615 ms stddev 10.055
        progress: 360.0 s, 733.1 tps, lat 10.892 ms stddev 9.246
        progress: 420.0 s, 707.8 tps, lat 11.276 ms stddev 9.514
        progress: 480.0 s, 696.0 tps, lat 11.478 ms stddev 9.670
        progress: 540.0 s, 698.9 tps, lat 11.422 ms stddev 9.493
        progress: 600.0 s, 756.5 tps, lat 10.553 ms stddev 8.585
        progress: 660.0 s, 674.0 tps, lat 11.845 ms stddev 9.738
        progress: 720.0 s, 635.6 tps, lat 12.568 ms stddev 10.735
        progress: 780.0 s, 707.1 tps, lat 11.291 ms stddev 9.460
        progress: 840.0 s, 602.1 tps, lat 13.266 ms stddev 11.766
        progress: 900.0 s, 685.4 tps, lat 11.648 ms stddev 10.159
        progress: 960.0 s, 715.4 tps, lat 11.162 ms stddev 9.274
        progress: 1020.0 s, 651.9 tps, lat 12.246 ms stddev 10.299
        progress: 1080.0 s, 665.6 tps, lat 12.001 ms stddev 9.895
        progress: 1140.0 s, 667.7 tps, lat 11.956 ms stddev 9.825
        progress: 1200.0 s, 477.8 tps, lat 16.727 ms stddev 14.144
        progress: 1260.0 s, 669.0 tps, lat 11.937 ms stddev 10.235
        progress: 1320.0 s, 682.6 tps, lat 11.698 ms stddev 10.199
        progress: 1380.0 s, 691.4 tps, lat 11.548 ms stddev 9.418
        progress: 1440.0 s, 715.0 tps, lat 11.163 ms stddev 9.211
        progress: 1500.0 s, 640.1 tps, lat 12.482 ms stddev 11.067
        progress: 1560.0 s, 706.9 tps, lat 11.292 ms stddev 9.256
        progress: 1620.0 s, 720.7 tps, lat 11.080 ms stddev 9.223
        progress: 1680.0 s, 683.6 tps, lat 11.682 ms stddev 9.350
        progress: 1740.0 s, 730.5 tps, lat 10.925 ms stddev 8.957
        progress: 1800.0 s, 691.9 tps, lat 11.544 ms stddev 9.395
        progress: 1860.0 s, 733.9 tps, lat 10.880 ms stddev 9.183
        progress: 1920.0 s, 768.4 tps, lat 10.388 ms stddev 8.419
        progress: 1980.0 s, 760.7 tps, lat 10.496 ms stddev 8.575
        progress: 2040.0 s, 636.6 tps, lat 12.544 ms stddev 10.268
        progress: 2100.0 s, 677.2 tps, lat 11.791 ms stddev 9.380
        progress: 2160.0 s, 649.0 tps, lat 12.306 ms stddev 10.093
        progress: 2220.0 s, 677.9 tps, lat 11.778 ms stddev 9.867
        progress: 2280.0 s, 680.0 tps, lat 11.743 ms stddev 9.680
        progress: 2340.0 s, 746.3 tps, lat 10.696 ms stddev 9.131
        progress: 2400.0 s, 676.5 tps, lat 11.804 ms stddev 10.533
        progress: 2460.0 s, 704.7 tps, lat 11.330 ms stddev 9.300
        progress: 2520.0 s, 691.6 tps, lat 11.545 ms stddev 9.734
        progress: 2580.0 s, 697.4 tps, lat 11.449 ms stddev 9.782
        progress: 2640.0 s, 752.8 tps, lat 10.605 ms stddev 8.870
        progress: 2700.0 s, 717.0 tps, lat 11.137 ms stddev 9.333
        progress: 2760.0 s, 632.5 tps, lat 12.624 ms stddev 10.956
        progress: 2820.0 s, 562.7 tps, lat 14.199 ms stddev 11.958
        progress: 2880.0 s, 629.1 tps, lat 12.696 ms stddev 10.826
        progress: 2940.0 s, 710.8 tps, lat 11.235 ms stddev 9.442
        progress: 3000.0 s, 700.4 tps, lat 11.401 ms stddev 9.705
        progress: 3060.0 s, 715.8 tps, lat 11.154 ms stddev 9.670
        progress: 3120.0 s, 565.7 tps, lat 14.122 ms stddev 11.870
        progress: 3180.0 s, 656.3 tps, lat 12.169 ms stddev 10.249
        progress: 3240.0 s, 712.9 tps, lat 11.201 ms stddev 9.529
        progress: 3300.0 s, 690.5 tps, lat 11.565 ms stddev 9.652
        progress: 3360.0 s, 676.7 tps, lat 11.798 ms stddev 10.088
        progress: 3420.0 s, 673.3 tps, lat 11.863 ms stddev 9.604
        progress: 3480.0 s, 710.0 tps, lat 11.247 ms stddev 9.091
        progress: 3540.0 s, 686.6 tps, lat 11.630 ms stddev 10.039
        progress: 3600.0 s, 698.4 tps, lat 11.431 ms stddev 9.415
        transaction type: <builtin: TPC-B (sort of)>
        scaling factor: 1
        query mode: simple
        number of clients: 8
        number of threads: 1
        duration: 3600 s
        number of transactions actually processed: 2453594
        latency average = 11.716 ms
        latency stddev = 9.937 ms
        initial connection time = 14.638 ms
        tps = 681.546082 (without initial connection time)

6.  Изменил настройки Autovacuum: 

         log_autovacuum_min_duration = 0 (Задаёт время выполнения действия автоочистки, при превышении которого информация об этом действии записывается в журнал. При значении = 0 в журнале фиксируются все действия автоочистки.)
         autovacuum_max_workers = 6 (Задаёт максимальное число процессов автоочистки (не считая процесс, запускающий автоочистку), которые могут выполняться одновременно.)
         autovacuum_naptime = 15s (Задаёт минимальную задержку между двумя запусками автоочистки для отдельной базы данных)
         autovacuum_vacuum_threshold = 25 (Задаёт минимальное число изменённых или удалённых кортежей, при котором будет выполняться VACUUM для отдельно взятой таблицы)
         autovacuum_vacuum_scale_factor = 0.05 (Задаёт процент от размера таблицы, который будет добавляться к autovacuum_vacuum_threshold при выборе порога срабатывания команды VACUUM)
         autovacuum_vacuum_cost_delay = 10 (Задаёт задержку при превышении предела стоимости, которая будет применяться при автоматических операциях VACUUM)
         autovacuum_vacuum_cost_limit = 1000 (Задаёт предел стоимости, который будет учитываться при автоматических операциях VACUUM)

    Значения pgbench изменились: 

        pgbench (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
        starting vacuum...end.
        progress: 60.0 s, 704.9 tps, lat 11.322 ms stddev 9.756
        progress: 120.0 s, 689.6 tps, lat 11.579 ms stddev 9.655
        progress: 180.0 s, 663.3 tps, lat 12.040 ms stddev 9.669
        progress: 240.0 s, 686.5 tps, lat 11.631 ms stddev 9.426
        progress: 300.0 s, 701.2 tps, lat 11.389 ms stddev 9.571
        progress: 360.0 s, 700.2 tps, lat 11.403 ms stddev 9.534
        progress: 420.0 s, 703.1 tps, lat 11.356 ms stddev 9.711
        progress: 480.0 s, 687.7 tps, lat 11.610 ms stddev 9.894
        progress: 540.0 s, 690.6 tps, lat 11.562 ms stddev 9.918
        progress: 600.0 s, 678.8 tps, lat 11.764 ms stddev 9.611
        progress: 660.0 s, 724.0 tps, lat 11.028 ms stddev 9.254
        progress: 720.0 s, 454.5 tps, lat 17.579 ms stddev 13.177
        progress: 780.0 s, 319.6 tps, lat 25.005 ms stddev 11.283
        progress: 840.0 s, 320.0 tps, lat 24.977 ms stddev 11.073
        progress: 900.0 s, 317.9 tps, lat 25.140 ms stddev 11.477
        progress: 960.0 s, 319.5 tps, lat 25.009 ms stddev 11.211
        progress: 1020.0 s, 320.7 tps, lat 24.920 ms stddev 11.102
        progress: 1080.0 s, 319.6 tps, lat 25.012 ms stddev 12.583
        progress: 1140.0 s, 320.6 tps, lat 24.938 ms stddev 12.371
        progress: 1200.0 s, 315.3 tps, lat 25.343 ms stddev 11.983
        progress: 1260.0 s, 319.5 tps, lat 25.022 ms stddev 11.928
        progress: 1320.0 s, 320.9 tps, lat 24.904 ms stddev 10.946
        progress: 1380.0 s, 315.5 tps, lat 25.331 ms stddev 13.109
        progress: 1440.0 s, 320.6 tps, lat 24.935 ms stddev 12.278
        progress: 1500.0 s, 313.8 tps, lat 25.475 ms stddev 12.968
        progress: 1560.0 s, 320.1 tps, lat 24.967 ms stddev 11.336
        progress: 1620.0 s, 320.6 tps, lat 24.926 ms stddev 10.894
        progress: 1680.0 s, 319.6 tps, lat 25.005 ms stddev 10.960
        progress: 1740.0 s, 319.8 tps, lat 24.991 ms stddev 11.135
        progress: 1800.0 s, 318.3 tps, lat 25.105 ms stddev 11.807
        progress: 1860.0 s, 316.5 tps, lat 25.256 ms stddev 12.575
        progress: 1920.0 s, 320.7 tps, lat 24.927 ms stddev 12.171
        progress: 1980.0 s, 317.4 tps, lat 25.186 ms stddev 13.678
        progress: 2040.0 s, 320.4 tps, lat 24.955 ms stddev 12.805
        progress: 2100.0 s, 318.5 tps, lat 25.091 ms stddev 11.679
        progress: 2160.0 s, 319.4 tps, lat 25.027 ms stddev 12.272
        progress: 2220.0 s, 320.3 tps, lat 24.955 ms stddev 11.681
        progress: 2280.0 s, 319.3 tps, lat 25.035 ms stddev 12.743
        progress: 2340.0 s, 320.3 tps, lat 24.963 ms stddev 12.322
        progress: 2400.0 s, 318.2 tps, lat 25.118 ms stddev 12.013
        progress: 2460.0 s, 320.1 tps, lat 24.969 ms stddev 11.657
        progress: 2520.0 s, 317.0 tps, lat 25.208 ms stddev 12.184
        progress: 2580.0 s, 320.1 tps, lat 24.974 ms stddev 12.210
        progress: 2640.0 s, 320.4 tps, lat 24.955 ms stddev 12.489
        progress: 2700.0 s, 318.2 tps, lat 25.117 ms stddev 11.951
        progress: 2760.0 s, 319.9 tps, lat 24.985 ms stddev 11.816
        progress: 2820.0 s, 320.6 tps, lat 24.933 ms stddev 12.142
        progress: 2880.0 s, 319.6 tps, lat 25.009 ms stddev 11.782
        progress: 2940.0 s, 320.4 tps, lat 24.951 ms stddev 11.793
        progress: 3000.0 s, 318.0 tps, lat 25.133 ms stddev 12.402
        progress: 3060.0 s, 319.9 tps, lat 24.989 ms stddev 12.560
        progress: 3120.0 s, 320.5 tps, lat 24.938 ms stddev 11.536
        progress: 3180.0 s, 319.6 tps, lat 25.007 ms stddev 11.949
        progress: 3240.0 s, 320.5 tps, lat 24.939 ms stddev 12.384
        progress: 3300.0 s, 317.4 tps, lat 25.181 ms stddev 11.306
        progress: 3360.0 s, 319.9 tps, lat 24.980 ms stddev 11.820
        progress: 3420.0 s, 320.0 tps, lat 24.979 ms stddev 12.685
        progress: 3480.0 s, 319.6 tps, lat 25.009 ms stddev 12.148
        progress: 3540.0 s, 319.3 tps, lat 25.032 ms stddev 11.887
        progress: 3600.0 s, 314.4 tps, lat 25.426 ms stddev 12.989
        transaction type: <builtin: TPC-B (sort of)>
        scaling factor: 1
        query mode: simple
        number of clients: 8
        number of threads: 1
        duration: 3600 s
        number of transactions actually processed: 1404166
        latency average = 20.488 ms
        latency stddev = 12.970 ms
        initial connection time = 15.323 ms
        tps = 390.045191 (without initial connection time)

    Т.е. как можно судить по данным показателям, процесс автоочистки мертвых строк стал запускаться значительно интенсивней, что в свою очередь повлияло на пропускную способность (кол-во транзакций в секндру снизилось практически в 2 раза)

    
