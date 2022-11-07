### Задание: ###
На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение. Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2. На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1. 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ). Небольшое описание, того, что получилось.

Задание со * : реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.

### Выполнение основного задания: ###
Создано 3 виртуальный машыны

На всех развернут ПГ 14

На ВМ 1 устанавливаем уровень репликации - логический: 

    testdb=# alter system set wal_level = logical;
    ALTER SYSTEM

Рестартуем сервер: 

    pg_ctlcluster 14 main restart

Проверяем что уровень репликации изменился

    testdb=# show wal_level;
    wal_level
    -----------
    logical
    (1 row)


На первой ВМ создаем таблица test: 

    create table test (id int,data varchar);

    testdb=# select * From test;
    id | data
    ----+------
    (0 rows)

И занесем запись: 

    testdb=# insert into test values (1, 'запись1');
    INSERT 0 1
    testdb=# select * From test;
    id |  data
    ----+---------
      1 | запись1
    (1 row)

Создаем публикацию таблицы test на ВМ 1:

    testdb=# CREATE PUBLICATION test_pub FOR TABLE test;
    CREATE PUBLICATION

Убеждаемся что она создана: 

    testdb=# \dRp+
                                Publication test_pub
      Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
    ----------+------------+---------+---------+---------+-----------+----------
    postgres | f          | t       | t       | t       | t         | f
    Tables:
        "public.test"

Теперь для этой таблицы на ВМ 2 создадим подписку, которая будет вычитывать данные из таблицы test на ВМ 1: 

Для этого на ВМ 2 была создана такая же таблица test, но записи в нее не вносились: 

    create table test (id int,data varchar);
    CREATE TABLE

Создаем подписку: 

    testdb=# CREATE SUBSCRIPTION test_sub
    CONNECTION 'host=51.250.101.213 port=5432 user=postgres password=2803 dbname=testdb'
    PUBLICATION test_pub WITH (copy_data = true);
    NOTICE:  created replication slot "test_sub" on publisher
    CREATE SUBSCRIPTION

В настройках подключения прописываю адрес своей ВМ 1, а так же пароль для пользователя который был задан ранее на ВМ 1

    testdb=# select * From test;
    id |  data
    ----+---------
      1 | запись1
    (1 row)

    testdb=# \dRs
                List of subscriptions
      Name   |  Owner   | Enabled | Publication
    ----------+----------+---------+-------------
    test_sub | postgres | t       | {test_pub}
    (1 row)

Теперь на ВМ 1 в таблицу test добавлю еще одну запись, и проверю что она пролилась в таблицу test на ВМ 2. 

ВМ 1: 

    root@denis-vm1:/etc/postgresql/14/main# sudo -u postgres psql -d testdb
    psql (14.5 (Ubuntu 14.5-2.pgdg22.04+2))
    Type "help" for help.

    testdb=# insert into test values(2,'запись2');
    INSERT 0 1

Смотрим на ВМ 2: 

    root@denis-vm2:/etc/postgresql/14/main# sudo -u postgres psql -d testdb
    psql (14.5 (Ubuntu 14.5-2.pgdg22.04+2))
    Type "help" for help.

    testdb=# select * From test;
    id |  data
    ----+---------
      1 | запись1
      2 | запись2
    (2 rows)

Готово, таблица test на ВМ1 у нас создана для записи данных, таблица test на ВМ2 для чтения, логическая репликация между ними поднята.
Повторяем такие же действя для таблицы test2, только на этот раз у нас на ВМ2 она будет для записи, т.е. создаем публиуацию, а на ВМ1 для чтения, т.е. делаем подписку: 

    testdb=# create table test2 (id int, data1 varchar, data2 varchar);
    CREATE TABLE
    testdb=# insert into test2 values(1,'запись','номер 1');
    INSERT 0 1
    testdb=# select * From test2;
    id | data1  |  data2
    ----+--------+---------
      1 | запись | номер 1
    (1 row)

    testdb=# CREATE PUBLICATION test2_pub FOR TABLE test2;
    CREATE PUBLICATION
    testdb=# \dRp+
                              Publication test2_pub
      Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
    ----------+------------+---------+---------+---------+-----------+----------
    postgres | f          | t       | t       | t       | t         | f
    Tables:
        "public.test2"

Переходим на ВМ1, создаем таблицу и делаем подписку на таблицу test2 ВМ2: 

    testdb=# create table test2 (id int, data1 varchar, data2 varchar);
    CREATE TABLE
    testdb=# select * From test2;
    id | data1 | data2
    ----+-------+-------
    (0 rows)

    testdb=# CREATE SUBSCRIPTION test2_sub
    CONNECTION 'host=158.160.15.139 port=5432 user=postgres password=2803 dbname=testdb'
    PUBLICATION test2_pub WITH (copy_data = true);
    NOTICE:  created replication slot "test2_sub" on publisher
    CREATE SUBSCRIPTION
    testdb=# select * From test2;
    id | data1  |  data2
    ----+--------+---------
      1 | запись | номер 1
    (1 row)

    testdb=# \dRs
                List of subscriptions
      Name    |  Owner   | Enabled | Publication
    -----------+----------+---------+-------------
    test2_sub | postgres | t       | {test2_pub}
    (1 row)



Готово, теперь у нас есть: 

1) Для таблицы test - публикация на ВМ1 и подписка на ВМ2

2) Для таблицы test2 - публикация на ВМ2 и подписка на ВМ1

Переходим к ВМ3 - на ней нам фактически надо создать подписки на таблицы test и test2, для этого проделываем следующее: 

Создаем таблицы на ВМ3:

    testdb=# create table test (id int,data varchar);
    CREATE TABLE
    testdb=# create table test2 (id int, data1 varchar, data2 varchar);
    CREATE TABLE

