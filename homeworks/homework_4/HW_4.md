1. Cоздайте новый кластер PostgresSQL 14

    Развернута ВМ на Яндекс Облако, установлен PG 14

        denis@denis-vm1:~$ pg_lsclusters
        Ver Cluster Port Status Owner    Data directory              Log file
        14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

2. Зайдите в созданный кластер под пользователем postgres

        denis@denis-vm1:~$ sudo -u postgres psql
        psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
        Type "help" for help.

        postgres=#

3. Создайте новую базу данных testdb

        postgres=# create database testdb;
        CREATE DATABASE
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


4. Зайдите в созданную базу данных под пользователем postgres

        postgres=# \c testdb
        You are now connected to database "testdb" as user "postgres".
        testdb=#

5. Создайте новую схему testnm

        testdb=# create schema testnm;
        CREATE SCHEMA

6. Создайте новую таблицу t1 с одной колонкой c1 типа integer

        testdb=# create table t1 (c1 integer);
        CREATE TABLE

7. Вставьте строку со значением c1=1

        testdb=# insert into t1 values (1);
        INSERT 0 1

        testdb=# select * from t1;
        c1
        ----
        1
        (1 row)

8. Создайте новую роль readonly

        testdb=# create role readonly;
        CREATE ROLE

9. Дайте новой роли право на подключение к базе данных testdb

        testdb=# grant connect on database testdb to readonly;
        GRANT

10. Дайте новой роли право на использование схемы testnm

        testdb=# grant usage on schema testnm to readonly;
        GRANT

11. Дайте новой роли право на select для всех таблиц схемы testnm

        testdb=# grant select on all tables in schema testnm to readonly;
        GRANT

12. Создайте пользователя testread с паролем test123

        testdb=# create user testread with password 'test123';
        CREATE ROLE

    *CREATE ROLE написало потому, что фактически в ПГ роль от пользователя отличается наличием пароля*

13. Дайте роль readonly пользователю testread

        testdb=# grant readonly to testread;
        GRANT ROLE

14. Зайдите под пользователем testread в базу данных testdb

    При попытке зайти вышла ошибка: 

        testdb=# \c testdb testread
        connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
        Previous connection kept   

    Смотрел в шпаргалке, что сделать, пошел по пути изменения в файле pg_hba.conf, в нем поменял строку: 

        # "local" is for Unix domain socket connections only
        local   all             all                                     md5

    После чего перезапустил кластер и зашел под пользователем

        denis@denis-vm1:/etc/postgresql/14/main$ sudo nano ./pg_hba.conf
        denis@denis-vm1:/etc/postgresql/14/main$ sudo -u postgres pg_ctlcluster 14 main stop
        denis@denis-vm1:/etc/postgresql/14/main$ sudo -u postgres pg_ctlcluster 14 main start
        Warning: the cluster will not be running as a systemd service. Consider using systemctl:
        sudo systemctl start postgresql@14-main
        denis@denis-vm1:/etc/postgresql/14/main$ sudo -u postgres psql
        psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
        Type "help" for help.

        postgres=# \c testdb testread
        Password for user testread:
        You are now connected to database "testdb" as user "testread".
        testdb=>

    *Честно признаюсь, не сразу осознал что изменения вступят в силу после рестарта кластера, но да, смог зайти только после рестарта кластера...=)*

15. Сделайте select * from t1;

        testdb=> select * from t1;
        ERROR:  permission denied for table t1

    *Ответы на пункты с 16 по 21 домашнего задания*

    Не получилось, так как на этапе создания таблицы мы не указывали схему в которой она должна будет создаться, а значит "по умолчанию" она создалась в схеме public: 

           search_path
        -----------------
        "$user", public
        (1 row)
    
    создавали мы под пользователем postgres, схемы с такими именем нет ("$user"), т.е. создаем в следующей по списку, а именно public

            List of relations
        Schema | Name | Type  |  Owner
        --------+------+-------+----------
        public | t1   | table | postgres
        (1 row)
    
    а, так как права на public мы для нашей роли readonly не давали, то соответственно и выбрать из данной схемы мы не можем.

    

22. Вернитесь в базу данных testdb под пользователем postgres

        testdb=> \c testdb postgres
        You are now connected to database "testdb" as user "postgres".
        testdb=#

23. Удалите таблицу t1

        testdb=# drop table t1;
        DROP TABLE

24. Создайте ее заново но уже с явным указанием имени схемы testnm

        testdb=# create table testnm.t1 (c1 integer);
        CREATE TABLE

25. Вставьте строку со значением c1=1

        testdb=# insert into testnm.t1 values (1);
        INSERT 0 1

26. Зайдите под пользователем testread в базу данных testdb

        testdb=# \c testdb testread
        Password for user testread:
        You are now connected to database "testdb" as user "testread".
        testdb=>

27. Сделайте select * from testnm.t1;

        testdb=> select * From testnm.t1;
        ERROR:  permission denied for table t1

    *Ответы на пункты с 28 по 33 домашнего задания*

    Не получилось, т.к. выдача прав роли readonly была в момент, когда таблица в схеме еще не существовала.

    Пробуем выдать права через alter default:

        testdb=> \c testdb postgres
        You are now connected to database "testdb" as user "postgres".
        testdb=# alter default privileges in schema testnm grant select on tables to readonly;
        ALTER DEFAULT PRIVILEGES
        testdb=# \c testdb testread
        Password for user testread:
        You are now connected to database "testdb" as user "testread".

    Выбираем данные:

        testdb=> select * From testnm.t1;
        ERROR:  permission denied for table t1

    Все равно ошибка, т.к. alter default будет действовать для вновь созданных таблиц, а не текущих.

    Пробуем перевыдать права на чтение из таблиц для нашей роли: 

        testdb=> \c testdb postgres
        You are now connected to database "testdb" as user "postgres".
        testdb=# grant select on all tables in schema testnm to readonly;
        GRANT
        testdb=# \c testdb testread
        Password for user testread:
        You are now connected to database "testdb" as user "testread".
        testdb=> 
        

34. Сделайте select * from testnm.t1;
        
        testdb=> select * From testnm.t1;
        c1
        ----
        1
        (1 row)

35. Получилось?

    #### Получилось! ####

36. ура!

    #### УРААААААА!!! =)) ####

37. Теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);

        testdb=> create table t2 (c1 integer); insert into t2 values (2);
        CREATE TABLE
        INSERT 0 1

38. А как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?

    Таблица создана в схеме public

        testdb=> \dt
        List of relations
        Schema | Name | Type  |  Owner
        --------+------+-------+----------
        public | t2   | table | testread
        (1 row) 
    
    *Ответы на пункты с 39 по 40 домашнего задания*

    Данная схема создается в каждой базе по умолчанию, при этом grant на все действия дается роли public, а данная роль выдается всем пользователям при создании по умолчанию.

    Для того, чтобы убрать данные права и в дальнейшем подобное не повторилось, необходимо забрать привилегии у роли public на создание в схеме public, а так же забрать все привилегии относящиеся к роли public внутри нашей БД: 

        testdb=> \c testdb postgres
        You are now connected to database "testdb" as user "postgres".
        testdb=# revoke create on schema public from public;
        REVOKE
        testdb=# revoke all on database testdb from public;
        REVOKE

  

41. Теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);

        testdb=> create table t3(c1 integer); insert into t2 values(1);
        ERROR:  permission denied for schema public
        LINE 1: create table t3(c1 integer);
                            ^
        INSERT 0 1
    
42. Расскажите что получилось и почему 

    Если я все правильно понял, то insert сработал по той причине, что когда мы создавали таблицу t2 под пользователем testread, то для этого пользователя установились все права для созданной таблицы, т.е. не смотря на то, что мы из роли public изъяли права, на уже созданные таблицы это не повлияло, и под пользователем testread мы можем манипулировать данными в созданной таблице, но не сможем создавать новые.

        testdb=# SELECT table_catalog, table_schema, table_name, privilege_type FROM information_schema.table_privileges WHERE grantee = 'testread';
        table_catalog | table_schema | table_name | privilege_type
        ---------------+--------------+------------+----------------
        testdb        | public       | t2         | INSERT
        testdb        | public       | t2         | SELECT
        testdb        | public       | t2         | UPDATE
        testdb        | public       | t2         | DELETE
        testdb        | public       | t2         | TRUNCATE
        testdb        | public       | t2         | REFERENCES
        testdb        | public       | t2         | TRIGGER
        (7 rows)

    Таким образом, если забрать права доступа на объекты схемы public уже у пользователя testread, то больше манипулировать с данными и видеть таблицу t2 мы не сможем: 

        testdb=# revoke all on all tables in schema public from testread;
        REVOKE
        testdb=# SELECT table_catalog, table_schema, table_name, privilege_type FROM information_schema.table_privileges WHERE grantee = 'testread';
        table_catalog | table_schema | table_name | privilege_type
        ---------------+--------------+------------+----------------
        (0 rows)

        testdb=# \c testdb testread
        Password for user testread:
        You are now connected to database "testdb" as user "testread".
        testdb=> select * from t2;
        ERROR:  permission denied for table t2

### Конец! ###

### Спасибо за внимание! ###

#### Автор - Кравченко Денис ####