И теперь создаем подписки на test с ВМ1: 

    testdb=# CREATE SUBSCRIPTION test_sub1
    CONNECTION 'host=51.250.101.213 port=5432 user=postgres password=2803 dbname=testdb'
    PUBLICATION test_pub WITH (copy_data = true);
    NOTICE:  created replication slot "test_sub1" on publisher
    CREATE SUBSCRIPTION

И с ВМ2: 

    testdb=# CREATE SUBSCRIPTION test2_sub1
    CONNECTION 'host=158.160.15.139 port=5432 user=postgres password=2803 dbname=testdb'
    PUBLICATION test2_pub WITH (copy_data = true);
    NOTICE:  created replication slot "test2_sub1" on publisher
    CREATE SUBSCRIPTION

    testdb=# \dRs
                List of subscriptions
        Name    |  Owner   | Enabled | Publication
    ------------+----------+---------+-------------
    test2_sub1 | postgres | t       | {test2_pub}
    test_sub1  | postgres | t       | {test_pub}
    (2 rows)

Проверям что таблицы заполнились:

    testdb=# select * from test;
    id |  data
    ----+---------
      1 | запись1
      2 | запись2
    (2 rows)

    testdb=# select * From test2;
    id | data1  |  data2
    ----+--------+---------
      1 | запись | номер 1
    (1 row)

Готово, мы организовали логическое реплицирование на 3х ВМ где ВМ3 содержит только таблицы для чтения, ВМ1 имеет таблицу для записи test и таблицу для чтения test2, а ВМ2 имеет таблицу test для чтения и test2 для записи.

### Выполнение задания со *: ###

Теперь попробуем реализовать горячее реплицирование данных с ВМ3 на ВМ4, для этого будем использовать физическое риплицирование, для этого: 

На ВМ3, которая будет мастером задаем пароль для postgres для дальнейшего подключения по нему при настройке реплики: 

    testdb=# alter role postgres password '2803';

Выходим из БД, идем в файлик pg_hba.conf и дописываем строку: 

    host    replication     postgres        10.129.0.5/32           md5

где 10.129.0.5/32 - внутренний IP адрес реплики (ВМ4) (они у меня находятся в одной сети на Яндекс Облаке)

Далее идем в postgres.conf и задаем: 

    listen_addresses = '*' - задал все адреса, в практике так не стоит делать, лучше указывать конкретные, но я для простоты выполнения ДЗ прописал все...
    wal_level = replica 
    archive_mode = on 
    max_wal_senders = 10
    hot_standby = on

После чего перезагрузил кластер

    pg_ctlcluster 14 main restart

На ВМ4 который выступает репликой: 

Остановили кластер:

    pg_ctlcluster 14 main stop

В файле pg_hba.conf задаем строку: 

    host    replication     postgres        10.129.0.18/32          md5

Где 10.129.0.18/32 - внетренний IP адрес марстера (ВМ3)

Файл postgres.conf настроил аналогично ВМ3

Далее заходим под пользователем postgres: 

    su - postgres

Удаляем данные по пути:

    /var/lib/postgres/14

И пересоздаем каталог main: 

    mkdir main 
    chmod go-rwx main

Теперь находясь по данному пути под пользователем postgres выполняем команду: 

    postgres@denis-vm4:~/14$ pg_basebackup -P -R -X stream -c fast -h 10.129.0.18 -U postgres -D ./main
    Password:
    34914/34914 kB (100%), 1/1 tablespace

Пароль вводится тот, что был ранее задан на ВМ3

Возвращаемся в рута (exit)

Запускаем кластер

    pg_ctlcluster 14 main start

Смотрим его состояние: 

    root@denis-vm4:/etc/postgresql/14/main# pg_lsclusters
    Ver Cluster Port Status          Owner    Data directory              Log file
    14  main    5432 online,recovery postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

Готово, физическое реплицирование с ВМ3 на ВМ4 настроено, проверяем:

Заходим в БД на ВМ 4 и видим: 

    root@denis-vm4:/etc/postgresql/14/main# sudo -u postgres psql;
    psql (14.5 (Ubuntu 14.5-2.pgdg22.04+2))
    Type "help" for help.

    postgres=# \l
                                      List of databases
      Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
    -----------+----------+----------+-------------+-------------+-----------------------
    postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
    template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
              |          |          |             |             | postgres=CTc/postgres
    template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
              |          |          |             |             | postgres=CTc/postgres
    testdb    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
    (4 rows)

что наша БД testdb есть, заходим в нее: 

    postgres=# \c testdb
    You are now connected to database "testdb" as user "postgres".
    testdb=# select * From test;
    id |  data
    ----+---------
      1 | запись1
      2 | запись2
      3 | запись3
      4 | запись4
    (4 rows)

    testdb=# select * From test2;
    id | data1  |  data2
    ----+--------+---------
      1 | запись | номер 1
    (1 row)

Видим что данные пролились из таблиц.

Так же на ВМ1 для "верности" добавляем еще одну запись в таблицу test: 

    root@denis-vm1:~# sudo -u postgres psql -d testdb
    could not change directory to "/root": Permission denied
    psql (14.5 (Ubuntu 14.5-2.pgdg22.04+2))
    Type "help" for help.

    testdb=# insert into test values (5,'запись5');
    INSERT 0 1

Заходим на ВМ4 и проверяем: 

    testdb=# select * From test;
    id |  data
    ----+---------
      1 | запись1
      2 | запись2
      3 | запись3
      4 | запись4
      5 | запись5
    (5 rows)

Запись так же появилась.

### Конец! ###

### Спасибо за внимание! ###

#### Автор - Кравченко Денис ####    
